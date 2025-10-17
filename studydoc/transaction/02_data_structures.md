# Transaction Manager 核心数据结构

## 1. TransactionStateData - 事务状态结构

### 1.1 结构定义

```c
// src/backend/access/transam/xact.c:191-216
typedef struct TransactionStateData
{
    /* 事务标识 */
    FullTransactionId fullTransactionId;  // 完整64位事务ID
    SubTransactionId subTransactionId;    // 子事务ID
    char *name;                           // SAVEPOINT 名称 (如果是子事务)
    int savepointLevel;                   // SAVEPOINT 嵌套层次
    
    /* 状态信息 */
    TransState state;                     // 低层事务状态
    TBlockState blockState;               // 高层事务块状态
    int nestingLevel;                     // 事务嵌套深度 (1=顶层)
    int gucNestLevel;                     // GUC 上下文嵌套深度
    
    /* 资源管理 */
    MemoryContext curTransactionContext;  // 事务生命周期内存上下文
    ResourceOwner curTransactionOwner;    // 资源所有者 (锁、文件等)
    
    /* 子事务管理 */
    TransactionId *childXids;             // 已提交子事务ID数组
    int nChildXids;                       // 当前子事务数量
    int maxChildXids;                     // 数组分配容量
    
    /* 用户权限 */
    Oid prevUser;                         // 之前的用户ID
    int prevSecContext;                   // 之前的安全上下文
    bool prevXactReadOnly;                // 进入时的只读状态
    
    /* 恢复相关 */
    bool startedInRecovery;               // 是否在恢复模式启动
    
    /* WAL相关 */
    bool didLogXid;                       // 事务ID是否已写入WAL
    bool topXidLogged;                    // 顶层XID是否已记录 (子事务用)
    
    /* 并行查询 */
    int parallelModeLevel;                // 并行模式计数器
    bool parallelChildXact;               // 父事务是否在并行模式
    
    /* 事务链 */
    bool chain;                           // COMMIT AND CHAIN 标志
    
    /* 栈结构 */
    struct TransactionStateData *parent;  // 父事务指针 (子事务栈)
} TransactionStateData;

typedef TransactionStateData *TransactionState;
```

### 1.2 字段详解

#### 1.2.1 fullTransactionId - 完整事务ID

```c
typedef struct FullTransactionId
{
    uint64 value;  // 64位完整事务ID
} FullTransactionId;

// 组成: Epoch (高32位) + XID (低32位)
// 例如: value = 0x0000000200001234
//       Epoch = 2
//       XID = 0x1234 (4660)
```

**特性**:
- ✅ 解决32位XID回卷问题
- ✅ 不需要冻结操作
- ✅ 支持 2^64 个事务 (~18 quintillion)

#### 1.2.2 state 和 blockState - 双层状态

```c
TransState state;        // 低层状态 (6种)
TBlockState blockState;  // 高层状态 (22种)

// 示例:
// 自动提交模式执行SELECT:
//   state = TRANS_INPROGRESS
//   blockState = TBLOCK_STARTED
//
// 显式事务块中:
//   state = TRANS_INPROGRESS
//   blockState = TBLOCK_INPROGRESS
```

#### 1.2.3 nestingLevel - 嵌套层次

```c
int nestingLevel;  // 1 = 顶层事务, 2+ = 子事务

// 示例:
BEGIN;                    // nestingLevel = 1
    SAVEPOINT sp1;        // nestingLevel = 2
        SAVEPOINT sp2;    // nestingLevel = 3
        ROLLBACK TO sp2;  // nestingLevel = 3 → 2
    RELEASE sp1;          // nestingLevel = 2 → 1
COMMIT;                   // nestingLevel = 1 → 0
```

#### 1.2.4 childXids - 子事务ID数组

```c
TransactionId *childXids;  // 动态数组
int nChildXids;            // 当前元素数
int maxChildXids;          // 容量

// 用途: 记录所有已提交的子事务ID，用于可见性判断
// 例如:
// BEGIN;  XID=100
//   SAVEPOINT sp1;  XID=101 ← 子事务
//   ... 修改数据 ...
//   RELEASE sp1;    → childXids[0] = 101
// COMMIT;  → 将childXids记录到WAL
```

### 1.3 全局事务状态变量

