# Transaction Manager 实现流程

本文档详细说明 BEGIN、COMMIT、ROLLBACK、SAVEPOINT 等事务命令的完整实现流程。

---

## 1. BEGIN - 开始事务

### 1.1 用户执行 BEGIN

```sql
BEGIN;  -- 或 BEGIN TRANSACTION / START TRANSACTION
```

### 1.2 三层调用流程

```
第1层: 用户命令处理
    utility.c:609  BeginTransactionBlock()
        ↓
第2层: 事务命令层
    (postgres.c 每个查询前调用)
    xact.c:2698  StartTransactionCommand()
        ↓
第3层: 低层事务实现
    xact.c:2011  StartTransaction()
```

### 1.3 第1层: BeginTransactionBlock()

```c
// src/backend/access/transam/xact.c:2841
void BeginTransactionBlock(void)
{
    TransactionState s = CurrentTransactionState;
    
    /* 检查当前状态 */
    switch (s->blockState)
    {
        case TBLOCK_STARTED:
            // 自动提交模式中的单查询事务
            // 转换为显式事务块
            s->blockState = TBLOCK_BEGIN;
            break;
            
        case TBLOCK_INPROGRESS:
        case TBLOCK_PARALLEL_INPROGRESS:
        case TBLOCK_SUBINPROGRESS:
            // 已经在事务块中，报警告
            ereport(WARNING,
                    (errcode(ERRCODE_ACTIVE_SQL_TRANSACTION),
                     errmsg("there is already a transaction in progress")));
            break;
            
        // ... 其他状态处理
    }
}
```

**状态转换**:

```
自动提交模式:
DEFAULT → STARTED → BEGIN (收到BEGIN命令)

显式事务块:
BEGIN → INPROGRESS (下次查询时)
```

### 1.4 第2层: StartTransactionCommand()

```c
// src/backend/access/transam/xact.c:2698
void StartTransactionCommand(void)
{
    TransactionState s = CurrentTransactionState;
    
    switch (s->blockState)
    {
        case TBLOCK_DEFAULT:
            // 默认状态，启动新事务
            StartTransaction();
            s->blockState = TBLOCK_STARTED;
            break;
            
        case TBLOCK_BEGIN:
            // BEGIN命令已执行，现在真正进入事务块
            StartTransaction();
            s->blockState = TBLOCK_INPROGRESS;
            break;
            
        case TBLOCK_INPROGRESS:
        case TBLOCK_SUBINPROGRESS:
            // 事务已在进行中，什么都不做
            break;
            
        // ... 其他情况
    }
}
```

### 1.5 第3层: StartTransaction()

```c
// src/backend/access/transam/xact.c:2011
static void StartTransaction(void)
{
    TransactionState s = &TopTransactionStateData;
    CurrentTransactionState = s;
    
    // 1. 检查状态
    Assert(s->state == TRANS_DEFAULT);
    
    // 2. 设置状态为 TRANS_START
    s->state = TRANS_START;
    s->fullTransactionId = InvalidFullTransactionId;  // 尚未分配XID
    
    // 3. 初始化事务字段
    s->nestingLevel = 1;
    s->gucNestLevel = 1;
    s->childXids = NULL;
    s->nChildXids = 0;
    s->maxChildXids = 0;
    
    // 4. 保存用户ID和安全上下文
    GetUserIdAndSecContext(&s->prevUser, &s->prevSecContext);
    
    // 5. 设置只读标志 (恢复期间强制只读)
    if (RecoveryInProgress())
    {
        s->startedInRecovery = true;
        XactReadOnly = true;
    }
    else
    {
        s->startedInRecovery = false;
        XactReadOnly = DefaultXactReadOnly;
    }
    
    // 6. 设置隔离级别
    XactIsoLevel = DefaultXactIsoLevel;
    XactDeferrable = DefaultXactDeferrable;
    
    // 7. 初始化计数器
    currentSubTransactionId = TopSubTransactionId;
    currentCommandId = FirstCommandId;
    currentCommandIdUsed = false;
    
    // 8. 设置时间戳
    xactStartTimestamp = GetCurrentTimestamp();
    stmtStartTimestamp = xactStartTimestamp;
    
    // 9. 分配虚拟事务ID (vxid)
    if (!IsBootstrapProcessingMode())
    {
        VirtualTransactionId vxid;
        
        LWLockAcquire(ProcArrayLock, LW_EXCLUSIVE);
        vxid.procNumber = MyProc->vxid.procNumber;
        vxid.localTransactionId = ++(MyProc->vxid.lxid);
        LWLockRelease(ProcArrayLock);
    }
    
    // 10. 初始化资源管理
    AtStart_Memory();              // 创建事务内存上下文
    AtStart_ResourceOwner();       // 创建资源所有者
    
    // 11. 通知其他子系统
    AfterTriggerBeginXact();
    AtStart_Cache();
    // ... 更多初始化
    
    // 12. 设置状态为 TRANS_INPROGRESS
    s->state = TRANS_INPROGRESS;
    
    // 13. 显示状态 (如果开启调试)
    ShowTransactionState("StartTransaction");
}
```

