# Lock Manager 核心数据结构

本文档详细说明Lock Manager的核心数据结构。

---

## 1. LOCK - 锁对象结构

### 1.1 结构定义

```c
// src/include/storage/lock.h:308
typedef struct LOCK
{
    /* 哈希键 */
    LOCKTAG tag;                    // 唯一标识被锁对象
    
    /* 锁状态 */
    LOCKMASK grantMask;             // 已授予的锁类型位掩码
    LOCKMASK waitMask;              // 等待中的锁类型位掩码
    
    /* 链表 */
    dlist_head procLocks;           // 持有此锁的PROCLOCK链表
    dclist_head waitProcs;          // 等待此锁的进程队列
    
    /* 计数器 */
    int requested[MAX_LOCKMODES];   // 各模式请求计数 (含已授予)
    int nRequested;                 // 总请求数
    int granted[MAX_LOCKMODES];     // 各模式已授予计数
    int nGranted;                   // 总授予数
} LOCK;
```

### 1.2 字段详解

#### LOCKTAG - 锁标识

```c
// src/include/storage/lock.h:85
typedef struct LOCKTAG
{
    uint32 locktag_field1;      // 主标识 (如: relation OID)
    uint32 locktag_field2;      // 次标识 (如: database OID)
    uint32 locktag_field3;      // 扩展标识
    uint16 locktag_field4;      // 额外标识 (如: tuple offset)
    uint8  locktag_type;        // 锁类型 (0-9)
    uint8  locktag_lockmethodid; // 锁方法ID (通常=1)
} LOCKTAG;
```

**锁类型 (locktag_type)**:

```c
#define LOCKTAG_RELATION              0  // 表锁
#define LOCKTAG_RELATION_EXTEND       1  // 表扩展锁
#define LOCKTAG_PAGE                  2  // 页锁
#define LOCKTAG_TUPLE                 3  // 元组锁
#define LOCKTAG_TRANSACTION           4  // 事务ID锁
#define LOCKTAG_VIRTUALTRANSACTION    5  // 虚拟事务ID锁
#define LOCKTAG_SPECULATIVE_TOKEN     6  // 推测插入令牌
#define LOCKTAG_OBJECT                7  // 数据库对象锁
#define LOCKTAG_USERLOCK              8  // 用户锁
#define LOCKTAG_ADVISORY              9  // 咨询锁
```

**LOCKTAG 示例**:

```
表锁:
  locktag_field1 = relation OID (如: 16384)
  locktag_field2 = database OID (如: 16385)
  locktag_type = LOCKTAG_RELATION (0)

行锁:
  locktag_field1 = relation OID
  locktag_field2 = database OID
  locktag_field3 = block number
  locktag_field4 = tuple offset
  locktag_type = LOCKTAG_TUPLE (3)

事务ID锁:
  locktag_field1 = transaction ID
  locktag_type = LOCKTAG_TRANSACTION (4)
```

#### LOCKMASK - 锁位掩码

```c
typedef int LOCKMASK;

// 每个锁模式对应一个位
#define LOCKBIT_ON(lockmode) (1 << (lockmode))

// 示例:
LOCKBIT_ON(AccessShareLock)     = 0x0002  // 1 << 1
LOCKBIT_ON(RowExclusiveLock)    = 0x0008  // 1 << 3
LOCKBIT_ON(AccessExclusiveLock) = 0x0100  // 1 << 8

// grantMask 示例:
// 如果已授予 AccessShareLock 和 RowExclusiveLock:
grantMask = 0x0002 | 0x0008 = 0x000A
```

#### requested[] 和 granted[] 数组

```c
// 计数数组 (索引0未使用，1-8对应8种锁模式)
int requested[MAX_LOCKMODES];  // MAX_LOCKMODES = 9
int granted[MAX_LOCKMODES];

// 示例:
// 3个进程持有 AccessShareLock
// 1个进程持有 RowExclusiveLock
// 1个进程等待 ExclusiveLock
requested[AccessShareLock]   = 3  // 已授予
requested[RowExclusiveLock]  = 1  // 已授予
requested[ExclusiveLock]     = 1  // 等待中

granted[AccessShareLock]     = 3
granted[RowExclusiveLock]    = 1
granted[ExclusiveLock]       = 0  // 尚未授予

nRequested = 5  // 总请求数
nGranted   = 4  // 已授予数
```