```c
// src/backend/access/transam/xact.c

// 顶层事务状态 (静态分配)
static TransactionStateData TopTransactionStateData = {
    .state = TRANS_DEFAULT,
    .blockState = TBLOCK_DEFAULT,
    .topXidLogged = false,
};

// 当前事务状态指针 (指向栈顶)
static TransactionState CurrentTransactionState = &TopTransactionStateData;

// 子事务ID计数器
static SubTransactionId currentSubTransactionId;

// 命令ID计数器
static CommandId currentCommandId;
static bool currentCommandIdUsed;  // 是否已使用
```

---

## 2. PGPROC - 进程事务信息

### 2.1 结构定义

```c
// src/include/storage/proc.h:162-309
struct PGPROC
{
    /* 必须是第一个字段 - 用于链表 */
    dlist_node links;                    // 链表节点
    dlist_head *procgloballist;          // 所属全局链表
    
    /* 同步原语 */
    PGSemaphore sem;                     // 信号量 (用于睡眠/唤醒)
    ProcWaitStatus waitStatus;           // 等待状态
    Latch procLatch;                     // Latch (用于事件通知)
    
    /* 事务标识 (镜像到ProcGlobal->xids[]) */
    TransactionId xid;                   // 当前事务ID (如果已分配)
    TransactionId xmin;                  // 最小活跃事务ID
    
    int pid;                             // 后端进程ID
    int pgxactoff;                       // 在ProcGlobal数组中的偏移
    
    /* 虚拟事务ID */
    struct {
        ProcNumber procNumber;           // 进程编号
        LocalTransactionId lxid;         // 本地事务ID
    } vxid;
    
    /* 数据库和用户 */
    Oid databaseId;                      // 数据库OID
    Oid roleId;                          // 角色OID
    Oid tempNamespaceId;                 // 临时模式OID
    
    /* 后端类型标识 */
    bool isBackgroundWorker;             // 是否后台worker
    bool isWalSender;                    // 是否WAL sender
    
    /* 锁信息 */
    int delayChkptFlags;                 // 延迟checkpoint标志
    uint8 statusFlags;                   // 状态标志 (镜像)
    
    /* 子事务缓存 */
    int subxidStatus;                    // 子事务溢出状态
    struct XidCache {
        TransactionId xids[PGPROC_MAX_CACHED_SUBXIDS];  // 子事务ID缓存
    } subxids;
    
    /* 等待锁信息 */
    LOCKMASK waitLock;                   // 等待的锁
    PROCLOCK *waitProcLock;              // 等待的PROCLOCK
    
    /* 组提交 (Group Commit) */
    pg_atomic_uint32 procArrayGroupNext; // 下一个进程
    
    /* 锁组 (并行查询) */
    PGPROC *lockGroupLeader;             // 锁组组长
    dlist_head lockGroupMembers;         // 组成员链表
    dlist_node lockGroupLink;            // 组成员链接
};

extern PGDLLIMPORT PGPROC *MyProc;  // 当前进程的PGPROC
```

### 2.2 关键字段详解

#### 2.2.1 xid 和 xmin

```c
TransactionId xid;   // 当前事务ID
TransactionId xmin;  // 最小活跃XID

// xid:
// - 事务开始时为InvalidTransactionId (0)
// - 执行第一个修改操作时调用GetNewTransactionId()分配
// - 提交/回滚后重新设为InvalidTransactionId
//
// xmin:
// - 取快照时设置为当前最小活跃XID
// - 用于VACUUM: 不能清理 xmin 之后的死元组
// - 长事务的xmin会阻碍VACUUM
```

**示例**:

```
时刻 T1: Backend A 启动事务
    MyProc->xid = InvalidTransactionId
    MyProc->xmin = 100 (当前最小活跃XID)
    
时刻 T2: 执行 UPDATE
    GetNewTransactionId() → xid = 105
    MyProc->xid = 105
    
时刻 T3: COMMIT
    MyProc->xid = InvalidTransactionId
```

#### 2.2.2 vxid - 虚拟事务ID

