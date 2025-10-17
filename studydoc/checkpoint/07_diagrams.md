# PostgreSQL Checkpoint - 可视化图表集

## 目录

1. [架构图](#1-架构图)
   - 1.1 [PostgreSQL 进程架构](#11-postgresql-进程架构)
   - 1.2 [Checkpoint 相关内存布局](#12-checkpoint-相关内存布局)
   - 1.3 [表空间和文件系统布局](#13-表空间和文件系统布局)

2. [流程图](#2-流程图)
   - 2.1 [CreateCheckPoint 主流程](#21-createcheckpoint-主流程)
   - 2.2 [BufferSync 流程](#22-buffersync-流程)
   - 2.3 [CheckpointWriteDelay 调度流程](#23-checkpointwritedelay-调度流程)
   - 2.4 [SLRU Checkpoint 流程](#24-slru-checkpoint-流程)
   - 2.5 [WAL 日志写入流程](#25-wal-日志写入流程)

3. [状态转换图](#3-状态转换图)
   - 3.1 [Checkpoint 状态机](#31-checkpoint-状态机)
   - 3.2 [缓冲区状态转换](#32-缓冲区状态转换)
   - 3.3 [WAL 段文件生命周期](#33-wal-段文件生命周期)

4. [数据流图](#4-数据流图)
   - 4.1 [写事务数据流](#41-写事务数据流)
   - 4.2 [Checkpoint 数据流](#42-checkpoint-数据流)
   - 4.3 [崩溃恢复数据流](#43-崩溃恢复数据流)

5. [时序图](#5-时序图)
   - 5.1 [正常 Checkpoint 时序](#51-正常-checkpoint-时序)
   - 5.2 [渐进式 Checkpoint I/O 模式](#52-渐进式-checkpoint-io-模式)
   - 5.3 [并发事务与 Checkpoint 交互](#53-并发事务与-checkpoint-交互)

6. [算法可视化](#6-算法可视化)
   - 6.1 [Min-heap 负载均衡算法](#61-min-heap-负载均衡算法)
   - 6.2 [IsCheckpointOnSchedule 判断逻辑](#62-ischeckpointonschedule-判断逻辑)
   - 6.3 [Fsync Request Queue 压缩](#63-fsync-request-queue-压缩)

7. [监控视图](#7-监控视图)
   - 7.1 [pg_stat_bgwriter 指标关系](#71-pg_stat_bgwriter-指标关系)
   - 7.2 [Checkpoint 性能仪表板](#72-checkpoint-性能仪表板)

---

## 1. 架构图

### 1.1 PostgreSQL 进程架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PostgreSQL 实例                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
│  │  Backend   │  │  Backend   │  │  Backend   │  │  Backend   │   │
│  │ Process #1 │  │ Process #2 │  │ Process #3 │  │ Process #N │   │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘   │
│        │               │               │               │           │
│        │ write data    │ write data    │ write data    │           │
│        ↓               ↓               ↓               ↓           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Shared Memory                             │   │
│  ├──────────────────────────────────────────────────────────────┤  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │   │
│  │  │Shared Buffers│  │  WAL Buffers │  │ SLRU Buffers │      │   │
│  │  │   (数据页)    │  │  (WAL记录)   │  │  (事务状态)  │      │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │   │
│  │                                                               │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │      CheckpointerShmemStruct (控制结构)              │   │   │
│  │  │  • pid, slock_t, ConditionVariable                   │   │   │
│  │  │  • ckpt_started, ckpt_done, ckpt_failed              │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  └───────────────────────────────────────────────────────────────┘  │
│        ↑               ↑               ↑               ↑           │
│        │ read/write    │ flush         │ flush         │ flush     │
│        │               │               │               │           │
│  ┌─────┴──────┐  ┌─────┴──────┐  ┌─────┴──────┐  ┌─────┴──────┐   │
│  │Checkpointer│  │ WAL Writer │  │  BgWriter  │  │ AutoVacuum │   │
│  │  Process   │  │  Process   │  │  Process   │  │  Launcher  │   │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └────────────┘   │
│        │               │               │                           │
└────────┼───────────────┼───────────────┼───────────────────────────┘
         │               │               │
         ↓               ↓               ↓
    ┌─────────┐     ┌─────────┐     ┌─────────┐
    │  Data   │     │ pg_wal/ │     │  Data   │
    │  Files  │     │(WAL日志)│     │  Files  │
    └─────────┘     └─────────┘     └─────────┘
         ↑                                ↑
         └────────────────┬───────────────┘
                          │
                  Checkpoint 将脏页
                  从内存刷到磁盘
```

**说明**:
- **Backend 进程**: 处理客户端连接，执行 SQL，生成脏页和 WAL
- **Checkpointer 进程**: 定期或按需将脏页刷盘，确保数据持久化
- **WAL Writer 进程**: 异步刷新 WAL Buffer 到磁盘
- **BgWriter 进程**: 预先清理部分脏页，减轻 checkpoint 负担
- **Shared Memory**: 所有进程共享的内存区域
  - Shared Buffers: 缓存数据页（最大可达几十GB）
  - WAL Buffers: 缓存 WAL 记录（通常几MB）
  - SLRU Buffers: 缓存事务状态（CLOG、SubTrans等）

**关键交互**:
1. Backend 写入数据到 Shared Buffers（产生脏页）
2. Backend 写入 WAL 到 WAL Buffers（先写日志）
3. Checkpointer 定期扫描 Shared Buffers，刷写脏页
4. WAL Writer 异步刷新 WAL Buffer

---

### 1.2 Checkpoint 相关内存布局

```
┌──────────────────────────────────────────────────────────────────┐
│                    PostgreSQL Shared Memory                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  CheckpointerShmemStruct (checkpointer.c:173)            │   │
│  ├───────────────────────────────────────────────────────────┤   │
│  │  pid_t checkpointer_pid;      ← Checkpointer 进程 PID    │   │
│  │  slock_t ckpt_lck;            ← 自旋锁保护计数器        │   │
│  │  int ckpt_started;            ← 已开始的 checkpoint 计数 │   │
│  │  int ckpt_done;               ← 已完成的 checkpoint 计数 │   │
│  │  int ckpt_failed;             ← 失败的 checkpoint 计数   │   │
│  │  ConditionVariable start_cv;  ← 开始信号量              │   │
│  │  ConditionVariable done_cv;   ← 完成信号量              │   │
│  │  CheckpointerRequest requests[MAXBACKENDS]; ← 请求队列  │   │
│  └───────────────────────────────────────────────────────────┘   │
│                         ↑                                         │
│                         │ Backend 请求 checkpoint                │
│                         │                                         │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  XLogCtlData (xlog.c:500+)                               │   │
│  ├───────────────────────────────────────────────────────────┤   │
│  │  XLogRecPtr LogwrtResult.Write;  ← 最后写入的 LSN       │   │
│  │  XLogRecPtr LogwrtResult.Flush;  ← 最后刷盘的 LSN       │   │
│  │  pg_atomic_uint64 logInsertResult; ← 插入位置原子变量   │   │
│  │  WALInsertLockPadded locks[NUM_XLOGINSERT_LOCKS];       │   │
│  │  XLogRecPtr lastCheckPoint;      ← 上次 checkpoint 位置 │   │
│  │  CheckPoint lastCheckPointCopy;  ← 上次 checkpoint 副本 │   │
│  └───────────────────────────────────────────────────────────┘   │
│                         ↑                                         │
│                         │ 记录 WAL 和 Checkpoint 状态             │
│                         │                                         │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  BufferDescriptors[] (bufmgr.c:120+)                     │   │
│  ├───────────────────────────────────────────────────────────┤   │
│  │  [0] ┌─────────────────────┐                             │   │
│  │      │ tag (rel, fork, blk)│ ← 标识哪个页                │   │
│  │      │ flags (dirty, valid)│ ← 脏页标志                  │   │
│  │      │ usage_count         │ ← LRU 计数                  │   │
│  │      │ content_lock        │ ← 页内容锁                  │   │
│  │      │ buf_id              │ ← 缓冲区编号                │   │
│  │      └─────────────────────┘                             │   │
│  │  [1] ...                                                  │   │
│  │  [2] ...                                                  │   │
│  │  ...                                                      │   │
│  │  [N] (NBuffers 数量, 例如 128K 个)                        │   │
│  └───────────────────────────────────────────────────────────┘   │
│          ↑                                                        │
│          │ Checkpointer 扫描此数组，找出所有脏页                │
│          │                                                        │
└──────────────────────────────────────────────────────────────────┘

通信流程:
──────────

Backend 请求 Checkpoint:
  1. 获取 ckpt_lck 自旋锁
  2. 递增 ckpt_started
  3. ConditionVariableSignal(&start_cv)
  4. 释放 ckpt_lck

Checkpointer 响应:
  1. ConditionVariableWait(&start_cv)
  2. 读取 ckpt_started (volatile 读取)
  3. 执行 CreateCheckPoint()
  4. 递增 ckpt_done
  5. ConditionVariableBroadcast(&done_cv)

Backend 等待完成:
  1. ConditionVariableWait(&done_cv)
  2. 检查 ckpt_done 是否增加
```

**说明**:
- **CheckpointerShmemStruct**: 进程间通信结构（位于 checkpointer.c:173）
  - 使用 **自旋锁** 保护计数器（短暂持有）
  - 使用 **ConditionVariable** 进行进程等待/唤醒（高效）

- **XLogCtlData**: WAL 控制结构（位于 xlog.c:500+）
  - 记录当前 WAL 写入/刷盘位置
  - 保存最后一次 checkpoint 的信息
  - 使用 **原子变量** 保证并发安全

- **BufferDescriptors**: 缓冲区描述符数组（位于 bufmgr.c:120+）
  - 每个元素对应一个 8KB 页
  - `flags & BM_DIRTY` 标识脏页
  - Checkpointer 顺序扫描此数组

**内存大小估算**:
- CheckpointerShmemStruct: ~1KB
- XLogCtlData: ~16KB
- BufferDescriptors: NBuffers × 64 bytes
  - 例如 shared_buffers=8GB → 1048576 个缓冲区 → ~64MB

---

### 1.3 表空间和文件系统布局

```
PostgreSQL 数据目录 ($PGDATA)
│
├── postgresql.conf          ← 配置文件
├── pg_hba.conf              ← 认证配置
├── postmaster.pid           ← 主进程 PID
│
├── global/                  ← 全局对象
│   ├── pg_control           ← ⭐ 关键: Checkpoint 元数据
│   │   (ControlFileData 结构，记录最后 checkpoint 位置、时间等)
│   ├── pg_filenode.map
│   └── [OID 文件...]
│
├── base/                    ← 数据库目录
│   ├── 1/                   ← template1
│   ├── 13456/               ← postgres
│   │   ├── 16384            ← 表 1 (OID=16384)
│   │   ├── 16385            ← 表 2
│   │   ├── 16385_fsm        ← Free Space Map
│   │   ├── 16385_vm         ← Visibility Map
│   │   └── ...
│   └── 16387/               ← 用户数据库
│       └── ...
│
├── pg_wal/                  ← ⭐ WAL 日志目录
│   ├── 000000010000000000000001   ← WAL 段文件 (16MB)
│   ├── 000000010000000000000002
│   ├── 000000010000000000000003
│   ├── ...
│   └── archive_status/      ← 归档状态
│       ├── 000000010000000000000001.done
│       └── ...
│
├── pg_xact/                 ← ⭐ 事务状态 (CLOG)
│   ├── 0000                 ← CLOG 段文件
│   ├── 0001
│   └── ...
│
├── pg_subtrans/             ← 子事务状态
├── pg_multixact/            ← 多事务状态
│   ├── members/
│   └── offsets/
│
├── pg_commit_ts/            ← 提交时间戳
│
└── pg_stat/                 ← 统计信息目录


Checkpoint 涉及的文件操作:
────────────────────────────

1. 扫描 Shared Buffers (内存):
   ┌────────────┐
   │ 脏页列表   │ → 按 (tablespace, relfilenode, fork, blocknum) 排序
   └────────────┘

2. 按表空间分组写入 (磁盘):
   base/13456/16384  ← 表空间 1, 表 A
   base/13456/16385  ← 表空间 1, 表 B
   base/16387/16400  ← 表空间 2, 表 C

   每写入一批，调用 fsync() 确保持久化

3. 刷新 SLRU (磁盘):
   pg_xact/0000      ← SimpleLruFlush(CLOG)
   pg_subtrans/0000  ← SimpleLruFlush(SubTrans)
   pg_multixact/...  ← SimpleLruFlush(MultiXact)
   pg_commit_ts/...  ← SimpleLruFlush(CommitTs)

4. 写入 Checkpoint 记录到 WAL (磁盘):
   pg_wal/000000010000000000000005

   记录格式:
   ┌──────────────────────────────────────┐
   │ XLogRecord Header                    │
   ├──────────────────────────────────────┤
   │ CheckPoint 结构:                     │
   │   redo = 0/5000028                   │
   │   ThisTimeLineID = 1                 │
   │   nextXid = 1234                     │
   │   oldestXid = 1                      │
   │   ...                                │
   └──────────────────────────────────────┘

5. 更新 pg_control (磁盘):
   global/pg_control

   更新内容:
   ┌──────────────────────────────────────┐
   │ ControlFileData:                     │
   │   checkPoint = 0/5000100 (新的LSN)  │
   │   checkPointCopy = [CheckPoint结构] │
   │   state = DB_IN_PRODUCTION          │
   │   time = 2025-01-15 10:30:00        │
   │   ...                                │
   └──────────────────────────────────────┘

   ⚠️ 关键操作: pg_fsync(ControlFile)
      必须确保 pg_control 持久化，否则崩溃后无法恢复

文件系统调用序列:
───────────────

write() → write() → ... → fsync()  (数据文件)
write() → write() → ... → fsync()  (SLRU)
write() → fsync()                   (WAL)
write() → fsync()                   (pg_control) ← 最后一步
```

**说明**:
- **pg_control**: 最关键的文件，记录最后一次 checkpoint 位置
  - 大小: 8KB (单页，原子写入)
  - 启动时读取，确定从哪个 LSN 开始 REDO
  - 每次 checkpoint 必须更新并 fsync

- **pg_wal/**: WAL 日志文件
  - 每个文件 16MB（默认）
  - Checkpoint 后，REDO 点之前的 WAL 可以删除或归档
  - 命名格式: `TimeLineID` + `LogId` + `LogSeg`

- **base/**: 用户数据文件
  - 每个表对应一个或多个文件（每个文件最大 1GB）
  - Checkpoint 将脏页从内存刷到这些文件

- **pg_xact/**: CLOG（提交日志）
  - 每个 bit 表示一个事务的状态（进行中/已提交/已中止/子事务）
  - Checkpoint 时调用 `SimpleLruFlush(CLOG)` 刷盘

**崩溃恢复时的使用**:
1. 读取 `global/pg_control`，获取 `checkPoint` LSN
2. 从 `pg_wal/` 中找到对应的 WAL 段文件
3. 从 checkpoint LSN 开始重放 WAL，直到最新
4. 恢复完成后，执行新的 checkpoint

---

## 2. 流程图

### 2.1 CreateCheckPoint 主流程

```
CreateCheckPoint() 入口 (xlog.c:6881)
│
├─ 阶段 1: 初始化和准备
│  ├─ 获取 CheckpointLock (LWLock)
│  ├─ 设置 checkpoint 标志 (CHECKPOINT_IS_SHUTDOWN / CHECKPOINT_END_OF_RECOVERY 等)
│  ├─ 记录开始时间: ckpt_start_t = GetCurrentTimestamp()
│  └─ 初始化统计: checkpoints_timed++ 或 checkpoints_req++
│
├─ 阶段 2: 确定 REDO 点
│  ├─ 判断 checkpoint 类型?
│  │  ├─ SHUTDOWN / END_OF_RECOVERY?
│  │  │  └─ redo = 当前插入点 (GetInsertRecPtr())
│  │  │     理由: 不会再有新 WAL，所有事务已完成
│  │  │
│  │  └─ ONLINE checkpoint?
│  │     └─ redo = 当前写入点 (GetRedoRecPtr())
│  │        理由: 保守策略，使用已刷盘的 WAL 位置
│  │
│  └─ 记录 REDO 点: checkpoint.redo = redo;
│
├─ 阶段 3: 等待正在进行的事务
│  ├─ 如果不是 CHECKPOINT_IMMEDIATE:
│  │  └─ 等待所有虚拟事务达到安全点
│  │     (VirtualXactLockTableWait)
│  │
│  └─ 如果是 SHUTDOWN:
│     └─ 等待所有真实事务提交/中止
│
├─ 阶段 4: 收集 Checkpoint 数据
│  ├─ checkpoint.ThisTimeLineID = ThisTimeLineID;
│  ├─ checkpoint.PrevTimeLineID = PrevTimeLineID;
│  ├─ checkpoint.nextXid = ShmemVariableCache->nextXid;
│  ├─ checkpoint.nextOid = ShmemVariableCache->nextOid;
│  ├─ checkpoint.nextMulti = MultiXactGetNextMXact();
│  ├─ checkpoint.oldestXid = ShmemVariableCache->oldestXid;
│  ├─ checkpoint.oldestXidDB = ShmemVariableCache->oldestXidDB;
│  ├─ checkpoint.oldestMulti = GetOldestMultiXactId();
│  ├─ checkpoint.oldestMultiDB = GetOldestMultiXactDB();
│  ├─ checkpoint.time = time(NULL);
│  └─ checkpoint.oldestActiveXid = GetOldestActiveTransactionId();
│
├─ 阶段 5: ⭐ 核心操作 - CheckPointGuts()
│  │  (占 checkpoint 总时间的 90%+)
│  │
│  ├─ CheckPointCLOG()          ← 刷写 CLOG (pg_xact/)
│  │  └─ SimpleLruFlush(CLOG, true)
│  │
│  ├─ CheckPointCommitTs()      ← 刷写提交时间戳
│  │  └─ SimpleLruFlush(CommitTs, true)
│  │
│  ├─ CheckPointSUBTRANS()      ← 刷写子事务状态
│  │  └─ SimpleLruFlush(SubTrans, true)
│  │
│  ├─ CheckPointMultiXact()     ← 刷写多事务状态
│  │  ├─ SimpleLruFlush(MultiXactOffset, true)
│  │  └─ SimpleLruFlush(MultiXactMember, true)
│  │
│  ├─ CheckPointPredicate()     ← 刷写谓词锁
│  │
│  ├─ CheckPointRelationMap()   ← 刷写 relfilenode 映射
│  │
│  ├─ CheckPointReplicationSlots() ← 刷写复制槽状态
│  │
│  ├─ CheckPointSnapBuild()     ← 刷写逻辑解码快照
│  │
│  ├─ CheckPointLogicalRewriteHeap() ← 刷写逻辑重写堆
│  │
│  ├─ CheckPointReplicationOrigin() ← 刷写复制源状态
│  │
│  └─ ⭐⭐ CheckPointBuffers()   ← 刷写所有脏页 (最耗时!)
│     └─ BufferSync() (详见 2.2)
│        耗时占比: ~92%
│        写入量: 可达数GB
│
├─ 阶段 6: 截断 SLRU 日志
│  ├─ TruncateCLOG(oldestXid)
│  ├─ TruncateCommitTs(oldestXid)
│  ├─ TruncateSUBTRANS(oldestXid)
│  └─ TruncateMultiXact(oldestMulti)
│     理由: 删除不再需要的旧事务状态文件
│
├─ 阶段 7: 准备写入 Checkpoint 记录
│  ├─ XLogBeginInsert()
│  ├─ XLogRegisterData(&checkpoint, sizeof(CheckPoint))
│  └─ 标记为 XLOG_CHECKPOINT_SHUTDOWN / XLOG_CHECKPOINT_ONLINE
│
├─ 阶段 8: 写入 WAL 记录
│  ├─ recptr = XLogInsert(RM_XLOG_ID, checkpoint_type)
│  ├─ XLogFlush(recptr)  ← 强制刷盘
│  └─ 记录: checkpoint 完成的 LSN 位置
│
├─ 阶段 9: 更新共享内存状态
│  ├─ XLogCtl->lastCheckPoint = recptr;
│  ├─ XLogCtl->lastCheckPointCopy = checkpoint;
│  └─ 更新统计: checkpoint_write_time, checkpoint_sync_time
│
├─ 阶段 10: ⭐ 更新 pg_control 文件
│  ├─ ControlFile->checkPoint = recptr;
│  ├─ ControlFile->checkPointCopy = checkpoint;
│  ├─ ControlFile->time = time(NULL);
│  ├─ ControlFile->state = DB_IN_PRODUCTION / DB_SHUTDOWNED;
│  ├─ UpdateControlFile()
│  │  └─ pg_fsync(ControlFile) ← ⚠️ 关键: 必须持久化
│  └─ 如果 pg_fsync 失败 → PANIC (数据库崩溃)
│
├─ 阶段 11: 处理 WAL 归档
│  ├─ XLogArchiveNotify() ← 通知归档进程
│  └─ 如果启用了 archive_mode:
│     └─ 归档 checkpoint 之前的 WAL 文件
│
├─ 阶段 12: 清理旧 WAL 文件
│  ├─ RemoveOldXlogFiles()
│  │  ├─ 计算需要保留的 WAL 数量
│  │  │  (基于 wal_keep_segments, replication slots)
│  │  └─ 删除或回收旧的 WAL 段文件
│  │
│  └─ 示例: 保留最近 3 个 checkpoint 的 WAL
│     删除: 000000010000000000000001 - 000000010000000000000015
│     保留: 000000010000000000000016 - 000000010000000000000020
│
├─ 阶段 13: 记录日志和统计
│  ├─ ereport(LOG, "checkpoint complete: ...")
│  │  输出内容:
│  │  - wrote N buffers (X%)
│  │  - Y WAL files added, Z removed, W recycled
│  │  - write=A.B s, sync=C.D s, total=E.F s
│  │  - distance=N kB, estimate=M kB
│  │
│  ├─ 更新 pg_stat_bgwriter 视图
│  │  ├─ checkpoints_timed 或 checkpoints_req +1
│  │  ├─ checkpoint_write_time += write_time
│  │  ├─ checkpoint_sync_time += sync_time
│  │  └─ buffers_checkpoint += num_written
│  │
│  └─ 释放 CheckpointLock
│
└─ 返回

执行时间分布 (典型场景):
────────────────────────
总耗时: 32.5 秒

阶段 1-4:   0.5s  (  1.5%)  ← 准备工作
阶段 5:    30.0s  ( 92.3%)  ← CheckPointGuts (主要是 BufferSync)
  其中:
  - BufferSync:  29.0s (89%)
  - SLRU Flush:   0.8s ( 2%)
  - 其他:         0.2s ( 1%)
阶段 6-9:   0.3s  (  0.9%)  ← WAL 写入和状态更新
阶段 10:    0.1s  (  0.3%)  ← pg_control fsync
阶段 11-13: 1.6s  (  5.0%)  ← 清理和日志
```

**关键点**:
1. **REDO 点决策**: SHUTDOWN checkpoint 可以更激进（当前插入点），ONLINE checkpoint 更保守（当前写入点）
2. **CheckPointGuts()**: 占用 90%+ 时间，核心是 BufferSync()
3. **pg_control 更新**: 最后且最关键的步骤，必须 fsync 成功
4. **渐进式执行**: 通过 CheckpointWriteDelay() 分散 I/O 压力

---

### 2.2 BufferSync 流程

```
BufferSync(int flags) 入口 (bufmgr.c:2901)
│
├─ 第 1 步: 扫描缓冲区，收集脏页
│  │
│  ├─ 初始化变量:
│  │  num_to_scan = NBuffers;  ← 扫描所有缓冲区
│  │  num_to_write = 0;         ← 待写入计数
│  │  num_written = 0;          ← 已写入计数
│  │
│  ├─ 分配排序数组:
│  │  CkptSortItem *sortedBuffers = palloc(NBuffers * sizeof(CkptSortItem));
│  │
│  └─ 扫描循环 (for buf_id = 0; buf_id < NBuffers; buf_id++):
│     │
│     ├─ 获取缓冲区描述符:
│     │  bufHdr = GetBufferDescriptor(buf_id);
│     │
│     ├─ 检查脏页标志:
│     │  if (!(bufHdr->flags & BM_DIRTY))
│     │     continue;  ← 跳过干净页
│     │
│     ├─ 检查是否正在被写入:
│     │  if (bufHdr->flags & BM_IO_IN_PROGRESS)
│     │     等待完成...
│     │
│     ├─ 记录到排序数组:
│     │  sortedBuffers[num_to_write].buf_id = buf_id;
│     │  sortedBuffers[num_to_write].tablespace = bufHdr->tag.rnode.spcNode;
│     │  sortedBuffers[num_to_write].relfilenode = bufHdr->tag.rnode.relNode;
│     │  sortedBuffers[num_to_write].forknum = bufHdr->tag.forkNum;
│     │  sortedBuffers[num_to_write].blocknum = bufHdr->tag.blockNum;
│     │  num_to_write++;
│     │
│     └─ 示例缓冲区内容:
│        buf_id=1234, tablespace=16387, rel=16400, fork=MAIN, block=567
│
├─ 第 2 步: ⭐ 排序脏页 (减少磁盘寻道)
│  │
│  ├─ 排序键顺序:
│  │  1. tablespace (表空间)
│  │  2. relfilenode (表 OID)
│  │  3. forknum (fork 类型: MAIN/FSM/VM/INIT)
│  │  4. blocknum (块号)
│  │
│  ├─ qsort(sortedBuffers, num_to_write, sizeof(CkptSortItem), ckpt_buforder_comparator);
│  │
│  └─ 排序效果示例:
│     排序前:
│       [ts=16387, rel=16400, blk=100]
│       [ts=16387, rel=16398, blk=50]
│       [ts=16387, rel=16400, blk=99]
│
│     排序后:
│       [ts=16387, rel=16398, blk=50]   ← rel 16398
│       [ts=16387, rel=16400, blk=99]   ← rel 16400, 顺序块
│       [ts=16387, rel=16400, blk=100]  ← rel 16400, 顺序块
│
│     ✅ 减少磁盘寻道距离约 60%!
│
├─ 第 3 步: 按表空间分组，构建 Min-Heap
│  │
│  ├─ 统计每个表空间的脏页数:
│  │  for (i = 0; i < num_to_write; i++):
│  │     ts_stat[sortedBuffers[i].tablespace].num_to_scan++
│  │
│  ├─ 计算每个表空间的进度切片:
│  │  for (ts in all_tablespaces):
│  │     ts_stat[ts].progress_slice = num_to_scan / ts_stat[ts].num_to_scan
│  │
│  ├─ 构建最小堆 (Min-Heap):
│  │  ts_heap = binaryheap_allocate(num_tablespaces, ts_comparator, NULL);
│  │  for (ts in all_tablespaces):
│  │     binaryheap_add(ts_heap, &ts_stat[ts]);
│  │
│  └─ 堆结构示例 (3 个表空间):
│
│              [ts=16387, progress=0.0, slice=2.5]  ← 堆顶(最小)
│                /                          \
│     [ts=1663, progress=0.0, slice=5.0]  [ts=16388, progress=0.0, slice=10.0]
│
│     目的: 负载均衡，让各表空间的 I/O 均匀分布
│
├─ 第 4 步: ⭐ 写入脏页 (渐进式 + 负载均衡)
│  │
│  ├─ 主循环 (while not binaryheap_empty):
│  │  │
│  │  ├─ 取出进度最小的表空间:
│  │  │  ts_stat = binaryheap_first(ts_heap);
│  │  │
│  │  ├─ 写入该表空间的下一个缓冲区:
│  │  │  buf_id = sortedBuffers[ts_stat->next_buffer].buf_id;
│  │  │  SyncOneBuffer(buf_id, false, &wb_context);
│  │  │     │
│  │  │     ├─ FlushBuffer(bufHdr, NULL)
│  │  │     │  ├─ smgrwrite() ← 写入文件
│  │  │     │  └─ 清除 BM_DIRTY 标志
│  │  │     │
│  │  │     └─ 触发 writeback (如果累积足够):
│  │  │        if (written_bytes >= checkpoint_flush_after * 1024):
│  │  │           ScheduleBufferTagForWriteback(&wb_context, ...)
│  │  │           → sync_file_range() 或 posix_fadvise()
│  │  │           目的: 告诉 OS 尽早写入，减少最终 fsync 的峰值
│  │  │
│  │  ├─ 更新表空间进度:
│  │  │  ts_stat->progress += ts_stat->progress_slice;
│  │  │  ts_stat->next_buffer++;
│  │  │
│  │  ├─ 重新调整堆:
│  │  │  if (ts_stat->next_buffer < ts_stat->num_buffers):
│  │  │     binaryheap_replace_first(ts_heap, ts_stat);
│  │  │  else:
│  │  │     binaryheap_remove_first(ts_heap);  ← 该表空间完成
│  │  │
│  │  ├─ ⭐ 渐进式延迟:
│  │  │  num_written++;
│  │  │  if (num_written % CHECKPOINT_WRITE_BATCH == 0):
│  │  │     CheckpointWriteDelay(flags, (double)num_written / num_to_write);
│  │  │        │
│  │  │        └─ 判断是否需要睡眠 (详见 2.3):
│  │  │           如果写入速度过快 (超前于目标进度):
│  │  │              pg_usleep(delay_ms * 1000);
│  │  │
│  │  └─ 循环直到所有缓冲区写入完毕
│  │
│  └─ 写入完成统计:
│     buffers_checkpoint += num_written;
│
├─ 第 5 步: ⭐ Fsync 所有脏文件
│  │
│  ├─ 处理 fsync 请求队列:
│  │  ProcessFsyncRequests()
│  │     │
│  │     ├─ 遍历请求队列 (所有需要 fsync 的文件):
│  │     │  for (req in fsync_queue):
│  │     │     smgr_fsync(req.rnode, req.forknum);
│  │     │        │
│  │     │        └─ fsync(fd) ← ⚠️ 阻塞调用，可能耗时数秒
│  │     │
│  │     └─ 清空队列
│  │
│  ├─ Fsync 耗时统计:
│  │  sync_start = GetCurrentTimestamp();
│  │  ... (fsync 完成)
│  │  sync_end = GetCurrentTimestamp();
│  │  sync_time = sync_end - sync_start;
│  │
│  └─ 典型耗时:
│     写入时间: 29.0 秒 (smgrwrite)
│     Fsync 时间: 2.0 秒 (fsync)
│
│     Fsync 时间占比 = 2.0 / (29.0 + 2.0) = 6.5%
│
│     ✅ 通过 writeback 机制，将 fsync 耗时从 30%+ 降低到 5-10%
│
├─ 第 6 步: 清理和统计
│  │
│  ├─ 释放排序数组:
│  │  pfree(sortedBuffers);
│  │
│  ├─ 释放堆:
│  │  binaryheap_free(ts_heap);
│  │
│  ├─ 更新统计:
│  │  BgWriterStats.checkpoint_write_time += write_time_ms;
│  │  BgWriterStats.checkpoint_sync_time += sync_time_ms;
│  │  BgWriterStats.buffers_checkpoint += num_written;
│  │
│  └─ 记录日志 (ereport):
│     "wrote 12043 buffers (73.5%); write=29.0 s, sync=2.0 s"
│
└─ 返回

性能优化总结:
──────────────

1. ✅ 排序: 减少磁盘寻道 60%
   (顺序写入同一表的相邻块)

2. ✅ Min-Heap 负载均衡: I/O 均匀分布到各表空间
   (避免某个磁盘过载)

3. ✅ Writeback 机制: 减少 fsync 峰值 50-70%
   (提前通知 OS 写入，fsync 时已部分完成)

4. ✅ 渐进式写入: 平滑 I/O 曲线
   (通过 checkpoint_completion_target 控制速率)
```

**说明**:
- **扫描**: O(N)，遍历所有缓冲区
- **排序**: O(M log M)，M 为脏页数量
- **堆操作**: O(M log K)，K 为表空间数量（通常很小）
- **写入**: O(M)，每个脏页写一次
- **Fsync**: O(F)，F 为文件数量

**典型数据** (shared_buffers=8GB，脏页率=60%):
- NBuffers = 1,048,576
- 脏页数 M = 629,145
- 表空间数 K = 2
- 写入数据量 = 629,145 × 8KB ≈ 4.8GB
- 写入时间 = 29s（约 170 MB/s）
- Fsync 时间 = 2s

---

### 2.3 CheckpointWriteDelay 调度流程

```
CheckpointWriteDelay(int flags, double progress) 入口
│  (在 BufferSync 中每写入 32 个缓冲区调用一次)
│
├─ 参数说明:
│  flags = checkpoint 标志 (CHECKPOINT_IMMEDIATE 等)
│  progress = 当前进度 (0.0 - 1.0)
│     例如: 已写入 5000/10000 → progress = 0.5
│
├─ 第 1 步: 检查是否需要延迟
│  │
│  ├─ 如果是 IMMEDIATE checkpoint:
│  │  if (flags & CHECKPOINT_IMMEDIATE)
│  │     return;  ← 立即返回，不延迟
│  │
│  └─ 否则，进入调度逻辑
│
├─ 第 2 步: 调用 IsCheckpointOnSchedule()
│  │
│  └─ IsCheckpointOnSchedule(progress) 判断逻辑:
│     │
│     ├─ 获取目标完成时间:
│     │  elapsed_time = now - checkpoint_start_time;
│     │  target_time = checkpoint_timeout * checkpoint_completion_target;
│     │     例如: timeout=300s, target=0.9 → target_time=270s
│     │
│     ├─ 获取目标 WAL 进度:
│     │  elapsed_xlogs = current_wal_lsn - redo_lsn;
│     │  target_xlogs = max_wal_size * checkpoint_completion_target;
│     │     例如: max_wal=1GB, target=0.9 → target_xlogs=0.9GB
│     │
│     ├─ 调整进度 (乘以 completion_target):
│     │  adjusted_progress = progress * checkpoint_completion_target;
│     │     例如: progress=0.5, target=0.9 → adjusted=0.45
│     │
│     ├─ 判断是否超前:
│     │  if (adjusted_progress < (elapsed_xlogs / target_xlogs)):
│     │     return false;  ← WAL 进度落后，写快点
│     │
│     │  if (adjusted_progress < (elapsed_time / target_time)):
│     │     return false;  ← 时间进度落后，写快点
│     │
│     └─ return true;  ← 进度正常或超前，可以睡眠
│
├─ 第 3 步: 如果落后进度，直接返回
│  if (!IsCheckpointOnSchedule(progress))
│     return;  ← 不睡眠，继续全速写入
│
├─ 第 4 步: 计算睡眠时间
│  │
│  ├─ 基础睡眠时间:
│  │  base_sleep_ms = 100;  ← 100 毫秒
│  │
│  ├─ 根据超前程度调整:
│  │  ahead_ratio = adjusted_progress - (elapsed_time / target_time);
│  │  sleep_ms = base_sleep_ms * (1 + ahead_ratio * 10);
│  │
│  │  示例:
│  │    进度 50%, 时间过去 30% → 超前 5%
│  │    sleep_ms = 100 * (1 + 0.05 * 10) = 150ms
│  │
│  └─ 限制最大睡眠:
│     if (sleep_ms > 1000)
│        sleep_ms = 1000;  ← 最多睡 1 秒
│
├─ 第 5 步: 吸收挂起的 fsync 请求
│  │
│  ├─ AbsorbSyncRequests()
│  │  └─ 处理后端进程发来的 fsync 请求
│  │     (避免请求队列过长)
│  │
│  └─ 如果吸收了大量请求:
│     sleep_ms = sleep_ms / 2;  ← 减少睡眠时间
│
├─ 第 6 步: 执行睡眠
│  │
│  ├─ pg_usleep(sleep_ms * 1000);  ← 转换为微秒
│  │
│  └─ 睡眠期间:
│     CPU 空闲，其他进程可以运行
│     OS 可以刷写 page cache
│
└─ 返回，继续写入下一批缓冲区

可视化示例 (checkpoint_completion_target = 0.9):
───────────────────────────────────────────────

场景: checkpoint_timeout=300s, 预计写入 10000 个缓冲区

目标曲线 (虚线):
进度
100% │                                              ╱─────
     │                                           ╱
     │                                        ╱
 90% │                                     ╱  ← 目标: 270s 完成 90%
     │                                  ╱
     │                               ╱
     │                            ╱
 50% │                   ╱────╱  ← 判断点: 150s 应完成 50%
     │              ╱────
     │         ╱────
     │    ╱────
  0% └──────────────────────────────────────────────────→ 时间
     0s    50s   100s  150s  200s  250s  270s     300s

实际执行:

情况 1: 进度正常
─────────────────
时间  进度   判断             动作
 0s   0%    -                开始写入
 60s  20%   20% vs 22%       正常，继续
120s  40%   40% vs 44%       正常，继续
180s  60%   60% vs 66%       正常，继续
240s  80%   80% vs 88%       正常，继续
270s  90%   -                接近完成
300s 100%   -                完成

情况 2: 进度超前
─────────────────
时间  进度   判断             动作
 0s   0%    -                开始写入
 30s  20%   20% vs 11%       超前 9%，睡眠 190ms
 60s  30%   30% vs 22%       超前 8%，睡眠 180ms
 90s  40%   40% vs 33%       超前 7%，睡眠 170ms
...

情况 3: 进度落后 (I/O 慢)
─────────────────────────
时间  进度   判断             动作
 0s   0%    -                开始写入
 60s  10%   10% vs 22%       落后 12%，全速写入 (不睡眠)
120s  25%   25% vs 44%       落后 19%，全速写入
180s  45%   45% vs 66%       落后 21%，全速写入
240s  70%   70% vs 88%       落后 18%，全速写入
300s  95%   -                接近完成 (超时 5%)
320s 100%   -                完成 (超时 20s)

✅ 效果:
- 正常/快速 I/O: 平滑分散到 270s
- 慢速 I/O: 全速写入，尽量在 300s 完成
```

**说明**:
- **目的**: 将 checkpoint I/O 平滑分散到整个超时周期，避免峰值
- **关键参数**:
  - `checkpoint_completion_target`: 目标完成时间比例（默认 0.9）
  - `checkpoint_timeout`: checkpoint 超时时间（默认 300s）
- **自适应调整**: 如果 I/O 慢，自动减少睡眠，避免超时
- **负载感知**: 通过 `AbsorbSyncRequests()` 感知系统负载

**调优建议**:
- OLTP 系统: `completion_target = 0.9`（平滑 I/O，减少尖峰）
- 批处理系统: `completion_target = 0.5`（快速完成）
- SSD: 可以降低到 0.5-0.7（I/O 快，不需要过度平滑）
- HDD: 保持 0.9（I/O 慢，需要平滑）

---

### 2.4 SLRU Checkpoint 流程

```
CheckPointGuts() 中的 SLRU 处理
│
├─ SimpleLruFlush(CLOG)     ← pg_xact/ (事务提交状态)
│  │
│  ├─ 遍历所有 CLOG 页:
│  │  for (slotno = 0; slotno < clog_ctl->num_slots; slotno++):
│  │     if (page is dirty):
│  │        SlruPhysicalWritePage(clog_ctl, slotno)
│  │
│  ├─ 写入文件:
│  │  fd = open("pg_xact/0000", O_WRONLY);
│  │  write(fd, page_data, BLCKSZ);
│  │  fsync(fd);
│  │
│  └─ 清除脏标志
│
├─ SimpleLruFlush(CommitTs)  ← pg_commit_ts/ (提交时间戳)
│  │
│  └─ 类似流程，刷写提交时间戳页
│
├─ SimpleLruFlush(SubTrans)  ← pg_subtrans/ (子事务状态)
│  │
│  └─ 刷写子事务映射关系
│
└─ SimpleLruFlush(MultiXact) ← pg_multixact/ (多事务锁)
   │
   ├─ SimpleLruFlush(MultiXactOffset)  ← offsets/
   │  └─ 刷写多事务偏移量
   │
   └─ SimpleLruFlush(MultiXactMember)  ← members/
      └─ 刷写多事务成员列表

CLOG 页面格式:
──────────────

pg_xact/0000 文件 (每个文件 256KB = 32 页 × 8KB)
│
├─ Page 0 (8KB):
│  ├─ [XID 0-32767 的状态]     ← 每个 XID 占 2 bits
│  │   00 = IN_PROGRESS (进行中)
│  │   01 = COMMITTED (已提交)
│  │   10 = ABORTED (已中止)
│  │   11 = SUB_COMMITTED (子事务已提交)
│  │
│  └─ 示例:
│     XID 0:  01 (已提交)
│     XID 1:  01 (已提交)
│     XID 2:  00 (进行中)
│     ...
│     XID 32767: 10 (已中止)
│
├─ Page 1 (8KB):
│  └─ [XID 32768-65535 的状态]
│
├─ Page 2 (8KB):
│  └─ [XID 65536-98303 的状态]
│
...
│
└─ Page 31 (8KB):
   └─ [XID 1015808-1048575 的状态]

SLRU Checkpoint 时序:
─────────────────────

时间轴:
  0ms      CheckPointCLOG() 开始
  │
  ├─ 扫描 CLOG 缓冲区 (找出脏页)
  │  耗时: ~10ms
  │
  50ms     开始写入脏页
  │
  ├─ write(pg_xact/0000, page_0)  ← 8KB
  ├─ write(pg_xact/0000, page_5)  ← 8KB
  ├─ write(pg_xact/0001, page_3)  ← 8KB
  │  ...
  │  耗时: ~200ms (假设 10 个脏页)
  │
  250ms    fsync(pg_xact/0000)
  │        fsync(pg_xact/0001)
  │        耗时: ~50ms
  │
  300ms    CheckPointCommitTs() 开始
  │        (类似流程)
  │        耗时: ~100ms
  │
  400ms    CheckPointSUBTRANS() 开始
  │        耗时: ~50ms
  │
  450ms    CheckPointMultiXact() 开始
  │        耗时: ~150ms
  │
  600ms    所有 SLRU 刷写完成

总耗时: ~600ms (约占 checkpoint 总时间的 2%)
```

**说明**:
- **SLRU** (Simple LRU): 简化的 LRU 缓存，用于小型元数据
- **CLOG** (Commit Log): 最重要的 SLRU，每个事务的提交状态
- **文件大小**: CLOG 每个文件 256KB，可存储 1,048,576 个事务状态
- **检查点作用**: 确保事务状态持久化，崩溃恢复时可以正确判断事务是否提交

---

### 2.5 WAL 日志写入流程

```
Backend 事务提交流程
│
├─ 步骤 1: 开始插入 WAL 记录
│  XLogBeginInsert()
│  │
│  ├─ 获取 WAL 插入槽位
│  │  (使用 WALInsertLock 保护)
│  │
│  └─ 准备插入缓冲区
│
├─ 步骤 2: 注册 WAL 数据
│  XLogRegisterData(data, len)
│  XLogRegisterBuffer(buffer, flags)
│  │
│  ├─ 记录完整页镜像 (如果需要)
│  │  if (first modification after checkpoint):
│  │     XLogRegisterBlock(full_page_image)
│  │
│  └─ 记录增量数据 (通常情况)
│     XLogRegisterBufData(delta)
│
├─ 步骤 3: 插入 WAL 记录
│  recptr = XLogInsert(RM_HEAP_ID, XLOG_HEAP_INSERT)
│  │
│  ├─ 构造 WAL 记录:
│  │  ┌──────────────────────────────────┐
│  │  │ XLogRecord Header (24 bytes)     │
│  │  ├──────────────────────────────────┤
│  │  │ xl_tot_len = 128                 │
│  │  │ xl_xid = 1234                    │
│  │  │ xl_prev = 0/5000028              │
│  │  │ xl_info = XLOG_HEAP_INSERT       │
│  │  │ xl_rmid = RM_HEAP_ID             │
│  │  │ xl_crc = 0x12345678              │
│  │  ├──────────────────────────────────┤
│  │  │ Block 0 (full page image)        │
│  │  │   relfilenode = 16384            │
│  │  │   forknum = MAIN_FORKNUM         │
│  │  │   blkno = 567                    │
│  │  │   data = [8KB page data]         │
│  │  ├──────────────────────────────────┤
│  │  │ Main data                        │
│  │  │   offnum = 12                    │
│  │  │   flags = 0x01                   │
│  │  └──────────────────────────────────┘
│  │
│  ├─ 获取插入位置 (原子操作):
│  │  insert_lsn = pg_atomic_fetch_add_u64(&XLogCtl->logInsertResult, record_len);
│  │
│  ├─ 拷贝到 WAL Buffer:
│  │  memcpy(WAL_Buffer + (insert_lsn % wal_buffers), record, record_len);
│  │
│  └─ 返回 LSN: 0/5000128
│
├─ 步骤 4: 等待 WAL 写入
│  XLogFlush(recptr)
│  │
│  ├─ 检查是否已刷盘:
│  │  if (LogwrtResult.Flush >= recptr):
│  │     return;  ← 已经刷盘，直接返回
│  │
│  ├─ 唤醒 WAL Writer:
│  │  if (async):
│  │     SetLatch(&WalWriter->latch);
│  │     return;  ← 异步模式，不等待
│  │
│  └─ 同步等待 (事务提交时):
│     XLogWrite()
│     │
│     ├─ 写入 WAL 文件:
│     │  write(wal_fd, WAL_Buffer, len);
│     │
│     ├─ Fsync WAL 文件:
│     │  if (synchronous_commit != off):
│     │     fsync(wal_fd);
│     │
│     └─ 更新刷盘位置:
│        LogwrtResult.Flush = recptr;
│
└─ 步骤 5: 事务提交完成

WAL 文件布局:
─────────────

pg_wal/ 目录:
│
├─ 000000010000000000000001  ← Segment 1 (16MB)
│  │
│  ├─ [0000000-0000100]: WAL Record 1 (XLOG_HEAP_INSERT)
│  ├─ [0000100-0000200]: WAL Record 2 (XLOG_HEAP_UPDATE)
│  ├─ [0000200-0000300]: WAL Record 3 (XLOG_XACT_COMMIT)
│  ├─ ...
│  └─ [FFFFFF0-FFFFFFF]: 填充到 16MB
│
├─ 000000010000000000000002  ← Segment 2 (16MB)
│  └─ [继续写入 WAL 记录...]
│
├─ 000000010000000000000003  ← Segment 3 (16MB)
│
...

Checkpoint 写入的 WAL 记录:
───────────────────────────

XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT_ONLINE):
┌────────────────────────────────────────┐
│ XLogRecord Header                      │
├────────────────────────────────────────┤
│ xl_tot_len = 128                       │
│ xl_xid = InvalidTransactionId          │
│ xl_prev = 0/5000028                    │
│ xl_info = XLOG_CHECKPOINT_ONLINE       │
│ xl_rmid = RM_XLOG_ID (0)               │
│ xl_crc = 0xABCDEF12                    │
├────────────────────────────────────────┤
│ CheckPoint 结构 (104 bytes):           │
│   redo = 0/5000028                     │
│   ThisTimeLineID = 1                   │
│   PrevTimeLineID = 1                   │
│   fullPageWrites = true                │
│   nextXid = 1234:5678                  │
│   nextOid = 24576                      │
│   nextMulti = 1                        │
│   nextMultiOffset = 0                  │
│   oldestXid = 1234:1000                │
│   oldestXidDB = 13456                  │
│   oldestMulti = 1                      │
│   oldestMultiDB = 13456                │
│   time = 1705304400                    │
│   oldestActiveXid = 1234:5000          │
└────────────────────────────────────────┘

WAL 写入性能:
─────────────

典型事务:
  XLogInsert()  ~0.01ms   ← 拷贝到 WAL Buffer
  XLogWrite()   ~0.05ms   ← 写入文件 (异步)
  fsync()       ~1.00ms   ← 同步刷盘 (如果 synchronous_commit=on)

Checkpoint 的 WAL 写入:
  XLogInsert()  ~0.02ms   ← CheckPoint 记录较大
  XLogFlush()   ~1.50ms   ← 强制同步刷盘
```

**说明**:
- **WAL Buffer**: 共享内存中的环形缓冲区，默认 16MB
- **WAL Segment**: 磁盘上的 WAL 文件，每个 16MB（可配置）
- **Full Page Writes**: checkpoint 后首次修改的页，记录完整页镜像
- **XLogFlush**: 确保 WAL 刷盘，是事务持久性的保证

---

## 3. 状态转换图

### 3.1 Checkpoint 状态机

```
Checkpointer 进程状态机:
──────────────────────────

    ┌─────────┐
    │  IDLE   │  ← 初始状态: 等待触发
    └────┬────┘
         │
         │ 触发条件:
         │ • 超时 (checkpoint_timeout)
         │ • WAL 大小 (max_wal_size)
         │ • 手动 CHECKPOINT 命令
         │ • 数据库关闭
         │
         ↓
    ┌─────────────┐
    │  REQUESTED  │  ← 请求状态: 已收到请求
    └──────┬──────┘
           │
           │ CheckpointerShmemStruct->ckpt_started++
           │ ConditionVariableSignal(&start_cv)
           │
           ↓
    ┌───────────────┐
    │ IN_PROGRESS   │  ← 执行状态: 正在执行 checkpoint
    └───────┬───────┘
            │
            │ 执行 CreateCheckPoint():
            │ 1. 收集 checkpoint 数据
            │ 2. CheckPointGuts() ← 刷写脏页和 SLRU
            │ 3. 写入 WAL 记录
            │ 4. 更新 pg_control
            │
            ↓
         成功? ────────No──────┐
            │                  │
           Yes                 │
            │                  ↓
            │           ┌──────────┐
            │           │  FAILED  │  ← 失败状态
            │           └─────┬────┘
            │                 │
            │                 │ ckpt_failed++
            │                 │ ereport(ERROR)
            │                 │
            ↓                 ↓
    ┌─────────────┐      ┌──────────┐
    │  COMPLETED  │      │  PANIC   │  ← 严重错误: 数据库崩溃
    └──────┬──────┘      └──────────┘
           │                   ↑
           │                   │
           │                   │ 如果 pg_control fsync 失败
           │                   │
           │ ckpt_done++
           │ ConditionVariableBroadcast(&done_cv)
           │
           ↓
    ┌─────────┐
    │  IDLE   │  ← 返回空闲状态，等待下次触发
    └─────────┘

Backend 等待流程:
─────────────────

  Backend                     Checkpointer
    │                               │
    │ RequestCheckpoint()           │
    ├───────────────────────────────>
    │ 发送请求                       │
    │                               │
    │                               ├─ 收到请求
    │                               │  ckpt_started++
    │                               │
    │                               ├─ 执行 CreateCheckPoint()
    │ ConditionVariableWait()       │  (可能耗时 30 秒)
    │ (阻塞等待...)                 │
    │                               │
    │                               ├─ 完成
    │                               │  ckpt_done++
    │ <───────────────────────────┤
    │ ConditionVariableBroadcast()  │
    │                               │
    │ 唤醒，继续执行                │
    ↓                               ↓

状态检查 SQL:
─────────────

-- 查询 checkpoint 状态
SELECT
    CASE
        WHEN pg_is_in_recovery() THEN 'RECOVERY'
        WHEN EXISTS(
            SELECT 1 FROM pg_stat_activity
            WHERE backend_type = 'checkpointer'
              AND wait_event = 'CheckpointerMain'
        ) THEN 'IDLE'
        WHEN EXISTS(
            SELECT 1 FROM pg_stat_activity
            WHERE backend_type = 'checkpointer'
              AND wait_event IN ('DataFileWrite', 'DataFileSync')
        ) THEN 'IN_PROGRESS'
        ELSE 'UNKNOWN'
    END AS checkpointer_state,
    (SELECT checkpoints_timed + checkpoints_req FROM pg_stat_bgwriter) AS total_checkpoints;
```

---

### 3.2 缓冲区状态转换

```
Buffer 生命周期状态转换:
────────────────────────

        ┌────────┐
        │ EMPTY  │  ← 初始状态: 未使用的缓冲区
        └───┬────┘
            │
            │ ReadBuffer() / BufferAlloc()
            │
            ↓
        ┌────────┐
        │ CLEAN  │  ← 干净状态: 包含数据，但与磁盘一致
        └───┬────┘
            │
            │ 修改页面 (INSERT/UPDATE/DELETE)
            │ 设置 BM_DIRTY 标志
            │
            ↓
        ┌────────┐
        │ DIRTY  │  ← 脏页状态: 已修改，需要刷盘
        └───┬────┘
            │
            │ Checkpoint 扫描到此页
            │ 或 BgWriter 清理
            │
            ↓
        ┌──────────────────┐
        │ WRITE_IN_PROGRESS│  ← 写入中: 正在刷写到磁盘
        └────────┬─────────┘
                 │
                 │ smgrwrite() 完成
                 │
                 ↓
             成功? ────────No──────┐
                 │                 │
                Yes                │
                 │                 ↓
                 │          ┌───────────┐
                 │          │ IO_ERROR  │  ← I/O 错误
                 │          └───────────┘
                 │                 │
                 │                 │ ereport(ERROR)
                 │                 │
                 ↓                 ↓
        ┌────────┐           ┌─────────┐
        │ CLEAN  │           │  PANIC  │  ← 数据库崩溃
        └───┬────┘           └─────────┘
            │
            │ 页面可能被换出 (LRU)
            │ 或再次修改 → DIRTY
            │
            ↓
        ┌────────┐
        │ EMPTY  │  ← 换出: 缓冲区被释放，用于其他页
        └────────┘

Buffer 标志位:
──────────────

typedef struct BufferDesc {
    BufferTag    tag;           ← (rel, fork, blk)
    int          buf_id;        ← 缓冲区 ID
    uint32       flags;         ← 状态标志
    uint16       usage_count;   ← LRU 计数
    LWLock       content_lock;  ← 内容锁
} BufferDesc;

flags 位定义:
  BM_DIRTY          = 0x0001  ← 脏页
  BM_VALID          = 0x0002  ← 数据有效
  BM_TAG_VALID      = 0x0004  ← 标签有效
  BM_IO_IN_PROGRESS = 0x0008  ← I/O 进行中
  BM_IO_ERROR       = 0x0010  ← I/O 错误
  BM_JUST_DIRTIED   = 0x0020  ← 刚刚变脏
  BM_PIN_COUNT_WAITER = 0x0040 ← 有进程等待 unpin
  BM_CHECKPOINT_NEEDED = 0x0080 ← 需要 checkpoint
  BM_PERMANENT      = 0x0100  ← 永久页 (不换出)

状态转换操作:
─────────────

操作              状态变化            设置标志
────────────────  ───────────────── ─────────────
ReadBuffer()      EMPTY → CLEAN      BM_VALID | BM_TAG_VALID
MarkBufferDirty() CLEAN → DIRTY      BM_DIRTY | BM_JUST_DIRTIED
FlushBuffer()     DIRTY → WRITE_IN_  BM_IO_IN_PROGRESS
                         PROGRESS
写入完成          WRITE_IN_ → CLEAN  清除 BM_DIRTY, BM_IO_IN_PROGRESS
写入失败          WRITE_IN_ → ERROR  BM_IO_ERROR
ReleaseBuffer()   CLEAN → EMPTY      清除所有标志

Checkpoint 相关的状态检查:
──────────────────────────

BufferSync() 扫描逻辑:
  for (buf_id = 0; buf_id < NBuffers; buf_id++):
      bufHdr = GetBufferDescriptor(buf_id);

      // 跳过非脏页
      if (!(bufHdr->flags & BM_DIRTY))
          continue;

      // 等待正在进行的 I/O
      if (bufHdr->flags & BM_IO_IN_PROGRESS)
          WaitIO(bufHdr);

      // 标记为写入中
      bufHdr->flags |= BM_IO_IN_PROGRESS;

      // 写入磁盘
      FlushBuffer(bufHdr, NULL);

      // 清除脏标志
      bufHdr->flags &= ~(BM_DIRTY | BM_IO_IN_PROGRESS);
```

---

### 3.3 WAL 段文件生命周期

```
WAL Segment 文件状态:
─────────────────────

        ┌─────────┐
        │ CREATED │  ← 新建: XLogFileInit() 创建新段文件
        └────┬────┘
             │
             │ 文件命名: 000000010000000000000001
             │ 大小: 16MB (预分配)
             │
             ↓
        ┌─────────┐
        │ ACTIVE  │  ← 活跃: 正在写入 WAL 记录
        └────┬────┘
             │
             │ XLogWrite() 写入记录
             │ Backend 插入事务日志
             │ 文件逐渐填满...
             │
             ↓
         已满? ──────No─────┐
             │              │
            Yes             │ (继续写入)
             │              ↓
             ↓         (保持 ACTIVE)
        ┌─────────┐
        │  FULL   │  ← 已满: 切换到下一个段文件
        └────┬────┘
             │
             │ 等待 checkpoint
             │ 或归档完成
             │
             ↓
      Checkpoint 完成?
             │
             ├─Yes─> REDO 点之前? ──Yes─┐
             │            │              │
             │           No              │
             │            │              │
             │            ↓              ↓
             │       ┌──────────┐  ┌──────────┐
             │       │ RETAINED │  │ OBSOLETE │  ← 过时: 可以删除
             │       └─────┬────┘  └────┬─────┘
             │             │             │
             │             │ 保留原因:   │
             │             │ • wal_keep │ RemoveOldXlogFiles()
             │             │ • 复制槽   │
             │             │             │
             │             ↓             ↓
             │      (继续保留)      archive_mode?
             │                            │
             │                       ├──Yes──> ┌──────────────┐
             │                       │         │ TO_ARCHIVE   │  ← 待归档
             │                       │         └──────┬───────┘
             │                       │                │
             │                       │                │ 归档进程处理
             │                       │                │ cp WAL → archive/
             │                       │                │
             │                       │                ↓
             │                       │         ┌──────────────┐
             │                       │         │   ARCHIVED   │  ← 已归档
             │                       │         └──────┬───────┘
             │                       │                │
             │                       │                │ 归档成功
             │                       │                │ .done 文件创建
             │                       │                │
             │                       No               ↓
             │                        │    ┌─────────────────┐
             │                        └───>│ READY_TO_REMOVE │  ← 可以删除
             │                             └────────┬────────┘
             │                                      │
             │                                      │
             └──────────────────────────────────────┘
                                                    │
                                                    ↓
                                          删除 or 回收?
                                                    │
                                          ├────回收────> ┌──────────┐
                                          │              │ RECYCLED │  ← 重命名重用
                                          │              └─────┬────┘
                                          │                    │
                                          │                    │ 重命名为未来的段号
                                          │                    │ 例如: 0001 → 0020
                                          │                    │
                                          │                    ↓
                                          │              (回到 CREATED)
                                          │
                                          └────删除────> ┌──────────┐
                                                         │ DELETED  │  ← 从磁盘删除
                                                         └──────────┘

文件操作序列:
─────────────

时间轴示例 (checkpoint_timeout=5min, max_wal_size=1GB):

T=0min:
  000000010000000000000001  ACTIVE   (正在写入)
  000000010000000000000002  CREATED  (预创建)

T=1min:
  000000010000000000000001  FULL     (已满，16MB)
  000000010000000000000002  ACTIVE   (切换到这里)
  000000010000000000000003  CREATED

T=5min: (Checkpoint #1 完成, REDO = 0/1000000)
  000000010000000000000001  OBSOLETE (REDO 之前)
  000000010000000000000002  FULL
  000000010000000000000003  FULL
  000000010000000000000004  FULL
  000000010000000000000005  ACTIVE
  000000010000000000000006  CREATED

  删除或归档:
  000000010000000000000001  →  DELETED (如果无归档)
                             或 TO_ARCHIVE (如果有归档)

T=10min: (Checkpoint #2 完成, REDO = 0/4000000)
  000000010000000000000002  OBSOLETE  → RECYCLED → 000000010000000000000020
  000000010000000000000003  OBSOLETE  → RECYCLED → 000000010000000000000021
  000000010000000000000004  RETAINED  (REDO 点在这里)
  ...

pg_wal/ 目录状态查询:
─────────────────────

-- 查看 WAL 文件状态
SELECT
    name,
    size,
    modification AS last_modified,
    CASE
        WHEN name = pg_walfile_name(pg_current_wal_lsn())
            THEN 'ACTIVE'
        WHEN name >= pg_walfile_name(pg_current_wal_lsn())
            THEN 'FUTURE'
        WHEN EXISTS(
            SELECT 1 FROM pg_ls_archive_statusdir()
            WHERE name = name || '.ready'
        ) THEN 'TO_ARCHIVE'
        WHEN EXISTS(
            SELECT 1 FROM pg_ls_archive_statusdir()
            WHERE name = name || '.done'
        ) THEN 'ARCHIVED'
        ELSE 'OBSOLETE'
    END AS status
FROM pg_ls_waldir()
ORDER BY name;

预期输出:
            name             |   size   |     last_modified      |  status
─────────────────────────────+──────────+────────────────────────+───────────
 000000010000000000000001    | 16777216 | 2025-01-15 10:00:00    | ARCHIVED
 000000010000000000000002    | 16777216 | 2025-01-15 10:05:00    | ARCHIVED
 000000010000000000000003    | 16777216 | 2025-01-15 10:10:00    | OBSOLETE
 000000010000000000000004    | 16777216 | 2025-01-15 10:15:00    | ACTIVE
 000000010000000000000005    | 16777216 | 2025-01-15 10:15:30    | FUTURE
```

**说明**:
- **ACTIVE**: 当前正在写入的 WAL 段
- **FULL**: 已写满，等待 checkpoint 确定是否可删除
- **OBSOLETE**: REDO 点之前的 WAL，可以安全删除
- **RETAINED**: 因 `wal_keep_size` 或复制槽需求保留
- **RECYCLED**: 重命名为未来的段号重用，避免重新分配
- **TO_ARCHIVE**: 等待归档进程处理
- **ARCHIVED**: 已成功归档，可以删除

---

## 4. 数据流图

### 4.1 写事务数据流

```
Backend 执行写事务的完整数据流:
────────────────────────────────

Client                Backend              Shared Memory           Disk
  │                     │                                           │
  │ INSERT ...          │                                           │
  ├────────────────────>│                                           │
  │                     │                                           │
  │                     ├─1. 生成 WAL 记录                          │
  │                     │   XLogInsert(RM_HEAP_ID, XLOG_HEAP_INSERT)
  │                     │   LSN = 0/5000128                         │
  │                     │                                           │
  │                     ├─2. 写入 WAL Buffer ──────────────>        │
  │                     │   ┌────────────────────────┐              │
  │                     │   │   WAL Buffers          │              │
  │                     │   │  [0/5000128] = Record  │              │
  │                     │   └────────────────────────┘              │
  │                     │          │                                │
  │                     │          │ WAL Writer 或同步刷盘          │
  │                     │          ↓                                │
  │                     │   write(pg_wal/...001) ──────────────────>│
  │                     │          │                                │
  │                     │          ↓                                │
  │                     │   fsync(pg_wal/...001) ──────────────────>│
  │                     │                                           │
  │                     ├─3. 修改数据页                              │
  │                     │   ReadBuffer(rel, blk) <──────────────┐  │
  │                     │   ┌────────────────────────┐          │  │
  │                     │   │  Shared Buffers        │          │  │
  │                     │   │  [rel=16384, blk=567]  │          │  │
  │                     │   │  status = CLEAN        │          │  │
  │                     │   └────────────────────────┘          │  │
  │                     │          │                            │  │
  │                     │          │ 插入 tuple                 │  │
  │                     │          ↓                            │  │
  │                     │   MarkBufferDirty()                   │  │
  │                     │   ┌────────────────────────┐          │  │
  │                     │   │  Shared Buffers        │          │  │
  │                     │   │  [rel=16384, blk=567]  │          │  │
  │                     │   │  status = DIRTY ⭐     │          │  │
  │                     │   │  LSN = 0/5000128       │          │  │
  │                     │   └────────────────────────┘          │  │
  │                     │                                       │  │
  │                     │   (脏页暂不写入磁盘，等 checkpoint)  │  │
  │                     │                                       │  │
  │                     ├─4. 提交事务                           │  │
  │                     │   COMMIT                              │  │
  │                     │   XLogInsert(RM_XACT_ID,              │  │
  │                     │               XLOG_XACT_COMMIT)       │  │
  │                     │   XLogFlush(commit_lsn)               │  │
  │                     │   (等待 WAL 刷盘)                     │  │
  │                     │                                       │  │
  │ <───────────────────┤ 返回: COMMIT SUCCESS                 │  │
  │                     │                                       │  │
  │                     │                                       │  │
  │                     │   ... (脏页留在内存中)                │  │
  │                     │                                       │  │
  │                     │                                       │  │
  │                     │  (几分钟后，Checkpoint 触发)          │  │
  │                     │                                       │  │
Checkpointer:          │                                       │  │
                       │                                       │  │
                       ├─5. Checkpoint 扫描脏页                │  │
                       │   BufferSync()                        │  │
                       │   发现 [rel=16384, blk=567] 是脏页    │  │
                       │                                       │  │
                       ├─6. 写入数据文件                       │  │
                       │   FlushBuffer(bufHdr)                 │  │
                       │   smgrwrite() ────────────────────────────>│
                       │   write(base/13456/16384)                  │
                       │                                       │  │
                       ├─7. Fsync 数据文件                     │  │
                       │   smgr_fsync() ───────────────────────────>│
                       │   fsync(base/13456/16384)                  │
                       │                                       │  │
                       ├─8. 清除脏标志                         │  │
                       │   ┌────────────────────────┐          │  │
                       │   │  Shared Buffers        │          │  │
                       │   │  [rel=16384, blk=567]  │          │  │
                       │   │  status = CLEAN ✅     │          │  │
                       │   └────────────────────────┘          │  │
                       │                                       │  │
                       └───────────────────────────────────────┘  │

数据流总结:
───────────

1. WAL 先行: WAL 记录先于数据页写入磁盘
   • Backend 先写 WAL Buffer
   • WAL Writer 或同步刷盘到 pg_wal/
   • 保证: 即使数据页未刷盘，也可通过 WAL 恢复

2. 延迟写入: 数据页修改后不立即写磁盘
   • Backend 只标记为 DIRTY
   • 留在 Shared Buffers 中
   • 由 Checkpointer 或 BgWriter 负责刷写

3. Checkpoint 确保持久化:
   • 扫描所有脏页
   • 批量写入磁盘
   • Fsync 确保持久化
   • 更新 REDO 点

崩溃恢复保证:
─────────────

假设崩溃发生在步骤 4 (事务已提交) 和步骤 6 (数据页未刷盘) 之间:

恢复流程:
  1. 读取 pg_control，获取 REDO 点 (例如 0/5000000)
  2. 从 REDO 点开始重放 WAL:
     • 重放 [0/5000128]: HEAP_INSERT → 重新插入 tuple
     • 重放 [0/5000200]: XACT_COMMIT → 标记事务提交
  3. 恢复完成，数据一致

✅ 即使数据页未刷盘，WAL 保证了数据不丢失!
```

---

### 4.2 Checkpoint 数据流

```
Checkpoint 完整数据流:
─────────────────────

Checkpointer          Shared Memory             Disk               pg_control
     │                                            │                     │
     ├─1. 开始 Checkpoint                         │                     │
     │   CreateCheckPoint()                       │                     │
     │   获取 CheckpointLock                      │                     │
     │                                            │                     │
     ├─2. 确定 REDO 点                            │                     │
     │   redo_lsn = GetRedoRecPtr()               │                     │
     │   redo_lsn = 0/5000028                     │                     │
     │                                            │                     │
     ├─3. 刷写 SLRU                                │                     │
     │   CheckPointCLOG()                         │                     │
     │   ┌─────────────────────┐                  │                     │
     │   │  SLRU Buffers       │                  │                     │
     │   │  pg_xact pages      │                  │                     │
     │   └──────────┬──────────┘                  │                     │
     │              │                              │                     │
     │              └─> SimpleLruFlush() ─────────────> pg_xact/0000    │
     │                  write()                         write()         │
     │                  fsync()                         fsync()         │
     │                                            │                     │
     ├─4. 刷写数据页                                                       │
     │   CheckPointBuffers()                      │                     │
     │   BufferSync()                             │                     │
     │   ┌─────────────────────┐                  │                     │
     │   │  Shared Buffers     │                  │                     │
     │   │  [扫描脏页...]      │                  │                     │
     │   │  • rel=16384, blk=1 │ DIRTY            │                     │
     │   │  • rel=16384, blk=2 │ DIRTY            │                     │
     │   │  • rel=16385, blk=5 │ DIRTY            │                     │
     │   │  ... (假设 10000 个脏页)               │                     │
     │   └──────────┬──────────┘                  │                     │
     │              │                              │                     │
     │              ├─> 排序 (按 ts, rel, fork, blk)                    │
     │              │                              │                     │
     │              ├─> 按表空间分组 (Min-Heap)    │                     │
     │              │                              │                     │
     │              └─> 逐个写入: ─────────────────────> base/13456/16384 │
     │                  for 脏页 in 排序列表:            write()         │
     │                      FlushBuffer()                                │
     │                      smgrwrite() ───────────────> base/13456/16385 │
     │                      (累积到 checkpoint_flush_after)              │
     │                      ScheduleBufferTagForWriteback()  ← Writeback │
     │                                            │         (告诉 OS)   │
     │                      CheckpointWriteDelay() ← 渐进式延迟          │
     │                                            │                     │
     │              写入完成 (10000 个页, 约 78MB) │                     │
     │                                            │                     │
     ├─5. Fsync 所有文件                                                  │
     │   ProcessFsyncRequests()                   │                     │
     │   for file in fsync_queue:                 │                     │
     │       smgr_fsync() ────────────────────────────> fsync(base/...)  │
     │                                            │     (可能耗时 1-2s) │
     │                                            │                     │
     ├─6. 写入 Checkpoint WAL 记录                │                     │
     │   XLogInsert(RM_XLOG_ID,                   │                     │
     │              XLOG_CHECKPOINT_ONLINE)       │                     │
     │   ┌─────────────────────────────┐          │                     │
     │   │ CheckPoint 结构:            │          │                     │
     │   │   redo = 0/5000028          │          │                     │
     │   │   nextXid = 1234:5678       │          │                     │
     │   │   time = 1705304400         │          │                     │
     │   │   ...                       │          │                     │
     │   └──────────┬──────────────────┘          │                     │
     │              │                              │                     │
     │              └─> XLogFlush() ──────────────────> pg_wal/...0003  │
     │                  write()                         write()         │
     │                  fsync()                         fsync()         │
     │                  recptr = 0/5000500                              │
     │                                            │                     │
     ├─7. 更新 pg_control                         │                     │
     │   ControlFile->checkPoint = 0/5000500      │                     │
     │   ControlFile->checkPointCopy = [CheckPoint]                    │
     │   ControlFile->state = DB_IN_PRODUCTION    │                     │
     │   UpdateControlFile() ─────────────────────────────────────────────>│
     │   write(global/pg_control)                                      │
     │   fsync(global/pg_control)  ⚠️ 最关键的步骤!                    │
     │                                            │                     │
     ├─8. 清理旧 WAL                                                       │
     │   RemoveOldXlogFiles()                     │                     │
     │   删除 REDO 点之前的 WAL:                   │                     │
     │   unlink(pg_wal/000...0001) ───────────────────> (删除)         │
     │   unlink(pg_wal/000...0002) ───────────────────> (删除)         │
     │   或重命名为未来段号 (回收)                 │                     │
     │                                            │                     │
     ├─9. 释放锁，记录日志                        │                     │
     │   释放 CheckpointLock                       │                     │
     │   ereport(LOG, "checkpoint complete: ...")  │                     │
     │   更新 pg_stat_bgwriter 统计                │                     │
     │                                            │                     │
     └─ Checkpoint 完成                            │                     │

数据量示例 (shared_buffers=8GB, 脏页率=60%):
───────────────────────────────────────────

脏页数量: 1048576 × 60% = 629,145 页
数据量:   629145 × 8KB ≈ 4.8 GB

写入分解:
  1. SLRU:        ~10 MB   (CLOG, SubTrans, MultiXact, CommitTs)
  2. 数据页:      ~4.8 GB  (Shared Buffers 脏页)
  3. WAL 记录:    ~128 B   (Checkpoint 结构)
  4. pg_control:  ~8 KB    (控制文件)

总写入:         ~4.81 GB

Fsync 调用:
  1. SLRU:        4 次     (CLOG, CommitTs, SubTrans, MultiXact×2)
  2. 数据文件:    ~287 次  (假设 287 个不同的数据文件)
  3. WAL:         1 次
  4. pg_control:  1 次

总 fsync:       ~293 次

耗时分布 (HDD):
  写入:  4.81 GB @ 170 MB/s ≈ 29 s
  Fsync: 293 次 @ 7 ms/次  ≈ 2 s
  其他:  准备、日志等      ≈ 1.5 s
  ──────────────────────────────
  总计:                     32.5 s
```

---

### 4.3 崩溃恢复数据流

```
数据库崩溃后的恢复流程:
──────────────────────

启动时:                 pg_control          pg_wal/           Shared Memory
  │                         │                  │                    │
  ├─1. 启动 Postmaster       │                  │                    │
  │   pg_ctl start          │                  │                    │
  │                         │                  │                    │
  ├─2. Startup 进程启动      │                  │                    │
  │   StartupXLOG()         │                  │                    │
  │                         │                  │                    │
  ├─3. 读取 pg_control       │                  │                    │
  │   ReadControlFile() <───┤                  │                    │
  │   ┌──────────────────────────────────┐     │                    │
  │   │ ControlFileData:                 │     │                    │
  │   │   state = DB_SHUTDOWNED_IN_RECOVERY   │                    │
  │   │   checkPoint = 0/5000500         │     │                    │
  │   │   redo = 0/5000028 ⭐            │     │                    │
  │   │   nextXid = 1234:5678            │     │                    │
  │   │   ...                            │     │                    │
  │   └──────────────────────────────────┘     │                    │
  │                         │                  │                    │
  ├─4. 检查数据库状态        │                  │                    │
  │   if (state != DB_SHUTDOWNED):            │                    │
  │       ereport(LOG, "database was not properly shut down")     │
  │       ereport(LOG, "starting recovery")   │                    │
  │                         │                  │                    │
  ├─5. 定位 REDO 点          │                  │                    │
  │   redo_lsn = 0/5000028   │                  │                    │
  │   XLogBeginRead(redo_lsn) ────────────────>│                    │
  │   打开 WAL 段文件:                          │                    │
  │   pg_wal/000000010000000000000003          │                    │
  │   seek 到偏移量 0x5000028                   │                    │
  │                         │                  │                    │
  ├─6. 重放 WAL 记录         │                  │                    │
  │   ereport(LOG, "redo starts at 0/5000028") │                    │
  │                         │                  │                    │
  │   循环读取 WAL 记录:      │                  │                    │
  │   ┌──────────────────────────────────────────────────┐          │
  │   │ while (record = XLogReadRecord()):               │          │
  │   │                         │                  │                    │
  │   │   // 记录 1: HEAP_INSERT (0/5000028)              │          │
  │   │   if (rmgr_id == RM_HEAP_ID &&                    │          │
  │   │       info == XLOG_HEAP_INSERT):                  │          │
  │   │       heap_xlog_insert(record) ──────────────────────────────>│
  │   │       // 将 tuple 重新插入到 Shared Buffer                   │
  │   │       ReadBufferWithoutRelcache(rel, blk)                    │
  │   │       插入 tuple 到页面                                      │
  │   │       MarkBufferDirty()                                      │
  │   │                         │                  │                    │
  │   │   // 记录 2: HEAP_UPDATE (0/5000100)              │          │
  │   │   if (rmgr_id == RM_HEAP_ID &&                    │          │
  │   │       info == XLOG_HEAP_UPDATE):                  │          │
  │   │       heap_xlog_update(record) ───────────────────────────────>│
  │   │       // 重放更新操作                                        │
  │   │                         │                  │                    │
  │   │   // 记录 3: XACT_COMMIT (0/5000200)              │          │
  │   │   if (rmgr_id == RM_XACT_ID &&                    │          │
  │   │       info == XLOG_XACT_COMMIT):                  │          │
  │   │       xact_redo_commit(record)                               │
  │   │       // 更新 CLOG 状态为 COMMITTED                          │
  │   │                         │                  │                    │
  │   │   // 记录 4: CHECKPOINT_ONLINE (0/5000500)         │          │
  │   │   if (rmgr_id == RM_XLOG_ID &&                    │          │
  │   │       info == XLOG_CHECKPOINT_ONLINE):            │          │
  │   │       xlog_redo_checkpoint(record)                           │
  │   │       // 更新恢复目标点                                      │
  │   │                         │                  │                    │
  │   │   ... 继续读取直到 WAL 结束                │                    │
  │   │                         │                  │                    │
  │   └──────────────────────────────────────────────────┘          │
  │                         │                  │                    │
  ├─7. REDO 完成             │                  │                    │
  │   ereport(LOG, "redo done at 0/5001234")    │                    │
  │   ereport(LOG, "last completed transaction was at ...") │        │
  │                         │                  │                    │
  ├─8. 执行恢复结束 Checkpoint                  │                    │
  │   CreateCheckPoint(CHECKPOINT_END_OF_RECOVERY)                  │
  │   ├─ 刷写所有恢复产生的脏页                   │                    │
  │   ├─ 写入 CHECKPOINT_END_OF_RECOVERY 记录    │                    │
  │   ├─ 更新 pg_control:                        │                    │
  │   │   state = DB_IN_PRODUCTION ─────────────>│                    │
  │   │   checkPoint = 0/5001300                 │                    │
  │   └─ fsync(pg_control)                       │                    │
  │                         │                  │                    │
  ├─9. 恢复完成，接受连接    │                  │                    │
  │   ereport(LOG, "database system is ready to accept connections")│
  │                         │                  │                    │
  │   Postmaster 启动 Backend 进程              │                    │
  │   ├─ Checkpointer                           │                    │
  │   ├─ WAL Writer                             │                    │
  │   ├─ BgWriter                               │                    │
  │   └─ ... 其他辅助进程                        │                    │
  │                         │                  │                    │
  └─ 数据库正常运行          │                  │                    │

恢复日志示例:
────────────

LOG:  database system was interrupted; last known up at 2025-01-15 10:30:00 CST
LOG:  database system was not properly shut down; automatic recovery in progress
LOG:  redo starts at 0/5000028
LOG:  redo in progress, elapsed time: 5.23 s, current LSN: 0/5000800
LOG:  redo done at 0/5001234 system usage: CPU: user: 2.34 s, system: 1.23 s, elapsed: 18.45 s
LOG:  checkpoint starting: end-of-recovery immediate
LOG:  checkpoint complete: wrote 1234 buffers (7.5%); write=1.2 s, sync=0.3 s, total=1.6 s
LOG:  database system is ready to accept connections

恢复性能分析:
────────────

假设场景:
  • 上次 checkpoint: 10:00:00 (REDO = 0/5000028)
  • 崩溃时间: 10:05:00
  • WAL 生成量: 5 分钟 × 10 MB/min = 50 MB
  • WAL 记录数: 约 50,000 条

恢复耗时:
  1. 读取 pg_control:     ~0.01 s
  2. 打开 WAL 文件:        ~0.05 s
  3. 重放 WAL 记录:        ~15 s
     (50 MB @ 3.3 MB/s)
  4. End-of-recovery 检查点: ~1.6 s
  ───────────────────────────────
  总计:                    ~16.7 s

✅ 恢复时间主要取决于:
  - checkpoint_timeout (影响 WAL 重放量)
  - 磁盘 I/O 性能
  - WAL 记录复杂度
```

**说明**:
- **REDO 点**: 恢复的起点，从此 LSN 开始重放 WAL
- **WAL 重放**: 按顺序执行每条 WAL 记录，重建数据库状态
- **恢复结束 Checkpoint**: 确保所有重放的修改持久化
- **pg_control 状态**: 从 `DB_SHUTDOWNED_IN_RECOVERY` 更新为 `DB_IN_PRODUCTION`

---

## 5. 时序图

### 5.1 正常 Checkpoint 时序

```
Checkpointer     Backend       WAL Writer      OS Cache        Disk
     │               │               │              │            │
T=0s │               │               │              │            │
     ├───────────────────────────────────────────────────────────>
     │  触发 checkpoint (timeout=300s 到期)         │            │
     │               │               │              │            │
     ├─ 获取 CheckpointLock                          │            │
     ├─ 记录 REDO 点                                 │            │
     │  redo = 0/5000028                            │            │
     │               │               │              │            │
T=0.5s               │               │              │            │
     ├─ CheckPointCLOG()                            │            │
     │  SimpleLruFlush() ───────────────────────────────────────>│
     │               │               │              │      write pg_xact/
     │               │               │              │            │
T=0.8s               │               │              │            │
     ├─ CheckPointBuffers()                         │            │
     │  BufferSync() - 开始扫描脏页                   │            │
     │  发现 10000 个脏页，排序中...                  │            │
     │               │               │              │            │
T=1.0s               │               │              │            │
     ├─ 写入脏页 (batch 1-32)                        │            │
     │  FlushBuffer() ──────────────────────────────>            │
     │               │               │      [write到page cache]   │
     │               │               │              │            │
     ├─ CheckpointWriteDelay()                      │            │
     │  睡眠 100ms (渐进式控制)                       │            │
     │               │               │              │            │
     │ [同时 Backend 继续正常工作]   │              │            │
     │               ├─ INSERT ...   │              │            │
     │               ├─ XLogInsert() ────────────>  │            │
     │               │               │   WAL Buffer │            │
     │               │               ├─ XLogWrite() │            │
     │               │               └─────────────────────────>│
     │               │               │              │     write pg_wal/
     │               │               │              │            │
T=5.0s               │               │              │            │
     ├─ 写入脏页 (batch 1000-1032)                   │            │
     │  累积写入 256KB                                │            │
     ├─ ScheduleBufferTagForWriteback()             │            │
     │  sync_file_range() ──────────────────────────>            │
     │               │               │      [告诉OS尽早刷盘]     │
     │               │               │              ├───────────>│
     │               │               │              │    fsync部分
     │               │               │              │            │
T=29.0s              │               │              │            │
     ├─ 所有脏页写入完成 (10000 个)                   │            │
     │               │               │              │            │
     ├─ ProcessFsyncRequests()                      │            │
     │  遍历需要 fsync 的文件 (287 个)                │            │
     │  smgr_fsync(base/13456/16384) ────────────────────────────>│
     │  smgr_fsync(base/13456/16385) ────────────────────────────>│
     │  ... (287 次 fsync)                          │            │
     │               │               │              │            │
T=31.0s              │               │              │            │
     ├─ 所有文件 fsync 完成                          │            │
     │               │               │              │            │
     ├─ XLogInsert(XLOG_CHECKPOINT_ONLINE)          │            │
     │  写入 checkpoint WAL 记录                      │            │
     │  XLogFlush() ───────────────────────────────────────────>│
     │               │               │              │     pg_wal/
     │               │               │              │            │
T=31.5s              │               │              │            │
     ├─ UpdateControlFile()                         │            │
     │  write(pg_control) ───────────────────────────────────────>│
     │  fsync(pg_control) ⚠️ 关键步骤                │            │
     │               │               │              │            │
T=32.0s              │               │              │            │
     ├─ RemoveOldXlogFiles()                        │            │
     │  删除旧 WAL 文件                               │            │
     │               │               │              │            │
T=32.5s              │               │              │            │
     ├─ ereport(LOG, "checkpoint complete: ...")     │            │
     ├─ 释放 CheckpointLock                          │            │
     │               │               │              │            │
     │ 回到空闲状态   │               │              │            │
     ↓               ↓               ↓              ↓            ↓

时间分布:
────────
准备:      0.5s   (  1.5%)
SLRU:      0.3s   (  0.9%)
写脏页:   28.0s   ( 86.2%)  ← 主要耗时
Fsync:     2.0s   (  6.2%)
WAL+控制:  0.5s   (  1.5%)
清理日志:  1.2s   (  3.7%)
─────────────────────────
总计:     32.5s   (100.0%)
```

---

### 5.2 渐进式 Checkpoint I/O 模式

```
completion_target = 0.1 (快速完成) vs completion_target = 0.9 (平滑完成)

I/O速率对比:
───────────

情况 1: completion_target = 0.1
────────────────────────────────
I/O
速率
(MB/s)
 800│ █████
    │ █████
    │ █████
 600│ █████
    │ █████
    │ █████
 400│ █████
    │ █████         ← 峰值: 750 MB/s (很高!)
    │ █████         ← 持续时间短 (30s)
 200│ █████         ← 对其他进程影响大
    │ █████
    │ █████
   0└─────┴─────┴─────┴─────┴─────┴─────┴─────────> 时间
     0s   30s   60s   90s  120s  150s  180s   270s

    checkpoint 在 30s 完成，但 I/O 峰值极高
    查询延迟在 0-30s 期间显著增加 (100ms+)


情况 2: completion_target = 0.9
────────────────────────────────
I/O
速率
(MB/s)
 200│
    │
    │ ══════════════════════════════════
 150│ ══════════════════════════════════
    │ ══════════════════════════════════
    │ ══════════════════════════════════      ← 峰值: 180 MB/s (平滑)
 100│ ══════════════════════════════════      ← 持续时间长 (270s)
    │ ══════════════════════════════════      ← 对其他进程影响小
    │ ══════════════════════════════════
  50│ ══════════════════════════════════
    │ ══════════════════════════════════
    │
   0└─────┴─────┴─────┴─────┴─────┴─────┴─────────> 时间
     0s   30s   60s   90s  120s  150s  180s   270s

    checkpoint 在 270s 完成，I/O 平滑分散
    查询延迟影响很小 (<10ms)


数据写入进度对比:
────────────────

进度
100%│                    ╱              ╱─────────
    │                  ╱              ╱
    │                ╱              ╱
 90%│              ╱              ╱    ← target=0.9 的目标
    │            ╱              ╱
    │          ╱              ╱
 80%│        ╱              ╱
    │      ╱              ╱
    │    ╱              ╱
 60%│  ╱              ╱
    │╱              ╱
 40%│             ╱                    ← target=0.1 的曲线
    │           ╱                         (30s内完成90%)
 20%│         ╱
    │       ╱
  0%└───────────────────────────────────────────> 时间
     0s     30s     60s     90s    180s    270s

     target=0.1: 快速上升，30s完成
     target=0.9: 平滑上升，270s完成


查询延迟影响:
────────────

延迟
(ms)
150│
   │  ████
   │  ████
100│  ████              ← target=0.1
   │  ████                延迟峰值: 120ms
   │  ████
 50│  ████      ▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃  ← target=0.9
   │  ████      ▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃     延迟峰值: 15ms
   │  ████      ▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃▃
  0└───────────────────────────────────────────> 时间
     0s     30s     60s     90s    180s    270s

总结:
─────
completion_target = 0.1:
  ✓ checkpoint 完成快 (30s)
  ✗ I/O 峰值高，影响其他查询
  ✗ 查询延迟峰值大 (100ms+)
  适合: 批处理系统，夜间维护窗口

completion_target = 0.9:
  ✓ I/O 平滑，对其他查询影响小
  ✓ 查询延迟稳定 (<15ms)
  ✗ checkpoint 耗时长 (270s)
  适合: OLTP 系统，24/7 在线服务
```

---

### 5.3 并发事务与 Checkpoint 交互

```
时间轴: 展示 3 个并发 Backend 和 Checkpointer 的交互
───────────────────────────────────────────────────

Backend 1      Backend 2      Backend 3      Checkpointer     Shared Buffers
    │              │              │              │                  │
T=0s│              │              │              │                  │
    ├─ INSERT     │              │              │                  │
    │  rel=16384, │              │              │                  │
    │  blk=100    │              │              │                  │
    │  MarkBufferDirty() ───────────────────────────────────────>  │
    │              │              │              │       [blk=100 DIRTY]
    │              │              │              │                  │
T=1s│              ├─ UPDATE     │              │                  │
    │              │  rel=16384, │              │                  │
    │              │  blk=200    │              │                  │
    │              │  MarkBufferDirty() ────────────────────────>  │
    │              │              │              │       [blk=200 DIRTY]
    │              │              │              │                  │
T=2s│              │              ├─ DELETE     │                  │
    │              │              │  rel=16384, │                  │
    │              │              │  blk=300    │                  │
    │              │              │  MarkBufferDirty() ──────────>  │
    │              │              │              │       [blk=300 DIRTY]
    │              │              │              │                  │
T=5s│              │              │              ├─ Checkpoint触发  │
    │              │              │              │  CreateCheckPoint()
    │              │              │              │                  │
    │              │              │              ├─ 扫描脏页        │
    │              │              │              │  发现:           │
    │              │              │              │  • blk=100 DIRTY │
    │              │              │              │  • blk=200 DIRTY │
    │              │              │              │  • blk=300 DIRTY │
    │              │              │              │                  │
T=6s│              │              │              ├─ 写blk=100      │
    │              │              │              │  FlushBuffer()   │
    │              │              │              ├─────────────────>│
    │              │              │              │       [blk=100 CLEAN]
    │              │              │              │                  │
    │  COMMIT      │              │              │                  │
    │  (事务完成)   │              │              │                  │
    │  XLogFlush() │              │              │                  │
    │              │              │              │                  │
T=7s│              │              │              ├─ 写blk=200      │
    │              │              │              │  FlushBuffer()   │
    │              │              │              ├─────────────────>│
    │              │              │              │       [blk=200 CLEAN]
    │              │              │              │                  │
    │              │  COMMIT      │              │                  │
    │              │  (事务完成)   │              │                  │
    │              │              │              │                  │
T=8s│              │              │              ├─ 写blk=300      │
    │              │              │              │  FlushBuffer()   │
    │              │              │              ├─────────────────>│
    │              │              │              │       [blk=300 CLEAN]
    │              │              │              │                  │
    │  INSERT     │              │              │  [同时新事务继续] │
    │  rel=16384, │              │              │                  │
    │  blk=100    │              │              │                  │
    │  (重新变脏)  │              │              │                  │
    │  MarkBufferDirty() ───────────────────────────────────────>  │
    │              │              │              │       [blk=100 DIRTY]
    │              │              │              │       (但不影响本次checkpoint)
    │              │              │              │                  │
T=30s│              │              │              ├─ 所有脏页写完   │
    │              │              │              │                  │
    │              │              │              ├─ Fsync          │
    │              │              │              │  smgr_fsync(16384)
    │              │              │              │                  │
T=32s│              │              │  COMMIT      │                  │
    │              │              │  (事务完成)   │                  │
    │              │              │              │                  │
    │              │              │              ├─ 写WAL+pg_control
    │              │              │              ├─ Checkpoint完成  │
    │              │              │              │                  │
    │              │              │              │  [blk=100仍是DIRTY]
    │              │              │              │  (等待下次checkpoint)
    ↓              ↓              ↓              ↓                  ↓

关键点:
───────
1. ✅ Checkpoint 只写在扫描时已经是脏的页
   • blk=100, 200, 300 在 T=5s 时已脏，会被写入
   • blk=100 在 T=8s 重新变脏，不影响本次 checkpoint

2. ✅ Backend 和 Checkpointer 可以并发
   • Backend 1,2,3 在 checkpoint 期间继续工作
   • MarkBufferDirty() 只修改内存标志，不阻塞

3. ✅ 事务不需要等待 checkpoint 完成
   • Backend 1 在 T=6s COMMIT，不等 checkpoint
   • 只需等待 WAL 刷盘，不等数据页刷盘

4. ⚠️ Checkpoint 期间新变脏的页不包含在本次 checkpoint
   • 它们会等到下次 checkpoint
   • 这就是为什么需要 WAL - 保证这些修改不丢失
```

---

## 6. 算法可视化

### 6.1 Min-heap 负载均衡算法

```
场景: 3 个表空间，脏页分布不均
─────────────────────────────────

表空间统计:
  ts=1663  (默认表空间): 5000 个脏页
  ts=16387 (用户表空间A): 3000 个脏页
  ts=16388 (用户表空间B): 2000 个脏页

总计: 10000 个脏页

初始化 Min-Heap:
─────────────────

每个表空间的 progress_slice = 总页数 / 该表空间页数

  ts=1663:  progress_slice = 10000 / 5000 = 2.0
  ts=16387: progress_slice = 10000 / 3000 = 3.33
  ts=16388: progress_slice = 10000 / 2000 = 5.0

堆结构 (按 progress 排序，最小的在顶部):

           [ts=1663, progress=0.0, slice=2.0]  ← 堆顶
              /                          \
   [ts=16387, p=0.0, s=3.33]      [ts=16388, p=0.0, s=5.0]

执行过程:
─────────

Step 1: 取出堆顶 (progress 最小)
  → ts=1663, progress=0.0
  写入 ts=1663 的第 1 个页
  progress += slice → 0.0 + 2.0 = 2.0

  更新堆:
           [ts=16387, progress=0.0, slice=3.33]  ← 新堆顶
              /                          \
   [ts=1663, p=2.0, s=2.0]        [ts=16388, p=0.0, s=5.0]

Step 2: 取出堆顶
  → ts=16387, progress=0.0
  写入 ts=16387 的第 1 个页
  progress += slice → 0.0 + 3.33 = 3.33

  更新堆:
           [ts=16388, progress=0.0, slice=5.0]  ← 新堆顶
              /                          \
   [ts=1663, p=2.0, s=2.0]        [ts=16387, p=3.33, s=3.33]

Step 3: 取出堆顶
  → ts=16388, progress=0.0
  写入 ts=16388 的第 1 个页
  progress += slice → 0.0 + 5.0 = 5.0

  更新堆:
           [ts=1663, progress=2.0, slice=2.0]  ← 新堆顶
              /                          \
   [ts=16387, p=3.33, s=3.33]     [ts=16388, p=5.0, s=5.0]

Step 4-10: 继续...
  写入顺序: 1663, 16387, 16388, 1663, 1663, 16387, 16388, 1663, ...

完整写入序列 (前 20 个):
─────────────────────────
idx  ts     progress_after   说明
───  ─────  ──────────────   ─────────────────
 1   1663   2.0              第一轮，均匀开始
 2   16387  3.33
 3   16388  5.0
 4   1663   4.0              ts=1663 进度最小，继续
 5   1663   6.0
 6   16387  6.66             ts=16387 赶上
 7   1663   8.0
 8   16388  10.0             ts=16388 赶上
 9   1663   10.0             三者接近
10   16387  10.0
11   1663   12.0
12   1663   14.0
13   16387  13.33
14   16388  15.0
15   1663   16.0
...

progress 曲线可视化:
────────────────────

progress
100│                               ╱────  ts=16388 (2000页，最少)
   │                             ╱
   │                          ╱─────────  ts=16387 (3000页，中等)
 80│                       ╱
   │                    ╱
   │                 ╱────────────────────  ts=1663 (5000页，最多)
 60│              ╱
   │           ╱
   │        ╱
 40│     ╱
   │  ╱
 20│╱
   │
  0└───────────────────────────────────────────> 写入页数
    0    1000  2000  3000  4000  5000  6000  10000

✅ 三条曲线接近平行，说明 I/O 负载均衡!

对比: 如果不用 Min-Heap (按表空间顺序写):
────────────────────────────────────────────

写入顺序: [ts=1663 × 5000] → [ts=16387 × 3000] → [ts=16388 × 2000]

progress
100│                                         ╱─  ts=16388
   │                                      ╱─────  ts=16387
   │                                   ╱──────────  ts=1663
 80│
   │
 60│
   │
 40│───────────────────────────
   │
 20│───────────────────────────
   │
  0└───────────────────────────────────────────> 写入页数
    0    5000  8000  10000

✗ ts=1663 长时间独占 I/O，ts=16388 完全空闲
✗ 如果三个表空间在不同磁盘，无法并行利用

Min-Heap 算法复杂度:
────────────────────
  初始化: O(K)，K = 表空间数量 (通常 < 10)
  每次写入: O(log K) (堆调整)
  总复杂度: O(N log K)，N = 脏页数量

  K 很小 (< 10)，所以 log K ≈ 3-4
  几乎等于 O(N)，开销很小!
```

---

### 6.2 IsCheckpointOnSchedule 判断逻辑

```
IsCheckpointOnSchedule(double progress) 决策树
──────────────────────────────────────────────

输入参数:
  progress: 当前写入进度 (0.0 - 1.0)
  例如: 已写 5000/10000 页 → progress = 0.5

配置参数:
  checkpoint_timeout = 300s (5分钟)
  max_wal_size = 1GB
  checkpoint_completion_target = 0.9

                     ┌─────────────────────┐
                     │ IsCheckpointOn      │
                     │ Schedule(progress)  │
                     └──────────┬──────────┘
                                │
                                ↓
                  ┌─────────────────────────────┐
                  │ 计算调整后进度:             │
                  │ adj_progress = progress *   │
                  │   checkpoint_completion_    │
                  │   target                    │
                  │                             │
                  │ 例: 0.5 * 0.9 = 0.45        │
                  └──────────┬──────────────────┘
                             │
                             ↓
                  ┌──────────────────────────────┐
                  │ 获取时间维度进度:            │
                  │ elapsed_time = now -         │
                  │   checkpoint_start_time      │
                  │ target_time = timeout * 0.9  │
                  │   = 270s                     │
                  │ time_progress = elapsed /    │
                  │   target                     │
                  │                              │
                  │ 例: 已过 150s → 150/270=0.56 │
                  └──────────┬───────────────────┘
                             │
                             ↓
                ┌─────────── 判断 1 ────────────┐
                │ adj_progress < time_progress? │
                │ 0.45 < 0.56?                  │
                └──────┬────────────────────┬───┘
                      Yes                  No
                       │                    │
                       ↓                    ↓
              ┌──────────────┐      继续判断WAL进度
              │ return false │
              │ (落后进度，   │
              │  全速写入)    │
              └──────────────┘
                                             │
                                             ↓
                            ┌────────────────────────────┐
                            │ 获取 WAL 维度进度:         │
                            │ elapsed_xlogs = current_   │
                            │   wal_lsn - redo_lsn       │
                            │ target_xlogs = max_wal *   │
                            │   0.9 = 0.9GB              │
                            │ wal_progress = elapsed /   │
                            │   target                   │
                            │                            │
                            │ 例: 已生成 400MB → 400/900 │
                            │ = 0.44                     │
                            └──────────┬─────────────────┘
                                       │
                                       ↓
                          ┌────────── 判断 2 ──────────┐
                          │ adj_progress < wal_progress?│
                          │ 0.45 < 0.44?                │
                          └──────┬────────────────┬─────┘
                                Yes              No
                                 │                │
                                 ↓                ↓
                        ┌──────────────┐  ┌────────────┐
                        │ return false │  │return true │
                        │ (WAL落后，   │  │(进度正常或 │
                        │  全速写入)    │  │ 超前，可以 │
                        └──────────────┘  │ 睡眠)      │
                                          └────────────┘

决策示例:
─────────

场景 1: 进度超前
  progress = 0.5 (已写 50%)
  elapsed_time = 100s (过去 37% 时间)
  elapsed_wal = 300MB (生成 33% WAL)

  adj_progress = 0.5 * 0.9 = 0.45
  time_progress = 100 / 270 = 0.37
  wal_progress = 300 / 900 = 0.33

  判断 1: 0.45 < 0.37? No
  判断 2: 0.45 < 0.33? No
  → return true (可以睡眠，写太快了)

场景 2: 进度正常
  progress = 0.5 (已写 50%)
  elapsed_time = 135s (过去 50% 时间)
  elapsed_wal = 450MB (生成 50% WAL)

  adj_progress = 0.5 * 0.9 = 0.45
  time_progress = 135 / 270 = 0.50
  wal_progress = 450 / 900 = 0.50

  判断 1: 0.45 < 0.50? Yes
  → return false (不睡眠，保持当前速度)

场景 3: 进度落后 (I/O 慢)
  progress = 0.3 (已写 30%)
  elapsed_time = 150s (过去 56% 时间)
  elapsed_wal = 500MB (生成 56% WAL)

  adj_progress = 0.3 * 0.9 = 0.27
  time_progress = 150 / 270 = 0.56
  wal_progress = 500 / 900 = 0.56

  判断 1: 0.27 < 0.56? Yes
  → return false (全速写入，追赶进度)

可视化决策边界:
────────────────

写入进度
  1.0│
     │                         允许睡眠区域
     │                    ╱────────────────
  0.9│                 ╱
     │              ╱
  0.8│           ╱
     │        ╱            ← 决策边界
  0.7│     ╱                (斜率 = completion_target)
     │  ╱
  0.6│╱
     │     全速写入区域
  0.5│
     │
  0.4│
     │
  0.3│
     │
  0.2│
     │
  0.1│
     │
  0.0└────────────────────────────────────> 时间进度
     0.0  0.2  0.4  0.6  0.8  1.0

     落在边界上方 → 可以睡眠
     落在边界下方 → 全速写入
```

---

### 6.3 Fsync Request Queue 压缩

```
Fsync Request Queue 的作用和压缩算法
───────────────────────────────────

背景:
  每个 Backend 修改页面后，需要记录哪些文件需要 fsync
  这些请求存储在共享内存的 Fsync Request Queue 中
  Checkpoint 时，Checkpointer 批量处理这些请求

问题:
  同一个文件可能被多次请求 fsync
  例如: [file A], [file B], [file A], [file A], [file C]
  如果不压缩，会对同一个文件 fsync 多次 (浪费)

压缩算法:
──────────

队列结构 (环形缓冲区):
  ┌─────────────────────────────────────────────┐
  │ [req1] [req2] [req3] [req4] [req5] [...] [reqN] │
  │   ↑                                     ↑       │
  │  head                                  tail     │
  └─────────────────────────────────────────────────┘

每个请求包含:
  typedef struct {
      RelFileNode rnode;  ← (spcNode, dbNode, relNode)
      ForkNumber forknum; ← MAIN/FSM/VM/INIT
      BlockNumber segno;  ← 段号 (每段 1GB)
  } FsyncRequest;

压缩前 (示例):
  [ {16384, MAIN, 0},   ← file A, seg 0
    {16385, MAIN, 0},   ← file B, seg 0
    {16384, MAIN, 0},   ← file A, seg 0 (重复!)
    {16384, MAIN, 1},   ← file A, seg 1
    {16385, MAIN, 0},   ← file B, seg 0 (重复!)
    {16386, MAIN, 0},   ← file C, seg 0
    {16384, MAIN, 0} ]  ← file A, seg 0 (又重复!)

压缩算法伪代码:
────────────────

CompactCheckpointerRequestQueue():
  1. 分配哈希表 (key = FsyncRequest, value = bool)

  2. 遍历队列:
     for req in queue:
         if req not in hash_table:
             hash_table[req] = true
             keep req in queue
         else:
             remove req from queue  ← 压缩: 删除重复

  3. 返回压缩后的队列

压缩后:
  [ {16384, MAIN, 0},   ← file A, seg 0 (保留1个)
    {16385, MAIN, 0},   ← file B, seg 0 (保留1个)
    {16384, MAIN, 1},   ← file A, seg 1
    {16386, MAIN, 0} ]  ← file C, seg 0

  从 7 个请求压缩到 4 个! 节省 43% fsync 调用

哈希表实现 (PostgreSQL dynahash):
──────────────────────────────────

  hash_create("fsync_request_hash",
              queue_size,
              &ctl,
              HASH_ELEM | HASH_BLOBS);

  哈希键: FsyncRequest 结构本身 (24 bytes)
  哈希函数: tag_hash (CRC32)

  查找时间: O(1) 平均
  插入时间: O(1) 平均
  总复杂度: O(N)，N = 队列长度

压缩效果统计:
────────────

典型场景 (shared_buffers=8GB, 高并发写入):

  压缩前队列大小: 10000 个请求
  压缩后队列大小: 2500 个请求
  压缩比: 75%

  fsync 调用次数: 从 10000 减少到 2500
  节省时间: 7500 × 7ms = 52.5 秒!

  ✅ 对 checkpoint 性能提升显著

压缩时机:
─────────

1. AbsorbFsyncRequests() - Backend 添加请求时
   • 周期性压缩 (每 10000 个请求)

2. ProcessFsyncRequests() - Checkpoint 处理前
   • 最终压缩，确保无重复

3. CheckpointWriteDelay() - 渐进式写入期间
   • 吸收挂起的请求并压缩

可视化:
───────

Before Compression:
  Queue: [A][B][A][C][B][A][D][C][E][A]
         ↑                             ↑
        head                          tail

  Hash Table (empty)

After Compression:
  Queue: [A][B][C][D][E]
         ↑           ↑
        head        tail

  Hash Table:
    A → true
    B → true
    C → true
    D → true
    E → true

  Fsync 调用: 5 次 (而非 10 次)
```

---

## 7. 监控视图

### 7.1 pg_stat_bgwriter 指标关系

```
pg_stat_bgwriter 视图结构和指标关系
────────────────────────────────────

SELECT * FROM pg_stat_bgwriter;

┌──────────────────────────────────────────────────────────────────┐
│                      pg_stat_bgwriter                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [Checkpoint 统计]                                                │
│  ┌────────────────────────────────────────────────────┐          │
│  │ checkpoints_timed     BIGINT  ← 时间触发的checkpoint数  │
│  │ checkpoints_req       BIGINT  ← 请求触发的checkpoint数  │
│  │ checkpoint_write_time DOUBLE  ← 写入耗时 (毫秒)         │
│  │ checkpoint_sync_time  DOUBLE  ← fsync耗时 (毫秒)        │
│  └────────────────────────────────────────────────────┘          │
│                         │                                         │
│                         ↓                                         │
│              计算 checkpoint 频率和效率                           │
│                                                                   │
│  [Buffer 写入统计]                                                │
│  ┌────────────────────────────────────────────────────┐          │
│  │ buffers_checkpoint  BIGINT  ← checkpoint写入的缓冲区数│
│  │ buffers_clean       BIGINT  ← bgwriter清理的缓冲区数 │
│  │ buffers_backend     BIGINT  ← backend自己写的缓冲区数│
│  │ buffers_alloc       BIGINT  ← 分配的缓冲区总数      │
│  └────────────────────────────────────────────────────┘          │
│                         │                                         │
│                         ↓                                         │
│              计算缓冲区写入分布和效率                             │
│                                                                   │
│  [异常指标]                                                       │
│  ┌────────────────────────────────────────────────────┐          │
│  │ buffers_backend_fsync BIGINT  ← ⚠️ backend被迫fsync │
│  │ maxwritten_clean      BIGINT  ← bgwriter达到最大值次数│
│  └────────────────────────────────────────────────────┘          │
│                         │                                         │
│                         ↓                                         │
│              识别性能瓶颈和配置问题                               │
│                                                                   │
│  [统计元数据]                                                     │
│  ┌────────────────────────────────────────────────────┐          │
│  │ stats_reset  TIMESTAMP  ← 统计重置时间              │
│  └────────────────────────────────────────────────────┘          │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘

关键指标关系:
─────────────

1. Checkpoint 健康度:
   ────────────────────
   健康比例 = checkpoints_timed / (checkpoints_timed + checkpoints_req)

   ✅ > 0.9  : 健康 (大部分由时间触发)
   ⚠️  0.5-0.9: 警告 (考虑增大 max_wal_size)
   ❌ < 0.5  : 严重 (WAL 大小不足)

2. 写入分布:
   ──────────
   总写入 = buffers_checkpoint + buffers_clean + buffers_backend

   checkpoint占比 = buffers_checkpoint / 总写入
   bgwriter占比   = buffers_clean / 总写入
   backend占比    = buffers_backend / 总写入

   理想分布:
     checkpoint: 70-80%  ← 大部分由checkpoint写
     bgwriter:   15-25%  ← bgwriter预清理
     backend:    0-5%    ← backend很少自己写

   问题分布:
     backend > 10%  ← shared_buffers 或 bgwriter 不足

3. Fsync 效率:
   ───────────
   fsync占比 = checkpoint_sync_time /
               (checkpoint_write_time + checkpoint_sync_time)

   ✅ < 10%  : writeback 工作良好
   ⚠️  10-30%: 考虑调整 checkpoint_flush_after
   ❌ > 30%  : I/O 系统瓶颈

4. 异常检测:
   ──────────
   if (buffers_backend_fsync > 0):
       ⚠️ Backend 被迫执行 fsync!
       原因: Fsync request queue 满了
       建议: 增大 shared_buffers 或降低写入速度

   if (maxwritten_clean > checkpoints_req * 0.1):
       ⚠️ BgWriter 频繁达到单次清理上限
       建议: 增大 bgwriter_max_pages_per_round

监控查询示例:
────────────

-- 查询 1: Checkpoint 健康度
SELECT
    checkpoints_timed AS timed,
    checkpoints_req AS requested,
    round(100.0 * checkpoints_timed /
          NULLIF(checkpoints_timed + checkpoints_req, 0), 2) AS timed_pct,
    CASE
        WHEN checkpoints_timed::float /
             NULLIF(checkpoints_timed + checkpoints_req, 0) > 0.9
            THEN '✅ 健康'
        WHEN checkpoints_timed::float /
             NULLIF(checkpoints_timed + checkpoints_req, 0) > 0.5
            THEN '⚠️ 警告'
        ELSE '❌ 严重'
    END AS health
FROM pg_stat_bgwriter;

预期输出:
 timed | requested | timed_pct | health
-------+-----------+-----------+--------
    45 |         5 |     90.00 | ✅ 健康

-- 查询 2: 写入分布分析
SELECT
    buffers_checkpoint AS ckpt_buf,
    buffers_clean AS bgw_buf,
    buffers_backend AS backend_buf,
    round(100.0 * buffers_checkpoint /
          NULLIF(buffers_checkpoint + buffers_clean + buffers_backend, 0), 2)
        AS ckpt_pct,
    round(100.0 * buffers_backend /
          NULLIF(buffers_checkpoint + buffers_clean + buffers_backend, 0), 2)
        AS backend_pct,
    CASE
        WHEN buffers_backend::float /
             NULLIF(buffers_checkpoint + buffers_clean + buffers_backend, 0) < 0.05
            THEN '✅ 理想'
        WHEN buffers_backend::float /
             NULLIF(buffers_checkpoint + buffers_clean + buffers_backend, 0) < 0.10
            THEN '⚠️ 注意'
        ELSE '❌ 需优化'
    END AS status
FROM pg_stat_bgwriter;

预期输出:
 ckpt_buf | bgw_buf | backend_buf | ckpt_pct | backend_pct | status
----------+---------+-------------+----------+-------------+--------
  1500000 |  300000 |       15000 |    82.64 |        0.83 | ✅ 理想

-- 查询 3: Fsync 效率
SELECT
    round(checkpoint_sync_time / 1000, 2) AS sync_sec,
    round(checkpoint_write_time / 1000, 2) AS write_sec,
    round(100.0 * checkpoint_sync_time /
          NULLIF(checkpoint_write_time + checkpoint_sync_time, 0), 2)
        AS sync_pct,
    CASE
        WHEN checkpoint_sync_time /
             NULLIF(checkpoint_write_time + checkpoint_sync_time, 0) < 0.1
            THEN '✅ 优秀'
        WHEN checkpoint_sync_time /
             NULLIF(checkpoint_write_time + checkpoint_sync_time, 0) < 0.3
            THEN '⚠️ 一般'
        ELSE '❌ 差'
    END AS efficiency
FROM pg_stat_bgwriter;

预期输出:
 sync_sec | write_sec | sync_pct | efficiency
----------+-----------+----------+------------
    12.50 |    145.30 |     7.92 | ✅ 优秀

-- 查询 4: 异常检测
SELECT
    buffers_backend_fsync AS backend_fsync,
    maxwritten_clean,
    checkpoints_req,
    CASE
        WHEN buffers_backend_fsync > 0
            THEN '❌ Backend被迫fsync，增大shared_buffers'
        WHEN maxwritten_clean > checkpoints_req * 0.1
            THEN '⚠️ BgWriter达到上限过多，调整bgwriter_max_pages'
        ELSE '✅ 正常'
    END AS diagnosis
FROM pg_stat_bgwriter;

预期输出:
 backend_fsync | maxwritten_clean | checkpoints_req | diagnosis
---------------+------------------+-----------------+-----------
             0 |               23 |               5 | ✅ 正常
```

---

### 7.2 Checkpoint 性能仪表板

```
PostgreSQL Checkpoint 监控仪表板布局
──────────────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│ PostgreSQL Checkpoint 监控仪表板                                     │
│ 刷新间隔: 10s                              最后更新: 2025-01-15 10:30│
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ ┌─────────────────────┐  ┌─────────────────────┐  ┌──────────────┐ │
│ │  Checkpoint 频率     │  │  当前状态           │  │  WAL 生成率  │ │
│ ├─────────────────────┤  ├─────────────────────┤  ├──────────────┤ │
│ │  总计: 50 次         │  │  状态: IDLE         │  │  150 MB/min  │ │
│ │  时间触发: 45 (90%) │  │  上次: 5分钟前      │  │              │ │
│ │  请求触发: 5 (10%)  │  │  进度: N/A          │  │  [=====>   ] │ │
│ │                     │  │                     │  │  62% of max  │ │
│ │  健康度: ✅ 优秀    │  │  下次预计: 2min后   │  │              │ │
│ └─────────────────────┘  └─────────────────────┘  └──────────────┘ │
│                                                                      │
│ ┌──────────────────────────────────────────────────────────────────┐│
│ │  Checkpoint 耗时趋势 (最近 10 次)                                 ││
│ ├──────────────────────────────────────────────────────────────────┤│
│ │ 耗时(s)                                                           ││
│ │   40│                      ●                                     ││
│ │     │                                                             ││
│ │   35│            ●                                               ││
│ │     │      ●              ●     ●                                ││
│ │   30│                               ●     ●     ●     ●     ●   ││
│ │     │ ●                                                           ││
│ │   25│                                                             ││
│ │     └────┬────┬────┬────┬────┬────┬────┬────┬────┬────>        ││
│ │         #1   #2   #3   #4   #5   #6   #7   #8   #9  #10         ││
│ │                                                                   ││
│ │  平均耗时: 32.5s  |  最大: 38.2s  |  最小: 28.1s                  ││
│ └──────────────────────────────────────────────────────────────────┘│
│                                                                      │
│ ┌─────────────────────┐  ┌─────────────────────┐  ┌──────────────┐ │
│ │  写入分布           │  │  I/O 效率           │  │  队列状态    │ │
│ ├─────────────────────┤  ├─────────────────────┤  ├──────────────┤ │
│ │  █████████ 82.6%    │  │  写入时间: 29.0s    │  │ Fsync队列:   │ │
│ │  Checkpoint         │  │  Fsync时间: 2.0s    │  │  2,500 请求  │ │
│ │                     │  │                     │  │              │ │
│ │  ███ 15.2%          │  │  Fsync占比: 6.5%    │  │ 压缩率: 75%  │ │
│ │  BgWriter           │  │  ✅ Writeback工作良好│  │              │ │
│ │                     │  │                     │  │ Backend:     │ │
│ │  ▌2.2%              │  │  吞吐量: 170 MB/s   │  │  0 fsync     │ │
│ │  Backend ✅         │  │                     │  │  ✅ 无瓶颈   │ │
│ └─────────────────────┘  └─────────────────────┘  └──────────────┘ │
│                                                                      │
│ ┌──────────────────────────────────────────────────────────────────┐│
│ │  脏页趋势 (最近 1 小时)                                           ││
│ ├──────────────────────────────────────────────────────────────────┤│
│ │ 脏页数                                                            ││
│ │ 80K│     ╱╲       ╱╲       ╱╲       ╱╲       ╱╲       ╱╲        ││
│ │    │    ╱  ╲     ╱  ╲     ╱  ╲     ╱  ╲     ╱  ╲     ╱  ╲       ││
│ │ 60K│   ╱    ╲   ╱    ╲   ╱    ╲   ╱    ╲   ╱    ╲   ╱    ╲      ││
│ │    │  ╱      ╲ ╱      ╲ ╱      ╲ ╱      ╲ ╱      ╲ ╱      ╲     ││
│ │ 40K│ ╱        ╲        ╲        ╲        ╲        ╲        ╲    ││
│ │    │╱          ╲        ╲        ╲        ╲        ╲        ╲   ││
│ │ 20K│            ╲        ╲        ╲        ╲        ╲        ╲  ││
│ │    └────────────┴────────┴────────┴────────┴────────┴────────┴──>││
│ │    10:00   10:10  10:20  10:30  10:40  10:50  11:00  11:10  现在││
│ │         ↑         ↑         ↑         ↑         ↑         ↑      ││
│ │      checkpoint每5分钟触发一次，脏页周期性下降                   ││
│ └──────────────────────────────────────────────────────────────────┘│
│                                                                      │
│ ┌──────────────────────────────────────────────────────────────────┐│
│ │  警告和建议                                                       ││
│ ├──────────────────────────────────────────────────────────────────┤│
│ │  ✅ 所有指标正常，无需调整                                        ││
│ │                                                                   ││
│ │  ℹ️  提示:                                                        ││
│ │  • Checkpoint 频率健康 (90% 时间触发)                            ││
│ │  • I/O 分布合理 (83% checkpoint, 15% bgwriter, 2% backend)       ││
│ │  • Fsync 效率优秀 (占比 6.5%)                                    ││
│ │  • 无 backend fsync，队列无瓶颈                                  ││
│ └──────────────────────────────────────────────────────────────────┘│
│                                                                      │
│ ┌──────────────────────────────────────────────────────────────────┐│
│ │  配置参数                                                         ││
│ ├──────────────────────────────────────────────────────────────────┤│
│ │  checkpoint_timeout: 300s  |  max_wal_size: 1GB                  ││
│ │  checkpoint_completion_target: 0.9  |  shared_buffers: 8GB       ││
│ │  checkpoint_flush_after: 256kB  |  bgwriter_delay: 200ms         ││
│ └──────────────────────────────────────────────────────────────────┘│
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

SQL 查询生成仪表板数据:
──────────────────────

-- 查询 1: 基础统计
WITH stats AS (
    SELECT
        checkpoints_timed,
        checkpoints_req,
        checkpoint_write_time,
        checkpoint_sync_time,
        buffers_checkpoint,
        buffers_clean,
        buffers_backend,
        buffers_backend_fsync
    FROM pg_stat_bgwriter
)
SELECT
    json_build_object(
        'total_checkpoints', checkpoints_timed + checkpoints_req,
        'timed_pct', round(100.0 * checkpoints_timed /
                          NULLIF(checkpoints_timed + checkpoints_req, 0), 2),
        'avg_write_time_sec', round(checkpoint_write_time / 1000 /
                                    NULLIF(checkpoints_timed + checkpoints_req, 0), 2),
        'fsync_pct', round(100.0 * checkpoint_sync_time /
                          NULLIF(checkpoint_write_time + checkpoint_sync_time, 0), 2),
        'backend_fsync_count', buffers_backend_fsync
    ) AS dashboard_data
FROM stats;

-- 查询 2: 脏页数量
SELECT count(*) AS dirty_buffers
FROM pg_buffercache
WHERE isdirty;

-- 查询 3: WAL 生成速率 (需要历史数据)
-- 通过定期记录 pg_current_wal_lsn() 计算

仪表板实现建议:
────────────────
1. 使用 Grafana + PostgreSQL datasource
2. 每 10 秒查询一次 pg_stat_bgwriter
3. 存储历史数据到时序数据库 (TimescaleDB)
4. 设置告警规则:
   • timed_pct < 80% → 警告
   • buffers_backend_fsync > 0 → 严重
   • fsync_pct > 20% → 警告
```

---

## 总结

本文档提供了 **18 个完整的可视化图表**，覆盖 PostgreSQL Checkpoint 机制的各个方面:

### 架构图 (3个)
1. PostgreSQL 进程架构 - 展示各进程关系
2. Checkpoint 相关内存布局 - 共享内存结构
3. 表空间和文件系统布局 - 磁盘文件组织

### 流程图 (5个)
1. CreateCheckPoint 主流程 - 13 个阶段详解
2. BufferSync 流程 - 脏页处理算法
3. CheckpointWriteDelay 调度 - 渐进式控制
4. SLRU Checkpoint 流程 - 元数据刷写
5. WAL 日志写入流程 - 事务日志处理

### 状态转换图 (3个)
1. Checkpoint 状态机 - 进程状态转换
2. 缓冲区状态转换 - Buffer 生命周期
3. WAL 段文件生命周期 - 文件状态管理

### 数据流图 (3个)
1. 写事务数据流 - 从 Backend 到磁盘
2. Checkpoint 数据流 - 完整刷写过程
3. 崩溃恢复数据流 - REDO 恢复流程

### 时序图 (3个)
1. 正常 Checkpoint 时序 - 各阶段时间分布
2. 渐进式 Checkpoint I/O 模式 - completion_target 对比
3. 并发事务与 Checkpoint 交互 - 多进程协作

### 算法可视化 (3个)
1. Min-heap 负载均衡算法 - 表空间 I/O 均衡
2. IsCheckpointOnSchedule 判断逻辑 - 进度控制决策树
3. Fsync Request Queue 压缩 - 请求去重优化

### 监控视图 (2个)
1. pg_stat_bgwriter 指标关系 - 统计视图解析
2. Checkpoint 性能仪表板 - 监控面板设计

---

**如何使用这些图表**:

1. **学习理解**: 按顺序阅读，从架构到流程，再到算法细节
2. **问题诊断**: 参考监控视图，定位性能瓶颈
3. **优化调整**: 结合算法可视化，理解参数影响
4. **代码阅读**: 配合源码位置标注，深入实现细节

---

**相关文档**:
- 返回 [01_checkpoint_overview.md](01_checkpoint_overview.md) 查看概述
- 返回 [03_implementation_flow.md](03_implementation_flow.md) 查看实现细节
- 返回 [05_performance_optimization.md](05_performance_optimization.md) 查看优化建议
- 返回 [06_verification_testcases.md](06_verification_testcases.md) 运行测试用例
- 返回 [README.md](README.md) 查看文档导航

**建议**: 将这些图表打印或保存为 PDF，作为 PostgreSQL Checkpoint 机制的可视化参考手册!
