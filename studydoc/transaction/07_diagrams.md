# Transaction Manager 架构图表集

本文档包含事务管理器的完整架构图表，帮助理解其工作原理。

---

## 1. 三层事务系统架构图

```
PostgreSQL 三层事务系统
========================

                用户 SQL 命令
                      │
    ┌─────────────────┼─────────────────┐
    │                 ▼                 │
    │   BEGIN / COMMIT / ROLLBACK       │
    │   SAVEPOINT / RELEASE             │
    │   PREPARE TRANSACTION             │
    └─────────────────┬─────────────────┘
                      │
        ┌─────────────┴─────────────┐
        ▼                           ▼
┌──────────────────────┐   ┌──────────────────────┐
│ utility.c:StandardProcessUtility()             │
│                                                 │
│  switch (stmt->kind)                           │
│    case TRANS_STMT_BEGIN:                       │
│        BeginTransactionBlock() ────────┐       │
│    case TRANS_STMT_COMMIT:              │       │
│        EndTransactionBlock() ──────────┤       │
│    case TRANS_STMT_ROLLBACK:            │       │
│        UserAbortTransactionBlock() ────┤       │
│    case TRANS_STMT_SAVEPOINT:           │       │
│        DefineSavepoint() ──────────────┤       │
└──────────────────────────────────────────┘     │
                      │                           │
                      ▼                           │
┌─────────────────────────────────────────────────┘
│         高层 (Transaction Block Layer)
│         xact.c:2800-4400
│  ┌────────────────────────────────────┐
│  │ BeginTransactionBlock()            │ ← BEGIN
│  │ EndTransactionBlock()              │ ← COMMIT
│  │ UserAbortTransactionBlock()        │ ← ROLLBACK
│  │ DefineSavepoint()                  │ ← SAVEPOINT
│  │ RollbackToSavepoint()              │ ← ROLLBACK TO
│  │ ReleaseSavepoint()                 │ ← RELEASE
│  └────────────────────────────────────┘
└──────────────────┬─────────────────────
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│         中层 (Transaction Command Layer)         │
│         xact.c:2698-2810                         │
│  ┌─────────────────────────────────────────┐    │
│  │ StartTransactionCommand()               │    │
│  │   ├─ 每个查询开始时调用                 │    │
│  │   └─ 根据blockState决定是否启动事务     │    │
│  │                                         │    │
│  │ CommitTransactionCommand()              │    │
│  │   ├─ 每个查询结束时调用                 │    │
│  │   ├─ 自动提交: 立即提交事务             │    │
│  │   └─ 事务块中: 递增命令ID               │    │
│  │                                         │    │
│  │ AbortCurrentTransaction()               │    │
│  │   └─ 错误处理时调用                     │    │
│  └─────────────────────────────────────────┘    │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│         低层 (Low-Level Transaction Layer)       │
│         xact.c:2011-2630                         │
│  ┌─────────────────────────────────────────┐    │
│  │ StartTransaction()                      │    │
│  │   ├─ 初始化事务状态                     │    │
│  │   ├─ 分配虚拟事务ID (vxid)              │    │
│  │   ├─ 创建内存上下文                     │    │
│  │   └─ state: DEFAULT → START → INPROGRESS│   │
│  │                                         │    │
│  │ CommitTransaction()                     │    │
│  │   ├─ 记录WAL (XLOG_XACT_COMMIT)         │    │
│  │   ├─ 更新CLOG (XID → COMMITTED)         │    │
│  │   ├─ 释放锁和资源                       │    │
│  │   └─ state: INPROGRESS → COMMIT → DEFAULT│  │
│  │                                         │    │
│  │ AbortTransaction()                      │    │
│  │   ├─ 记录WAL (XLOG_XACT_ABORT)          │    │
│  │   ├─ 更新CLOG (XID → ABORTED)           │    │
│  │   ├─ 清理资源                           │    │
│  │   └─ state: INPROGRESS → ABORT → DEFAULT│   │
│  │                                         │    │
│  │ PrepareTransaction()                    │    │
│  │   └─ 两阶段提交准备                     │    │
│  └─────────────────────────────────────────┘    │
└──────────────────┬──────────────────────────────┘
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
    ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
    │ MVCC │  │ WAL  │  │ Lock │  │Buffer│
    │      │  │      │  │ Mgr  │  │ Mgr  │
    └──────┘  └──────┘  └──────┘  └──────┘
```

