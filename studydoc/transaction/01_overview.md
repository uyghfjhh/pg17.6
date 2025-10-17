# Transaction Manager 事务管理概述

## 1. 事务管理系统简介

### 1.1 什么是事务?

事务(Transaction)是数据库操作的逻辑单元，将一组操作视为一个不可分割的整体。PostgreSQL 的事务管理器确保所有操作要么全部成功，要么全部失败。

**ACID 特性保证**:

```
┌─────────────────────────────────────────────────────────────┐
│                    ACID 特性实现                             │
├─────────────────────────────────────────────────────────────┤
│ A - Atomicity (原子性)                                       │
│     → Transaction Manager: 全部成功或全部回滚                │
│                                                              │
│ C - Consistency (一致性)                                     │
│     → Transaction Manager + Constraints: 保持数据一致        │
│                                                              │
│ I - Isolation (隔离性)                                       │
│     → MVCC + Lock Manager: 事务间相互隔离                    │
│                                                              │
│ D - Durability (持久性)                                      │
│     → WAL: 提交的数据永久保存                                │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 事务管理器的职责

| 职责 | 说明 | 实现方式 |
|------|------|----------|
| **生命周期管理** | 从BEGIN到COMMIT/ROLLBACK的完整流程 | 三层架构 + 状态机 |
| **事务ID分配** | 为每个事务分配唯一ID | GetNewTransactionId() |
| **状态维护** | 跟踪事务状态 (活跃/提交/回滚) | TransactionState + PGPROC |
| **子事务支持** | SAVEPOINT实现嵌套事务 | 事务栈结构 |
| **两阶段提交** | 分布式事务支持 | 2PC协议 |
| **资源管理** | 内存、锁、文件等资源清理 | ResourceOwner |
| **错误恢复** | 事务失败时的清理 | AbortTransaction() |

---

## 2. 三层事务系统架构

PostgreSQL 采用**三层架构**，实现了优雅的分层设计：

```
用户层 (SQL)
    │
    │  BEGIN / COMMIT / ROLLBACK / SAVEPOINT
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  高层 (Transaction Block Layer)                              │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  处理用户事务命令                                            │
│  函数: BeginTransactionBlock()                              │
│       EndTransactionBlock()                                 │
│       UserAbortTransactionBlock()                           │
│       DefineSavepoint()                                     │
│       RollbackToSavepoint()                                 │
│       ReleaseSavepoint()                                    │
│                                                              │
│  文件: src/backend/access/transam/xact.c (2800-4400行)     │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  中层 (Transaction Command Layer)                            │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  每个查询前后调用                                            │
│  函数: StartTransactionCommand()                            │
│       CommitTransactionCommand()                            │
│       AbortCurrentTransaction()                             │
│                                                              │
│  文件: src/backend/access/transam/xact.c (2698-2810行)     │
│                                                              │
│  调用时机:                                                   │
│  ┌─────────────────────────────────────────┐               │
│  │ StartTransactionCommand()                │               │
│  │     ↓                                    │               │
│  │ 执行查询 (SELECT / UPDATE / ...)         │               │
│  │     ↓                                    │               │
│  │ CommitTransactionCommand()               │               │
│  └─────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  低层 (Low-Level Transaction Layer)                          │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  实际的事务操作                                              │
│  函数: StartTransaction()                                   │
│       CommitTransaction()                                   │
│       PrepareTransaction()                                  │
│       AbortTransaction()                                    │
│       CleanupTransaction()                                  │
│                                                              │
│       StartSubTransaction()                                 │
│       CommitSubTransaction()                                │
│       AbortSubTransaction()                                 │
│       CleanupSubTransaction()                               │
│                                                              │
│  文件: src/backend/access/transam/xact.c (2011-2630行)     │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
底层系统 (MVCC / WAL / Lock Manager / Buffer Manager)
```

### 2.1 为什么需要三层?

| 层次 | 作用 | 原因 |
|------|------|------|
| **高层** | 处理用户命令语义 | BEGIN可能在事务中执行(报错)，COMMIT可能在非事务中执行(忽略) |
| **中层** | 管理查询级事务 | 自动提交模式下每个查询是一个事务，显式事务块内多个查询共享事务 |
| **低层** | 执行真正的事务逻辑 | 与底层系统(WAL、MVCC)交互，保证原子性和持久性 |

---

## 3. 双层状态机设计

PostgreSQL 使用两个状态机协同工作：

### 3.1 TransState - 低层事务状态

```c
// src/backend/access/transam/xact.c:139
typedef enum TransState
{
    TRANS_DEFAULT,      // 空闲状态，没有事务
    TRANS_START,        // 事务正在启动
    TRANS_INPROGRESS,   // 事务进行中 (稳定状态)
    TRANS_COMMIT,       // 提交进行中
    TRANS_ABORT,        // 回滚进行中
    TRANS_PREPARE,      // 两阶段提交准备中
} TransState;
```

**状态转换图**:

```
           StartTransaction()
    DEFAULT ────────────────→ START
                                │
                                │ (初始化完成)
                                ▼
                          INPROGRESS ←──────┐
                                │           │
              ┌─────────────────┼───────────┤
              │                 │           │
              │ Commit          │ Abort     │ 2PC
              ▼                 ▼           ▼
           COMMIT             ABORT      PREPARE
              │                 │           │
              └────→ (完成) ←───┴───────────┘
                       │
                       ▼
                    DEFAULT