```c
struct {
    ProcNumber procNumber;         // 进程编号 (0~N)
    LocalTransactionId lxid;       // 本地事务ID (递增计数器)
} vxid;

// 组合成虚拟事务ID: procNumber/lxid
// 例如: 3/42 表示进程3的第42个事务
//
// 用途:
// - 锁定: LOCK TABLE ... NOWAIT
// - 监控: pg_stat_activity.backend_xid
// - 分析: pg_locks.virtualtransaction
```

#### 2.2.3 subxids - 子事务缓存

```c
#define PGPROC_MAX_CACHED_SUBXIDS 64  // 最多缓存64个子事务ID

struct XidCache {
    TransactionId xids[PGPROC_MAX_CACHED_SUBXIDS];
} subxids;

int subxidStatus;  // 缓存状态
// SUBXIDS_CLEAR = 0      没有子事务
// SUBXIDS_IN_ARRAY = 1   子事务在数组中
// SUBXIDS_OVERFLOW = 2   子事务溢出到磁盘

// 当子事务超过64个时:
// 1. 设置 subxidStatus = SUBXIDS_OVERFLOW
// 2. 记录到 pg_subtrans SLRU
// 3. 后续可见性判断需要查询 pg_subtrans
```

---

## 3. PROC_HDR - 进程全局信息

### 3.1 结构定义

```c
// src/include/storage/proc.h:370-412
typedef struct PROC_HDR
{
    /* PGPROC数组 */
    PGPROC *allProcs;                     // 所有PGPROC结构数组
    uint32 allProcCount;                  // 总数量
    
    /* 密集数组 (镜像PGPROC关键字段，提高缓存效率) */
    TransactionId *xids;                  // XID数组
    XidCacheStatus *subxidStates;         // 子事务状态数组
    uint8 *statusFlags;                   // 状态标志数组
    
    /* 空闲PGPROC链表 */
    dlist_head freeProcs;                 // 普通后端空闲链表
    dlist_head autovacFreeProcs;          // autovacuum worker 空闲链表
    dlist_head bgworkerFreeProcs;         // 后台worker 空闲链表
    dlist_head walsenderFreeProcs;        // WAL sender 空闲链表
    
    /* 组提交 */
    pg_atomic_uint32 procArrayGroupFirst;  // XID清理组头
    pg_atomic_uint32 clogGroupFirst;       // CLOG更新组头
    
    /* 关键进程Latch */
    Latch *walwriterLatch;                // WAL writer latch
    Latch *checkpointerLatch;             // checkpointer latch
    
    /* 性能 */
    int spins_per_delay;                  // 自旋锁参数
    
    /* Startup进程等待的buffer */
    int startupBufferPinWaitBufId;        // -1表示未等待
} PROC_HDR;

extern PGDLLIMPORT PROC_HDR *ProcGlobal;  // 全局指针
```

### 3.2 密集数组设计

**为什么需要密集数组?**

```
传统设计 (仅使用PGPROC数组):
┌─────────┬─────────┬─────────┬─────────┬─────────┐
│ PGPROC0 │ PGPROC1 │ PGPROC2 │ PGPROC3 │ PGPROC4 │
│ (308B)  │ (308B)  │ (308B)  │ (308B)  │ (308B)  │
└─────────┴─────────┴─────────┴─────────┴─────────┘
问题: GetSnapshotData()需要遍历所有PGPROC，缓存不友好

优化设计 (密集数组):
┌────┬────┬────┬────┬────┐  ← xids数组 (4B each)
│100 │101 │102 │103 │104 │
└────┴────┴────┴────┴────┘
好处: 
- 缓存友好 (连续内存)
- 减少访问延迟
- 提高快照创建速度 50%+
```

**镜像机制**:

```c
// 更新PGPROC->xid时，同时更新密集数组
void AssignTransactionId(TransactionState s)
{
    // 1. 获取锁
    LWLockAcquire(XidGenLock, LW_EXCLUSIVE);
    
    // 2. 分配XID
    s->fullTransactionId = GetNewTransactionId(false);
    
    // 3. 更新PGPROC
    MyProc->xid = XidFromFullTransactionId(s->fullTransactionId);
    
    // 4. 更新密集数组 (关键!)
    ProcGlobal->xids[MyProc->pgxactoff] = MyProc->xid;
    
    // 5. 释放锁
    LWLockRelease(XidGenLock);
}
```

---

## 4. TransactionId 相关类型