---

## 2. 双层状态机图

### 2.1 TransState (低层状态)

```
低层事务状态转换 (TransState)
==============================

    DEFAULT (空闲)
       │
       │ StartTransaction()
       ▼
     START (启动中)
       │
       │ (初始化完成)
       ▼
   INPROGRESS (进行中) ───────────┐
       │                          │
       │ CommitTransaction()       │ PrepareTransaction()
       ▼                          ▼
     COMMIT                    PREPARE
       │                          │
       │ (完成)                   │ (2PC准备)
       ▼                          ▼
    DEFAULT                    DEFAULT
       │
       │ AbortTransaction()
       ▼
     ABORT (回滚中)
       │
       │ (完成)
       ▼
    DEFAULT

注意: 
- 稳定状态只有 DEFAULT 和 INPROGRESS
- 其他状态都是瞬时的过渡状态
```

### 2.2 TBlockState (高层状态)

```
高层事务块状态转换 (TBlockState)
==================================

【自动提交模式】
    DEFAULT
       │
       │ 收到查询
       ▼
    STARTED (单查询事务)
       │
       │ 查询完成
       ▼
    DEFAULT
       │
       └─→ 循环...

【显式事务块】
    DEFAULT
       │
       │ BEGIN
       ▼
     BEGIN (BEGIN刚执行)
       │
       │ 下一个查询
       ▼
  INPROGRESS (事务块进行中)
       │
       ├────→ COMMIT收到 ──→ END ──→ 提交 ──→ DEFAULT
       │
       ├────→ ROLLBACK收到 ──→ ABORT_END ──→ 回滚 ──→ DEFAULT
       │
       └────→ 出错 ──→ ABORT ──→ 等待ROLLBACK ──→ ABORT_END ──→ DEFAULT

【子事务状态】
  INPROGRESS
       │
       │ SAVEPOINT
       ▼
   SUBBEGIN (子事务开始)
       │
       ▼
 SUBINPROGRESS (子事务进行中)
       │
       ├────→ RELEASE ──→ SUBRELEASE ──→ 提交子事务 ──→ INPROGRESS
       │
       ├────→ ROLLBACK TO ──→ SUBRESTART ──→ 回滚子事务 ──→ SUBINPROGRESS
       │
       └────→ 出错 ──→ SUBABORT ──→ 等待处理 ──→ ...
```

---

## 3. 事务生命周期完整流程

```
事务完整生命周期 (从BEGIN到COMMIT)
====================================

用户     高层状态      低层状态    PGPROC       WAL/CLOG
 │         │            │           │            │
BEGIN ────→ DEFAULT      DEFAULT     xid=0        -
         ──→ BEGIN                               
 │                                   
第一个────→ BEGIN        START       vxid=分配    -
查询    ──→ INPROGRESS   INPROGRESS             
 │                                   
 │                                   
UPDATE ───→ (触发XID分配)            XID=100      ExtendCLOG(100)
 │         INPROGRESS   INPROGRESS   xid=100      -
 │                                   MyProc->xid=100
 │                                   ProcGlobal->xids[i]=100
 │                                   
 │         (写入元组: t_xmin=100)    
 │                                   
SELECT ───→ (命令ID递增)             
 │         INPROGRESS   INPROGRESS   
 │         CommandCounterIncrement()
 │                                   
COMMIT ───→ INPROGRESS   INPROGRESS  
         ──→ END         COMMIT       (保持xid=100)  RecordTransactionCommit()
 │                                                   ├─ XLogInsert(COMMIT)
 │                                                   ├─ XLogFlush() [sync_commit=on]
 │                                                   └─ TransactionIdCommitTree()
 │                                                      └─ CLOG: 100→COMMITTED
 │                                   
 │                      (ProcArrayEndTransaction)
 │                                   xid=0         (其他事务可见!)
 │                                   从活跃数组移除
 │                                   
 │                      (释放锁)     
 │                      (清理资源)   
 │                                   
 │         DEFAULT      DEFAULT      xid=0         -
 │                                   
```