```

### 3.2 TBlockState - 高层事务块状态

```c
// src/backend/access/transam/xact.c:155
typedef enum TBlockState
{
    /* 非事务块状态 */
    TBLOCK_DEFAULT,               // 空闲
    TBLOCK_STARTED,               // 单查询事务 (自动提交)
    
    /* 事务块状态 */
    TBLOCK_BEGIN,                 // BEGIN 刚执行
    TBLOCK_INPROGRESS,            // 事务块进行中
    TBLOCK_IMPLICIT_INPROGRESS,   // 隐式 BEGIN 后
    TBLOCK_PARALLEL_INPROGRESS,   // 并行 worker 中
    TBLOCK_END,                   // 收到 COMMIT 命令
    TBLOCK_ABORT,                 // 事务失败，等待 ROLLBACK
    TBLOCK_ABORT_END,             // 事务失败，收到 ROLLBACK
    TBLOCK_ABORT_PENDING,         // 活跃事务，收到 ROLLBACK
    TBLOCK_PREPARE,               // 收到 PREPARE TRANSACTION
    
    /* 子事务状态 */
    TBLOCK_SUBBEGIN,              // SAVEPOINT 刚创建
    TBLOCK_SUBINPROGRESS,         // 子事务进行中
    TBLOCK_SUBRELEASE,            // 收到 RELEASE SAVEPOINT
    TBLOCK_SUBCOMMIT,             // 子事务中收到 COMMIT
    TBLOCK_SUBABORT,              // 子事务失败
    TBLOCK_SUBABORT_END,          // 子事务失败，收到 ROLLBACK
    TBLOCK_SUBABORT_PENDING,      // 活跃子事务，收到 ROLLBACK
    TBLOCK_SUBRESTART,            // 收到 ROLLBACK TO
    TBLOCK_SUBABORT_RESTART,      // 失败的子事务，收到 ROLLBACK TO
} TBlockState;
```

**主要状态转换**:

```
自动提交模式 (每个查询一个事务):
DEFAULT → STARTED → DEFAULT → STARTED → ...

显式事务块:
DEFAULT → BEGIN → INPROGRESS → END → DEFAULT
                     │
                     │ (出错)
                     ▼
                   ABORT → ABORT_END → DEFAULT

带子事务:
INPROGRESS → SUBBEGIN → SUBINPROGRESS → SUBRELEASE → INPROGRESS
                            │
                            │ (ROLLBACK TO)
                            ▼
                       SUBRESTART → SUBINPROGRESS
```

---

## 4. 核心数据结构概览

### 4.1 TransactionStateData - 事务状态栈

```c
// src/backend/access/transam/xact.c:191
typedef struct TransactionStateData
{
    FullTransactionId fullTransactionId;  // 完整64位事务ID
    SubTransactionId subTransactionId;    // 子事务ID (32位)
    char *name;                           // SAVEPOINT 名称
    int savepointLevel;                   // SAVEPOINT 嵌套层次
    
    TransState state;                     // 低层状态
    TBlockState blockState;               // 高层状态
    int nestingLevel;                     // 事务嵌套深度 (1=顶层)
    
    MemoryContext curTransactionContext;  // 事务内存上下文
    ResourceOwner curTransactionOwner;    // 资源所有者
    
    TransactionId *childXids;             // 已提交的子事务ID数组
    int nChildXids;                       // 子事务数量
    int maxChildXids;                     // 数组容量
    
    bool didLogXid;                       // 是否已写入WAL
    int parallelModeLevel;                // 并行模式层次
    bool chain;                           // COMMIT AND CHAIN
    
    struct TransactionStateData *parent;  // 父事务指针 (栈结构)
} TransactionStateData;
```

**事务栈示例**:

```
当前状态: BEGIN; SAVEPOINT sp1; SAVEPOINT sp2;