**关键点**:
- ✅ **懒惰XID分配**: 此时并未分配真正的事务ID (XID)
- ✅ **虚拟事务ID**: 分配vxid用于锁管理和监控
- ✅ **资源初始化**: 创建内存上下文和资源所有者

---

## 2. 事务中执行SQL

### 2.1 第一个修改操作触发XID分配

```sql
BEGIN;
SELECT * FROM t;  -- 只读，不分配XID
UPDATE t SET ...;  -- 修改操作，触发XID分配
```

### 2.2 XID分配流程

```c
// src/backend/access/transam/xact.c:454
void AssignTransactionId(TransactionState s)
{
    bool isSubXact = (s->nestingLevel > 1);
    
    // 1. 获取新的XID
    s->fullTransactionId = GetNewTransactionId(isSubXact);
    
    // 2. 如果是子事务，记录到父事务
    if (isSubXact)
    {
        SubTransAdd(s->fullTransactionId, TopTransactionState->fullTransactionId);
        
        // 缓存到 MyProc->subxids (最多64个)
        if (MyProc->subxidStatus == SUBXIDS_IN_ARRAY)
        {
            if (MyProc->subxids.xcnt < PGPROC_MAX_CACHED_SUBXIDS)
                MyProc->subxids.xids[MyProc->subxids.xcnt++] = XidFromFullTransactionId(s->fullTransactionId);
            else
                MyProc->subxidStatus = SUBXIDS_OVERFLOW;
        }
    }
    
    // 3. 设置到PGPROC (让其他进程可见)
    MyProc->xid = XidFromFullTransactionId(s->fullTransactionId);
    ProcGlobal->xids[MyProc->pgxactoff] = MyProc->xid;
    
    // 4. 标记已分配
    s->didLogXid = false;
}
```

### 2.3 GetNewTransactionId() 详细流程

```c
// src/backend/access/transam/varsup.c:76
FullTransactionId GetNewTransactionId(bool isSubXact)
{
    FullTransactionId full_xid;
    TransactionId xid;
    
    // 1. 禁止在并行模式中分配XID
    if (IsInParallelMode())
        elog(ERROR, "cannot assign TransactionIds during a parallel operation");
    
    // 2. 引导模式使用特殊XID
    if (IsBootstrapProcessingMode())
    {
        MyProc->xid = BootstrapTransactionId;
        return FullTransactionIdFromEpochAndXid(0, BootstrapTransactionId);
    }
    
    // 3. 获取 XidGenLock 排他锁
    LWLockAcquire(XidGenLock, LW_EXCLUSIVE);
    
    // 4. 从全局计数器获取下一个XID
    full_xid = TransamVariables->nextXid;
    xid = XidFromFullTransactionId(full_xid);
    
    // 5. 递增全局计数器
    TransamVariables->nextXid = FullTransactionIdFromEpochAndXid(
        EpochFromFullTransactionId(full_xid),
        xid + 1);
    
    // 6. 检查XID回卷警告
    if (TransactionIdFollowsOrEquals(xid, TransamVariables->xidVacLimit))
        SetTransactionIdLimit(...);  // 触发VACUUM
    
    // 7. 扩展CLOG、SUBTRANS等SLRU
    ExtendCLOG(xid);
    ExtendCommitTs(xid);
    if (isSubXact)
        ExtendSUBTRANS(xid);
    
    // 8. 更新统计
    if (!isSubXact)
        TransamVariables->oldestXid = ...;
    
    // 9. 释放锁
    LWLockRelease(XidGenLock);
    
    return full_xid;
}
```

