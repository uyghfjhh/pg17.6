# Transaction Manager 关键算法

## 1. 事务ID分配算法

### 1.1 懒惰分配策略

PostgreSQL采用**懒惰分配(Lazy Allocation)**策略，只在事务真正需要写入数据时才分配XID。

**好处**:
- ✅ 只读事务不消耗XID，大幅延长XID使用周期
- ✅ 减少XID回卷压力
- ✅ 降低CLOG、SUBTRANS等SLRU的增长速度
- ✅ 提高只读查询性能

**触发时机**:

```c
// src/backend/access/transam/xact.c:360
TransactionId GetCurrentTransactionId(void)
{
    TransactionState s = CurrentTransactionState;
    
    if (!FullTransactionIdIsValid(s->fullTransactionId))
        AssignTransactionId(s);  // 第一次调用时分配
    
    return XidFromFullTransactionId(s->fullTransactionId);
}

// 在以下操作中会调用GetCurrentTransactionId():
// - heap_insert() - 插入元组
// - heap_update() - 更新元组
// - heap_delete() - 删除元组
// - 修改系统目录
```

### 1.2 64位 FullTransactionId

PostgreSQL 17 引入64位事务ID，彻底解决回卷问题：

```c
typedef struct FullTransactionId
{
    uint64 value;  // Epoch(32位) + XID(32位)
} FullTransactionId;

// 转换函数
FullTransactionId FullTransactionIdFromEpochAndXid(uint32 epoch, TransactionId xid)
{
    FullTransactionId result;
    result.value = ((uint64) epoch << 32) | xid;
    return result;
}

// 比较: 直接比较64位值，不需要循环比较
bool FullTransactionIdPrecedes(FullTransactionId a, FullTransactionId b)
{
    return a.value < b.value;
}
```

**优势**:
- 支持 2^64 个事务 (~1844京个)
- 不需要VACUUM冻结操作
- 简化可见性判断逻辑

### 1.3 XID分配流程

```
GetNewTransactionId(isSubXact)
    │
    ├─ 1. 获取 XidGenLock 排他锁
    │
    ├─ 2. nextXid = TransamVariables->nextXid
    │      TransamVariables->nextXid++
    │
    ├─ 3. 检查是否需要扩展SLRU
    │      ExtendCLOG(xid)
    │      ExtendCommitTs(xid)
    │      if (isSubXact) ExtendSUBTRANS(xid)
    │
    ├─ 4. 检查XID回卷警告
    │      if (xid >= xidVacLimit)
    │          SetTransactionIdLimit()  // 触发autovacuum
    │
    ├─ 5. 更新 PGPROC
    │      MyProc->xid = xid
    │      ProcGlobal->xids[MyProc->pgxactoff] = xid
    │
    └─ 6. 释放 XidGenLock
```

### 1.4 XID回卷检测

```c
// src/backend/access/transam/varsup.c
void SetTransactionIdLimit(TransactionId oldest_datfrozenxid,
                           Oid oldest_datoid)
{
    TransactionId xidVacLimit;
    TransactionId xidWarnLimit;
    TransactionId xidStopLimit;
    TransactionId xidWrapLimit;
    
    // 计算各个阈值 (假设最老XID=100)
    xidWrapLimit = oldest_datfrozenxid + 0x80000000;  // 100 + 21亿
    xidStopLimit = xidWrapLimit - 1000000;            // 停止接受新事务
    xidWarnLimit = xidStopLimit - 10000000;           // 发出警告
    xidVacLimit = oldest_datfrozenxid + autovacuum_freeze_max_age;  // 触发VACUUM
    
    // 保存到共享内存
    TransamVariables->xidVacLimit = xidVacLimit;
    TransamVariables->xidWarnLimit = xidWarnLimit;
    TransamVariables->xidStopLimit = xidStopLimit;
    TransamVariables->xidWrapLimit = xidWrapLimit;
}

// 分配XID时检查
if (TransactionIdFollowsOrEquals(xid, TransamVariables->xidVacLimit))
{
    // 触发紧急autovacuum
    SendPostmasterSignal(PMSIGNAL_START_AUTOVAC_LAUNCHER);
}

if (TransactionIdFollowsOrEquals(xid, TransamVariables->xidWarnLimit))
{
    // 发出警告
    ereport(WARNING, ...);
}

if (TransactionIdFollowsOrEquals(xid, TransamVariables->xidStopLimit))
{
    // 拒绝新事务!
    ereport(ERROR, 
            (errcode(ERRCODE_PROGRAM_LIMIT_EXCEEDED),
             errmsg("database is not accepting commands to avoid wraparound")));
}
```

