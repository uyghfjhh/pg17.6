# Buffer Manager 核心数据结构详解

> Buffer Manager 的所有功能都围绕几个核心数据结构展开，理解这些结构是掌握 Buffer Manager 的关键。

---

## 目录

1. [共享缓冲池结构](#一共享缓冲池结构)
2. [Buffer Descriptor详解](#二buffer-descriptor详解)
3. [Buffer Tag 体系](#三buffer-tag-体系)
4. [并发控制结构](#四并发控制结构)
5. [置换算法数据结构](#五置换算法数据结构)
6. [页面内容结构](#六页面内容结构)
7. [统计信息结构](#七统计信息结构)

---

## 一、共享缓冲池结构

### 1.1 Buffer Control 数据

**源码**: `src/backend/storage/buffer/buf_table.c:80`

```c
/* Buffer control block */
typedef struct BufferControl
{
    /* Buffer descriptors table */
    BufferDesc   *buffer_descriptors;    // Buffer 描述符数组
    
    /* Buffer table (hash table) */
    HTAB         *buffer_table;          // Buffer 哈希表
    
    /* Buffer blocks (actual data) */
    char         *buffer_blocks;         // 实际数据页数组
    
    /* Free list */
    int           *free_list;            // 空闲 Buffer 链表
    
    /* Statistics */
    BufferStrategyControl *strategy;     // Buffer 策略控制
    
} BufferControl;
```

### 1.2 缓冲池布局详解

```
物理内存布局 (shared_buffers = 128MB):

0x00000000 ┌─────────────────────────────────────────┐
           │         Buffer Control Header          │
           │  - 全局控制信息                         │
           │  - 大小: ~1KB                          │
           ├─────────────────────────────────────────┤
0x00000400 │     Buffer Descriptor Array             │
           │  - BufferDescriptors[NBuffers]          │
           │  - 每个 BufferDesc: 64 Bytes            │
           │  - 总大小: NBuffers * 64 Bytes          │
           │                                           │
           │  示例 (128MB):                          │
           │  - NBuffers = 128MB / 8KB = 16384       │
           │  - BufferDesc 数组: 16384 * 64B = 1MB    │
           ├─────────────────────────────────────────┤
0x00100000 │       Buffer Blocks Pool                │
           │  - BufferBlocks[NBuffers]               │
           │  - 每个 Block: 8KB                      │
           │  - 总大小: NBuffers * 8KB               │
           │                                           │
           │  示例 (128MB):                          │
           │  - Data Pool: 128MB                     │
           ├─────────────────────────────────────────┤
           │           (未使用区域)                   │
           │           (可能有对齐)                   │
           └─────────────────────────────────────────┘

总共: ~129MB (128MB 数据 + 1MB 描述符 + 控制信息)
```

### 1.3 暴露的接口变量

**源码**: `src/backend/storage/buffer/bufmgr.c:156`

```c
/* Global variables */
typedef struct BufferDescPadded
{
    BufferDesc bufferdesc;
    /* Pad to a full cache line size */
    char        padding[PG_CACHE_LINE_SIZE - sizeof(BufferDesc)];
} BufferDescPadded;

BufferDescPadded *BufferDescriptors = NULL;  // Buffer 描述符数组
Block         *BufferBlocks = NULL;          // 数据页指针
int           NBuffers = 0;                 // Buffer 总数
```

---

## 二、Buffer Descriptor详解

### 2.1 BufferDesc 核心结构

**源码**: `src/include/storage/buf_internals.h:94`

```c
typedef struct BufferDesc
{
    /* 标识符 */
    BufferTag    tag;                       // Buffer 标签 (唯一标识)
    
    /* Buffer ID */
    int          buf_id;                    // Buffer ID (0-based)
    
    /* State flags */
    uint32       flags;                     // 状态标志位 (BM_*)
    
    /* Usage count for clock sweep */
    unsigned char usage_count;              // 使用计数 (0-5)
    
    /* Reference count */
    uint32       refcount;                  // 引用计数 (Pin count)
    
    /* Waiting backend */
    int          wait_backend_pgprocno;     // 等待的 Backend 进程号
    
    /* Free list pointer */
    int          free_next;                 // 空闲链表指针
    
    /* Buffer content lock */
    LWLock     *content_lock;               // 内容锁
    
    /* Recently used flag for buffer replacement strategy */
    bool         recently_used;             // 最近使用标志
    
    /* IO state */
    uint8       io_in_progress_lock;        // I/O 进行中锁
    bool       io_in_progress;             // I/O 进行中标志
    
    /* WAL LSN when buffer was last dirtied */
    XLogRecPtr  rec_lsn;                    // 记录 LSN
    
} BufferDesc;
```

### 2.2 标志位详解

**源码**: `src/include/storage/buf_internals.h:60`

```c
/* Buffer flag definitions */
#define BM_LOCKED               (1U << 22)      // Buffer 头部锁
#define BM_DIRTY                (1U << 23)      // 页面脏了
#define BM_VALID                (1U << 24)      // 页面有效
#define BM_TAG_VALID            (1U << 25)      // Tag 有效
#define BM_IO_IN_PROGRESS       (1U << 26)      // I/O 进行中
#define BM_IO_ERROR             (1U << 27)      // I/O 错误
#define BM_PIN_COUNT_WAITERS    (1U << 28)      // 有等待 Pin 的进程
#define BM_CHECKPOINT_NEEDED    (1U << 29)      // 需要 Checkpoint
#define BM_PERMANENT            (1U << 30)      // 永久 Buffer
#define BM_USAGE_COUNT          (1U << 31)      // Usage Count 字段有效

/* Mask that contains all flag bits except for usage_count */
#define BM_FLAG_MASK  \
    (~(BM_USAGE_COUNT))
```

### 2.3 Buffer Desc 实例分析

```
BufferDesc 示例 (buf_id = 100):

┌─────────────────────────────────────────────────────────────┐
│                    BufferDesc #100                          │
├─────────────────────────────────────────────────────────────┤
│ tag.buffer.relNode.relId = 16385        // 表 OID            │
│ tag.buffer.relNode.dbId = 16384         // 数据库 OID         │
│ tag.buffer.forkNum = MAIN_FORKNUM       // 主 fork           │
│ tag.buffer.blockNum = 1234              // 块号              │
├─────────────────────────────────────────────────────────────┤
│ buf_id = 100                                             │
│ flags = BM_VALID | BM_DIRTY | BM_TAG_VALID              │
├─────────────────────────────────────────────────────────────┤
│ usage_count = 3                                        │
│ refcount = 2                                          │
│ wait_backend_pgprocno = 0                               │
│ free_next = 105                                        │
├─────────────────────────────────────────────────────────────┤
│ content_lock = LWLock #87                               │
│ recently_used = true                                    │
│ io_in_progress = false                                  │
│ rec_lsn = 0/1A000128                                  │
└─────────────────────────────────────────────────────────────┘

状态解读:
- 页面有效 (BM_VALID)
- 页面脏了 (BM_DIRTY)  
- Tag 信息有效 (BM_TAG_VALID)
- 使用计数为 3 (被频繁使用)
- 被两个进程 Pin 住 (refcount = 2)
- 没有进程等待 I/O
- 需要被 Checkpoint 处理
```

---

## 三、Buffer Tag 体系

### 3.1 BufferTag 结构

**源码**: `src/include/storage/buf_internals.h:25`

```c
/* Buffer tag: identifies a buffer in the buffer table */
typedef struct BufferTag
{
    RelFileNode rnode;      // 关系文件节点
    ForkNumber    forkNum;  // Fork 编号
    BlockNumber   blockNum; // 块号
} BufferTag;

/* RelFileNode: 组合标识符 */
typedef struct RelFileNode
{
    Oid     spcNode;    // 表空间 OID
    Oid     dbNode;     // 数据库 OID  
    Oid     relNode;    // 关系 OID
} RelFileNode;
```

### 3.2 Fork Number 类型

**源码**: `src/common/relpath.h:37`

```c
typedef enum ForkNumber
{
    InvalidForkNumber = -1,
    
    MAIN_FORKNUM = 0,      // 主数据文件
    FSM_FORKNUM = 1,       // Free Space Map
    VISIBILITYMAP_FORKNUM = 2,  // Visibility Map
    INIT_FORKNUM = 3,      // 初始化分支

    MAX_FORKNUM = INIT_FORKNUM
} ForkNumber;
```

### 3.3 Buffer Tag 示例

```
示例 1: 普通表数据页
┌─────────────────────────────────────────────────┐
│ BufferTag                                      │
├─────────────────────────────────────────────────┤
│ rnode.spcNode = 1663 (pg_default 表空间)        │
│ rnode.dbNode = 16384 (postgres 数据库)          │
│ rnode.relNode = 16385 (users 表)                │
│ forkNum = 0 (MAIN_FORKNUM)                      │
│ blockNum = 1000 (第 1000 个数据页)              │
└─────────────────────────────────────────────────┘
对应文件: $PGDATA/base/16384/16385

示例 2: FSM 页面
┌─────────────────────────────────────────────────┐
│ BufferTag                                      │
├─────────────────────────────────────────────────┤
│ rnode.spcNode = 1663                            │
│ rnode.dbNode = 16384                            │
│ rnode.relNode = 16385                           │
│ forkNum = 1 (FSM_FORKNUM)                       │
│ blockNum = 50                                   │
└─────────────────────────────────────────────────┘
对应文件: $PGDATA/base/16384/16385_fsm

示例 3: Visibility Map 页面
┌─────────────────────────────────────────────────┐
│ BufferTag                                      │
├─────────────────────────────────────────────────┤
│ rnode.spcNode = 1663                            │
│ rnode.dbNode = 16384                            │
│ rnode.relNode = 16385                           │
│ forkNum = 2 (VISIBILITYMAP_FORKNUM)             │
│ blockNum = 0                                    │
└─────────────────────────────────────────────────┘
对应文件: $PGDATA/base/16384/16385_vm
```

### 3.4 Buffer 哈希表

**源码**: `src/backend/storage/buffer/buf_table.c:150`

```c
/* Buffer hash table implementation */
static HTAB *SharedBufHash = NULL;

/* Hash key structure */
typedef struct BufTableLookupKey
{
    BufferTag  key;
} BufTableLookupKey;

/* Initialize buffer hash table */
void InitBufTable(void)
{
    HASHCTL     info;
    
    /* Initialize hash table */
    MemSet(&info, 0, sizeof(info));
    info.keysize = sizeof(BufferTag);
    info.entrysize = sizeof(BufferLookupEnt);
    info.num_partitions = NUM_BUFFER_PARTITIONS;
    
    SharedBufHash = ShmemInitHash("Shared Buffer Lookup Table",
                                  NBuffers, NBuffers,
                                  &info,
                                  HASH_ELEM | HASH_BLOBS | HASH_PARTITION);
}
```

---

## 四、并发控制结构

### 4.1 LWLock 体系

**源码**: `src/backend/storage/lmgr/lwlock.c:86`

```c
/* Buffer-related LWLocks */
typedef enum BuiltinLWLockId
{
    /* Buffer content locks: one per buffer */
    LWTRANCHE_BUFFER_CONTENT,
    
    /* Buffer mapping lock */
    LWTRANCHE_BUFFER_MAPPING,
    
    /* Buffer strategy lock */
    LWTRANCHE_BUFFER_STRATEGY,
    
    /* Buffer I/O operations */
    LWTRANCHE_BUFFER_IO,
    
    /* Freelist management */
    LWTRANCHE_BUFFER_FREELIST,
    
    /* ... 其他锁 ... */
} BuiltinLWLockId;

/* 组织方式:
- Buffer Content Locks: 每个一个 (动态分配)
- 其他锁: 全局单例
*/

/* 锁的使用时机:
1. Content Lock: 读写 Buffer 内容时
2. Mapping Lock: 查找/创建 BufferDesc 时  
3. Strategy Lock: Buffer 置换算法执行时
4. I/O Lock: 执行磁盘 I/O 时
5. Freelist Lock: 管理 Buffer 空闲链表时
*/
```

### 4.2 Pin 机制 (引用计数)

```c
/* Buffer Pin 操作 */
static inline void
PinBuffer(BufferDesc *buf)
{
    uint32      old_refcount;
    uint32      new_refcount;
    
    /* 递增 refcount */
    old_refcount = pg_atomic_fetch_add_u32(&buf->refcount, 1);
    
    /* 检查溢出 */
    Assert(old_refcount < UINT32_MAX);
    
    new_refcount = old_refcount + 1;
    
    /* 如果从 0 到 1，清除 BM_USAGE_COUNT */
    if (old_refcount == 0)
    {
        buf->flags &= ~BM_USAGE_COUNT;
    }
}

/* Buffer Unpin 操作 */
static inline void
UnpinBuffer(BufferDesc *buf, bool fix_owner)
{
    uint32      old_refcount;
    
    /* 递减 refcount */
    old_refcount = pg_atomic_fetch_sub_u32(&buf->refcount, 1);
    
    /* 检查下溢 */
    Assert(old_refcount > 0);
    
    new_refcount = old_refcount - 1;
    
    /* 如果有等待者，唤醒 */
    if (new_refcount == 0 && 
        (buf->flags & BM_PIN_COUNT_WAITERS) != 0)
    {
        /* 唤醒等待的进程 */
        ProcSendSignal(buf->wait_backend_pgprocno);
    }
}

/* Pin 的作用:
1. 防止 Buffer 被置换出去
2. 确保访问期间内容稳定
3. 作为 Buffer 验证机制
*/
```

---

## 五、置换算法数据结构

### 5.1 Strategy Control

**源码**: `src/backend/storage/buffer/freelist.c:75`

```c
/* Buffer replacement strategy control */
typedef struct BufferStrategyControl
{
    /* Clock sweep hand */
    int           next_victim_buffer;   // Clock 指针
    
    /* Spare buffer list */
    int           *first_spares_buffer; // 空闲 Buffer 列表头
    int           *last_spares_buffer;  // 空闲 Buffer 列表尾
    
    /* Statistics */
    uint32        num_buffer_allocs;    // Buffer 分配次数
    uint32        num_buffer_writes;    // Buffer 写入次数
    
    /* Lock for strategy control */
    LWLock       *strategy_lock;        // 策略保护锁
    
} BufferStrategyControl;
```

### 5.2 Clock-Sweep 算法实现

```c
/* Clock-Sweep 置换算法 */
static BufferDesc *
StrategyGetBuffer(BufferAccessStrategy strategy)
{
    BufferDesc  *buf;
    int          trycounter;
    bool        have_passes;
    
    /*
     * Clock-sweep algorithm: iterate over buffers, 
     * looking for one with usage_count = 0
     */
    
    for (;;)
    {
        /* 获取下一个 Buffer */
        buf = GetBufferDescriptor(StrategyControl->next_victim_buffer);
        
        /* 时钟指针前进 */
        StrategyControl->next_victim_buffer++;
        if (StrategyControl->next_victim_buffer >= NBuffers)
            StrategyControl->next_victim_buffer = 0;
        
        /* 检查 Buffer 是否可用 */
        if (buf->flags & BM_VALID)
        {
            /* 有效页面，检查 usage_count */
            if (buf->usage_count > 0)
            {
                /* 使用中，递减计数，继续扫描 */
                buf->usage_count--;
                continue;
            }
        }
        
        /* 尝试 Pin Buffer */
        if (PinBufferWithoutBlock(buf, strategy))
        {
            /* 成功，返回 */
            return buf;
        }
        
        /* 失败，继续扫描 */
    }
}

/* Clock-Sweep 优势:
1. O(1) 平均时间复杂度
2. 良好的时间局部性
3. 实现简单、稳定
4. 可以模拟 LRU 效果
*/
```

### 5.3 空闲链表管理

```c
/* Free list management */
typedef struct BufferListData
{
    int           free_list_len;      // 空闲链表长度
    int           *free_list;         // 空闲 Buffer 数组
    
    /* Free list positions */
    int           free_head;          // 链表头
    int           free_tail;          // 链表尾
} BufferListData;

/* 初始化空闲链表 */
static void
InitBufferList(void)
{
    int         i;
    
    /* 将所有 Buffer 加入空闲链表 */
    for (i = 0; i < NBuffers; i++)
    {
        BufferDesc *buf = GetBufferDescriptor(i);
        
        buf->free_next = (i < NBuffers - 1) ? i + 1 : -1;
        
        if (i == 0)
            BufferList.free_head = 0;
        if (i == NBuffers - 1)
            BufferList.free_tail = i;
    }
    
    BufferList.free_list_len = NBuffers;
}

/* 从空闲链表获取 Buffer */
static int
GetBufferFromFreeList(void)
{
    int         buf_id;
    
    if (BufferList.free_list_len == 0)
        return -1;  /* 无空闲 Buffer */
    
    buf_id = BufferList.free_head;
    BufferList.free_head = BufferDescriptors[buf_id].bufferdesc.free_next;
    BufferList.free_list_len--;
    
    return buf_id;
}
```

---

## 六、页面内容结构

### 6.1 Page Header

**源码**: `src/include/storage/bufpage.h:160`

```c
/* Page header data structure */
typedef struct PageHeaderData
{
    /* 7.4 版本后的 Page Header (24 字节) */
    PageXLogRecPtr pd_lsn;        // 最后修改的 LSN
    uint16        pd_checksum;    // 页面校验和
    uint16        pd_flags;       // 页面标志
    uint16        pd_lower;       // 空闲空间起始位置
    uint16        pd_upper;       // 空闲空间结束位置
    uint16        pd_special;     // 特殊空间起始位置
    uint16        pd_pagesize_version; // 页面大小和版本

    /* 紧随其后的是 ItemId 数组 */
    ItemIdData    pd_linp[1];     // 行指针数组 (变长)
} PageHeaderData;

/* 标志位定义 */
#define PD_HAS_FREE_LINES    0x0001   // 有空闲行指针
#define PD_PAGE_FULL         0x0002   // 页面满了
#define PD_ALL_VISIBLE       0x0004   // 所有行可见
#define PD_VALID_FLAG_BITS   0x0007   // 有效标志位
```

### 6.2 Heap Tuple Header

**源码**: `src/include/access/htup_details.h:150`

```c
/* Heap tuple header */
typedef struct HeapTupleHeaderData
{
    /* 事务和命令标识 */
    TransactionId t_xmin;        // 插入事务 ID
    TransactionId t_xmax;        // 删除事务 ID
    CommandId     t_cid;         // 命令 ID
    CommandId     t_xvac;        // 旧版本删除事务 ID
    
    /* 元组位置信息 */
    ItemPointerData t_ctid;      // 元组位置 (页内或页间)
    
    /* 标志位 */
    uint16        t_infomask;    // 信息标志
    uint8         t_infomask2;   // 信息标志 2
    uint8         t_hoff;        // 头部偏移
    
    /* bitmap跟着这里（如果有的话） */
    /* 然后是实际的用户数据 */
} HeapTupleHeaderData;

/* 信息标志位 (t_infomask) */
#define HEAP_HASNULL            0x0001  // 有 NULL 值
#define HEAP_HASVARWIDTH        0x0002  // 有变长字段
#define HEAP_HASKVARSIZE        0x0004  // 有变长 TOAST 值
#define HEAP_HASOID             0x0008  // 有 OID
#define HEAP_XMAX_LOCK_ONLY     0x0080  // xmax 只是锁，不是删除
#define HEAP_XMAX_EXCL_LOCK     0x0100  // 排他锁
#define HEAP_XMAX_SHARED_LOCK   0x0200  // 共享锁
#define HEAP_XMIN_COMMITTED     0x0800  // xmin 已提交
#define HEAP_XMIN_INVALID       0x1000  // xmin 无效
#define HEAP_XMAX_COMMITTED     02000   // xmax 已提交
#define HEAP_XMAX_INVALID       0x4000  // xmax 无效
#define HEAP_UPDATED            0x8000  // 元组被更新
```

### 6.3 Buffer 内容布局

```
完整的 Buffer 内容布局 (8KB = 8192 字节):

┌────────────────────────────────────────────────────────┐  ← 0 字节
│  Buffer 控制信息 (不在缓冲区中，在 BufferDesc 中)      │
│  - BufferDesc: 64 字节                                 │
├────────────────────────────────────────────────────────┤  ← 页面内容开始
│  PageHeaderData (24 字节)                              │
│  ├─ pd_lsn:        0/1A000128                         │
│  ├─ pd_checksum:   0x1234                              │
│  ├─ pd_flags:      PD_ALL_VISIBLE                     │
│  ├─ pd_lower:      80                                  │
│  ├─ pd_upper:      8000                                │
│  ├─ pd_special:    8192                                │
│  └─ pd_pagesize_version: 8192 | 0x04                  │
├────────────────────────────────────────────────────────┤  ← 24 字节
│  ItemIdData 数组 (行指针数组)                           │
│  每个 ItemId 4 字节:                                    │
│  ┌────────────────────────────────────┐               │
│  │ lp_off:   指向元组的偏移         │               │
│  │ lp_flags: 标志 (LP_NORMAL)        │               │
│  │ lp_len:   元组长度               │               │
│  └────────────────────────────────────┘               │
│  ... (假设 14 个 ItemId，共 56 字节)                    │
├────────────────────────────────────────────────────────┤  ← 80 字节 (pd_lower)
│  Free Space (空闲空间)                                  │
│  ← 向下增长 - ← 80 字节                                │
│                                                        │
│                                                        │
│                                                        │
│                                                        │
│  ↑ 向上增长 (元组数据) - ← 8000 字节                    │
│                                                        │
├────────────────────────────────────────────────────────┤  ← 8000 字节 (pd_upper)
│  HeapTuple 数据 (从页面末尾向前增长)                     │
│  ┌──────────────────────────────────────┐             │
│  │ HeapTupleHeaderData (23 字节)        │             │
│  │ ├─ t_xmin: 1000                       │             │
│  │ ├─ t_xmax: 0                         │             │
│  │ ├─ t_cid: 0                          │             │
│  │ ├─ t_ctid: (100, 5)                  │             │
│  │ └─ t_infomask: HEAP_HASVARWIDTH       │             │
│  ├──────────────────────────────────────┤             │
│  │ User Data (变长，例如 50 字节)        │             │
│  └──────────────────────────────────────┘             │
│  ... (多个元组)                                       │
├────────────────────────────────────────────────────────┤  ← 8192 字节 (页末)
│  Special Space (特殊空间 - 索引页使用)                 │
│  (Heap 页通常不使用特殊空间)                           │
└────────────────────────────────────────────────────────┘

内存对齐优化:
- BufferDesc 对齐到缓存行边界 (64 字节)
- 避免伪共享 (false sharing)
- 每个 Buffer 的热门数据在单独的缓存行
```

---

## 七、统计信息结构

### 7.1 Buffer Access Strategy

```c
/* Buffer access strategy */
typedef struct BufferAccessStrategyData
{
    /* 所属进程的Backend */
    BackendId  backend;
    
    /* 策略类型 */
    BufferAccessStrategyType btype;
    
    /* 下一个要检查的 Buffer */
    int         next_buf_in_cycle;
    
    /* 当前周期的 Buffer 数组 */
    int         buffers[NBuffers];
    
    /* 实际使用的 Buffer 数量 */
    int         nbuffers;
    
} BufferAccessStrategyData;

/* 策略类型 */
typedef enum BufferAccessStrategyType
{
    BAS_NORMAL,        // 正常策略
    BAS_BULKREAD,      // 大量读取策略 (VACUUM)
    BAS_BULKWRITE,     // 大量写入策略 (CREATE INDEX)
    BAS_VACUUM         // VACUUM 专用策略
} BufferAccessStrategyType;
```

### 7.2 Buffer 统计信息

```c
/* Buffer pool statistics */
typedef struct PgStat_BgWriter
{
    /* Checkpoint 统计 */
    PgStat_Counter checkpoints_timed;     // 超时触发的 checkpoint
    PgStat_Counter checkpoints_req;       // 请求触发的 checkpoint
    PgStat_Counter checkpoint_write_time; // checkpoint 写入时间
    PgStat_Counter checkpoint_sync_time;  // checkpoint 同步时间
    
    /* BgWriter 统计 */
    PgStat_Counter buffers_checkpoint;    // checkpoint 写入的页面
    PgStat_Counter buffers_clean;         // bgwriter 写入的页面
    PgStat_Counter maxwritten_clean;      // 单次最大写入
    PgStat_Counter buffers_backend;       // backend 写入的页面
    PgStat_Counter buffers_backend_fsync; // backend fsync 次数
    PgStat_Counter buffers_alloc;         // 分配的 Buffer 数
    
} PgStat_BgWriter;

/* 数据库 Buffer 统计 */
typedef struct PgStat_Database
{
    PgStat_Counter xact_commit;           // 提交事务数
    PgStat_Counter xact_rollback;         // 回滚事务数
    PgStat_Counter blks_read;             // 磁盘读取块数
    PgStat_Counter blks_hit;              // 内存命中块数
    PgStat_Counter tup_returned;          // 返回的行数
    PgStat_Counter tup_fetched;           // 获取的行数
    PgStat_Counter tup_inserted;          // 插入的行数
    PgStat_Counter tup_updated;           // 更新的行数
    PgStat_Counter tup_deleted;           // 删除的行数
} PgStat_Database;
```

---

## 总结

Buffer Manager 的核心数据结构体现了精心设计：

1. **分层设计**: BufferDesc → BufferTag → 实际页面
2. **高效索引**: 哈希表快速定位页面
3. **并发控制**: 多级锁机制保证安全
4. **智能置换**: Clock-Sweep 算法高效简单
5. **内存对齐**: 避免伪共享，提升性能
6. **统计监控**: 丰富的性能指标

**数据结构优势**:
- 低内存开销 (控制信息与数据分离)
- 高并发性能 (细粒度锁)
- 简单高效 (Clock-Sweep O(1))
- 易于扩展 (模块化设计)

**下一步**: 深入学习 Buffer Manager 的实现流程和算法细节。

---

**文档版本**: 1.0
**相关源码**: PostgreSQL 17.5
**创建日期**: 2025-01-15

**下一篇**: [03_implementation_flow.md](03_implementation_flow.md) - Buffer Manager 实现流程分析