---

## 3. COMMIT - 提交事务

### 3.1 用户执行 COMMIT

```sql
COMMIT;  -- 或 COMMIT TRANSACTION / COMMIT WORK
```

### 3.2 三层调用流程

```
第1层: 用户命令处理
    utility.c:631  EndTransactionBlock()
        ↓
第2层: 事务命令层
    xact.c:2751  CommitTransactionCommand()
        ↓
第3层: 低层事务实现
    xact.c:2167  CommitTransaction()
```

### 3.3 第1层: EndTransactionBlock()

```c
// src/backend/access/transam/xact.c:2904
bool EndTransactionBlock(bool chain)
{
    TransactionState s = CurrentTransactionState;
    bool result = false;
    
    switch (s->blockState)
    {
        case TBLOCK_INPROGRESS:
            // 正常情况: 事务块进行中
            s->blockState = TBLOCK_END;
            result = true;
            break;
            
        case TBLOCK_STARTED:
            // 警告: 没有事务块
            ereport(WARNING,
                    (errcode(ERRCODE_NO_ACTIVE_SQL_TRANSACTION),
                     errmsg("there is no transaction in progress")));
            result = true;
            break;
            
        case TBLOCK_ABORT:
            // 事务已失败，需要ROLLBACK
            ereport(ERROR,
                    (errcode(ERRCODE_IN_FAILED_SQL_TRANSACTION),
                     errmsg("current transaction is aborted")));
            break;
            
        // ... 其他情况
    }
    
    s->chain = chain;  // 支持 COMMIT AND CHAIN
    return result;
}
```

### 3.4 第2层: CommitTransactionCommand()

```c
// src/backend/access/transam/xact.c:2751
void CommitTransactionCommand(void)
{
    TransactionState s = CurrentTransactionState;
    
    switch (s->blockState)
    {
        case TBLOCK_STARTED:
            // 自动提交模式
            CommitTransaction();
            s->blockState = TBLOCK_DEFAULT;
            break;
            
        case TBLOCK_END:
            // 显式事务块的提交
            CommitTransaction();
            if (s->chain)
            {
                // COMMIT AND CHAIN
                StartTransaction();
                s->blockState = TBLOCK_INPROGRESS;
                s->chain = false;
            }
            else
            {
                s->blockState = TBLOCK_DEFAULT;
            }
            break;
            
        case TBLOCK_INPROGRESS:
        case TBLOCK_SUBINPROGRESS:
            // 事务块内的查询结束，递增命令ID
            CommandCounterIncrement();
            break;
            
        // ... 其他情况
    }
}
```

### 3.5 第3层: CommitTransaction()

```c
// src/backend/access/transam/xact.c:2167
static void CommitTransaction(void)
{
    TransactionState s = CurrentTransactionState;
    TransactionId latestXid;
    
    ShowTransactionState("CommitTransaction");
    
    // 1. 检查状态
    Assert(s->state == TRANS_INPROGRESS);
    
    // 2. 设置状态为 TRANS_COMMIT
    s->state = TRANS_COMMIT;
    
    // 3. 如果XID未记录到WAL，现在记录
    latestXid = RecordTransactionCommit();
    
    /*
     * 让别人知道我已提交。注意: 这个步骤后，
     * 其他后端可以看到我的修改了。
     */
    ProcArrayEndTransaction(MyProc, latestXid);
    
    // 4. 调用提交时的回调
    CallXactCallbacks(is_parallel_worker ? XACT_EVENT_PARALLEL_COMMIT
                                         : XACT_EVENT_COMMIT);
    
    // 5. 删除快照
    AtEOXact_Snapshot(true, false);
    
    // 6. 处理并发控制
    AtEOXact_Inval(true);      // 缓存失效
    smgrDoPendingDeletes(true); // 删除待删文件
    
    // 7. 释放资源
    AtEOXact_RelationCache(true);
    AtEOXact_MultiXact();
    AtEOXact_Files(true);
    AtEOXact_ComboCid();
    AtEOXact_HashTables(true);
    AtEOXact_Namespace(true, false);
    AtEOXact_LargeObject(true);
    AfterTriggerEndXact(true);
    
    // 8. 释放锁 (必须在资源释放后)
    AtCommit_Memory();
    AtEOXact_GUC(true, 1);
    
    // 9. 更新统计
    pgstat_report_xact_timestamp(xactStopTimestamp);
    
    // 10. 释放所有锁
    ProcArrayGroupClearXid(MyProc, latestXid);
    
    // 11. 重置状态
    AtCommit_Notify();
    PostCommit_inval();
    
    ResourceOwnerRelease(TopTransactionResourceOwner,
                        RESOURCE_RELEASE_LOCKS,
                        true, true);
    
    // 12. 设置状态回 TRANS_DEFAULT
    s->state = TRANS_DEFAULT;
    
    // 13. 清理
    AtCleanup_Memory();
}
```