---

## 2. 子事务算法

### 2.1 子事务栈结构

```
CurrentTransactionState
    ↓
┌─────────────────────────────────┐
│ Level 3: SubXID=3, name="sp2"  │
│ state=TRANS_INPROGRESS          │
│ nestingLevel=3                  │
│ parent ↓                        │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│ Level 2: SubXID=2, name="sp1"  │
│ state=TRANS_INPROGRESS          │
│ nestingLevel=2                  │
│ parent ↓                        │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│ Level 1: XID=100, SubXID=1     │
│ state=TRANS_INPROGRESS          │
│ nestingLevel=1                  │
│ parent = NULL                   │
└─────────────────────────────────┘
```

### 2.2 子事务ID缓存

PostgreSQL使用三级缓存结构存储子事务ID：

```
第1级: PGPROC->subxids (最多64个)
       ├─ 最快速访问
       └─ 用于快照创建时快速判断

第2级: TransactionState->childXids (动态数组)
       ├─ 存储已提交的子事务ID
       └─ 用于COMMIT时写入WAL

第3级: pg_subtrans SLRU (磁盘)
       ├─ 子事务超过64个时溢出到这里
       └─ 格式: SubXID → ParentXID 映射
```

**缓存管理算法**:

```c
// src/backend/access/transam/xact.c
void SubTransSetParent(TransactionId xid, TransactionId parent)
{
    // 1. 尝试缓存到PGPROC
    if (MyProc->subxidStatus == SUBXIDS_IN_ARRAY)
    {
        if (MyProc->subxids.xcnt < PGPROC_MAX_CACHED_SUBXIDS)
        {
            // 还有空间，直接添加
            MyProc->subxids.xids[MyProc->subxids.xcnt++] = xid;
            return;
        }
        else
        {
            // 缓存已满，标记溢出
            MyProc->subxidStatus = SUBXIDS_OVERFLOW;
            ereport(DEBUG1, 
                    (errmsg("子事务溢出，后续需要查询pg_subtrans")));
        }
    }
    
    // 2. 写入pg_subtrans SLRU
    SubTransSetParent(xid, parent);
}
```

### 2.3 子事务提交算法

```c
static void CommitSubTransaction(void)
{
    TransactionState s = CurrentTransactionState;
    TransactionState p = s->parent;
    
    // 1. 检查状态
    Assert(s->state == TRANS_INPROGRESS);
    s->state = TRANS_COMMIT;
    
    // 2. 如果分配了XID，记录到父事务
    if (TransactionIdIsValid(s->fullTransactionId))
    {
        xid = XidFromFullTransactionId(s->fullTransactionId);
        
        // 添加到父事务的childXids数组
        if (p->nChildXids >= p->maxChildXids)
        {
            // 扩展数组
            p->maxChildXids = Max(8, p->maxChildXids * 2);
            p->childXids = repalloc(p->childXids, 
                                   p->maxChildXids * sizeof(TransactionId));
        }
        p->childXids[p->nChildXids++] = xid;
        
        // 排序 (方便后续查找)
        if (p->nChildXids > 1)
            qsort(p->childXids, p->nChildXids, sizeof(TransactionId), xidComparator);
    }
    
    // 3. 合并资源到父事务
    AtSubCommit_Memory();
    AtSubCommit_childXids();
    AtSubCommit_ResourceOwner();
    
    // 4. 弹出栈
    PopTransaction();
    
    // 5. 回调
    CallSubXactCallbacks(SUBXACT_EVENT_COMMIT_SUB, ...);
}
```

### 2.4 子事务回滚算法