---

## 4. 子事务栈结构图

```
子事务栈结构 (SAVEPOINT 嵌套)
==============================

SQL 执行序列:
  BEGIN;
      INSERT INTO t VALUES (1);
      SAVEPOINT sp1;
          UPDATE t SET ...;
          SAVEPOINT sp2;
              DELETE FROM t WHERE ...;
          -- 当前状态如下:

内存中的栈结构:

CurrentTransactionState
    │
    ▼
┌──────────────────────────────────────────┐
│ Level 3: SubXID=3, name="sp2"           │
│ fullTransactionId = 102                  │
│ subTransactionId = 3                     │
│ nestingLevel = 3                         │
│ state = TRANS_INPROGRESS                 │
│ blockState = TBLOCK_SUBINPROGRESS        │
│ curTransactionContext = [子事务内存]     │
│ parent ↓                                 │
└──────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────┐
│ Level 2: SubXID=2, name="sp1"           │
│ fullTransactionId = 101                  │
│ subTransactionId = 2                     │
│ nestingLevel = 2                         │
│ state = TRANS_INPROGRESS                 │
│ blockState = TBLOCK_SUBINPROGRESS        │
│ childXids = []                           │
│ parent ↓                                 │
└──────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────┐
│ Level 1: XID=100, SubXID=1 (顶层事务)   │
│ fullTransactionId = 100                  │
│ subTransactionId = 1                     │
│ nestingLevel = 1                         │
│ state = TRANS_INPROGRESS                 │
│ blockState = TBLOCK_INPROGRESS           │
│ childXids = [] (子事务提交后会填充)     │
│ parent = NULL                            │
└──────────────────────────────────────────┘

操作示例:

ROLLBACK TO sp1:
  ├─ 弹出 Level 3 (调用 AbortSubTransaction)
  ├─ 弹出 Level 2 (调用 CleanupSubTransaction)
  └─ CurrentTransactionState → Level 1

RELEASE sp1:
  ├─ 提交 Level 2 (调用 CommitSubTransaction)
  │   └─ Level1.childXids = [101]
  ├─ 弹出 Level 2
  └─ CurrentTransactionState → Level 1
```

---

## 5. 两阶段提交流程图