### 3.6 RecordTransactionCommit() - 关键函数

```c
// src/backend/access/transam/xact.c:1296
static TransactionId RecordTransactionCommit(void)
{
    TransactionId xid = XidFromFullTransactionId(XactTopFullTransactionId);
    bool markXidCommitted = TransactionIdIsValid(xid);
    
    // 1. 如果没有分配XID，直接返回
    if (!markXidCommitted)
        return InvalidTransactionId;
    
    // 2. 准备WAL记录
    xl_xact_commit xlrec;
    xlrec.xact_time = xactStopTimestamp;
    xlrec.xinfo = 0;
    
    // 3. 添加子事务ID (如果有)
    if (nchildren > 0)
    {
        xlrec.xinfo |= XACT_XINFO_HAS_SUBXACTS;
        // 附加子事务ID数组到WAL
    }
    
    // 4. 写入WAL记录
    XLogRecPtr recptr = XLogInsert(RM_XACT_ID, XLOG_XACT_COMMIT);
    
    // 5. 等待WAL刷盘 (如果需要同步提交)
    if (forceSyncCommit || !XactSyncCommit)
    {
        XLogFlush(recptr);
    }
    
    // 6. 更新CLOG: 设置事务状态为已提交
    TransactionIdCommitTree(xid, nchildren, children);
    
    // 7. 设置提示位 (在元组头部)
    // (后续扫描元组时会设置HEAP_XMIN_COMMITTED)
    
    return xid;
}
```

**COMMIT 关键时间点**:

```
Timeline:
T1: 状态 = TRANS_COMMIT
T2: 写入WAL (XLOG_XACT_COMMIT)
T3: WAL刷盘 (如果sync_commit=on)
T4: 更新CLOG (XID → COMMITTED)     ← 可见性改变点!
T5: ProcArrayEndTransaction()       ← 从活跃数组移除
T6: 释放锁
T7: 状态 = TRANS_DEFAULT
```

---

## 4. ROLLBACK - 回滚事务

### 4.1 用户执行 ROLLBACK

```sql
ROLLBACK;  -- 或 ROLLBACK TRANSACTION / ROLLBACK WORK
```

### 4.2 三层调用流程

```
第1层: 用户命令处理
    utility.c:659  UserAbortTransactionBlock()
        ↓
第2层: 事务命令层
    xact.c:2786  AbortCurrentTransaction()
        ↓
第3层: 低层事务实现
    xact.c:2630  AbortTransaction()
```

### 4.3 AbortTransaction() 详细流程