### 4.1 类型定义

```c
// 32位事务ID
typedef uint32 TransactionId;

// 64位完整事务ID
typedef struct FullTransactionId
{
    uint64 value;
} FullTransactionId;

// 子事务ID
typedef uint32 SubTransactionId;

// 本地事务ID
typedef uint32 LocalTransactionId;

// 命令ID
typedef uint32 CommandId;
```

### 4.2 特殊事务ID

```c
// src/include/access/transam.h:27-31
#define InvalidTransactionId       ((TransactionId) 0)
#define BootstrapTransactionId     ((TransactionId) 1)
#define FrozenTransactionId        ((TransactionId) 2)
#define FirstNormalTransactionId   ((TransactionId) 3)
#define MaxTransactionId           ((TransactionId) 0xFFFFFFFF)
```

| 名称 | 值 | 含义 | 用途 |
|------|-----|------|------|
| InvalidTransactionId | 0 | 无效事务ID | 表示未分配XID |
| BootstrapTransactionId | 1 | 引导事务ID | 初始化时使用 |
| FrozenTransactionId | 2 | 冻结事务ID | VACUUM冻结后的t_xmin |
| FirstNormalTransactionId | 3 | 第一个正常XID | 用户事务从3开始 |

### 4.3 FullTransactionId 操作

```c
// 从 Epoch 和 XID 构造
static inline FullTransactionId
FullTransactionIdFromEpochAndXid(uint32 epoch, TransactionId xid)
{
    FullTransactionId result;
    result.value = ((uint64) epoch << 32) | xid;
    return result;
}

// 提取XID
static inline TransactionId
XidFromFullTransactionId(FullTransactionId x)
{
    return (TransactionId) x.value;
}

// 提取Epoch
static inline uint32
EpochFromFullTransactionId(FullTransactionId x)
{
    return (uint32) (x.value >> 32);
}

// 比较
static inline bool
FullTransactionIdPrecedes(FullTransactionId a, FullTransactionId b)
{
    return a.value < b.value;
}
```

---

## 5. GlobalTransactionData - 两阶段提交

### 5.1 结构定义

```c
// src/backend/access/transam/twophase.c:147-170
typedef struct GlobalTransactionData
{
    /* 链表 */
    GlobalTransaction next;               // 空闲链表指针
    
    /* PGPROC关联 */
    int pgprocno;                         // 关联的dummy PGPROC编号
    
    /* 时间戳 */
    TimestampTz prepared_at;              // 准备时间
    
    /* WAL位置 */
    XLogRecPtr prepare_start_lsn;         // PREPARE记录起始LSN
    XLogRecPtr prepare_end_lsn;           // PREPARE记录结束LSN
    
    /* 事务ID */
    TransactionId xid;                    // 事务ID
    
    /* 所有者 */
    Oid owner;                            // 执行PREPARE的用户OID
    
    /* 状态 */
    ProcNumber locking_backend;           // 当前处理该事务的后端
    bool valid;                           // PGPROC是否在proc array中
    bool ondisk;                          // 状态文件是否已持久化
    bool inredo;                          // 是否通过redo恢复
    
    /* 全局事务ID */
    char gid[GIDSIZE];                    // 全局事务ID (最多200字符)
} GlobalTransactionData;

typedef GlobalTransactionData *GlobalTransaction;

#define GIDSIZE 200
```

### 5.2 TwoPhaseStateData - 2PC全局状态

```c
// src/backend/access/transam/twophase.c:176-186
typedef struct TwoPhaseStateData
{
    /* 空闲链表 */
    GlobalTransaction freeGXacts;         // 空闲GlobalTransaction链表头
    
    /* 准备好的事务数量 */
    int numPrepXacts;                     // 当前准备好的事务数
    
    /* 准备好的事务数组 */
    GlobalTransaction prepXacts[FLEXIBLE_ARRAY_MEMBER];
    // 大小: max_prepared_transactions
} TwoPhaseStateData;

static TwoPhaseStateData *TwoPhaseState;  // 共享内存指针
```

### 5.3 生命周期