```c
static void AbortSubTransaction(void)
{
    TransactionState s = CurrentTransactionState;
    
    // 1. 设置状态
    s->state = TRANS_ABORT;
    
    // 2. 如果有XID，标记为已回滚
    if (TransactionIdIsValid(s->fullTransactionId))
    {
        xid = XidFromFullTransactionId(s->fullTransactionId);
        
        // 更新CLOG: XID → ABORTED
        TransactionIdAbortTree(xid, s->nChildXids, s->childXids);
        
        // 从PGPROC移除
        if (MyProc->subxids.xcnt > 0)
        {
            for (int i = 0; i < MyProc->subxids.xcnt; i++)
            {
                if (MyProc->subxids.xids[i] == xid)
                {
                    // 移除这个XID
                    memmove(&MyProc->subxids.xids[i],
                            &MyProc->subxids.xids[i+1],
                            (MyProc->subxids.xcnt - i - 1) * sizeof(TransactionId));
                    MyProc->subxids.xcnt--;
                    break;
                }
            }
        }
    }
    
    // 3. 清理资源
    AtSubAbort_Memory();
    AtSubAbort_ResourceOwner();
    AtSubAbort_childXids();
    
    // 4. 删除快照、释放锁等
    ResourceOwnerRelease(s->curTransactionOwner, ...);
    
    // 5. 状态重置
    s->state = TRANS_DEFAULT;
}
```

---

## 3. 两阶段提交 (2PC) 算法

### 3.1 2PC 协议概述

```
阶段1: PREPARE
    ├─ 协调者发送 PREPARE
    ├─ 参与者准备提交 (写WAL、持久化状态)
    ├─ 参与者回复 YES/NO
    └─ 如果所有参与者回复YES，进入阶段2

阶段2: COMMIT/ABORT
    ├─ 协调者决定 COMMIT 或 ABORT
    ├─ 协调者发送决定到所有参与者
    ├─ 参与者执行 COMMIT/ABORT
    └─ 参与者回复完成
```

### 3.2 PrepareTransaction 算法

```c
static void PrepareTransaction(void)
{
    GlobalTransaction gxact;
    TransactionId xid;
    
    // 1. 检查前置条件
    xid = XidFromFullTransactionId(XactTopFullTransactionId);
    if (!TransactionIdIsValid(xid))
        ereport(ERROR, 
                (errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
                 errmsg("cannot PREPARE a transaction that has not been assigned an XID")));
    
    // 2. 分配GlobalTransaction结构
    gxact = MarkAsPreparing(xid, prepareGID, prepared_at, owner, databaseid);
    
    // 3. 收集需要持久化的信息
    //    - 子事务ID列表
    //    - 持有的锁列表
    //    - 等待的锁信息
    //    - 资源管理器状态
    StartPrepare(gxact);
    
    // 子事务
    if (nchildren > 0)
        RegisterTwoPhaseRecord(TWOPHASE_RM_XACT_ID, ...);
    
    // 锁
    AtPrepare_Locks();
    AtPrepare_PredicateLocks();
    
    // 其他资源管理器
    for (rmid = 0; rmid <= RM_MAX_ID; rmid++)
    {
        if (RmgrTable[rmid].rm_twophase_prepare)
            RmgrTable[rmid].rm_twophase_prepare();
    }
    
    EndPrepare(gxact);
    
    // 4. 写入2PC状态文件
    //    文件路径: $PGDATA/pg_twophase/<XID>
    //    格式:
    //      TwoPhaseFileHeader
    //      TwoPhaseRecordOnDisk[] (变长)
    fd = BufFileOpenWrite(TwoPhaseFilePath(xid));
    BufFileWrite(fd, &hdr, sizeof(TwoPhaseFileHeader));
    for (each record)
        BufFileWrite(fd, record, record->len);
    BufFileClose(fd);
    
    // 5. 写入WAL (XLOG_XACT_PREPARE)
    XLogBeginInsert();
    XLogRegisterData((char *) &xlrec, sizeof(xl_xact_prepare));
    recptr = XLogInsert(RM_XACT_ID, XLOG_XACT_PREPARE);
    
    // 6. 刷盘WAL (必须!)
    XLogFlush(recptr);
    
    // 7. 标记为prepared并加入ProcArray
    MarkAsPrepared(gxact, true);
    
    // 8. 创建dummy PGPROC (代表prepared事务)
    //    - pid = 0 (标识为prepared事务)
    //    - xid = <事务ID>
    //    - 持有所有锁
    gxact->proc = PreparedXactProcs[gxact->pgprocno];
    gxact->proc->xid = xid;
    gxact->proc->xmin = TransactionXmin;
    ProcArrayAdd(gxact->proc);
    
    // 9. 当前进程释放事务状态 (但保留锁!)
    CurrentTransactionState->state = TRANS_DEFAULT;
}
```

