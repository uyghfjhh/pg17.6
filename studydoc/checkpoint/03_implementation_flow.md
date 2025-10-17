# PostgreSQL Checkpoint - 实现流程详解

## 目录

1. [CreateCheckPoint 完整流程](#1-createcheckpoint-完整流程)
2. [CheckPointGuts 详细步骤](#2-checkpointguts-详细步骤)
3. [BufferSync 实现分析](#3-buffersync-实现分析)
4. [ProcessSyncRequests 处理](#4-processsyncrequests-处理)
5. [关键同步点和锁](#5-关键同步点和锁)
6. [时序分析](#6-时序分析)
7. [错误处理和恢复](#7-错误处理和恢复)

---

## 1. CreateCheckPoint 完整流程

**位置**: `src/backend/access/transam/xlog.c:6881`

### 1.1 函数签名

```c
void CreateCheckPoint(int flags)
```

**参数 flags 的可能组合**:
```c
// 基本标志
CHECKPOINT_IS_SHUTDOWN      0x0001  // 关闭 checkpoint
CHECKPOINT_END_OF_RECOVERY  0x0002  // 恢复结束
CHECKPOINT_IMMEDIATE        0x0004  // 立即完成，不限流
CHECKPOINT_FORCE            0x0008  // 强制执行
CHECKPOINT_WAIT             0x0010  // 等待完成（用于后端）
CHECKPOINT_CAUSE_XLOG       0x0020  // WAL 大小触发
CHECKPOINT_CAUSE_TIME       0x0040  // 时间触发
CHECKPOINT_FLUSH_ALL        0x0080  // 刷写所有页（含 unlogged）
```

### 1.2 流程步骤详解

#### 阶段 1: 准备和初始化 (行 6881-6924)

```c
void CreateCheckPoint(int flags)
{
    bool        shutdown;
    CheckPoint  checkPoint;
    XLogRecPtr  recptr;
    XLogSegNo   _logSegNo;
    XLogCtlInsert *Insert = &XLogCtl->Insert;
    uint32      freespace;
    XLogRecPtr  PriorRedoPtr;
    XLogRecPtr  last_important_lsn;
    VirtualTransactionId *vxids;
    int         nvxids;
    int         oldXLogAllowed = 0;

    // 步骤 1.1: 判断是否为 shutdown checkpoint
    // xlog.c:6899
    if (flags & (CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_END_OF_RECOVERY))
        shutdown = true;
    else
        shutdown = false;

    // 步骤 1.2: 安全性检查
    // xlog.c:6905
    if (RecoveryInProgress() && (flags & CHECKPOINT_END_OF_RECOVERY) == 0)
        elog(ERROR, "can't create a checkpoint during recovery");

    // 步骤 1.3: 初始化统计信息
    // xlog.c:6915
    MemSet(&CheckpointStats, 0, sizeof(CheckpointStats));
    CheckpointStats.ckpt_start_t = GetCurrentTimestamp();

    // 步骤 1.4: 让 smgr 准备 checkpoint（在临界区外）
    // xlog.c:6924
    SyncPreCheckpoint();
    // 作用:
    // - 通知存储管理器即将开始 checkpoint
    // - 某些文件系统操作需要在临界区外完成
```

#### 阶段 2: 进入临界区，更新状态 (行 6929-6951)

```c
    // 步骤 2.1: 进入临界区
    // xlog.c:6929
    START_CRIT_SECTION();
    // 重要: 临界区内的错误会导致 PANIC，强制数据库重启

    // 步骤 2.2: Shutdown checkpoint 时更新控制文件状态
    // xlog.c:6931-6937
    if (shutdown)
    {
        LWLockAcquire(ControlFileLock, LW_EXCLUSIVE);
        ControlFile->state = DB_SHUTDOWNING;  // 标记正在关闭
        UpdateControlFile();                   // 立即写入磁盘
        LWLockRelease(ControlFileLock);
    }

    // 步骤 2.3: 初始化 CheckPoint 结构
    // xlog.c:6940-6941
    MemSet(&checkPoint, 0, sizeof(checkPoint));
    checkPoint.time = (pg_time_t) time(NULL);

    // 步骤 2.4: Hot Standby 需要记录最老活跃事务
    // xlog.c:6948-6951
    if (!shutdown && XLogStandbyInfoActive())
        checkPoint.oldestActiveXid = GetOldestActiveTransactionId();
    else
        checkPoint.oldestActiveXid = InvalidTransactionId;
```

#### 阶段 3: 获取 LSN 并检查是否跳过 (行 6953-6974)

```c
    // 步骤 3.1: 获取最后重要记录的 LSN
    // xlog.c:6957
    last_important_lsn = GetLastImportantRecPtr();
    // 遍历所有 WAL 插入锁，找到最大的 lastImportantAt

    // 步骤 3.2: 检查是否可以跳过 checkpoint
    // xlog.c:6964-6974
    if ((flags & (CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_END_OF_RECOVERY |
                  CHECKPOINT_FORCE)) == 0)
    {
        // 如果没有新的重要 WAL 记录，跳过 checkpoint
        if (last_important_lsn == ControlFile->checkPoint)
        {
            END_CRIT_SECTION();
            ereport(DEBUG1,
                    (errmsg_internal("checkpoint skipped because system is idle")));
            return;  // 直接返回，不执行 checkpoint
        }
    }

    // 场景示例:
    // 1. checkpoint_timeout 触发
    // 2. 但系统完全空闲，没有任何事务
    // 3. last_important_lsn == ControlFile->checkPoint
    // 4. 跳过 checkpoint，避免无意义的 I/O
```

#### 阶段 4: 确定 REDO 点 (行 6981-7062)

```c
    // 步骤 4.1: End-of-recovery checkpoint 需要临时允许 WAL 插入
    // xlog.c:6981-6982
    if (flags & CHECKPOINT_END_OF_RECOVERY)
        oldXLogAllowed = LocalSetXLogInsertAllowed();

    // 步骤 4.2: 设置时间线信息
    // xlog.c:6984-6988
    checkPoint.ThisTimeLineID = XLogCtl->InsertTimeLineID;
    if (flags & CHECKPOINT_END_OF_RECOVERY)
        checkPoint.PrevTimeLineID = XLogCtl->PrevTimeLineID;
    else
        checkPoint.PrevTimeLineID = checkPoint.ThisTimeLineID;

    // 步骤 4.3: 获取所有 WAL 插入锁（阻止并发写入）
    // xlog.c:6993
    WALInsertLockAcquireExclusive();
    // 此时没有任何进程能插入新的 WAL 记录

    // 步骤 4.4: 记录当前配置
    // xlog.c:6995-6996
    checkPoint.fullPageWrites = Insert->fullPageWrites;
    checkPoint.wal_level = wal_level;

    // 步骤 4.5: Shutdown checkpoint 的 REDO 点计算
    // xlog.c:6998-7030
    if (shutdown)
    {
        XLogRecPtr curInsert = XLogBytePosToRecPtr(Insert->CurrBytePos);

        // 计算下一个 XLOG 记录的位置
        freespace = INSERT_FREESPACE(curInsert);
        if (freespace == 0)
        {
            // 页面已满，需要跳到下一个页面的头部
            if (XLogSegmentOffset(curInsert, wal_segment_size) == 0)
                curInsert += SizeOfXLogLongPHD;   // 段开始，长页头
            else
                curInsert += SizeOfXLogShortPHD;  // 段中间，短页头
        }
        checkPoint.redo = curInsert;

        // 更新共享的 RedoRecPtr
        RedoRecPtr = XLogCtl->Insert.RedoRecPtr = checkPoint.redo;

        // 注意: 如果 checkpoint 失败，RedoRecPtr 会指向比实际需要更远的位置
        // 这是安全的，只是可能导致更多的全页写
    }

    // 步骤 4.6: 释放 WAL 插入锁
    // xlog.c:7037
    WALInsertLockRelease();
    // 现在其他进程可以继续插入 WAL，但它们的修改不会包含在本次 checkpoint 中

    // 步骤 4.7: Online checkpoint 的 REDO 点插入
    // xlog.c:7048-7062
    if (!shutdown)
    {
        // 插入 XLOG_CHECKPOINT_REDO 记录
        XLogBeginInsert();
        XLogRegisterData((char *) &wal_level, sizeof(wal_level));
        (void) XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT_REDO);

        // XLogInsert 已更新 RedoRecPtr
        checkPoint.redo = RedoRecPtr;
    }

    // 步骤 4.8: 更新 info_lck 保护的 RedoRecPtr 副本
    // xlog.c:7065-7067
    SpinLockAcquire(&XLogCtl->info_lck);
    XLogCtl->RedoRecPtr = checkPoint.redo;
    SpinLockRelease(&XLogCtl->info_lck);
```

**REDO 点的区别图示**:

```
Shutdown Checkpoint:
  时间线: [事务A] [事务B] [等待所有完成] [Checkpoint记录]
                                              ↑
                                          redo点

  特点:
  - 没有并发事务
  - checkpoint 记录本身就是 redo 点
  - 恢复时无需重放任何 WAL

Online Checkpoint:
  时间线: [事务A] [事务B] [REDO记录] [CheckPointGuts] [Checkpoint记录] [事务C]
                              ↑
                          redo点

  特点:
  - 有并发事务
  - REDO 记录的 LSN 是 redo 点
  - redo 点之后的所有修改都被 checkpoint 持久化
  - 恢复时从 redo 点开始重放
```

#### 阶段 5: 记录日志并收集元数据 (行 7073-7110)

```c
    // 步骤 5.1: 记录 checkpoint 开始日志
    // xlog.c:7073-7074
    if (log_checkpoints)
        LogCheckpointStart(flags, false);
    // 输出: "LOG:  checkpoint starting: time"

    // 步骤 5.2: 更新进程标题
    // xlog.c:7077
    update_checkpoint_display(flags, false, false);
    // ps 输出: "postgres: checkpointer performing checkpoint"

    // 步骤 5.3: 追踪点
    // xlog.c:7079
    TRACE_POSTGRESQL_CHECKPOINT_START(flags);

    // 步骤 5.4: 收集事务 ID 信息
    // xlog.c:7089-7093
    LWLockAcquire(XidGenLock, LW_SHARED);
    checkPoint.nextXid = TransamVariables->nextXid;
    checkPoint.oldestXid = TransamVariables->oldestXid;
    checkPoint.oldestXidDB = TransamVariables->oldestXidDB;
    LWLockRelease(XidGenLock);

    // 步骤 5.5: 收集提交时间戳信息
    // xlog.c:7095-7098
    LWLockAcquire(CommitTsLock, LW_SHARED);
    checkPoint.oldestCommitTsXid = TransamVariables->oldestCommitTsXid;
    checkPoint.newestCommitTsXid = TransamVariables->newestCommitTsXid;
    LWLockRelease(CommitTsLock);

    // 步骤 5.6: 收集对象 ID 信息
    // xlog.c:7100-7104
    LWLockAcquire(OidGenLock, LW_SHARED);
    checkPoint.nextOid = TransamVariables->nextOid;
    if (!shutdown)
        checkPoint.nextOid += TransamVariables->oidCount;  // 预留的 OID
    LWLockRelease(OidGenLock);

    // 步骤 5.7: 收集 MultiXact 信息
    // xlog.c:7106-7110
    MultiXactGetCheckptMulti(shutdown,
                             &checkPoint.nextMulti,
                             &checkPoint.nextMultiOffset,
                             &checkPoint.oldestMulti,
                             &checkPoint.oldestMultiDB);
```

#### 阶段 6: 等待事务临界区 (行 7120-7169)

```c
    // 步骤 6.1: 退出临界区（I/O 操作可能失败）
    // xlog.c:7120
    END_CRIT_SECTION();

    // 步骤 6.2: 等待事务提交临界区完成
    // xlog.c:7151-7169
    vxids = GetVirtualXIDsDelayingChkpt(&nvxids, DELAY_CHKPT_START);
    if (nvxids > 0)
    {
        do
        {
            // 持续吸收 fsync 请求，避免队列满导致死锁
            AbsorbSyncRequests();

            pgstat_report_wait_start(WAIT_EVENT_CHECKPOINT_DELAY_START);
            pg_usleep(10000L);  // 休眠 10ms
            pgstat_report_wait_end();
        } while (HaveVirtualXIDsDelayingChkpt(vxids, nvxids,
                                              DELAY_CHKPT_START));
    }
    pfree(vxids);

    // 原因:
    // 事务在提交时会:
    // 1. 写 WAL 记录 (在 redo 点之前)
    // 2. 更新 CLOG (在临界区内)
    // 如果 checkpoint 在步骤 1 和 2 之间发生，会导致不一致。
    // 必须等待所有这样的事务完成临界区。
```

#### 阶段 7: 核心刷写操作 (行 7171)

```c
    // 步骤 7.1: 执行核心的 checkpoint 操作
    // xlog.c:7171
    CheckPointGuts(checkPoint.redo, flags);
    // 详见下一节

    // 这是整个 checkpoint 最耗时的部分，可能需要几分钟
```

#### 阶段 8: 等待延迟完成的事务 (行 7173-7186)

```c
    // 步骤 8.1: 等待延迟完成的事务
    // xlog.c:7173-7186
    vxids = GetVirtualXIDsDelayingChkpt(&nvxids, DELAY_CHKPT_COMPLETE);
    if (nvxids > 0)
    {
        do
        {
            AbsorbSyncRequests();

            pgstat_report_wait_start(WAIT_EVENT_CHECKPOINT_DELAY_COMPLETE);
            pg_usleep(10000L);
            pgstat_report_wait_end();
        } while (HaveVirtualXIDsDelayingChkpt(vxids, nvxids,
                                              DELAY_CHKPT_COMPLETE));
    }
    pfree(vxids);

    // 某些操作需要确保 checkpoint 完成后才能继续，例如:
    // - CREATE DATABASE (复制模板数据库)
    // - DROP TABLESPACE
```

#### 阶段 9: 记录运行中事务快照 (行 7196-7197)

```c
    // 步骤 9.1: 为 Hot Standby 记录运行中事务快照
    // xlog.c:7196-7197
    if (!shutdown && XLogStandbyInfoActive())
        LogStandbySnapshot();

    // 作用:
    // - 备库可以知道哪些事务在主库上仍在运行
    // - 只读查询可以安全执行，不会看到不一致的数据
```

#### 阶段 10: 写入 Checkpoint WAL 记录 (行 7199-7210)

```c
    // 步骤 10.1: 重新进入临界区
    // xlog.c:7199
    START_CRIT_SECTION();

    // 步骤 10.2: 插入 checkpoint 记录到 WAL
    // xlog.c:7204-7208
    XLogBeginInsert();
    XLogRegisterData((char *) (&checkPoint), sizeof(checkPoint));
    recptr = XLogInsert(RM_XLOG_ID,
                       shutdown ? XLOG_CHECKPOINT_SHUTDOWN :
                                 XLOG_CHECKPOINT_ONLINE);

    // 步骤 10.3: 强制刷写 WAL 到磁盘
    // xlog.c:7210
    XLogFlush(recptr);
    // 确保 checkpoint 记录持久化，这是关键点

    // 步骤 10.4: Shutdown checkpoint 禁止后续 WAL 写入
    // xlog.c:7219-7225
    if (shutdown)
    {
        if (flags & CHECKPOINT_END_OF_RECOVERY)
            LocalXLogInsertAllowed = oldXLogAllowed;
        else
            LocalXLogInsertAllowed = 0;  // 永远不再写 WAL
    }

    // 步骤 10.5: 安全性检查
    // xlog.c:7231-7233
    if (shutdown && checkPoint.redo != ProcLastRecPtr)
        ereport(PANIC,
                (errmsg("concurrent write-ahead log activity while database system is shutting down")));
    // Shutdown 时不应该有并发 WAL 写入
```

#### 阶段 11: 更新 pg_control (行 7239-7266)

```c
    // 步骤 11.1: 记录前一个 checkpoint 的 redo 点
    // xlog.c:7239
    PriorRedoPtr = ControlFile->checkPointCopy.redo;

    // 步骤 11.2: 更新控制文件
    // xlog.c:7244-7261
    LWLockAcquire(ControlFileLock, LW_EXCLUSIVE);

    if (shutdown)
        ControlFile->state = DB_SHUTDOWNED;  // 标记已关闭

    ControlFile->checkPoint = ProcLastRecPtr;      // checkpoint 记录的 LSN
    ControlFile->checkPointCopy = checkPoint;       // checkpoint 的完整副本

    // 崩溃恢复总是恢复到 WAL 末尾
    ControlFile->minRecoveryPoint = InvalidXLogRecPtr;
    ControlFile->minRecoveryPointTLI = 0;

    // 持久化 unlogged 表的 LSN
    ControlFile->unloggedLSN = pg_atomic_read_membarrier_u64(&XLogCtl->unloggedLSN);

    UpdateControlFile();  // 原子写入磁盘
    LWLockRelease(ControlFileLock);

    // 步骤 11.3: 更新共享内存中的 checkpoint XID
    // xlog.c:7264-7266
    SpinLockAcquire(&XLogCtl->info_lck);
    XLogCtl->ckptFullXid = checkPoint.nextXid;
    SpinLockRelease(&XLogCtl->info_lck);

    // 步骤 11.4: 退出临界区
    // xlog.c:7272
    END_CRIT_SECTION();
    // 至此，checkpoint 已成功持久化
```

#### 阶段 12: 后续清理工作 (行 7293-7341)

```c
    // 步骤 12.1: 通知 WAL summarizer
    // xlog.c:7291
    SetWalSummarizerLatch();

    // 步骤 12.2: Smgr 后续清理
    // xlog.c:7296
    SyncPostCheckpoint();

    // 步骤 12.3: 更新 checkpoint 距离估计
    // xlog.c:7302-7303
    if (PriorRedoPtr != InvalidXLogRecPtr)
        UpdateCheckPointDistanceEstimate(RedoRecPtr - PriorRedoPtr);

    // 步骤 12.4: 删除旧的 WAL 文件
    // xlog.c:7309-7324
    XLByteToSeg(RedoRecPtr, _logSegNo, wal_segment_size);
    KeepLogSeg(recptr, &_logSegNo);

    // 使复制槽失效（如果它们落后太多）
    if (InvalidateObsoleteReplicationSlots(RS_INVAL_WAL_REMOVED,
                                           _logSegNo, InvalidOid,
                                           InvalidTransactionId))
    {
        // 重新计算
        XLByteToSeg(RedoRecPtr, _logSegNo, wal_segment_size);
        KeepLogSeg(recptr, &_logSegNo);
    }

    _logSegNo--;  // 保留 redo 点所在的段
    RemoveOldXlogFiles(_logSegNo, RedoRecPtr, recptr,
                       checkPoint.ThisTimeLineID);

    // 步骤 12.5: 预分配新的 WAL 段
    // xlog.c:7330-7331
    if (!shutdown)
        PreallocXlogFiles(recptr, checkPoint.ThisTimeLineID);

    // 步骤 12.6: 截断 pg_subtrans
    // xlog.c:7340-7341
    if (!RecoveryInProgress())
        TruncateSUBTRANS(GetOldestTransactionIdConsideredRunning());

    // 步骤 12.7: 记录日志和统计
    // xlog.c:7344
    LogCheckpointEnd(false);

    // 步骤 12.8: 重置进程标题
    // xlog.c:7347
    update_checkpoint_display(flags, false, true);

    // 步骤 12.9: 追踪点
    // xlog.c:7349-7353
    TRACE_POSTGRESQL_CHECKPOINT_DONE(CheckpointStats.ckpt_bufs_written,
                                     NBuffers,
                                     CheckpointStats.ckpt_segs_added,
                                     CheckpointStats.ckpt_segs_removed,
                                     CheckpointStats.ckpt_segs_recycled);
}  // CreateCheckPoint 结束
```

### 1.3 流程图总览

```
CreateCheckPoint() 完整流程:

开始
  │
  ├─[1]─> 准备和初始化
  │       - 判断 shutdown
  │       - 初始化统计
  │       - SyncPreCheckpoint()
  │
  ├─[2]─> 进入临界区
  │       - START_CRIT_SECTION()
  │       - 更新 ControlFile->state (如果 shutdown)
  │       - 初始化 CheckPoint 结构
  │
  ├─[3]─> 获取 LSN 并检查跳过
  │       - GetLastImportantRecPtr()
  │       - 如果系统空闲 -> 跳过并返回
  │
  ├─[4]─> 确定 REDO 点
  │       - WALInsertLockAcquireExclusive()
  │       - 如果 shutdown: redo = 当前位置
  │       - 如果 online: 插入 REDO 记录，redo = 记录 LSN
  │       - WALInsertLockRelease()
  │       - 更新 XLogCtl->RedoRecPtr
  │
  ├─[5]─> 记录日志并收集元数据
  │       - LogCheckpointStart()
  │       - 收集 XID, OID, MultiXact 信息
  │
  ├─[6]─> 退出临界区
  │       - END_CRIT_SECTION()
  │
  ├─[7]─> 等待事务临界区
  │       - GetVirtualXIDsDelayingChkpt(DELAY_CHKPT_START)
  │       - 循环等待，持续 AbsorbSyncRequests()
  │
  ├─[8]─> ★核心刷写★
  │       - CheckPointGuts(checkPoint.redo, flags)
  │         ├─> CheckPointRelationMap()
  │         ├─> CheckPointReplicationSlots()
  │         ├─> CheckPointCLOG()
  │         ├─> CheckPointCommitTs()
  │         ├─> CheckPointSUBTRANS()
  │         ├─> CheckPointMultiXact()
  │         ├─> CheckPointBuffers()  <-- 最耗时
  │         │   └─> BufferSync()
  │         ├─> ProcessSyncRequests()  <-- 批量 fsync
  │         └─> CheckPointTwoPhase()
  │
  ├─[9]─> 等待延迟完成事务
  │       - GetVirtualXIDsDelayingChkpt(DELAY_CHKPT_COMPLETE)
  │
  ├─[10]─> 记录运行中事务快照
  │        - LogStandbySnapshot() (如果 hot standby)
  │
  ├─[11]─> 写入 Checkpoint WAL 记录
  │        - START_CRIT_SECTION()
  │        - XLogInsert(XLOG_CHECKPOINT_*)
  │        - XLogFlush(recptr)
  │
  ├─[12]─> 更新 pg_control
  │        - LWLockAcquire(ControlFileLock)
  │        - ControlFile->checkPoint = recptr
  │        - ControlFile->checkPointCopy = checkPoint
  │        - UpdateControlFile()
  │        - END_CRIT_SECTION()
  │
  └─[13]─> 后续清理
          - SyncPostCheckpoint()
          - RemoveOldXlogFiles()
          - PreallocXlogFiles()
          - TruncateSUBTRANS()
          - LogCheckpointEnd()
结束
```

---

## 2. CheckPointGuts 详细步骤

**位置**: `src/backend/access/transam/xlog.c:7500`

### 2.1 函数实现

```c
static void
CheckPointGuts(XLogRecPtr checkPointRedo, int flags)
{
    // 第 1 部分: Checkpoint 元数据
    // xlog.c:7502-7506
    CheckPointRelationMap();              // 关系文件号映射
    CheckPointReplicationSlots(flags & CHECKPOINT_IS_SHUTDOWN);
    CheckPointSnapBuild();                 // 逻辑复制快照构建
    CheckPointLogicalRewriteHeap();        // 逻辑复制重写堆
    CheckPointReplicationOrigin();         // 复制源

    // 第 2 部分: 刷写 SLRU 和缓冲池
    // xlog.c:7509-7516
    TRACE_POSTGRESQL_BUFFER_CHECKPOINT_START(flags);
    CheckpointStats.ckpt_write_t = GetCurrentTimestamp();

    CheckPointCLOG();                      // 事务提交状态
    CheckPointCommitTs();                  // 提交时间戳
    CheckPointSUBTRANS();                  // 子事务
    CheckPointMultiXact();                 // 多事务 ID
    CheckPointPredicate();                 // 谓词锁（可串行化隔离）
    CheckPointBuffers(flags);              // ★核心：刷写共享缓冲池★

    // 第 3 部分: 批量 fsync
    // xlog.c:7519-7523
    TRACE_POSTGRESQL_BUFFER_CHECKPOINT_SYNC_START();
    CheckpointStats.ckpt_sync_t = GetCurrentTimestamp();
    ProcessSyncRequests();                 // 处理所有 fsync 请求
    CheckpointStats.ckpt_sync_end_t = GetCurrentTimestamp();
    TRACE_POSTGRESQL_BUFFER_CHECKPOINT_DONE();

    // 第 4 部分: 两阶段事务
    // xlog.c:7526
    CheckPointTwoPhase(checkPointRedo);
}
```

### 2.2 各步骤详解

#### 2.2.1 CheckPointRelationMap()

```c
// 作用: 刷写 pg_filenode.map 文件
// 位置: src/backend/utils/cache/relmapper.c

// pg_filenode.map 内容:
// - 系统表的 relfilenode 映射
// - 例如: pg_class -> 1259

// 为什么需要 checkpoint:
// - 某些系统表可能被 VACUUM FULL 或 CLUSTER 重建
// - relfilenode 会改变，需要更新映射
```

#### 2.2.2 CheckPointReplicationSlots()

```c
// 作用: 持久化复制槽状态
// 位置: src/backend/replication/slot.c

// 复制槽信息:
// - slot_name
// - restart_lsn (需要保留的最老 WAL)
// - confirmed_flush_lsn (逻辑复制确认位置)

// Shutdown checkpoint:
// - 保存所有槽的状态到磁盘 (pg_replslot/)
// - 确保重启后槽信息不丢失
```

#### 2.2.3 CheckPointCLOG() 等 SLRU

```c
// SLRU: Simple Least Recently Used
// 一种简单的缓存机制，用于事务状态等小数据

// CheckPointCLOG() - pg_xact/
// - 每个事务 2 位: committed, aborted, in-progress, sub-committed
// - 每个 SLRU 页 8KB，可容纳 32K 个事务

// CheckPointCommitTs() - pg_commit_ts/
// - 每个事务的提交时间戳 (如果 track_commit_timestamp = on)

// CheckPointSUBTRANS() - pg_subtrans/
// - 子事务的父事务 ID

// CheckPointMultiXact() - pg_multixact/
// - 多事务 ID 映射 (用于共享行锁)

// 实现: 遍历 SLRU 缓冲区，写出所有脏页
```

#### 2.2.4 CheckPointBuffers()

```c
void CheckPointBuffers(int flags)
{
    // xlog.c:7442
    CheckpointStats.ckpt_write_t = GetCurrentTimestamp();

    // 调用核心函数
    BufferSync(flags);

    // 记录统计
    CheckpointStats.ckpt_sync_t = GetCurrentTimestamp();
}
```

详见下一节 BufferSync 分析。

#### 2.2.5 ProcessSyncRequests()

详见第 4 节。

#### 2.2.6 CheckPointTwoPhase()

```c
// 作用: 持久化两阶段事务状态
// 位置: src/backend/access/transam/twophase.c

// 两阶段提交流程:
// 1. PREPARE TRANSACTION 'txn1';
//    - 创建状态文件 pg_twophase/XXXXXX
// 2. 系统崩溃或重启
// 3. 恢复时读取状态文件
// 4. COMMIT/ROLLBACK PREPARED 'txn1';

// Checkpoint 时:
// - 确保所有 prepared 事务的状态已写入磁盘
// - 在 checkpoint 记录中记录这些事务
// - 恢复时能够正确处理
```

---

## 3. BufferSync 实现分析

**位置**: `src/backend/storage/buffer/bufmgr.c:2901`

### 3.1 完整实现（带详细注释）

```c
void
BufferSync(int flags)
{
    uint32      buf_state;
    int         buf_id;
    int         num_to_scan;
    int         num_spaces;
    int         num_processed;
    int         num_written;
    CkptTsStatus *per_ts_stat = NULL;
    Oid         last_tsid;
    binaryheap *ts_heap;
    int         i;
    int         mask = BM_DIRTY;
    WritebackContext wb_context;

    // ============ 阶段 1: 扫描并标记脏页 ============
    // bufmgr.c:2916-2923

    // 确定需要刷写的缓冲区类型
    if (!((flags & (CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_END_OF_RECOVERY |
                    CHECKPOINT_FLUSH_ALL))))
        mask |= BM_PERMANENT;  // 常规 checkpoint: 只刷写永久表

    // bufmgr.c:2941-2971
    num_to_scan = 0;
    for (buf_id = 0; buf_id < NBuffers; buf_id++)
    {
        BufferDesc *bufHdr = GetBufferDescriptor(buf_id);

        // 使用 header spinlock 检查 BM_DIRTY
        buf_state = LockBufHdr(bufHdr);

        // 检查是否匹配条件
        if ((buf_state & mask) == mask)
        {
            CkptSortItem *item;

            // 设置 BM_CHECKPOINT_NEEDED 标志
            buf_state |= BM_CHECKPOINT_NEEDED;

            // 记录缓冲区信息用于排序
            item = &CkptBufferIds[num_to_scan++];
            item->buf_id = buf_id;
            item->tsId = bufHdr->tag.spcOid;        // 表空间 OID
            item->relNumber = BufTagGetRelNumber(&bufHdr->tag);  // 关系编号
            item->forkNum = BufTagGetForkNum(&bufHdr->tag);      // Fork 类型
            item->blockNum = bufHdr->tag.blockNum;               // 块号
        }

        UnlockBufHdr(bufHdr, buf_state);

        // 检查 barrier 事件（NBuffers 可能很大）
        if (ProcSignalBarrierPending)
            ProcessProcSignalBarrier();
    }

    if (num_to_scan == 0)
        return;  // 没有脏页

    // ============ 阶段 2: 排序 ============
    // bufmgr.c:2976-2987

    WritebackContextInit(&wb_context, &checkpoint_flush_after);

    TRACE_POSTGRESQL_BUFFER_SYNC_START(NBuffers, num_to_scan);

    // 排序: (表空间, 关系, fork, 块号)
    // 目的: 将随机 I/O 转换为顺序 I/O
    sort_checkpoint_bufferids(CkptBufferIds, num_to_scan);

    // ============ 阶段 3: 按表空间分组 ============
    // bufmgr.c:2989-3052

    num_spaces = 0;
    last_tsid = InvalidOid;

    for (i = 0; i < num_to_scan; i++)
    {
        CkptTsStatus *s;
        Oid cur_tsid;

        cur_tsid = CkptBufferIds[i].tsId;

        // 遇到新的表空间？
        if (last_tsid == InvalidOid || last_tsid != cur_tsid)
        {
            Size sz;

            num_spaces++;

            // 动态扩展 per_ts_stat 数组
            sz = sizeof(CkptTsStatus) * num_spaces;
            if (per_ts_stat == NULL)
                per_ts_stat = (CkptTsStatus *) palloc(sz);
            else
                per_ts_stat = (CkptTsStatus *) repalloc(per_ts_stat, sz);

            s = &per_ts_stat[num_spaces - 1];
            memset(s, 0, sizeof(*s));
            s->tsId = cur_tsid;
            s->index = i;  // 在 CkptBufferIds 中的起始索引

            last_tsid = cur_tsid;
        }
        else
        {
            s = &per_ts_stat[num_spaces - 1];
        }

        s->num_to_scan++;  // 该表空间的缓冲区计数

        if (ProcSignalBarrierPending)
            ProcessProcSignalBarrier();
    }

    Assert(num_spaces > 0);

    // ============ 阶段 4: 构建最小堆 ============
    // bufmgr.c:3061-3074

    // 创建堆用于表空间负载均衡
    ts_heap = binaryheap_allocate(num_spaces,
                                   ts_ckpt_progress_comparator,
                                   NULL);

    for (i = 0; i < num_spaces; i++)
    {
        CkptTsStatus *ts_stat = &per_ts_stat[i];

        // 计算单个缓冲区对该表空间进度的贡献
        // progress_slice 越大，表示该表空间缓冲区越少
        // 写一个缓冲区，进度增加越多
        ts_stat->progress_slice = (float8) num_to_scan / ts_stat->num_to_scan;

        binaryheap_add_unordered(ts_heap, PointerGetDatum(ts_stat));
    }

    binaryheap_build(ts_heap);

    // ============ 阶段 5: 平衡写入循环 ============
    // bufmgr.c:3082-3144

    num_processed = 0;
    num_written = 0;

    while (!binaryheap_empty(ts_heap))
    {
        BufferDesc *bufHdr = NULL;
        CkptTsStatus *ts_stat = (CkptTsStatus *)
            DatumGetPointer(binaryheap_first(ts_heap));

        buf_id = CkptBufferIds[ts_stat->index].buf_id;
        Assert(buf_id != -1);

        bufHdr = GetBufferDescriptor(buf_id);

        num_processed++;

        // 检查是否仍需要写入
        // 注意: 不获取锁，只查看单个标志位
        // 可能的竞态: 其他进程可能已经写入并清除了标志
        // 这是可接受的，SyncOneBuffer 会再次检查
        if (pg_atomic_read_u32(&bufHdr->state) & BM_CHECKPOINT_NEEDED)
        {
            if (SyncOneBuffer(buf_id, false, &wb_context) & BUF_WRITTEN)
            {
                TRACE_POSTGRESQL_BUFFER_SYNC_WRITTEN(buf_id);
                PendingCheckpointerStats.buffers_written++;
                num_written++;
            }
        }

        // 更新该表空间的进度
        // 无论是否实际写入，都更新进度
        // 这样可以保持负载均衡
        ts_stat->progress += ts_stat->progress_slice;
        ts_stat->num_scanned++;
        ts_stat->index++;

        // 该表空间处理完了？
        if (ts_stat->num_scanned == ts_stat->num_to_scan)
        {
            binaryheap_remove_first(ts_heap);
        }
        else
        {
            // 更新堆（重新调整优先级）
            binaryheap_replace_first(ts_heap, PointerGetDatum(ts_stat));
        }

        // ★ I/O 限流 ★
        // 根据进度和 checkpoint_completion_target 决定是否休眠
        CheckpointWriteDelay(flags, (double) num_processed / num_to_scan);
    }

    // ============ 阶段 6: 发出待处理的 writeback ============
    // bufmgr.c:3150

    IssuePendingWritebacks(&wb_context, IOCONTEXT_NORMAL);

    // 清理
    pfree(per_ts_stat);
    per_ts_stat = NULL;
    binaryheap_free(ts_heap);

    // 更新统计
    CheckpointStats.ckpt_bufs_written += num_written;

    TRACE_POSTGRESQL_BUFFER_SYNC_DONE(NBuffers, num_written, num_to_scan);
}
```

### 3.2 SyncOneBuffer 实现

```c
// bufmgr.c:3262
static int
SyncOneBuffer(int buf_id, bool skip_recently_used, WritebackContext *wb_context)
{
    BufferDesc *bufHdr = GetBufferDescriptor(buf_id);
    int         result = 0;
    uint32      buf_state;
    BufferTag   tag;

    ReserveAuxProcessResourceOwner();

    // 获取 buffer header 锁
    buf_state = LockBufHdr(bufHdr);

    // 再次检查是否需要刷写（可能已被其他进程处理）
    if (!(buf_state & BM_CHECKPOINT_NEEDED))
    {
        UnlockBufHdr(bufHdr, buf_state);
        UnreserveAuxProcessResourceOwner();
        return result;
    }

    // 如果设置了 skip_recently_used 且缓冲区最近被使用，跳过
    if (skip_recently_used && BUF_STATE_GET_USAGECOUNT(buf_state) > 0)
    {
        UnlockBufHdr(bufHdr, buf_state);
        UnreserveAuxProcessResourceOwner();
        return result;
    }

    // Pin 住缓冲区（防止被替换）
    PinBuffer_Locked(bufHdr);

    // 复制 tag（在锁外使用）
    tag = bufHdr->tag;

    // 清除 BM_CHECKPOINT_NEEDED 标志
    buf_state &= ~BM_CHECKPOINT_NEEDED;

    UnlockBufHdr(bufHdr, buf_state);

    // ★ 在锁外执行 I/O ★
    // 这是关键优化：锁持有时间最小化
    if (buf_state & BM_DIRTY)
    {
        // 刷写缓冲区到磁盘
        FlushBuffer(bufHdr, NULL, IOOBJECT_RELATION, IOCONTEXT_NORMAL);
        result |= BUF_WRITTEN;

        // 发出 writeback hint
        ScheduleBufferTagForWriteback(wb_context, IOOBJECT_RELATION, &tag);
    }

    // Unpin
    UnpinBuffer(bufHdr);

    UnreserveAuxProcessResourceOwner();

    return result;
}
```

### 3.3 负载均衡示例

```
假设 3 个表空间:
  ts1 (pg_default):  1000 个脏页
  ts2 (ts_data):     500 个脏页
  ts3 (ts_index):    300 个脏页
  总计:              1800 个脏页

progress_slice 计算:
  ts1: 1800 / 1000 = 1.8
  ts2: 1800 / 500 = 3.6
  ts3: 1800 / 300 = 6.0

写入过程 (简化):

步骤  选择   progress  操作
----  -----  --------  ----------------
 1    ts1    0.0       写 ts1[0], progress = 1.8
 2    ts2    0.0       写 ts2[0], progress = 3.6
 3    ts1    1.8       写 ts1[1], progress = 3.6
 4    ts2    3.6       写 ts2[1], progress = 7.2
 5    ts1    3.6       写 ts1[2], progress = 5.4
 6    ts3    0.0       写 ts3[0], progress = 6.0
 7    ts1    5.4       写 ts1[3], progress = 7.2
 8    ts3    6.0       写 ts3[1], progress = 12.0
 9    ts2    7.2       写 ts2[2], progress = 10.8
...

观察:
- 三个表空间的写入交替进行
- ts3 虽然页面少，但 progress_slice 大，不会落后太多
- 实现了负载均衡
```

---

## 4. ProcessSyncRequests 处理

**位置**: `src/backend/storage/sync/sync.c:183`

### 4.1 实现概述

```c
void
ProcessSyncRequests(void)
{
    AbsorbSyncRequests();  // 从共享队列吸收请求

    ProcessSyncRequestsInternal();  // 实际处理
}

static void
ProcessSyncRequestsInternal(void)
{
    // 遍历所有已记住的 sync 请求
    // 按 (relfilenode, fork, block) 排序
    // 批量调用 fsync()

    // 记录最长的 fsync 时间（用于日志）
    // 累计总 fsync 时间

    // 详细实现涉及多个数据结构，这里简化描述
}
```

### 4.2 Fsync 请求流程

```
1. 后端写脏页:
   Backend -> FlushBuffer()
           -> ForwardSyncRequest(ftag, SYNC_REQUEST)
           -> CheckpointerShmem->requests[n] = {ftag, SYNC_REQUEST}

2. Checkpointer 吸收:
   Checkpointer -> AbsorbSyncRequests()
                -> 复制 requests[] 数组
                -> CheckpointerShmem->num_requests = 0
                -> RememberSyncRequest(ftag, SYNC_REQUEST)
                   (内部哈希表)

3. Checkpoint 时处理:
   Checkpointer -> ProcessSyncRequests()
                -> ProcessSyncRequestsInternal()
                -> 排序所有请求
                -> 批量 fsync()
```

---

## 5. 关键同步点和锁

### 5.1 锁层次结构

```
锁类型              作用域                        持有时间
----------------    ---------------------------    -------------
Spinlock            很小的数据结构（几个字节）      微秒级
LWLock (Shared)     读共享数据                     毫秒级
LWLock (Exclusive)  修改共享数据                   毫秒级
LWLock + I/O        持有锁期间做 I/O              秒级 (避免!)
```

### 5.2 CreateCheckPoint 中的锁

| 锁名称 | 类型 | 位置 | 作用 | 持有时间 |
|--------|------|------|------|----------|
| **WALInsertLocks[]** | LWLock Exclusive (所有) | xlog.c:6993 | 阻止并发 WAL 插入，确定 redo 点 | ~1ms |
| **ckpt_lck** | Spinlock | checkpointer.c:397 | 保护 checkpoint 计数器 | ~1μs |
| **ControlFileLock** | LWLock Exclusive | xlog.c:7244 | 更新 pg_control 文件 | ~10ms |
| **info_lck** | Spinlock | xlog.c:7065 | 更新 XLogCtl 的小字段 | ~1μs |

### 5.3 BufferSync 中的锁

| 锁名称 | 类型 | 作用 | 持有时间 |
|--------|------|------|----------|
| **bufHdr spinlock** | Spinlock | 检查/设置 buffer 状态标志 | ~1μs |
| **buffer content_lock** | LWLock Shared | SyncOneBuffer 中读取数据 | ~100μs |
| **io_in_progress_lock** | LWLock Exclusive | 执行 I/O 时 | ~1ms |

### 5.4 临界区 (Critical Section)

```c
// 临界区内的错误会导致 PANIC，强制重启数据库

// CreateCheckPoint 中的临界区:
START_CRIT_SECTION();
    // 1. 更新 ControlFile->state
    UpdateControlFile();
END_CRIT_SECTION();

// ... 可能失败的 I/O 操作在临界区外 ...

START_CRIT_SECTION();
    // 2. 写 checkpoint WAL 记录
    XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT_*);
    XLogFlush(recptr);

    // 3. 更新 pg_control
    UpdateControlFile();
END_CRIT_SECTION();

// 为什么需要临界区？
// - 确保关键操作的原子性
// - 如果部分完成后崩溃，数据库状态会不一致
// - PANIC 可以回滚到之前的一致状态
```

---

## 6. 时序分析

### 6.1 完整时序图

```
时间 (相对)  Checkpointer              WAL               Backends
----------  -----------------------  ----------------  ------------------
T+0s        CheckpointerMain()
            等待触发...

T+300s      检测到 checkpoint_timeout
            CreateCheckPoint(CAUSE_TIME)

T+300.001s  START_CRIT_SECTION()
            ControlFile->state =
              DB_SHUTDOWNING (如果shutdown)

T+300.002s  WALInsertLockAcquireExclusive()
                                                      Backend 尝试插入WAL
                                                      -> 阻塞在 WAL 插入锁

T+300.003s  确定 redo 点
            插入 REDO 记录  ------>  [REDO记录]
            WALInsertLockRelease()
                                                      Backend 继续插入 WAL

T+300.010s  收集元数据 (XID, OID...)
            END_CRIT_SECTION()

T+300.020s  等待事务临界区
            GetVirtualXIDsDelaying...
                                                      Backend 在 commit 临界区
T+300.030s  (等待)
T+300.040s  事务完成临界区        <-----------------  Backend 退出临界区

T+300.050s  CheckPointGuts()
            - CheckPointCLOG()
            - CheckPointCommitTs()
            - CheckPointSUBTRANS()
            - CheckPointMultiXact()

T+300.100s  CheckPointBuffers()
            BufferSync() 开始

T+300.101s  扫描所有 buffer
            标记 10000 个脏页

T+300.150s  排序 10000 个项

T+300.200s  开始写入循环
            写 buffer[0]
            CheckpointWriteDelay()
            -> 检查进度
            -> 休眠 100ms (如果超前)

T+300.300s  写 buffer[1]
            ...

T+330.000s  写 buffer[9999]
            (共 30 秒，符合 completion_target=0.9)

T+330.050s  IssuePendingWritebacks()

T+330.100s  ProcessSyncRequests()
            批量 fsync()

T+332.000s  CheckPointTwoPhase()

T+332.050s  等待延迟完成事务

T+332.100s  LogStandbySnapshot()

T+332.150s  START_CRIT_SECTION()
            XLogInsert(CHECKPOINT_ONLINE)  ->  [Checkpoint记录]
            XLogFlush()

T+332.200s  UpdateControlFile()
            - checkPoint = LSN
            - checkPointCopy = checkPoint

T+332.250s  END_CRIT_SECTION()

T+332.300s  RemoveOldXlogFiles()
            PreallocXlogFiles()
            TruncateSUBTRANS()

T+332.500s  LogCheckpointEnd()
            "checkpoint complete: ..."

T+332.501s  返回 CheckpointerMain
            继续等待下一次触发

总耗时: 32.5 秒
其中: 写脏页 30 秒, fsync 2 秒, 其他 0.5 秒
```

### 6.2 性能分析

**典型耗时分布** (以 32.5 秒总时间为例):

| 阶段 | 耗时 | 占比 | 说明 |
|------|------|------|------|
| 准备阶段 | 0.05s | 0.2% | 很快 |
| 确定 REDO 点 | 0.01s | 0.03% | 几乎瞬间 |
| 收集元数据 | 0.02s | 0.06% | 快速 |
| 等待事务临界区 | 0.02s | 0.06% | 通常很快 |
| SLRU checkpoint | 0.05s | 0.15% | 快速 |
| **BufferSync 写脏页** | **30s** | **92%** | ★最耗时★ |
| **ProcessSyncRequests** | **2s** | **6%** | fsync 慢 |
| 两阶段事务 | 0.05s | 0.15% | 快速 |
| 写 WAL 记录 | 0.1s | 0.3% | 快速 |
| 更新 pg_control | 0.05s | 0.15% | 快速 |
| 后续清理 | 0.15s | 0.5% | 快速 |

**优化重点**:
1. BufferSync - 92% 时间
   - 通过 checkpoint_completion_target 平滑
   - 排序减少随机 I/O
   - 负载均衡避免 I/O 热点

2. ProcessSyncRequests - 6% 时间
   - 批量 fsync 减少系统调用
   - 排序优化 fsync 顺序

---

## 7. 错误处理和恢复

### 7.1 错误类型

```c
// 1. 临界区内错误 -> PANIC
START_CRIT_SECTION();
    if (write(fd, data, size) < 0)
        ereport(PANIC, "could not write control file");
END_CRIT_SECTION();
// PANIC: 立即终止所有进程，数据库重启进入恢复

// 2. 临界区外错误 -> ERROR
END_CRIT_SECTION();
if (some_operation() < 0)
    ereport(ERROR, "checkpoint operation failed");
// ERROR: checkpoint 失败，但数据库继续运行

// 3. Checkpointer 进程错误处理
sigsetjmp() {
    // 错误恢复点

    // 清理资源
    LWLockReleaseAll();
    UnlockBuffers();

    // 通知等待的后端 checkpoint 失败
    SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
    CheckpointerShmem->ckpt_failed++;
    CheckpointerShmem->ckpt_done = CheckpointerShmem->ckpt_started;
    SpinLockRelease(&CheckpointerShmem->ckpt_lck);

    ConditionVariableBroadcast(&CheckpointerShmem->done_cv);

    // 休眠 1 秒后重试
    pg_usleep(1000000L);
}
```

### 7.2 Checkpoint 失败的后果

```
场景 1: Checkpoint 在 BufferSync 中失败 (I/O 错误)
-------------------------------------------------------
后果:
- 部分脏页已写入磁盘
- Checkpoint WAL 记录未写入
- pg_control 未更新

恢复:
- 数据库继续运行
- 已写入的脏页是安全的 (有对应的 WAL)
- 下次 checkpoint 会再次尝试

场景 2: Checkpoint 在临界区内失败
-------------------------------------------------------
后果:
- PANIC, 所有进程被杀死
- 数据库重启进入恢复模式

恢复:
- 从最后一个成功的 checkpoint 的 redo 点开始
- 重放所有后续 WAL
- 重做所有已提交的事务

场景 3: 更新 pg_control 时系统崩溃
-------------------------------------------------------
后果:
- pg_control 可能部分写入 (但因单扇区原子性，这很罕见)

恢复:
- 启动时检测 pg_control CRC 失败
- 如果有备份控制文件，使用备份
- 否则，数据库无法启动 (需要 pg_resetwal)
```

### 7.3 恢复时使用 Checkpoint

```c
// StartupXLOG() 恢复流程:

1. 读取 pg_control
   ControlFile = ReadControlFile();
   CheckPoint checkPoint = ControlFile->checkPointCopy;

2. 定位 checkpoint 记录
   XLogBeginRead(checkPoint.LSN);
   record = XLogReadRecord();

3. 验证一致性
   if (memcmp(&record->CheckPoint, &checkPoint, sizeof(CheckPoint)) != 0)
       ereport(PANIC, "checkpoint record mismatch");

4. 从 redo 点开始恢复
   RedoRecPtr = checkPoint.redo;
   XLogBeginRead(RedoRecPtr);

5. 重放 WAL
   while ((record = XLogReadRecord()) != NULL)
   {
       RmgrTable[record->xl_rmid].rm_redo(record);

       // 更新恢复进度
       ereport(LOG, "redo at %X/%X", LSN_FORMAT_ARGS(record->LSN));
   }

6. 恢复完成
   ereport(LOG, "redo done at %X/%X", LSN_FORMAT_ARGS(EndOfLog));
```

---

## 总结

本文档详细分析了 PostgreSQL Checkpoint 的实现流程：

1. **CreateCheckPoint** - 主流程 13 个阶段
2. **CheckPointGuts** - 核心刷写操作
3. **BufferSync** - 最耗时的缓冲区同步
4. **ProcessSyncRequests** - 批量 fsync
5. **锁和同步** - 多层次的并发控制
6. **时序分析** - 典型 checkpoint 的时间分布
7. **错误处理** - 不同错误的影响和恢复

通过源码级别的分析，我们看到了 PostgreSQL 如何在保证数据一致性和系统可用性之间取得平衡。

---

**下一步**: 阅读 `04_key_algorithms.md` 深入理解关键算法。
