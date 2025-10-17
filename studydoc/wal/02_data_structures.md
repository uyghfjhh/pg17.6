# WAL 数据结构详解

> 深入分析 WAL 系统的核心数据结构，包括内存结构、磁盘格式和关键控制信息。

---

## 目录

1. [WAL 相关的共享内存结构](#一wal-相关的共享内存结构)
2. [WAL 记录的磁盘格式](#二wal-记录的磁盘格式)
3. [控制文件中的 WAL 信息](#三控制文件中的-wal-信息)
4. [关键缓存和状态结构](#四关键缓存和状态结构)
5. [WAL LSN 和定位机制](#五wal-lsn-和定位机制)
6. [内存对齐和优化](#六内存对齐和优化)

---

## 一、WAL 相关的共享内存结构

### 1.1 XLogCtlData - WAL 控制中心

**源码位置**: `src/include/access/xlog_internal.h:320`

**内存布局**:
```c
// 简化的 XLogCtlData 结构
typedef struct XLogCtlData {
    // -------- 插入控制 --------
    XLogCtlInsert Insert;           // WAL 记录插入控制
    
    // -------- 刷写状态 --------
    XLogCtlWrite Write;             // WAL 写入状态
    
    // -------- 检查点信息 --------
    CheckpointCheckpoints ckpt;     // 检查点状态缓存
    
    // -------- 恢复信息 --------
    XLogRecPtr minRecoveryPoint;    // 最小恢复点
    TimeLineID minRecoveryPointTLI; // 最小恢复点 Timeline
    
    // -------- WAL 缓冲区 --------
    char *pages;                    // 实际的 WAL 缓冲区
    int XLogCacheBlck;              // 缓冲块数
    
    // -------- 统计信息 --------
    WalStats stats;                 // WAL 操作统计
    
    // -------- 同步原语 --------
    slock_t info_lck;               // 信息锁
    slock_t insertpos_lck;          // 插入位置锁
    
} XLogCtlData;

// 共享内存指针
extern XLogCtlData *XLogCtl;
```

**详细分析**:

#### 1.1.1 XLogCtlInsert - 插入控制区

```c
typedef struct XLogCtlInsert {
    // 核心位置指针 (原子操作)
    pg_atomic_uint64 CurrBytePos;   // 当前字节位置
    pg_atomic_uint64 PrevBytePos;   // 前一个位置
    
    // 页面边界信息
    pg_atomic_uint32 currpage;      // 当前页号
    pg_atomic_uint32 currpos;       // 当前页内偏移
    
    // 插入锁 (XLogInsert 需要互斥)
    LWLock *insertpos_lck;
    
    // 保留空间 (用于检查点等)
    XLogRecPtr RedoRecPtr;          // REDO 起始位置
    TimeLineID RedoRecPtrTLI;       // REDO Timeline
    
    // 排队信息 (批量插入优化)
    XLogRecData *rdatas;            // 记录数据链表
    int num_rdatas;                 // 数据块数量
    int max_rdatas;                 // 最大数据块数
    
} XLogCtlInsert;
```

**操作原理**:

```
WAL 插入流程图:

Backend Process              XLogCtl->Insert
─────────────────┐          ┌──────────────────────┐
XLogInsert()      │          │                      │
                  │          │                      │
1. 获取Insert锁    ├──────────►│ insertpos_lck.lock() │
                  │          │                      │
2. 分配空间       │          │                      │
   CurrBytePos += │          │                      │
   record_size    │          │                      │
                  │          │                      │
3. 写入数据       │          │                      │
   填充 WAL 页面  │          │                      │
                  │          │                      │
4. 更新位置       │          │                      │
   PrevBytePos =  │          │                      │
   CurrBytePos    │          │                      │
                  │          │                      │
5. 释放锁         ├──────────►│ insertpos_lck.unlock()│
                  │          │                      │
─────────────────┘          └──────────────────────┘
```

#### 1.1.2 XLogCtlWrite - 写入控制区

```c
typedef struct XLogCtlWrite {
    // 刷写位置 (WAL Writer 更新)
    XLogRecPtr LogwrtRqst;         // 请求刷写的 LSN
    XLogRecPtr LogwrtResult;       // 已经刷写的 LSN
    
    // 同步状态
    XLogRecPtr SyncRepLSN[MAX_REPLICATION_SLOTS]; // 各槽同步点
    
    // 正在写入的页面
    XLogRecPtr LogwrtBuf;          // 缓冲区当前写入点
    XLogRecPtr LogwrtSeg;          // 段文件结束点
    
    // WAL Writer 状态
    pg_time_t lastSegSwitchTime;   // 上次段切换时间
    int segno;                     // 当前段号
    
} XLogCtlWrite;
```

#### 1.1.3 WalStats - 统计结构

```c
typedef struct WalStats {
    // 计数器
    uint64 wal_write;              // 写入次数
    uint64 wal_write_bytes;        // 写入字节数
    uint64 wal_write_time;         // 写入耗时
    uint64 wal_sync;               // fsync 次数
    uint64 wal_sync_bytes;         // fsync 字节数
    uint64 wal_sync_time;          // fsync 耗时
    
    // 缓冲区状态
    uint64 wal_buffers_full;       // 缓冲区满次数
    uint64 wal_write_lag;          // 写入延迟
    uint64 wal_sync_lag;           // 同步延迟
    
    // 记录统计
    uint64 wal_records;            // WAL 记录数
    uint64 wal_fpi;                // 全页镜像数
    uint64 wal_bytes_total;        // 总产生字节数
    
} WalStats;
```

### 1.2 WAL 缓冲区内存布局

**虚拟内存映射**:
```
共享内存中的 WAL 缓冲区:
┌──────────────────────────────────────────────────────────────┐
│                    XLogCtlData                              │
│                    (控制结构，约 4KB)                       │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│                  WAL Buffer Pages                           │
│                  (nominal_size/KB，默认 16MB)                 │
│                                                              │
│  Pages[0]:
│  ┌─────────────────────────────────────────────────────┐   │
│  │ XLOG_PAGE_SIZE = 8KB                             │   │
│  │ ┌─────────────────────────────────────────────┐   │   │
│  │ │ XLogPageHeaderData (变长)                   │   │   │
│  │ ├─────────────────────────────────────────────┤   │   │
│  │ │ WAL Records (变长)                           │   │   │
│  │ │ ┌─────────────────────────────────────┐      │   │   │
│  │ │ │ XLogRecord (24字节)                 │      │   │   │
│  │ │ ├─────────────────────────────────────┤      │   │   │
│  │ │ │ Main Data (变长)                     │      │   │   │
│  │ │ ├─────────────────────────────────────┤      │   │   │
│  │ │ │ Block Data (变长)                    │      │   │   │
│  │ │ └─────────────────────────────────────┘      │   │   │
│  │ └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Pages[1]: (下一个 8KB 页面)                               │
│  ...                                                         │
│                                                              │
│  Pages[2047]: (默认总共 2048 页，16MB)                     │
└──────────────────────────────────────────────────────────────┘
```

**实际物理布局** (在磁盘文件中):
```c
// WAL 文件的物理布局
struct XLogPageHeaderData {
    uint16      xlp_magic;          // 魔数 0xD10B
    uint16      xlp_info;           // 页面标志位
    TimeLineID  xlp_tli;            // Timeline ID
    XLogRecPtr  xlp_pageaddr;       // 页面起始 SNS
    uint32      xlp_rem_len;        // 继续记录的剩余长度
};

// 常见的 xlp_info 标志
#define XLP_FIRST_IS_CONTRECORD 0x0001  // 第一条是继续记录
#define XLP_LONG_HEADER         0x0002  // 长页头
#define XLP_BKP_REMOVABLE       0x0004  // 备份时可移除
```

---

## 二、WAL 记录的磁盘格式

### 2.1 XLogRecord - WAL 记录头

**源码位置**: `src/include/access/xlogrecord.h:100`

```c
typedef struct XLogRecord {
    uint32      xl_tot_len;        // 记录总长度 (包括头部)
    TransactionId xl_xid;          // 事务 ID (可选)
    XLogRecPtr  xl_prev;           // 前一条记录 LSN
    uint8       xl_info;           // 记录信息和标志
    RmgrId      xl_rmid;           // 资源管理器 ID
    uint8       xl_2pc;            // 2PC 信息
    pg_crc32c   xl_crc;            // CRC32C 校验和
} XLogRecord;

// xl_info 高 4 位是资源管理器特定的标志
#define XLR_INFO_MASK           0x0F
#define XLR_RMGR_INFO_MASK      0xF0
```

**大小计算**:
```
XLogRecord 头大小 = sizeof(XLogRecord) = 24 字节

包含校验和时的完整记录:
[ XLogRecord (24) ] 
[ 数据主体 (变长) ]
[ CRC32C (4) ] (可选，现代 PostgreSQL 都是使用)
```

### 2.2 XLogRecordBlockHeader - 块头

**用途**: 描述记录中引用的数据块

```c
typedef struct XLogRecordBlockHeader {
    uint8       fork;              // Fork 编号
    uint16      flags;             // 块标志
    uint16      data_length;       // 数据长度
    BlockNumber blkno;             // 块号
} XLogRecordBlockHeader;

// Fork 编号
#define MAIN_FORKNUM      0        // 主数据
#define FSM_FORKNUM       1        // Free Space Map  
#define VISIBILITYMAP_FORKNUM 2    // Visibility Map
#define INIT_FORKNUM      3        // 初始化分支

// 块标志
#define BKPBLOCK_HAS_IMAGE      0x0001  // 包含全页镜像
#define BKPBLOCK_HAS_DATA       0x0002  // 包含增量数据
#define BKPBLOCK_WILL_INIT     0x0004  // 页面将被初始化
#define BKPBLOCK_SAME_REL      0x0008  // 与前一块同一关系
```

### 2.3 XLogRecordBlockImage - 块镜像头

**用途**: 当需要全页镜像时的额外信息

```c
typedef struct XLogRecordBlockImage {
    uint16      length;            // 镜像长度 (通常是 BLCKSZ)
    uint16      hole_offset;       // hole 偏移 (压缩优化)
    uint16      hole_length;       // hole 长度
    uint8       bimg_info;         // 镜像信息
    /* 镜像数据紧随其后 */
} XLogRecordBlockImage;

// bimg_info 标志
#define BKPIMAGE_HAS_HOLE         0x01  // 有 hole
#define BKPIMAGE_IS_COMPRESSED    0x02  // 已压缩
#define BKPIMAGE_APPLY            0x04  // 恢复时需要应用
```

**全页镜像压缩原理**:
```
原始页面 (8KB):
┌────────────────────────────────────────┐
│ PageHeaderData (24字节)               │
├────────────────────────────────────────┤
│ LinePointers Array (可能有很多空)      │
│ [id1][id2][id3][empty][empty]...      │
├────────────────────────────────────────┤
│ Tuples Area (可能很稀疏)              │
│ [tuple1][tuple2][empty space]...      │
└────────────────────────────────────────┘

压缩后的镜像:
┌────────────────────────────────────────┐
│ 页面前面部分 (hole_offset)             │
├────────────────────────────────────────┤
│ 跳过 hole (不存储)                     │
├────────────────────────────────────────┤  
│ 页面后面部分 (BLCKSZ - hole_offset - hole_length) │
└────────────────────────────────────────┘
```

### 2.4 资源管理器特定格式

#### 2.4.1 Heap Insert 记录

```c
// XLOG_HEAP_INSERT 的主要信息
typedef struct xl_heap_insert {
    OffsetNumber offnum;             // 元组偏移
    uint8       flags;               // 标志
    /* HeapTupleHeaderData 和用户数据跟随 */
} xl_heap_insert;

// Heap 元组头部 (在 WAL 中)
typedef struct HeapTupleHeaderData {
    TransactionId t_xmin;           // 插入事务
    TransactionId t_xmax;           // 删除事务
    CommandId   t_cid;              // 命令 ID
    ItemPointerData t_ctid;         // TID
    uint16      t_infomask2;        // 信息掩码 2
    uint16      t_infomask;         // 信息掩码
    uint8       t_hoff;             // 头部偏移
} HeapTupleHeaderData;
```

**示例**: INSERT 记录的磁盘布局
```
XLogRecord {
    xl_rmid: RM_HEAP_ID
    xl_info: XLOG_HEAP_INSERT  
    xl_xid: 12345
    xl_tot_len: 150
}

BlockHeader {
    fork: MAIN_FORKNUM
    flags: BKPBLOCK_HAS_DATA
    data_length: 200
    blkno: 100
}

Main Data: xl_heap_insert {
    offnum: 5
    flags: 0
}

Block Data: HeapTuple {
    t_xmin: 12345
    t_xmax: 0
    t_cid: 0
    // ... 元组数据
}
```

#### 2.4.2 B-Tree Split 记录

```c
// 左分裂记录
typedef struct xl_btree_split {
    BlockNumber leftblk;             // 左页块号
    BlockNumber rightblk;            // 右页块号
    uint32      level;               // 树层级
    uint32      firstrightoff;       // 右页第一个项
    Oid         postingoff;         // posting 列表 (如果有)
} xl_btree_split;
```

#### 2.4.3 Transaction Commit 记录

```c
typedef struct xl_xact_commit {
    TimestampTz xact_time;           // 提交时间
    Oid         xact_dbid;           // 数据库 ID
    TransactionId nrels;             // 修改的关系数
    TransactionId nsubxacts;         // 子事务数
    TransactionId nmsgs;             // 共享消息数
    RelFileNode *rels;               // 修改的关系列表
    TransactionId *subxacts;         // 子事务列表
    SharedInvalidationMessage *msgs; // 无效消息列表
} xl_xact_commit;
```

---

## 三、控制文件中的 WAL 信息

### 3.1 ControlFileData 结构

**源码位置**: `src/include/catalog/pg_control.h:180`

```c
typedef struct ControlFileData {
    // 文件标识
    uint64      system_identifier;   // 系统唯一标识
    uint32      pg_control_version;  // 控制文件版本
    uint32      catalog_version_no;  // 系统目录版本

    // 状态信息
    DBState     state;               // 数据库状态
    pg_time_t   time;                // 最后更新时间
    
    // WAL 关键信息  
    XLogRecPtr  checkPoint;          // 最后一次检查点 LSN
    XLogRecPtr  prevCheckPoint;      // 上一次检查点 LSN
    CheckPoint   checkPointCopy;     // 检查点数据副本
    
    // WAL 定位信息
    XLogRecPtr  redo;                // REDO 起始 LSN
    TimeLineID  ThisTimeLineID;      // 当前 Timeline ID
    TimeLineID  PrevTimeLineID;      // 前一个 Timeline ID

    // WAL 文件信息
    XLogSegNo   prevSegNo;           // 上一个段文件号
    XLogSegNo   removeSegNo;         // 可移除的段文件号
    XLogSegNo   lastRemovedSegNo;     // 最后移除的段文件号
    
    // 配置参数 (启动时验证)
    uint32      maxAlign;            // 最大对齐
    double      floatFormat;         // 浮点格式
    uint32      blcksz;              // 块大小
    uint32      relseg_size;         // 关系段大小
    uint32      xlog_blcksz;         // WAL 块大小
    uint32      xlog_seg_size;       // WAL 段大小
    
    // 时间线历史
    TimeLineIDEntry *timeLineHistory; // Timeline 历史数组
    uint32      timeLineHistoryLen;   // 历史长度

    // 数据库标识
    char        bob_lsn[16];         // backup_label LSN (备份相关)
    
} ControlFileData;
```

### 3.2 检查点记录 CheckPoint

**源码位置**: `src/include/access/xlog.h:147`

```c
typedef struct CheckPoint {
    XLogRecPtr  redo;                // REDO 起始位置
    XLogRecPtr  redo_lsn;            // REDO LSN (同 redo)
    TimeLineID  ThisTimeLineID;      // 当前 Timeline
    FullTransactionId nextFullXid;   // 下一个事务 ID
    Oid         nextOid;             // 下一个 OID
    MultiXactId nextMulti;           // 下一个 MultiXact ID
    MultiXactOffset nextMultiOffset; // 下一个 MultiXact 偏移
    TransactionId oldestXid;         // 最老活跃事务 ID
    TransactionId oldestXidDB;       // 最老活跃事务所在数据库
    
    // 时间戳信息
    TimestampTz time;                // 检查点时间
    TimestampTz oldestActiveXid;     // 最老活跃事务时间
    
    // 数据库状态
    bool        oldestXidIsVacuumable; // 最老事务是否可清理
    
    // 计数器
    uint64      nextXidEpoch;        // 事务 ID epoch
    uint64      oldestCommitTsXid;   // 最老提交时间戳
    uint64      newestCommitTsXid;   // 最新提交时间戳
    uint64      oldestActiveXidFull; // 最老活跃事务完整 ID
    
    // 备份和恢复
    bool        backupStartPoint;    // 备份起始点 (用于 PITR)
    XLogRecPtr  backupStartEndLoc;   // 备份结束位置
    XLogRecPtr  minRecoveryPoint;    // 最小恢复点
    TimeLineID  minRecoveryPointTLI; // 最小恢复点 Timeline
    
} CheckPoint;
```

### 3.3 读取控制文件

**命令示例**:
```bash
$ pg_controldata $PGDATA | grep -E "(WAL|check|Log)"
pg_control version number:            1300
Catalog version number:               202307071
Database system identifier:           7335734480586991234
Database cluster state:               in production
pg_control last modified:             Mon 15 Jan 2024 10:30:22
Latest checkpoint location:           1/2A000028
Prior checkpoint location:            1/29FFFC10
Latest checkpoint's REDO location:    1/2A000028
Latest checkpoint's TimeLineID:       1
Latest checkpoint's PrevTimeLineID:   1
Latest checkpoint's full_page_writes: on
Latest checkpoint's NextXID:          0:2047
Latest checkpoint's NextOID:          24576
Latest checkpoint's NextMultiXactId:  1
Latest checkpoint's NextMultiOffset:  0
Latest checkpoint's oldestXID:        719
Latest checkpoint's oldestXID's DB:   1
Latest checkpoint's oldestActiveXID:  2047
Latest checkpoint's oldestMultiXactId: 1
Latest checkpoint's oldestMulti's DB: 1
Latest checkpoint's oldestCommitTsXid:0
Latest checkpoint's newestCommitTsXid:0
Time of latest checkpoint:            Mon 15 Jan 2024 10:28:10
Minimum recovery ending location:     0/0
Min recovery ending loc's timeline:   0
Backup start location:                0/0
Backup end location:                  0/0
WAL file size:                        16777216
WAL segment size:                     16MB
```

---

## 四、关键缓存和状态结构

### 4.1 WAL 段文件映射表

**源码位置**: `src/backend/access/transam/xlog.c:500`

```c
// WAL 段文件信息缓存
typedef struct XLogSegNoData {
    XLogSegNo   segno;               // 段文件号
    int         fd;                  // 文件描述符
    uint32      open_mode;           // 打开模式
    XLogRecPtr  seg_end;             // 段文件结束位置
    TimestampTz open_time;           // 打开时间
} XLogSegNoData;

// LRU 缓存 (通常缓存最近打开的几个段文件)
static XLogSegNoData seg_files[MAX_XLOG_SEG_BUFFERS];
```

### 4.2 TimeLine 历史管理

**源码位置**: `src/backend/access/transam/timeline.c`

```c
// Timeline 历史文件内容
typedef struct TimeLineHistoryEntry {
    TimeLineID  tli;                 // Timeline ID
    XLogRecPtr  switchpoint;         // 切换点
} TimeLineHistoryEntry;

// Timeline 历史 (从 .history 文件加载)
typedef struct TimeLineHistory {
    int         nentries;            // 条目数
    TimeLineHistoryEntry *entries;   // 条目数组
} TimeLineHistory;

// 示例: 00000002.history 文件内容
// 1   1/8B1E0000
// 2   1/8B1E0123
```

### 4.3 备份标签文件

**备份标签格式**:
```
backup_label 文件内容:
START WAL LOCATION: 1/2A000028 (file 000000010000000000000002)
CHECKPOINT LOCATION: 1/29FFFC10
BACKUP METHOD: pg_start_backup
BACKUP FROM: primary
START TIME: 2024-01-15 10:20:00 UTC
LABEL: "Daily backup"
START TIMELINE: 1
```

**结构定义**:
```c
typedef struct BackupLabel {
    XLogRecPtr  startLoc;            // WAL 起始位置
    XLogRecPtr  checkpointLoc;       // 检查点位置
    char        method[64];          // 备份方法
    char        from[64];            // 备份源
    TimestampTz startTime;           // 开始时间
    char        label[512];          // 标签
    TimeLineID  startTLI;            // 起始 Timeline
} BackupLabel;
```

### 4.4 复制槽状态

**源码位置**: `src/backend/replication/slot.c`

```c
// 物理复制槽
typedef struct ReplicationSlot {
    ReplicationSlotType type;        // 槽类型 (PHYSICAL/LOGICAL)
    
    // 槽标识
    char        name[NAMEDATALEN];   // 槽名称
    Oid         database;            // 数据库 OID (logical)
    
    // 位置信息
    XLogRecPtr  restart_lsn;         // 重启 LSN
    TransactionId xmin;              // 最小事务 ID
    TransactionId catalog_xmin;      // 最小系统表事务 ID
    
    // 状态信息
    TransactionId effective_xmin;    // 有效 xmin
    TransactionId effective_catalog_xmin; // 有效 catalog xmin
    
    // 配置
    bool        active;              // 是否活跃
    bool        failover;            // 是否故障转移到备库
    bool        two_phase;           // 是否支持两阶段
    
    // 时间戳
    TimestampTz last_active_time;    // 最后活跃时间
    
} ReplicationSlot;
```

---

## 五、WAL LSN 和定位机制

### 5.1 LSN (Log Sequence Number) 定义

**源码位置**: `src/include/access/xlogdefs.h`

```c
typedef uint64 XLogRecPtr;

// LSN 编码: 32位段号 + 32位偏移
// 高32位: 段文件中的页号 (基于 WAL_SEG_SIZE)
// 低32位: 页内字节偏移

// LSN 操作宏
#define LSN_FORMAT_ARGS(lsn) ((uint32) ((lsn) >> 32)), ((uint32) (lsn))
// LSN 输出格式: "0/1A000128" (十六进制)
```

### 5.2 LSN 到文件路径的转换

**算法实现**:

```c
// 源码: XLogFilePath() - src/backend/access/transam/xlog.c:4183
void
XLogFilePath(char *path, TimeLineID tli, XLogRecPtr segno, int seg_size)
{
    uint32      log, seg;
    
    // 将 segno 分解为 log 和 seg 部分
    XLByteToSeg(segno, log, seg, seg_size);
    
    // 构造路径: pg_wal/YYYYMMDD/XXXXXXXXXXXXXXXX
    snprintf(path, MAXPGPATH, "%s/%08X%08X%08X",
             XLOGDIR,
             tli,
             log,
             seg);
}

// 示例: LSN 0/1A000128
// segno = 1
// log = 0, seg = 1  
// tli = 1
// 路径: pg_wal/000000010000000000000001
```

**批量计算**:
```c
// 计算 LSN 对应的精确位置
typedef struct XLogRecPtr_Info {
    TimeLineID  tli;                 // 所在 Timeline
    XLogSegNo   segno;               // 段文件号
    uint32      offset;              // 段内偏移
    uint32      seg_offset;          // 段内页偏移  
} XLogRecPtr_Info;

// 解析示例: LSN = 0x0000000100000001A000128
//          tli = 1, segno = 26, offset = 0xA000128
//          seg_offset = offset % XLOG_SEG_SIZE
//          page_offset = offset % XLOG_BLCKSZ  
```

### 5.3 WAL 读取状态机

**读取状态**:
```c
typedef enum XLogReadState {
    XLOGREAD_STATE_UNINITIALIZED,   // 未初始化
    XLOGREAD_STATE_OPENING,          // 正在打开文件
    XLOGREAD_STATE_READING,          // 正在读取
    XLOGREAD_STATE_PAGE_HEADER,      // 正在读页头
    XLOGREAD_STATE_RECORD_HEADER,    // 正在读记录头
    XLOGREAD_STATE_RECORD_DATA,      // 正在读记录数据
    XLOGREAD_STATE_NEXT_RECORD,      // 准备下一条记录
    XLOGREAD_STATE_EOF,              // 文件结束
} XLogReadState;

// 读取上下文
typedef struct XLogReaderState {
    XLogReadState state;             // 当前状态
    XLogRecPtr  currRecPtr;          // 当前记录指针
    XLogRecPtr  EndRecPtr;           // 结束记录指针
    char       *readBuf;             // 读取缓冲区
    size_t      readLen;             // 已读取长度
    TimeLineID  currTLI;             // 当前 Timeline
    
    // 记录解析信息
    XLogRecord *decoded_record;      // 解码的记录
    List       *record_data;         // 记录数据列表
    List       *blocks;              // 块数据列表
    
    // 错误信息
    char        errormsg[512];       // 错误消息
    
} XLogReaderState;
```

### 5.4 LSN 比较和距离计算

**常用操作**:
```c
// 比较 LSN (带环绕处理)
#define XLogRecPtrIsInvalid(r)      ((r) == InvalidXLogRecPtr)
#define XLogRecPtrIsEqual(r1, r2)   ((r1) == (r2))
#define XLogRecPtrLess(r1, r2)      ((r1) < (r2))

// LSN 距离计算 (考虑前后 2^64 循环)
static inline uint64
XLogRecPtrDiff(XLogRecPtr r1, XLogRecPtr r2)
{
    if (r1 >= r2)
        return r1 - r2;
    else
        return (((uint64) 1) << 63) * 2 - (r2 - r1);
}

// 示例: 计算两个时间点之间产生的 WAL 量
-- SQL 方式
SELECT pg_wal_lsn_diff('1/2B000000', '1/2A000000');
-- 返回: 1048576 (1MB)

-- C 代码方式
uint64 wal_bytes = XLogRecPtrDiff(end_lsn, start_lsn);
```

---

## 六、内存对齐和优化

### 6.1 WAL 缓冲区对齐要求

**页面对齐**:
```c
// WAL 缓冲区需要按页面大小对齐
#define XLOG_BLCKSZ          8192        // WAL 块大小 (8KB)

// 缓冲区分配
XLogCtl->pages = (char *)
    ShmemInitStruct("WAL Buffer",
                    wal_segment_size * (WalSlicesCount),
                    &found);

// 确保页面对齐
Assert(((uintptr_t) XLogCtl->pages % XLOG_BLCKSZ) == 0);
```

### 6.2 LSN 的原子更新

**原子操作优化**:
```c
// 使用原子操作更新插入位置
static inline XLogRecPtr
AdvanceInsertPosition(uint32 nbytes)
{
    uint64 oldpos, newpos;
    
    // 原子获取并更新
    oldpos = pg_atomic_fetch_add_u64(&XLogCtl->Insert.CurrBytePos,
                                    nbytes);
    newpos = oldpos + nbytes;
    
    // 更新页面边界信息
    if (newpos % XLOG_BLCKSZ < oldpos % XLOG_BLCKSZ)
    {
        // 跨页了，更新当前页号
        uint32 page = newpos / XLOG_BLCKSZ;
        pg_atomic_write_u32(&XLogCtl->Insert.currpage, page);
    }
    
    return oldpos;
}
```

### 6.3 缓冲区压缩和优化

**全页镜像压缩算法**:
```c
// 源码: compress_page() 
static char *
compress_page(Page page, uint32 *compressed_size)
{
    char       *compressed;
    uint32      hole_offset;
    uint32      hole_length;
    uint32      tail_len;
    
    // 查找 "hole" - 连续的空区域
    find_hole(page, &hole_offset, &hole_length);
    
    if (hole_length < HOLE_THRESHOLD) {
        // hole 太小，不值得压缩
        compressed_size = NULL;
        return NULL;
    }
    
    // 保存 hole 信息
    *compressed_size = BLCKSZ - hole_length;
    
    // 分配压缩缓冲区
    compressed = palloc(*compressed_size);
    
    // 复制页面前面部分
    if (hole_offset > 0)
        memcpy(compressed, page, hole_offset);
    
    // 复制页面后面部分
    tail_len = BLCKSZ - hole_offset - hole_length;
    if (tail_len > 0)
        memcpy(compressed + hole_offset,
               page + hole_offset + hole_length,
               tail_len);
    
    return compressed;
}
```

### 6.4 批量插入优化

**批量写入机制**:
```c
// 批量插入上下文
typedef struct XLogInsertBatch {
    XLogRecData *rdatas[MAX_BATCH_RDATAS];  // 批量数据
    int         nrdatas;                    // 数据块数
    XLogRecPtr  start_lsn;                 // 批量起始 LSN
    size_t      total_size;                // 总大小
    
    // 同步点
    bool        need_sync;                 // 是否需要同步
    TimestampTz sync_time;                 // 同步时间
    
} XLogInsertBatch;

// 批量插入流程
static XLogRecPtr
XLogInsertBatch(XLogInsertBatch *batch)
{
    // 1. 预分配连续空间
    XLogRecPtr start_lsn = ReserveLogSpace(batch->total_size);
    
    // 2. 快速复制数据 (避免多次系统调用)
    CopyDataToBuffer(start_lsn, batch->rdatas, batch->nrdatas);
    
    // 3. 更新位置 (原子操作)
    AdvanceInsertPosition(batch->total_size);
    
    // 4. 必要时触发同步
    if (batch->need_sync)
        XLogFlush(start_lsn + batch->total_size);
    
    return start_lsn;
}
```

---

## 总结

WAL 的数据结构设计体现了 PostgreSQL 的几个重要设计原则：

1. **高效存储**: 通过对齐、压缩、批量等优化
2. **并发友好**: 使用原子操作、锁分离
3. **恢复可靠**: LSN 机制、校验和、全页镜像
4. **扩展性**: 资源管理器模式、复制槽

理解这些数据结构对于：
- 优化 WAL 性能
- 开发 WAL 相关扩展
- 故障定位和恢复
- 实现定制工具

**下一步**: 分析 WAL 的实现流程和关键算法 → [03_implementation_flow.md](03_implementation_flow.md)

---

**相关函数**:
- `XLogInsert()` - 插入 WAL 记录
- `XLogFlush()` - 刷新 WAL 到磁盘
- `XLogRead()` - 读取 WAL 记录
- `XLogRecPtrToSegment()` - LSN 转段号
- ` ReadControlFile()` - 读取控制文件
- ` UpdateControlFile()` - 更新控制文件