```c
// src/backend/access/transam/xact.c:2630
static void AbortTransaction(void)
{
    TransactionState s = CurrentTransactionState;
    TransactionId latestXid;
    
    // 1. 检查状态
    Assert(s->state == TRANS_INPROGRESS || s->state == TRANS_PREPARE);
    
    // 2. 设置状态为 TRANS_ABORT
    s->state = TRANS_ABORT;
    
    /*
     * 释放快照并失效关系缓存，这样我们就不会
     * 尝试在清理期间使用可能无效的缓存数据。
     */
    AtAbort_Memory();
    AtAbort_Snapshot();
    
    // 3. 记录回滚到WAL (如果已分配XID)
    if (TransactionIdIsValid(s->transactionId))
    {
        XLogBeginInsert();
        XLogRegisterData((char *) &xlrec, sizeof(xl_xact_abort));
        recptr = XLogInsert(RM_XACT_ID, XLOG_XACT_ABORT);
        
        // 不需要等待WAL刷盘 (回滚不需要持久化)
        
        // 更新CLOG: 设置为ABORTED
        TransactionIdAbortTree(xid, nchildren, children);
        
        latestXid = xid;
    }
    else
    {
        latestXid = InvalidTransactionId;
    }
    
    // 4. 让其他进程知道
    ProcArrayEndTransaction(MyProc, latestXid);
    
    // 5. 清理资源
    CallXactCallbacks(XACT_EVENT_ABORT);
    AtAbort_Portals();
    smgrDoPendingDeletes(false);  // 删除新创建的文件
    AtEOXact_LargeObject(false);
    AfterTriggerEndXact(false);
    AtAbort_Notify();
    AtEOXact_GUC(false, 1);
    
    // 6. 释放锁
    AtAbort_Locks(true);
    
    // 7. 删除临时文件
    AtEOXact_Files(false);
    
    // 8. 重置状态
    s->state = TRANS_DEFAULT;
    
    // 9. 清理内存
    AtCleanup_Memory();
}
```

**COMMIT vs ROLLBACK 对比**:

| 操作 | COMMIT | ROLLBACK |
|------|--------|----------|
| WAL刷盘 | 必须 (sync_commit=on) | 不需要 |
| CLOG更新 | COMMITTED | ABORTED |
| 文件操作 | 确认新文件 | 删除新文件 |
| 锁释放 | 资源释放后 | 尽快释放 |
| 时间开销 | 高 (I/O) | 低 (内存) |

---

## 5. SAVEPOINT - 子事务

### 5.1 创建保存点

```sql
BEGIN;
    INSERT INTO t VALUES (1);
    
    SAVEPOINT sp1;  -- 创建保存点
    UPDATE t SET ...;
    
    ROLLBACK TO sp1;  -- 回滚到保存点
COMMIT;
```

### 5.2 DefineSavepoint() 流程

```c
// src/backend/access/transam/xact.c:4143
void DefineSavepoint(const char *name)
{
    TransactionState s = CurrentTransactionState;
    
    // 1. 检查是否在事务块中
    switch (s->blockState)
    {
        case TBLOCK_INPROGRESS:
        case TBLOCK_SUBINPROGRESS:
            // 正常情况
            break;
        default:
            ereport(ERROR,
                    (errcode(ERRCODE_NO_ACTIVE_SQL_TRANSACTION),
                     errmsg("SAVEPOINT can only be used in transaction blocks")));
    }
    
    // 2. 推入新的子事务
    PushTransaction();
    s = CurrentTransactionState;  // 指向新节点
    
    // 3. 保存SAVEPOINT名称
    s->name = MemoryContextStrdup(TopTransactionContext, name);
    s->savepointLevel++;
    
    // 4. 设置状态
    s->blockState = TBLOCK_SUBBEGIN;
    
    // 5. 启动子事务
    StartSubTransaction();
    s->blockState = TBLOCK_SUBINPROGRESS;
}
```

### 5.3 StartSubTransaction() 详细流程

```c
// src/backend/access/transam/xact.c:5048
static void StartSubTransaction(void)
{
    TransactionState s = CurrentTransactionState;
    
    // 1. 检查嵌套层次限制
    if (s->nestingLevel >= MaxTransactionDepth)
        ereport(ERROR, ...);
    
    // 2. 设置低层状态
    s->state = TRANS_START;
    
    // 3. 分配子事务ID
    currentSubTransactionId++;
    s->subTransactionId = currentSubTransactionId;
    
    // 4. 分配事务ID (如果需要修改数据)
    // 注意: 只有在实际修改时才分配
    s->fullTransactionId = InvalidFullTransactionId;
    
    // 5. 创建资源
    AtSubStart_Memory();          // 子事务内存上下文
    AtSubStart_ResourceOwner();   // 子事务资源所有者
    
    // 6. 保存快照
    AtSubStart_Snapshot();
    
    // 7. 初始化其他子系统
    AfterTriggerBeginSubXact();
    
    // 8. 设置状态为进行中
    s->state = TRANS_INPROGRESS;
}
```