```
两阶段提交 (2PC) 完整流程
==========================

【参与者1】                【协调者】              【参与者2】
    │                          │                      │
BEGIN                      (决定开始2PC)          BEGIN
UPDATE t1                                         UPDATE t2
    │                          │                      │
    │                          │                      │
PREPARE ──────────────────────→│                      │
TRANSACTION 'gid1'             │◄─────────────────── PREPARE
    │                          │                   TRANSACTION 'gid2'
    │                          │                      │
    ├─ 写WAL(PREPARE)          │                      ├─ 写WAL(PREPARE)
    ├─ 写2PC状态文件           │                      ├─ 写2PC状态文件
    ├─ 创建dummy PGPROC        │                      ├─ 创建dummy PGPROC
    └─ 回复: YES ──────────────→│                      └─ 回复: YES
                               │◄────────────────────────────┘
                               │
                          (收到所有YES)
                          (决定COMMIT)
                               │
    ┌──────────────────────────│
    │                          │──────────────────────────┐
    ▼                          │                          ▼
COMMIT PREPARED 'gid1'         │                     COMMIT PREPARED 'gid2'
    │                          │                          │
    ├─ 读2PC状态文件           │                          ├─ 读2PC状态文件
    ├─ 写WAL(COMMIT_PREPARED)  │                          ├─ 写WAL(COMMIT_PREPARED)
    ├─ 更新CLOG                │                          ├─ 更新CLOG
    ├─ 释放锁                  │                          ├─ 释放锁
    ├─ 删除2PC文件             │                          ├─ 删除2PC文件
    └─ 回复: OK ──────────────→│                          └─ 回复: OK
                               │◄────────────────────────────┘
                               │
                          (全部完成)
                               ▼


【2PC 状态文件结构】
pg_twophase/<XID>:
┌─────────────────────────────────┐
│ TwoPhaseFileHeader              │
│   ├─ magic = 0x57F94531         │
│   ├─ total_len                  │
│   ├─ xid                        │
│   ├─ database                   │
│   ├─ prepared_at                │
│   ├─ owner                      │
│   └─ gid[200]                   │
├─────────────────────────────────┤
│ TwoPhaseRecordOnDisk[]          │
│   ├─ TWOPHASE_RM_LOCK_ID:       │
│   │   持有的锁列表               │
│   ├─ TWOPHASE_RM_XACT_ID:       │
│   │   子事务ID列表               │
│   └─ 其他资源管理器状态          │
└─────────────────────────────────┘
```

---

## 6. XID分配流程图

```
事务ID分配流程 (GetNewTransactionId)
=====================================

用户执行UPDATE
    │
    ▼
heap_update()
    │
    ▼
GetCurrentTransactionId()
    │
    ▼
检查: fullTransactionId有效?
    │
    NO
    ▼
AssignTransactionId()
    │
    ▼
GetNewTransactionId(isSubXact=false)
    │
    ├─────────────────────────────────┐
    │                                 │
    ▼                                 ▼
LWLockAcquire(XidGenLock, EXCLUSIVE)  (其他事务等待)
    │
    ▼
nextXid = TransamVariables->nextXid
    │
    ▼
TransamVariables->nextXid++
    │
    ▼
检查: nextXid >= xidVacLimit?
    │
    YES → SendPostmasterSignal(AUTOVAC) (触发紧急VACUUM)
    │
    ▼
ExtendCLOG(nextXid)
    ├─ 如果需要,分配新的CLOG页面
    └─ 初始化为 IN_PROGRESS
    │
    ▼
ExtendCommitTs(nextXid)
    │
    ▼
ExtendSUBTRANS(nextXid) [如果是子事务]
    │
    ▼
MyProc->xid = nextXid
ProcGlobal->xids[MyProc->pgxactoff] = nextXid
    │
    ▼
LWLockRelease(XidGenLock)
    │                                 │
    └─────────────────────────────────┘
    │
    ▼
返回 FullTransactionId
    │
    ▼
写入元组: tuple.t_xmin = nextXid
```

---

## 7. COMMIT详细流程图