---

## 2. PROCLOCK - 进程锁结构

### 2.1 结构定义

```c
// src/include/storage/lock.h:362-380
typedef struct PROCLOCKTAG
{
    LOCK   *myLock;     // 指向 LOCK 结构
    PGPROC *myProc;     // 指向 PGPROC 结构
} PROCLOCKTAG;

typedef struct PROCLOCK
{
    /* 哈希键 */
    PROCLOCKTAG tag;            // {LOCK*, PGPROC*} 组合键
    
    /* 锁状态 */
    PGPROC *groupLeader;        // 锁组组长 (并行查询)
    LOCKMASK holdMask;          // 持有的锁类型位掩码
    LOCKMASK releaseMask;       // 待释放的锁类型位掩码
    
    /* 链表节点 */
    dlist_node lockLink;        // 在 LOCK->procLocks 链表中
    dlist_node procLink;        // 在 PGPROC->myProcLocks 链表中
} PROCLOCK;
```

### 2.2 PROCLOCK 作用

PROCLOCK 是 **LOCK 和 PGPROC 的关联表**，记录"哪个进程持有哪个锁"。

```
关系示意:
  LOCK (被锁对象) ←→ PROCLOCK ←→ PGPROC (持有者)
  
  一个 LOCK 可以有多个 PROCLOCK (多个进程持有)
  一个 PGPROC 可以有多个 PROCLOCK (持有多个锁)
```

### 2.3 holdMask 和 releaseMask

```c
// holdMask: 当前持有的锁模式
// 例如: 进程同时持有 AccessShareLock 和 RowExclusiveLock
holdMask = LOCKBIT_ON(AccessShareLock) | LOCKBIT_ON(RowExclusiveLock)
         = 0x0002 | 0x0008 = 0x000A

// releaseMask: LockReleaseAll() 时使用
// 标记哪些锁需要释放
releaseMask = LOCKBIT_ON(AccessShareLock)  // 只释放 AccessShareLock
```

---

## 3. LOCALLOCK - 本地锁缓存

### 3.1 结构定义

```c
// src/include/storage/lock.h:408-441
typedef struct LOCALLOCKTAG
{
    LOCKTAG lock;           // 锁标识
    LOCKMODE mode;          // 锁模式
} LOCALLOCKTAG;

typedef struct LOCALLOCKOWNER
{
    ResourceOwner owner;    // 资源所有者
    int64 nLocks;           // 持有次数
} LOCALLOCKOWNER;

typedef struct LOCALLOCK
{
    /* 哈希键 */
    LOCALLOCKTAG tag;           // {LOCKTAG, LOCKMODE}
    
    /* 哈希值 */
    uint32 hashcode;            // LOCKTAG 的哈希值
    
    /* 指向共享内存 */
    LOCK *lock;                 // 指向共享 LOCK (如果存在)
    PROCLOCK *proclock;         // 指向共享 PROCLOCK (如果存在)
    
    /* 计数 */
    int64 nLocks;               // 总持有次数
    
    /* 资源所有者 */
    int numLockOwners;          // 所有者数量
    int maxLockOwners;          // 数组容量
    LOCALLOCKOWNER *lockOwners; // 动态数组
    
    /* 快速路径标志 */
    bool holdsStrongLockCount;  // 是否计入强锁计数
    bool lockCleared;           // 是否已读取失效消息
} LOCALLOCK;
```

### 3.2 LOCALLOCK 的作用

**本地缓存**，避免重复访问共享内存：

```
场景: 同一个进程多次获取同一个锁

第1次 LockAcquire():
  1. 查找 LOCALLOCK (未找到)
  2. 访问共享内存 (慢)
  3. 创建 LOCALLOCK 缓存
  4. nLocks = 1

第2次 LockAcquire() (同一个锁):
  1. 查找 LOCALLOCK (找到!)
  2. nLocks++  (快! 无需访问共享内存)
  3. 直接返回

释放时:
  1. nLocks--
  2. 如果 nLocks == 0，访问共享内存释放
```

**性能提升**: 避免重复的共享内存访问和锁争用。

### 3.3 lockOwners 数组

跟踪不同 ResourceOwner 持有的次数：