CurrentTransactionState ─→ [Level 3: sp2]
                               ↓ parent
                           [Level 2: sp1]
                               ↓ parent
                           [Level 1: 顶层事务]
                               ↓ parent
                           NULL
```

### 4.2 PGPROC - 进程事务信息

```c
// src/include/storage/proc.h:162
struct PGPROC
{
    TransactionId xid;        // 当前事务ID (如果已分配)
    TransactionId xmin;       // 事务开始时的最小活跃XID
    
    int pid;                  // 后端进程ID
    int pgxactoff;            // 在ProcGlobal数组中的偏移
    
    struct {
        ProcNumber procNumber;       // 进程编号
        LocalTransactionId lxid;     // 本地事务ID
    } vxid;                   // 虚拟事务ID
    
    Oid databaseId;           // 数据库OID
    Oid roleId;               // 角色OID
    
    // 锁、等待、信号量等字段...
};
```

### 4.3 全局事务变量

```c
// src/backend/access/transam/xact.c

// 当前事务状态指针
static TransactionState CurrentTransactionState = &TopTransactionStateData;

// 顶层事务状态 (静态分配)
static TransactionStateData TopTransactionStateData = {
    .state = TRANS_DEFAULT,
    .blockState = TBLOCK_DEFAULT,
};

// 当前子事务ID计数器
static SubTransactionId currentSubTransactionId;

// 当前命令ID
static CommandId currentCommandId;

// 事务时间戳
static TimestampTz xactStartTimestamp;    // transaction_timestamp()
static TimestampTz stmtStartTimestamp;    // statement_timestamp()
```

---

## 5. 事务ID管理

### 5.1 FullTransactionId - 64位事务ID

PostgreSQL 17 引入了64位的 `FullTransactionId`，解决了32位事务ID回卷问题：

```c
typedef struct FullTransactionId
{
    uint64 value;
} FullTransactionId;

// 组成: Epoch (高32位) + XID (低32位)
// 范围: 0 ~ 2^64-1 (约18,446,744,073,709,551,615)
```

**特殊事务ID**:

| XID | 值 | 含义 |
|-----|-----|------|
| InvalidTransactionId | 0 | 无效事务ID |
| BootstrapTransactionId | 1 | 引导事务ID |
| FrozenTransactionId | 2 | 冻结事务ID |
| FirstNormalTransactionId | 3 | 第一个正常事务ID |

### 5.2 事务ID分配时机

**关键点**: PostgreSQL 采用**懒惰分配**策略，只在需要时才分配事务ID。

```
事务开始 (BEGIN)
    ↓
执行查询 (SELECT) ──────────→ 不分配XID ✗
    ↓
执行更新 (UPDATE) ──────────→ 分配XID ✓
    ↓                     (GetNewTransactionId)
写入 tuple.t_xmin = XID
    ↓
提交事务 (COMMIT)
```

**为什么懒惰分配?**
- 只读事务不需要XID，节省ID空间
- 减少事务ID回卷压力
- 提高只读查询性能

### 5.3 GetNewTransactionId() 流程

```c
// src/backend/access/transam/varsup.c:76
FullTransactionId GetNewTransactionId(bool isSubXact)
{
    // 1. 获取 XidGenLock (排他锁)
    LWLockAcquire(XidGenLock, LW_EXCLUSIVE);
    
    // 2. 从全局计数器获取下一个XID
    xid = TransamVariables->nextXid;
    TransamVariables->nextXid++;
    
    // 3. 检查是否需要扩展CLOG
    ExtendCLOG(xid);
    ExtendCommitTs(xid);
    ExtendSUBTRANS(xid);
    
    // 4. 设置到PGPROC
    MyProc->xid = xid;
    ProcGlobal->xids[MyProc->pgxactoff] = xid;
    
    // 5. 释放锁
    LWLockRelease(XidGenLock);
    
    return FullTransactionIdFromEpochAndXid(epoch, xid);
}
```

---

## 6. 子事务 (SAVEPOINT) 机制

### 6.1 子事务概述

子事务允许在事务内创建嵌套的"保存点"，可以部分回滚而不影响整个事务。

```sql
BEGIN;
    INSERT INTO accounts (id, balance) VALUES (1, 1000);  -- 操作1
    
    SAVEPOINT sp1;
    UPDATE accounts SET balance = balance - 500 WHERE id = 1;  -- 操作2
    
    -- 如果余额不足，回滚到sp1
    ROLLBACK TO SAVEPOINT sp1;  -- 操作2被撤销，操作1保留
    
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 操作3
COMMIT;  -- 操作1和操作3生效
```

### 6.2 子事务ID分配

```c
typedef uint32 SubTransactionId;