```
COMMIT 提交流程
===============

用户执行 COMMIT
    │
    ▼
EndTransactionBlock()  [高层]
    │
    └─ s->blockState = TBLOCK_END
    │
    ▼
CommitTransactionCommand()  [中层]
    │
    └─ 检测到 blockState = TBLOCK_END
    │
    ▼
CommitTransaction()  [低层]
    │
    ├─ s->state = TRANS_COMMIT
    │
    ├─────────────────────────────────────┐
    │                                     │
    ▼                                     ▼
TransactionIdIsValid(xid)?          NO → 跳过WAL记录
    │                                     │
    YES                                   │
    ▼                                     │
RecordTransactionCommit()                │
    │                                     │
    ├─ 准备WAL记录                         │
    │  xl_xact_commit {                  │
    │    xact_time,                      │
    │    nchildren, children[],          │
    │    ...                             │
    │  }                                 │
    │                                     │
    ├─ XLogInsert(XLOG_XACT_COMMIT)       │
    │      ↓                              │
    │  [WAL Buffer]                       │
    │      ↓                              │
    │  XLogFlush() [如果sync_commit=on]  │
    │      ↓                              │
    │  [磁盘]                             │
    │      ↓                              │
    │  (持久化完成)                        │
    │                                     │
    ├─ TransactionIdCommitTree()          │
    │      ↓                              │
    │  更新CLOG: XID → COMMITTED          │
    │      ↓                              │
    │  (此时对其他事务可见!)  ◄───────────┘
    │                      
    ▼
ProcArrayEndTransaction(MyProc, xid)
    ├─ MyProc->xid = InvalidTransactionId
    ├─ ProcGlobal->xids[i] = InvalidTransactionId
    └─ 从活跃事务数组移除
    │
    ▼
CallXactCallbacks(XACT_EVENT_COMMIT)
    ├─ 触发器
    ├─ 缓存失效
    └─ 其他回调
    │
    ▼
AtEOXact_*() 系列函数
    ├─ AtEOXact_Snapshot()  (删除快照)
    ├─ AtEOXact_Files()     (关闭文件)
    ├─ smgrDoPendingDeletes() (确认文件操作)
    └─ ...
    │
    ▼
ResourceOwnerRelease(..., RESOURCE_RELEASE_LOCKS)
    └─ 释放所有锁
    │
    ▼
AtCommit_Memory()
    └─ 释放事务内存上下文
    │
    ▼
s->state = TRANS_DEFAULT
s->blockState = TBLOCK_DEFAULT
    │
    ▼
完成!
```

---

## 8. 数据结构关系图

```
事务管理器核心数据结构关系
==========================

                全局共享内存
                     │
    ┌────────────────┼────────────────┐
    │                │                │
    ▼                ▼                ▼
┌─────────┐  ┌──────────────┐  ┌────────────┐
│TransamVariables│ProcGlobal │TwoPhaseState│
│         │  │              │  │            │
│nextXid  │  │allProcs[]    │  │prepXacts[] │
│oldestXid│  │xids[]        │  │numPrepXacts│
│xidVacLimit│ │subxidStates[]│  │freeGXacts  │
└─────────┘  └───┬──────────┘  └────────────┘
                 │
            ┌────┴─────┬─────────┐
            ▼          ▼         ▼
        PGPROC[0]  PGPROC[1]  PGPROC[N]
        ┌────────────────────────┐
        │ xid                    │◄─ 镜像到 ProcGlobal->xids[]
        │ xmin                   │
        │ vxid.procNumber        │
        │ vxid.lxid              │
        │ subxids.xids[64]       │◄─ 子事务缓存
        │ subxidStatus           │
        └────────────────────────┘
                 │
            (每个后端一个)
                 │
                 ▼
        本地进程内存
                 │
        ┌────────┴────────┐
        ▼                 ▼
CurrentTransactionState  TopTransactionStateData
        │                 │
        ▼                 ▼
┌──────────────────────────────┐
│ TransactionStateData (栈)    │
│                              │
│ fullTransactionId            │
│ subTransactionId             │
│ state (TransState)           │
│ blockState (TBlockState)     │
│ nestingLevel                 │
│ childXids[]                  │◄─ 已提交子事务
│ parent ↓                     │
└──────────────────────────────┘
        │
        ▼ (子事务)
┌──────────────────────────────┐
│ TransactionStateData         │
│ (子事务节点)                 │
│ parent ↓                     │
└──────────────────────────────┘
        │
        ▼ (更深层子事务)
       ...
```

---

## 总结

本图表集包含：

1. ✅ 三层架构完整结构
2. ✅ 双层状态机转换
3. ✅ 事务完整生命周期
4. ✅ 子事务栈结构
5. ✅ 两阶段提交流程
6. ✅ XID分配流程
7. ✅ COMMIT详细流程
8. ✅ 数据结构关系

这些图表帮助理解：
- 事务管理器的分层设计
- 状态如何转换
- 数据如何流动
- 组件如何交互

---

**PostgreSQL Transaction Manager模块分析完成！**

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-01-16