```c
// 示例: 一个锁被两个 ResourceOwner 持有
lockOwners[0] = {owner: TopTransactionResourceOwner, nLocks: 2}
lockOwners[1] = {owner: SubTransactionResourceOwner, nLocks: 1}

// 总计
nLocks = 3
```

---

## 4. 三层哈希表关系图

```
┌─────────────────────────────────────────────────────────────┐
│                      共享内存                                │
│                                                              │
│  LOCK Hash Table (16 partitions)                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Key: LOCKTAG                                       │    │
│  │ Hash: tag → partition → bucket                     │    │
│  │                                                     │    │
│  │ [Partition 0]                                       │    │
│  │   Bucket 0: LOCK₁ → LOCK₂ → ...                   │    │
│  │   Bucket 1: LOCK₃ → ...                            │    │
│  │   ...                                               │    │
│  │                                                     │    │
│  │ [Partition 1]                                       │    │
│  │   ...                                               │    │
│  └────────────────────────────────────────────────────┘    │
│                      ↕                                       │
│  PROCLOCK Hash Table                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Key: (LOCK*, PGPROC*)                              │    │
│  │                                                     │    │
│  │ PROCLOCK₁: {LOCK₁, PGPROC_A, holdMask=0x0002}     │    │
│  │ PROCLOCK₂: {LOCK₁, PGPROC_B, holdMask=0x0008}     │    │
│  │ PROCLOCK₃: {LOCK₂, PGPROC_A, holdMask=0x0004}     │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                           ↕
┌─────────────────────────────────────────────────────────────┐
│                    本地内存 (每个进程)                       │
│                                                              │
│  LOCALLOCK Hash Table                                       │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Key: (LOCKTAG, LOCKMODE)                           │    │
│  │                                                     │    │
│  │ LOCALLOCK₁:                                         │    │
│  │   tag = {LOCKTAG_RELATION, tableOID=16384}        │    │
│  │   mode = AccessShareLock                           │    │
│  │   nLocks = 3                                       │    │
│  │   lock = &LOCK₁                                    │    │
│  │   proclock = &PROCLOCK₁                            │    │
│  │                                                     │    │
│  │ LOCALLOCK₂:                                         │    │
│  │   tag = {LOCKTAG_RELATION, tableOID=16385}        │    │
│  │   mode = RowExclusiveLock                          │    │
│  │   nLocks = 1                                       │    │
│  │   lock = &LOCK₂                                    │    │
│  │   proclock = &PROCLOCK₃                            │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. 快速路径数据结构

### 5.1 PGPROC 中的快速路径字段

```c
// src/include/storage/proc.h
struct PGPROC
{
    // ... 其他字段
    
    /* 快速路径锁 (仅限弱表锁) */
    uint64 fpLockBits;                  // 锁位掩码 (64位)
    Oid fpRelId[FP_LOCK_SLOTS_PER_BACKEND];  // 关系OID数组 (16个)
    
    // fpLockBits 编码:
    // 每个slot占4位: [unused:2][mode:2]
    // mode: 0=empty, 1=AccessShareLock, 2=RowShareLock
    
    /* 快速路径计数 */
    int fpLocalLockCount;               // 本地快速路径锁数
    int fpRelationIdHash;               // 哈希缓存
};

#define FP_LOCK_SLOTS_PER_BACKEND 16    // 每进程16个快速路径槽
```

### 5.2 快速路径编码示例

```c
// 假设进程持有以下快速路径锁:
// Slot 0: table OID 16384, AccessShareLock (mode=1)
// Slot 1: table OID 16385, RowShareLock (mode=2)
// Slot 2: 空
// ...

fpRelId[0] = 16384;
fpRelId[1] = 16385;
fpRelId[2] = InvalidOid;
// ...

// fpLockBits 编码 (每个slot 4位):
// Slot 0: mode=1 → 0001
// Slot 1: mode=2 → 0010
// Slot 2: empty → 0000
// ...
fpLockBits = 0x...0021  // (从右往左: slot0=1, slot1=2)
```

---

## 6. 锁等待队列

### 6.1 等待队列结构

```c
// LOCK 结构中的等待队列
typedef struct LOCK
{
    // ...
    dclist_head waitProcs;  // 双向循环链表
    // ...
} LOCK;