#define InvalidSubTransactionId  ((SubTransactionId) 0)
#define TopSubTransactionId      ((SubTransactionId) 1)

// 每个子事务分配一个递增的SubTransactionId
// 主事务: SubTransactionId = 1
// 子事务1: SubTransactionId = 2
// 子事务2: SubTransactionId = 3
// ...
```

### 6.3 子事务栈结构

```
BEGIN;
    [Level 1: XID=100, SubXID=1, nestingLevel=1]
    
    SAVEPOINT sp1;
    [Level 2: XID=101, SubXID=2, nestingLevel=2, parent→Level1]
    
    SAVEPOINT sp2;
    [Level 3: XID=102, SubXID=3, nestingLevel=3, parent→Level2]
    
    ROLLBACK TO sp1;  -- 弹出 Level3 和 Level2
    [Level 1: XID=100, SubXID=1, nestingLevel=1]
    
COMMIT;
```

---

## 7. 两阶段提交 (2PC)

### 7.1 2PC 协议

两阶段提交用于分布式事务，确保多个数据库要么全部提交，要么全部回滚。

```
协调者                           参与者1                    参与者2
   │                               │                          │
   │  1. PREPARE TRANSACTION      │                          │
   ├──────────────────────────────→                          │
   │                               │ (准备成功)               │
   │  ← OK                         │                          │
   │                               │                          │
   ├───────────────────────────────────────────────────────→ │
   │                               │                          │ (准备成功)
   │  ← OK ──────────────────────────────────────────────────┤
   │                               │                          │
   │  2. COMMIT PREPARED           │                          │
   ├──────────────────────────────→                          │
   │                               │ (提交成功)               │
   ├───────────────────────────────────────────────────────→ │
   │                               │                          │ (提交成功)
   │                               │                          │
```

### 7.2 PostgreSQL 2PC 实现

```sql
-- 参与者1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
PREPARE TRANSACTION 'txn_001';  -- 准备阶段

-- 参与者2
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
PREPARE TRANSACTION 'txn_002';  -- 准备阶段

-- 协调者决定提交
COMMIT PREPARED 'txn_001';  -- 提交阶段
COMMIT PREPARED 'txn_002';

-- 或决定回滚
ROLLBACK PREPARED 'txn_001';
ROLLBACK PREPARED 'txn_002';
```

### 7.3 准备好的事务存储

```c
// src/backend/access/transam/twophase.c:147
typedef struct GlobalTransactionData
{
    int pgprocno;                 // PGPROC编号
    TimestampTz prepared_at;      // 准备时间
    XLogRecPtr prepare_start_lsn; // WAL起始位置
    XLogRecPtr prepare_end_lsn;   // WAL结束位置
    TransactionId xid;            // 事务ID
    Oid owner;                    // 执行用户
    bool valid;                   // 是否有效
    bool ondisk;                  // 是否已持久化
    char gid[GIDSIZE];            // 全局事务ID (最多200字符)
} GlobalTransactionData;
```

---

## 8. 事务与其他子系统的交互

### 8.1 事务 + MVCC

```
Transaction Manager          MVCC
      │                       │
      │ 1. 分配 XID=100       │
      ├──────────────────────→│
      │                       │ 设置 tuple.t_xmin=100
      │                       │
      │ 2. COMMIT             │
      ├──────────────────────→│
      │                       │ 更新 CLOG: XID=100 → COMMITTED
      │                       │ 设置 HEAP_XMIN_COMMITTED 提示位