```
1. PREPARE TRANSACTION 'gid':
   ├─ 分配 GlobalTransaction
   ├─ 设置 valid=false, gid='gid'
   ├─ 锁定 (locking_backend=MyProcNumber)
   └─ 写入状态文件到磁盘

2. 准备成功:
   ├─ 设置 valid=true
   ├─ 创建 dummy PGPROC
   └─ 加入 ProcArray

3. COMMIT/ROLLBACK PREPARED 'gid':
   ├─ 检查 valid=true && !locked
   ├─ 锁定 (locking_backend=MyProcNumber)
   ├─ 执行提交/回滚
   ├─ 从 ProcArray 移除
   ├─ 删除状态文件
   └─ 释放 GlobalTransaction
```

---

## 6. 辅助数据结构

### 6.1 SerializedTransactionState - 并行查询

```c
// src/backend/access/transam/xact.c:224-233
typedef struct SerializedTransactionState
{
    int xactIsoLevel;                    // 隔离级别
    bool xactDeferrable;                 // 是否可延迟
    FullTransactionId topFullTransactionId;       // 顶层事务ID
    FullTransactionId currentFullTransactionId;   // 当前事务ID
    CommandId currentCommandId;          // 当前命令ID
    int nParallelCurrentXids;            // 并行worker数量
    TransactionId parallelCurrentXids[FLEXIBLE_ARRAY_MEMBER];  // 并行XID数组
} SerializedTransactionState;

// 用途: 将事务状态序列化后传递给并行worker进程
```

### 6.2 TransamVariablesData - 全局事务变量

```c
// src/backend/access/transam/varsup.c
typedef struct TransamVariablesData
{
    /* 下一个XID */
    FullTransactionId nextXid;           // 下一个待分配的XID
    TransactionId oldestXid;             // 系统中最老的XID
    TransactionId xidVacLimit;           // VACUUM触发点
    TransactionId xidWarnLimit;          // 警告触发点
    TransactionId xidStopLimit;          // 停止接受新事务的点
    TransactionId xidWrapLimit;          // 回卷点
    
    /* 下一个OID */
    Oid nextOid;                         // 下一个待分配的OID
    
    /* CLOG oldest */
    TransactionId oldestClogXid;         // CLOG中最老的XID
} TransamVariablesData;

extern TransamVariablesData *TransamVariables;  // 共享内存指针
```

---

## 7. 数据结构关系图

```
用户执行事务
    │
    ▼
┌──────────────────────────────────────┐
│  TransactionStateData (栈结构)       │
│  ┌────────────────────────────────┐  │
│  │ Level 1: 顶层事务              │  │
│  │ fullTransactionId = 100        │  │
│  │ state = TRANS_INPROGRESS       │  │
│  │ parent = NULL                  │  │
│  └────────────────────────────────┘  │
│               ▲                       │
│               │ parent                │
│  ┌────────────────────────────────┐  │
│  │ Level 2: 子事务 (SAVEPOINT)    │  │
│  │ subTransactionId = 2           │  │
│  │ name = "sp1"                   │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
    │
    │ 存储进程级事务信息
    ▼
┌──────────────────────────────────────┐
│  MyProc (PGPROC)                     │
│  ┌────────────────────────────────┐  │
│  │ xid = 100                      │◀─┼─ 镜像到
│  │ xmin = 95                      │  │    ProcGlobal->xids[pgxactoff]
│  │ pid = 12345                    │  │
│  │ pgxactoff = 42                 │  │
│  │ subxids.xids[0] = 101          │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
    │
    │ 被管理于
    ▼
┌──────────────────────────────────────┐
│  ProcGlobal (PROC_HDR)               │
│  ┌────────────────────────────────┐  │
│  │ allProcs[0..N]                 │  │
│  │ xids[0..N]       ← 密集数组    │  │
│  │ subxidStates[0..N]             │  │
│  │ statusFlags[0..N]              │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
    │
    │ 分配XID时访问
    ▼
┌──────────────────────────────────────┐
│  TransamVariables                    │
│  ┌────────────────────────────────┐  │
│  │ nextXid = 103                  │  │
│  │ oldestXid = 50                 │  │
│  │ xidVacLimit = ...              │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

---

## 8. 内存布局和大小

### 8.1 结构体大小

| 结构体 | 大小 (64位系统) | 数量 | 总内存 |
|--------|------------------|------|--------|
| TransactionStateData | ~200B | 每个嵌套层次1个 | 动态 |
| PGPROC | ~312B | max_connections + autovacuum + ... | 约 N×312B |
| PROC_HDR | ~100B + 数组 | 1个 (共享内存) | ~100B + N×12B |
| GlobalTransactionData | ~300B | max_prepared_transactions | M×300B |

**示例计算** (max_connections=100):

```
PGPROC数组:      100 × 312B = 31.2 KB
密集数组:
  xids:          100 × 4B   = 400 B
  subxidStates:  100 × 1B   = 100 B
  statusFlags:   100 × 1B   = 100 B