### 3.3 COMMIT/ROLLBACK PREPARED 算法

```c
void FinishPreparedTransaction(const char *gid, bool isCommit)
{
    GlobalTransaction gxact;
    PGPROC *proc;
    
    // 1. 查找并锁定GlobalTransaction
    gxact = LockGXact(gid, GetUserId());
    xid = gxact->xid;
    proc = gxact->proc;
    
    // 2. 从2PC状态文件恢复事务信息
    //    读取: 子事务、锁、资源状态
    buf = ReadTwoPhaseFile(xid);
    hdr = (TwoPhaseFileHeader *) buf;
    
    // 恢复子事务信息
    if (hdr->nsubxacts > 0)
        memcpy(children, ..., hdr->nsubxacts * sizeof(TransactionId));
    
    // 恢复锁信息
    for (each record)
    {
        if (record->rmid == TWOPHASE_RM_LOCK_ID)
            lock_twophase_recover(record);
    }
    
    // 3. 决定提交或回滚
    if (isCommit)
    {
        // 记录COMMIT到WAL
        RecordTransactionCommitPrepared(xid, nchildren, children, ...);
        
        // 更新CLOG
        TransactionIdCommitTree(xid, nchildren, children);
        
        // 调用资源管理器的提交回调
        for (each record)
            RmgrTable[record->rmid].rm_twophase_postcommit(record);
    }
    else
    {
        // 记录ABORT到WAL
        RecordTransactionAbortPrepared(xid, nchildren, children);
        
        // 更新CLOG
        TransactionIdAbortTree(xid, nchildren, children);
        
        // 调用资源管理器的回滚回调
        for (each record)
            RmgrTable[record->rmid].rm_twophase_postabort(record);
    }
    
    // 4. 从ProcArray移除dummy PGPROC
    ProcArrayRemove(proc, InvalidTransactionId);
    
    // 5. 释放锁
    LockReleaseAll(DEFAULT_LOCKMETHOD, false);
    
    // 6. 删除2PC状态文件
    RemoveTwoPhaseFile(xid, true);
    
    // 7. 释放GlobalTransaction
    RemoveGXact(gxact);
}
```

### 3.4 2PC 崩溃恢复

```c
// 启动时恢复prepared事务
void RecoverPreparedTransactions(void)
{
    DIR *cldir;
    struct dirent *clde;
    
    // 1. 扫描pg_twophase目录
    cldir = AllocateDir("pg_twophase");
    while ((clde = ReadDir(cldir, "pg_twophase")) != NULL)
    {
        // 2. 读取每个2PC状态文件
        xid = atoi(clde->d_name);
        buf = ReadTwoPhaseFile(xid);
        hdr = (TwoPhaseFileHeader *) buf;
        
        // 3. 重建GlobalTransaction
        gxact = MarkAsPreparing(xid, hdr->gid, ...);
        
        // 4. 恢复锁和资源状态
        for (each record in buf)
        {
            RmgrTable[record->rmid].rm_twophase_recover(record);
        }
        
        // 5. 创建dummy PGPROC并加入ProcArray
        proc = gxact->proc;
        proc->xid = xid;
        proc->xmin = ...;
        ProcArrayAdd(proc);
        
        // 6. 标记为有效
        MarkAsPrepared(gxact, false);  // ondisk=true
    }
    
    FreeDir(cldir);
}
```

---

## 4. 组提交 (Group Commit) 优化

### 4.1 组提交原理

多个并发事务的COMMIT操作可以共享一次WAL刷盘，大幅提高吞吐量。

```
时刻   事务1      事务2      事务3      WAL
T1:   COMMIT
T2:            COMMIT
T3:                      COMMIT
T4:   [等待WAL刷盘] [等待] [等待]
T5:   [一次XLogFlush刷盘所有事务的WAL]
T6:   完成       完成       完成
```

### 4.2 组提交实现