### 5.4 事务栈结构

```c
// 子事务通过链表实现栈结构
static void PushTransaction(void)
{
    TransactionState p = CurrentTransactionState;
    TransactionState s;
    
    // 分配新节点
    s = (TransactionState)
        MemoryContextAllocZero(TopTransactionContext,
                              sizeof(TransactionStateData));
    
    // 复制父事务的一些字段
    s->fullTransactionId = InvalidFullTransactionId;
    s->subTransactionId = currentSubTransactionId;
    s->nestingLevel = p->nestingLevel + 1;
    s->gucNestLevel = NewGUCNestLevel();
    s->savepointLevel = p->savepointLevel;
    s->state = TRANS_DEFAULT;
    s->blockState = TBLOCK_SUBBEGIN;
    
    // 链接到父事务
    s->parent = p;
    
    // 设置为当前事务
    CurrentTransactionState = s;
}
```

### 5.5 ROLLBACK TO SAVEPOINT

```c
// src/backend/access/transam/xact.c:4249
void RollbackToSavepoint(const char *name)
{
    TransactionState s = CurrentTransactionState;
    TransactionState target = NULL;
    
    // 1. 查找指定名称的SAVEPOINT
    for (;;)
    {
        if (s->name && strcmp(s->name, name) == 0)
        {
            target = s;
            break;
        }
        
        if (s->nestingLevel == 1)
            ereport(ERROR, ...);  // 未找到
        
        s = s->parent;
    }
    
    // 2. 回滚到目标层次
    do
    {
        if (s->blockState == TBLOCK_SUBINPROGRESS)
            AbortSubTransaction();
        else if (s->blockState == TBLOCK_SUBABORT)
            CleanupSubTransaction();
        else
            elog(FATAL, "unexpected state");
        
        s = CurrentTransactionState;
    } while (s != target);
    
    // 3. 准备重新开始
    s->blockState = TBLOCK_SUBRESTART;
}
```

### 5.6 AbortSubTransaction() 流程

```c
// src/backend/access/transam/xact.c:5323
static void AbortSubTransaction(void)
{
    TransactionState s = CurrentTransactionState;
    
    // 1. 设置状态
    s->state = TRANS_ABORT;
    
    // 2. 如果分配了XID，记录回滚
    if (TransactionIdIsValid(s->fullTransactionId))
    {
        // 记录到pg_subtrans
        SubTransSetParent(xid, parentxid, false);
        
        // 更新CLOG
        TransactionIdAbortTree(xid, nchildren, children);
        
        // 从PGPROC移除子事务
        RemoveSubXid(xid);
    }
    
    // 3. 清理资源
    AtSubAbort_Memory();
    AtSubAbort_ResourceOwner();
    AtSubAbort_Portals();
    AtSubAbort_Snapshot();
    
    // 4. 调用回调
    CallSubXactCallbacks(SUBXACT_EVENT_ABORT_SUB, ...);
    
    // 5. 设置状态
    s->state = TRANS_DEFAULT;
}
```

---

## 6. 两阶段提交 (2PC)