总计:                        ~32 KB
```

### 8.2 共享内存分配

```c
// 计算所需共享内存
Size ProcGlobalShmemSize(void)
{
    Size size = 0;
    
    // PROC_HDR
    size = add_size(size, sizeof(PROC_HDR));
    
    // PGPROC数组
    size = add_size(size, mul_size(TotalProcs, sizeof(PGPROC)));
    
    // 密集数组
    size = add_size(size, mul_size(TotalProcs, sizeof(TransactionId)));     // xids
    size = add_size(size, mul_size(TotalProcs, sizeof(XidCacheStatus)));    // subxidStates
    size = add_size(size, mul_size(TotalProcs, sizeof(uint8)));             // statusFlags
    
    return size;
}
```

---

## 9. 关键宏和内联函数

### 9.1 PGPROC 访问宏

```c
// 通过ProcNumber获取PGPROC
#define GetPGProcByNumber(n) (&ProcGlobal->allProcs[(n)])

// 通过PGPROC获取ProcNumber
#define GetNumberFromPGProc(proc) ((proc) - &ProcGlobal->allProcs[0])

// 当前进程的PGPROC
extern PGPROC *MyProc;
```

### 9.2 事务ID判断

```c
// 是否为有效事务ID
#define TransactionIdIsValid(xid) ((xid) != InvalidTransactionId)

// 是否为普通事务ID
#define TransactionIdIsNormal(xid) ((xid) >= FirstNormalTransactionId)

// 是否为特殊事务ID
#define TransactionIdEquals(id1, id2) ((id1) == (id2))
#define TransactionIdPrecedes(id1, id2) (...)  // 循环比较
#define TransactionIdFollows(id1, id2) (...)   // 循环比较
```

---

## 10. 使用示例

### 10.1 获取当前事务状态

```c
TransactionState s = CurrentTransactionState;

elog(LOG, "Transaction: XID=%u, nestingLevel=%d, state=%d",
     XidFromFullTransactionId(s->fullTransactionId),
     s->nestingLevel,
     s->state);
```

### 10.2 遍历子事务栈

```c
void ShowTransactionStack(void)
{
    TransactionState s = CurrentTransactionState;
    
    while (s != NULL)
    {
        elog(LOG, "Level %d: XID=%u, SubXID=%u, name=%s",
             s->nestingLevel,
             XidFromFullTransactionId(s->fullTransactionId),
             s->subTransactionId,
             s->name ? s->name : "");
        
        s = s->parent;
    }
}
```

### 10.3 检查进程的事务ID

```c
for (int i = 0; i < ProcGlobal->allProcCount; i++)
{
    PGPROC *proc = GetPGProcByNumber(i);
    TransactionId xid = ProcGlobal->xids[proc->pgxactoff];
    
    if (TransactionIdIsValid(xid))
    {
        elog(LOG, "Backend PID %d has XID %u", proc->pid, xid);
    }
}
```

---

## 11. 总结

### 11.1 核心要点

1. **TransactionStateData**: 栈结构，支持嵌套子事务
2. **PGPROC**: 每个后端一个，存储进程级事务信息
3. **PROC_HDR**: 全局进程管理，使用密集数组优化性能
4. **FullTransactionId**: 64位事务ID，解决回卷问题
5. **GlobalTransactionData**: 支持两阶段提交的持久化事务

### 11.2 性能优化点

- ✅ 密集数组减少缓存缺失
- ✅ 镜像关键字段提高访问速度
- ✅ 子事务缓存减少SLRU访问
- ✅ 懒惰XID分配节省资源

---

**下一步**: 阅读 [03_implementation_flow.md](03_implementation_flow.md) 了解事务执行流程

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-01-16