```

### 8.2 事务 + WAL

```
Transaction Manager          WAL System
      │                       │
      │ 1. 修改数据           │
      │                       │
      │ 2. CommitTransaction  │
      ├──────────────────────→│ XLogInsert(XLOG_XACT_COMMIT)
      │                       │ ┌─ WAL Record ─┐
      │                       │ │ XID: 100     │
      │                       │ │ Type: COMMIT │
      │                       │ │ LSN: 0/12345 │
      │                       │ └──────────────┘
      │ ← WAL已刷盘           │
      │                       │
      │ 3. 更新CLOG           │
```

### 8.3 事务 + Lock Manager

```
事务的每个阶段都会获取和释放锁：

BEGIN
  ↓
获取表锁 (AccessShareLock / RowExclusiveLock)
  ↓
执行查询/更新
  ↓
COMMIT
  ↓
释放所有锁 (LockReleaseAll)
```

---

## 9. 性能关键点

### 9.1 事务开销分析

| 操作 | 开销 | 优化方法 |
|------|------|----------|
| BEGIN | 低 (仅设置状态) | 无需优化 |
| 分配XID | 中 (需要LWLock) | 只读事务不分配XID |
| COMMIT | 高 (WAL写入 + fsync) | 异步提交、组提交 |
| ROLLBACK | 中 (清理资源) | 避免不必要的回滚 |
| SAVEPOINT | 低 (创建栈节点) | 合理嵌套层次 |

### 9.2 长事务的危害

```
长事务 (xact_start: 10:00, now: 11:00)
    ↓
阻碍 VACUUM 清理 (xmin=长事务XID)
    ↓
死元组积累 → 表膨胀 → 性能下降
    ↓
索引膨胀 → 查询变慢
    ↓
autovacuum 无法工作
```

**监控长事务**:

```sql
SELECT pid, now() - xact_start AS duration, state, query
FROM pg_stat_activity
WHERE xact_start < now() - interval '10 minutes'
ORDER BY xact_start;
```

---

## 10. 源码文件导航

### 10.1 核心文件

| 文件 | 行数 | 主要内容 |
|------|------|----------|
| `xact.c` | 6300+ | 事务管理主逻辑 |
| `varsup.c` | 700+ | 事务ID分配 |
| `twophase.c` | 2600+ | 两阶段提交 |
| `procarray.c` | 3000+ | 进程数组管理 |

### 10.2 关键函数速查

```
事务生命周期:
  StartTransaction()       xact.c:2011
  CommitTransaction()      xact.c:2167
  PrepareTransaction()     xact.c:2456
  AbortTransaction()       xact.c:2630

用户命令处理:
  BeginTransactionBlock()        xact.c:2841
  EndTransactionBlock()          xact.c:2904
  UserAbortTransactionBlock()    xact.c:3069
  DefineSavepoint()              xact.c:4143
  RollbackToSavepoint()          xact.c:4249
  ReleaseSavepoint()             xact.c:4371

子事务:
  StartSubTransaction()    xact.c:5048
  CommitSubTransaction()   xact.c:5168
  AbortSubTransaction()    xact.c:5323

事务ID:
  GetNewTransactionId()    varsup.c:76
  GetTopTransactionId()    xact.c:421
  GetCurrentTransactionId() xact.c:463

两阶段提交:
  PrepareTransactionBlock()      xact.c:3246
  FinishPreparedTransaction()    twophase.c:1169
  RecordTransactionCommitPrepared() twophase.c:1531
```

---

## 11. 总结

### 11.1 核心要点

1. **三层架构**: 高层处理用户命令、中层管理查询级事务、低层执行真正操作
2. **双层状态机**: TransState跟踪低层状态、TBlockState跟踪高层状态
3. **懒惰分配XID**: 只在需要时分配事务ID，节省ID空间
4. **子事务栈**: 通过链表实现SAVEPOINT的嵌套
5. **2PC支持**: 通过PREPARE TRANSACTION实现分布式事务
6. **与MVCC集成**: 事务ID是MVCC可见性判断的核心

### 11.2 学习路径

```
第1步: 理解三层架构 (本文档)
   ↓
第2步: 学习核心数据结构 (02_data_structures.md)
   ↓
第3步: 掌握事务流程 (03_implementation_flow.md)
   ↓
第4步: 深入关键算法 (04_key_algorithms.md)
   ↓
第5步: 性能优化 (05_performance.md)
```

---

**下一步**: 阅读 [02_data_structures.md](02_data_structures.md) 深入了解核心数据结构

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-01-16