// PGPROC 作为等待队列节点
struct PGPROC
{
    // ...
    dlist_node links;           // 链表节点
    LOCK *waitLock;             // 正在等待的 LOCK
    LOCKMODE waitLockMode;      // 等待的锁模式
    PROCLOCK *waitProcLock;     // 等待的 PROCLOCK
    // ...
};
```

### 6.2 等待队列示例

```
LOCK (tableOID=16384)
  waitProcs: PGPROC₁ ⇄ PGPROC₂ ⇄ PGPROC₃
             │         │         │
             │         │         └→ 等待 ExclusiveLock
             │         └→ 等待 ShareLock
             └→ 等待 RowExclusiveLock

等待顺序 = 到达顺序 (FIFO)
```

---

## 7. 锁方法表 (Lock Method Table)

### 7.1 LockMethodData 结构

```c
// src/backend/storage/lmgr/lock.c:56
typedef struct LockMethodData
{
    int numLockModes;           // 锁模式数量 (8)
    const LOCKMASK *conflictTab; // 冲突矩阵
    const char * const *lockModeNames; // 模式名称
    const bool *trace_flag;     // 跟踪标志
} LockMethodData;
```

### 7.2 冲突矩阵

```c
// src/backend/storage/lmgr/lock.c:73
static const LOCKMASK LockConflicts[] = {
    0,  // 0: NoLock
    
    // 1: AccessShareLock
    LOCKBIT_ON(AccessExclusiveLock),
    
    // 2: RowShareLock
    LOCKBIT_ON(ExclusiveLock) | LOCKBIT_ON(AccessExclusiveLock),
    
    // 3: RowExclusiveLock
    LOCKBIT_ON(ShareLock) | LOCKBIT_ON(ShareRowExclusiveLock) |
    LOCKBIT_ON(ExclusiveLock) | LOCKBIT_ON(AccessExclusiveLock),
    
    // 4: ShareUpdateExclusiveLock
    LOCKBIT_ON(ShareUpdateExclusiveLock) |
    LOCKBIT_ON(ShareLock) | LOCKBIT_ON(ShareRowExclusiveLock) |
    LOCKBIT_ON(ExclusiveLock) | LOCKBIT_ON(AccessExclusiveLock),
    
    // 5: ShareLock
    LOCKBIT_ON(RowExclusiveLock) | LOCKBIT_ON(ShareUpdateExclusiveLock) |
    LOCKBIT_ON(ShareRowExclusiveLock) |
    LOCKBIT_ON(ExclusiveLock) | LOCKBIT_ON(AccessExclusiveLock),
    
    // 6: ShareRowExclusiveLock
    LOCKBIT_ON(RowExclusiveLock) | LOCKBIT_ON(ShareUpdateExclusiveLock) |
    LOCKBIT_ON(ShareLock) | LOCKBIT_ON(ShareRowExclusiveLock) |
    LOCKBIT_ON(ExclusiveLock) | LOCKBIT_ON(AccessExclusiveLock),
    
    // 7: ExclusiveLock
    LOCKBIT_ON(RowShareLock) | LOCKBIT_ON(RowExclusiveLock) |
    LOCKBIT_ON(ShareUpdateExclusiveLock) |
    LOCKBIT_ON(ShareLock) | LOCKBIT_ON(ShareRowExclusiveLock) |
    LOCKBIT_ON(ExclusiveLock) | LOCKBIT_ON(AccessExclusiveLock),
    
    // 8: AccessExclusiveLock (与所有锁冲突)
    LOCKBIT_ON(AccessShareLock) | LOCKBIT_ON(RowShareLock) |
    LOCKBIT_ON(RowExclusiveLock) | LOCKBIT_ON(ShareUpdateExclusiveLock) |
    LOCKBIT_ON(ShareLock) | LOCKBIT_ON(ShareRowExclusiveLock) |
    LOCKBIT_ON(ExclusiveLock) | LOCKBIT_ON(AccessExclusiveLock)
};
```

### 7.3 冲突检查函数

```c
// 检查锁模式冲突
static inline bool
LockModeConflicts(LOCKMODE lockmode1, LOCKMODE lockmode2)
{
    return (LockConflicts[lockmode1] & LOCKBIT_ON(lockmode2)) != 0;
}