```c
// src/backend/access/transam/xlog.c
void XLogFlush(XLogRecPtr record)
{
    // 1. 检查是否已经刷盘
    if (record <= LogwrtResult.Flush)
        return;  // 已刷盘
    
    // 2. 尝试加入组提交
    //    如果有其他进程正在刷盘，等待它完成
    if (XLogCtl->LogwrtRqst.Flush >= record)
    {
        // 其他进程会帮我们刷盘
        while (LogwrtResult.Flush < record)
        {
            pg_usleep(1000);  // 1ms
            SpinLockAcquire(&XLogCtl->info_lck);
            // 检查是否已刷盘
            if (LogwrtResult.Flush >= record)
                break;
            SpinLockRelease(&XLogCtl->info_lck);
        }
        return;
    }
    
    // 3. 成为组长，执行刷盘
    WALInsertLockAcquireExclusive();
    
    // 收集所有待刷盘的WAL记录
    XLogCtl->LogwrtRqst.Flush = GetInsertRecPtr();
    
    // 执行刷盘
    XLogWrite(...);
    
    WALInsertLockRelease();
    
    // 4. 唤醒等待的进程
    WakeupWaiters();
}
```

---

## 5. 快照算法

### 5.1 快照创建

```c
// src/backend/storage/ipc/procarray.c
Snapshot GetSnapshotData(Snapshot snapshot)
{
    // 1. 获取ProcArrayLock共享锁
    LWLockAcquire(ProcArrayLock, LW_SHARED);
    
    // 2. 获取当前最大XID
    snapshot->xmax = TransamVariables->nextXid;
    
    // 3. 扫描所有活跃事务
    snapshot->xcnt = 0;
    for (int i = 0; i < numProcs; i++)
    {
        TransactionId xid = ProcGlobal->xids[i];
        
        if (TransactionIdIsNormal(xid))
        {
            // 添加到活跃事务数组
            snapshot->xip[snapshot->xcnt++] = xid;
            
            // 更新xmin (最小活跃XID)
            if (xid < snapshot->xmin)
                snapshot->xmin = xid;
            
            // 收集子事务ID
            if (ProcGlobal->subxidStates[i] == SUBXIDS_IN_ARRAY)
            {
                for (int j = 0; j < subxcnt; j++)
                    snapshot->subxip[snapshot->subxcnt++] = subxid[j];
            }
        }
    }
    
    // 4. 排序活跃事务数组 (用于二分查找)
    qsort(snapshot->xip, snapshot->xcnt, sizeof(TransactionId), xidComparator);
    
    // 5. 释放锁
    LWLockRelease(ProcArrayLock);
    
    return snapshot;
}
```

### 5.2 快照可见性判断

```sql
-- 快照: xmin=100, xmax=110, xip=[102, 105, 108]

XID=99:  < xmin          → 已提交，可见 ✓
XID=100: == xmin         → 已提交，可见 ✓
XID=102: in xip[]        → 活跃事务，不可见 ✗
XID=105: in xip[]        → 活跃事务，不可见 ✗
XID=107: not in xip[]    → 已提交，可见 ✓
XID=110: >= xmax         → 未来事务，不可见 ✗
```

---

## 6. 死锁检测算法

虽然主要由Lock Manager处理，但事务管理器需要配合检测死锁：

```c
// 等待图 (Wait-For Graph) 死锁检测
bool DeadLockCheck(PGPROC *proc)
{
    // 1. 构建等待图
    //    边: A → B 表示 A等待B持有的锁
    for (each process in ProcArray)
    {
        if (proc is waiting for lock held by other)
            AddEdge(proc, other);
    }
    
    // 2. DFS检测环
    visited = {};
    stack = {proc};
    
    while (stack not empty)
    {
        current = stack.pop();
        
        if (current in visited)
            return true;  // 检测到环，存在死锁!
        
        visited.add(current);
        
        for (each edge: current → next)
            stack.push(next);
    }
    
    return false;  // 无死锁
}
```

---

## 总结

### 核心算法要点

1. **懒惰XID分配**: 只读事务不消耗XID
2. **64位FullTransactionId**: 彻底解决回卷
3. **子事务三级缓存**: PGPROC → childXids → pg_subtrans
4. **2PC状态持久化**: 准备好的事务存活于崩溃
5. **组提交优化**: 多个COMMIT共享WAL刷盘
6. **快照算法**: O(N)时间创建，二分查找判断可见性

---

**下一步**: 阅读 [05_performance.md](05_performance.md) 学习性能优化技巧

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-01-16