### 6.1 PREPARE TRANSACTION

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
PREPARE TRANSACTION 'gid_001';
```

### 6.2 PrepareTransaction() 流程

```c
// src/backend/access/transam/xact.c:2456
static void PrepareTransaction(void)
{
    TransactionState s = CurrentTransactionState;
    GlobalTransaction gxact;
    
    // 1. 检查是否可以准备
    if (!TransactionIdIsValid(xid))
        ereport(ERROR, ...);  // 必须有XID
    
    if (MyProc->delayChkptFlags & DELAY_CHKPT_START)
        ereport(ERROR, ...);  // 不能在checkpoint延迟中
    
    // 2. 分配GlobalTransaction
    gxact = MarkAsPreparing(xid, prepareGID, ...);
    
    // 3. 收集2PC状态信息
    records = GXactLoadSubxactData(xact, &numchildren);
    
    // 4. 写入2PC状态文件到磁盘
    //    文件名: pg_twophase/gid
    //    内容: 事务信息、子事务、锁、等待信息
    StartPrepare(gxact);
    for (each record)
        RegisterTwoPhaseRecord(record);
    EndPrepare(gxact);
    
    // 5. 写入WAL (XLOG_XACT_PREPARE)
    XLogInsert(RM_XACT_ID, XLOG_XACT_PREPARE);
    XLogFlush(recptr);  // 必须刷盘!
    
    // 6. 标记为有效并加入ProcArray
    MarkAsPrepared(gxact, true);
    
    // 7. 释放当前进程的PGPROC
    //    但保留dummy PGPROC在ProcArray中
    ProcArrayRemove(MyProc, InvalidTransactionId);
    
    // 8. 释放大部分资源 (保留锁!)
    AtPrepare_Locks();
    AtPrepare_PredicateLocks();
    
    // 9. 设置状态
    s->state = TRANS_DEFAULT;
}
```

### 6.3 COMMIT PREPARED

```c
// src/backend/access/transam/twophase.c:1169
void FinishPreparedTransaction(const char *gid, bool isCommit)
{
    GlobalTransaction gxact;
    
    // 1. 查找准备好的事务
    gxact = LockGXact(gid, GetUserId());
    
    // 2. 恢复事务状态
    //    从2PC状态文件读取
    RecoverPreparedTransaction(gxact);
    
    // 3. 提交或回滚
    if (isCommit)
    {
        // 记录COMMIT到WAL
        RecordTransactionCommitPrepared(xid, ...);
        
        // 更新CLOG
        TransactionIdCommitTree(xid, nchildren, children);
    }
    else
    {
        // 记录ABORT到WAL
        RecordTransactionAbortPrepared(xid, ...);
        
        // 更新CLOG
        TransactionIdAbortTree(xid, nchildren, children);
    }
    
    // 4. 从ProcArray移除
    ProcArrayRemove(gxact->pgprocno, InvalidTransactionId);
    
    // 5. 释放锁
    AtEOXact_Locks(isCommit);
    
    // 6. 删除2PC状态文件
    RemoveTwoPhaseFile(xid, true);
    
    // 7. 释放GlobalTransaction
    RemoveGXact(gxact);
}
```

---

## 7. 流程总结

### 7.1 事务生命周期完整图

```
用户        高层              中层              低层              WAL/CLOG
 │           │                 │                 │                 │
BEGIN ──────→ BeginTxnBlock ──→ StartTxnCmd ────→ StartTxn ───────→ (分配vxid)
 │           │                 │                 │                 │
 │           │                 │         state: DEFAULT→START→INPROGRESS
 │           │                 │       blockState: DEFAULT→BEGIN→INPROGRESS
 │           │                 │                 │                 │
UPDATE ─────→ (触发XID分配) ───→ ────────────────→ AssignXID ─────→ (分配XID)
 │           │                 │                 │                 │
 │           │                 │                 │       MyProc->xid = 100
 │           │                 │                 │       ProcGlobal->xids[i] = 100
 │           │                 │                 │                 │
COMMIT ─────→ EndTxnBlock ─────→ CommitTxnCmd ───→ CommitTxn ─────→ XLogInsert(COMMIT)
 │           │                 │                 │                 │ XLogFlush()
 │           │                 │                 │                 │ CLOG: 100→COMMITTED
 │           │                 │                 │                 │
 │           │          state: INPROGRESS→COMMIT→DEFAULT          │
 │           │        blockState: INPROGRESS→END→DEFAULT           │
```

### 7.2 关键时间点

| 时刻 | 操作 | 可见性 | 资源状态 |
|------|------|--------|----------|
| T0 | BEGIN | - | 创建内存上下文 |
| T1 | 第一个UPDATE | - | 分配XID，写tuple |
| T2 | COMMIT开始 | - | state=TRANS_COMMIT |
| T3 | 写WAL | - | 记录到WAL buffer |
| T4 | WAL刷盘 | - | 持久化到磁盘 |
| T5 | 更新CLOG | ✅ 可见 | 其他事务可见 |
| T6 | 释放锁 | ✅ 可见 | 其他事务可获取锁 |
| T7 | 清理资源 | ✅ 可见 | 释放内存 |

---

**下一步**: 阅读 [04_key_algorithms.md](04_key_algorithms.md) 深入了解关键算法

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-01-16