// 检查与当前授予的锁是否冲突
static inline bool
LockCheckConflicts(LOCK *lock, LOCKMODE lockmode)
{
    return (lock->grantMask & LockConflicts[lockmode]) != 0;
}
```

---

## 8. 辅助数据结构

### 8.1 LockAcquireResult - 锁获取结果

```c
typedef enum
{
    LOCKACQUIRE_NOT_AVAIL,      // 锁不可用 (dontWait=true)
    LOCKACQUIRE_OK,             // 成功获取
    LOCKACQUIRE_ALREADY_HELD,   // 已持有，递增计数
    LOCKACQUIRE_ALREADY_CLEAR,  // 已清除，递增计数
} LockAcquireResult;
```

### 8.2 DeadLockState - 死锁状态

```c
typedef enum
{
    DS_NOT_YET_CHECKED,         // 尚未检查
    DS_NO_DEADLOCK,             // 无死锁
    DS_SOFT_DEADLOCK,           // 软死锁 (可能解决)
    DS_HARD_DEADLOCK,           // 硬死锁 (必须中止)
    DS_BLOCKED_BY_AUTOVACUUM,   // 被 autovacuum 阻塞
} DeadLockState;
```

### 8.3 VirtualTransactionId - 虚拟事务ID

```c
typedef struct
{
    ProcNumber procNumber;      // 进程编号
    LocalTransactionId lxid;    // 本地事务ID
} VirtualTransactionId;

// 组合成字符串: "3/42" (进程3的第42个事务)
```

---

## 9. 内存布局和大小

### 9.1 结构体大小 (64位系统)

| 结构体 | 大小 | 数量 | 位置 |
|--------|------|------|------|
| LOCK | ~200B | 动态 | 共享内存哈希表 |
| PROCLOCK | ~80B | 动态 | 共享内存哈希表 |
| LOCALLOCK | ~100B | 动态 | 本地内存哈希表 |
| LOCKTAG | 16B | - | 嵌入其他结构 |
| PGPROC快速路径 | ~100B | 固定 | PGPROC结构中 |

### 9.2 共享内存估算

```
max_locks_per_transaction = 64 (默认)
max_connections = 100

LOCK Hash Table:
  初始槽位 = max_connections * max_locks_per_transaction
           = 100 * 64 = 6400
  
PROCLOCK Hash Table:
  初始槽位 = max_connections * max_locks_per_transaction * 2
           = 100 * 64 * 2 = 12800

总共享内存 ≈ (6400 * 200B) + (12800 * 80B)
           ≈ 1.2MB + 1MB = 2.2MB
```

---

## 10. 数据结构关系总结

```
获取锁时的数据流:

1. 查找/创建 LOCALLOCK (本地哈希表)
   ↓
2. 如果 nLocks > 0: 递增计数，返回
   ↓
3. 否则，访问共享内存:
   ├─ 查找/创建 LOCK (共享哈希表)
   ├─ 查找/创建 PROCLOCK (共享哈希表)
   └─ 检查冲突
   ↓
4. 如果无冲突:
   ├─ 更新 LOCK (requested++, granted++, grantMask|=...)
   ├─ 更新 PROCLOCK (holdMask|=...)
   └─ 更新 LOCALLOCK (nLocks++, lock=, proclock=)
   ↓
5. 如果有冲突:
   └─ 加入等待队列 (LOCK->waitProcs)

释放锁时的数据流:

1. 查找 LOCALLOCK
   ↓
2. 递减 nLocks
   ↓
3. 如果 nLocks > 0: 返回
   ↓
4. 否则，访问共享内存:
   ├─ 更新 LOCK (granted--, grantMask&=...)
   ├─ 更新 PROCLOCK (holdMask&=...)
   └─ 唤醒等待进程
   ↓
5. 删除 LOCALLOCK 条目
```

---

## 总结

### 核心数据结构

1. **LOCK**: 被锁对象的信息 (共享内存)
2. **PROCLOCK**: LOCK和PGPROC的关联 (共享内存)
3. **LOCALLOCK**: 本地缓存，避免重复访问 (本地内存)
4. **快速路径**: PGPROC中的数组，优化弱表锁 (共享内存)

### 关键设计

- ✅ 三层哈希表: 清晰的职责分离
- ✅ 本地缓存: 大幅减少共享内存访问
- ✅ 快速路径: 常见场景零哈希查找
- ✅ 16个分区: 降低锁争用

---

**下一步**: 阅读 [03_implementation_flow.md](03_implementation_flow.md) 了解锁获取和释放流程

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-01-16

