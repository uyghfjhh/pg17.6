# PostgreSQL Checkpoint - 核心数据结构详解

## 目录

1. [CheckPoint 结构](#1-checkpoint-结构)
2. [CheckpointerShmemStruct 结构](#2-checkpointershmemstruct-结构)
3. [ControlFileData 结构](#3-controlfiledata-结构)
4. [XLogCtlData 结构](#4-xlogctldata-结构)
5. [CheckpointStatsData 结构](#5-checkpointstatsdata-结构)
6. [缓冲区相关结构](#6-缓冲区相关结构)
7. [数据结构关系图](#7-数据结构关系图)

---

## 1. CheckPoint 结构

**位置**: `src/include/catalog/pg_control.h:35`

### 1.1 完整定义

```c
typedef struct CheckPoint
{
    XLogRecPtr    redo;              /* 下一个可用的 RecPtr，即 REDO 起始点 */
    TimeLineID    ThisTimeLineID;    /* 当前时间线 ID */
    TimeLineID    PrevTimeLineID;    /* 前一个时间线 ID（时间线切换时不同）*/
    bool          fullPageWrites;    /* 当前 full_page_writes 设置 */
    int           wal_level;         /* 当前 wal_level */
    FullTransactionId nextXid;       /* 下一个空闲事务 ID */
    Oid           nextOid;           /* 下一个空闲对象 ID */
    MultiXactId   nextMulti;         /* 下一个空闲 MultiXactId */
    MultiXactOffset nextMultiOffset; /* 下一个空闲 MultiXact 偏移 */
    TransactionId oldestXid;         /* 集群范围最小 datfrozenxid */
    Oid           oldestXidDB;       /* 拥有 oldestXid 的数据库 */
    MultiXactId   oldestMulti;       /* 集群范围最小 datminmxid */
    Oid           oldestMultiDB;     /* 拥有 oldestMulti 的数据库 */
    pg_time_t     time;              /* checkpoint 时间戳 */
    TransactionId oldestCommitTsXid; /* 最老的有效提交时间戳 XID */
    TransactionId newestCommitTsXid; /* 最新的有效提交时间戳 XID */

    /*
     * 最老的运行中 XID。只用于从在线 checkpoint 初始化 hot standby 模式，
     * 所以只在 wal_level 是 replica 且是在线 checkpoint 时才计算。
     * 否则设置为 InvalidTransactionId。
     */
    TransactionId oldestActiveXid;
} CheckPoint;
```

### 1.2 字段详解

#### 1.2.1 redo (XLogRecPtr)

**作用**: 最关键的字段，指示崩溃恢复应从哪里开始重放 WAL。

**值的确定**:
- **Online Checkpoint**: 插入 `XLOG_CHECKPOINT_REDO` 记录的 LSN
- **Shutdown Checkpoint**: 当前 WAL 插入位置

**示例**:
```
WAL 日志:
[LSN 0/1000] 事务 A 修改
[LSN 0/2000] 事务 B 修改
[LSN 0/3000] XLOG_CHECKPOINT_REDO 记录  <-- redo 指向这里
[LSN 0/3100] 开始 CheckPointGuts()
[LSN 0/5000] XLOG_CHECKPOINT_ONLINE 记录
[LSN 0/6000] 事务 C 修改

崩溃恢复时:
- 从 LSN 0/3000 开始重放
- 跳过 0/1000 和 0/2000（已被 checkpoint 持久化）
```

#### 1.2.2 nextXid (FullTransactionId)

**作用**: 保存下一个可分配的事务 ID，恢复后从这里继续分配。

**结构**:
```c
typedef struct FullTransactionId
{
    uint64 value;  // 64 位完整事务 ID
} FullTransactionId;

// 拆分为 epoch 和 xid
epoch = value >> 32;  // 高 32 位
xid = value & 0xFFFFFFFF;  // 低 32 位
```

**重要性**:
- 防止事务 ID 回卷（wraparound）
- 确保恢复后不会重复分配事务 ID

**示例**:
```c
// Checkpoint 时
checkPoint.nextXid = TransamVariables->nextXid;  // 当前 1000

// 恢复后
TransamVariables->nextXid = checkPoint.nextXid;  // 从 1000 继续
```

#### 1.2.3 oldestXid / oldestXidDB

**作用**: 用于 VACUUM 和防止事务 ID 回卷。

**含义**:
- `oldestXid`: 集群中所有数据库的最老未冻结事务 ID
- `oldestXidDB`: 哪个数据库拥有这个最老 XID

**使用场景**:
```sql
-- 查看当前值
SELECT datname, datfrozenxid
FROM pg_database
ORDER BY datfrozenxid;

-- 如果某个数据库的 datfrozenxid 太老
-- 会触发 autovacuum 强制 VACUUM
-- 防止事务 ID 耗尽导致数据库停机
```

#### 1.2.4 oldestActiveXid

**作用**: Hot Standby 初始化时需要知道哪些事务仍在运行。

**计算时机**:
```c
// xlog.c:6948
if (!shutdown && XLogStandbyInfoActive())
    checkPoint.oldestActiveXid = GetOldestActiveTransactionId();
else
    checkPoint.oldestActiveXid = InvalidTransactionId;
```

**Hot Standby 使用**:
```
主库 Checkpoint 时:
- oldestActiveXid = 900 (有事务 900-999 在运行)

备库应用 Checkpoint:
- 知道事务 900 之前的都已提交
- 可以安全提供只读查询（不会看到不一致状态）
```

### 1.3 大小和存储

```c
sizeof(CheckPoint) = 72 字节

// 存储位置
1. WAL 日志中: 作为 XLOG_CHECKPOINT_ONLINE/SHUTDOWN 记录的数据部分
2. pg_control 文件中: 完整副本存储在 ControlFile->checkPointCopy
```

---

## 2. CheckpointerShmemStruct 结构

**位置**: `src/backend/postmaster/checkpointer.c:102`

### 2.1 完整定义

```c
typedef struct CheckpointerRequest
{
    SyncRequestType type;  /* 请求类型: SYNC_REQUEST, SYNC_UNLINK 等 */
    FileTag         ftag;  /* 文件标识 */
} CheckpointerRequest;

typedef struct CheckpointerShmemStruct
{
    pid_t  checkpointer_pid;  /* Checkpointer 进程的 PID (0 表示未启动) */

    slock_t ckpt_lck;         /* 保护所有 ckpt_* 字段的自旋锁 */

    int    ckpt_started;      /* checkpoint 开始计数器 */
    int    ckpt_done;         /* checkpoint 完成计数器 */
    int    ckpt_failed;       /* checkpoint 失败计数器 */

    int    ckpt_flags;        /* checkpoint 标志位 (xlog.h 中定义) */

    ConditionVariable start_cv;  /* ckpt_started 变化时发信号 */
    ConditionVariable done_cv;   /* ckpt_done 变化时发信号 */

    int    num_requests;      /* 当前 fsync 请求数量 */
    int    max_requests;      /* 请求数组的分配大小 */
    CheckpointerRequest requests[FLEXIBLE_ARRAY_MEMBER];  /* 请求队列 */
} CheckpointerShmemStruct;
```

### 2.2 通信协议详解

#### 2.2.1 Checkpoint 请求协议

**后端进程请求流程**:

```c
// 第 1 步: 设置标志并记录计数器
SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
old_started = CheckpointerShmem->ckpt_started;  // 例如: 10
old_failed = CheckpointerShmem->ckpt_failed;    // 例如: 1
CheckpointerShmem->ckpt_flags |= flags;         // OR 操作合并多个请求
SpinLockRelease(&CheckpointerShmem->ckpt_lck);

// 第 2 步: 发送信号
kill(CheckpointerShmem->checkpointer_pid, SIGINT);

// 第 3 步: 等待 checkpoint 开始 (可选，如果有 CHECKPOINT_WAIT)
ConditionVariablePrepareToSleep(&CheckpointerShmem->start_cv);
while (CheckpointerShmem->ckpt_started == old_started) {
    ConditionVariableSleep(&CheckpointerShmem->start_cv,
                          WAIT_EVENT_CHECKPOINT_START);
}
new_started = CheckpointerShmem->ckpt_started;  // 现在变成 11

// 第 4 步: 等待 checkpoint 完成
ConditionVariablePrepareToSleep(&CheckpointerShmem->done_cv);
while (CheckpointerShmem->ckpt_done < new_started) {  // ckpt_done 还是 10
    ConditionVariableSleep(&CheckpointerShmem->done_cv,
                          WAIT_EVENT_CHECKPOINT_DONE);
}
// ckpt_done 变成 11，循环退出

// 第 5 步: 检查是否失败
new_failed = CheckpointerShmem->ckpt_failed;
if (new_failed != old_failed)
    ERROR("checkpoint failed");
```

**Checkpointer 进程处理流程**:

```c
// 第 1 步: 检测到请求
if (CheckpointerShmem->ckpt_flags) {
    // 第 2 步: 原子地获取标志并递增计数器
    SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
    flags = CheckpointerShmem->ckpt_flags;    // 获取标志
    CheckpointerShmem->ckpt_flags = 0;        // 清空标志
    CheckpointerShmem->ckpt_started++;        // 10 -> 11
    SpinLockRelease(&CheckpointerShmem->ckpt_lck);

    // 第 3 步: 通知已开始
    ConditionVariableBroadcast(&CheckpointerShmem->start_cv);

    // 第 4 步: 执行 checkpoint
    CreateCheckPoint(flags);

    // 第 5 步: 通知完成
    SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
    CheckpointerShmem->ckpt_done = CheckpointerShmem->ckpt_started;  // 11
    SpinLockRelease(&CheckpointerShmem->ckpt_lck);

    ConditionVariableBroadcast(&CheckpointerShmem->done_cv);
}
```

**为什么使用计数器而非布尔值？**

```
场景: 两个后端同时请求 checkpoint

使用布尔值的问题:
Backend 1: 设置 requested = true, 等待 done = true
Backend 2: 设置 requested = true, 等待 done = true
Checkpointer: requested = true, 执行, 设置 done = true
结果: Backend 1 和 2 都认为完成了，但实际上只执行了一次

使用计数器的解决:
Backend 1: old_started = 10, 请求
Backend 2: old_started = 10, 请求
Checkpointer: ckpt_started 递增到 11, 执行
Backend 1: 检测到 ckpt_started (11) != old_started (10), 继续等待 ckpt_done >= 11
Backend 2: 同样等待 ckpt_done >= 11
Checkpointer: 完成后设置 ckpt_done = 11
结果: 两个后端都正确等待到同一次 checkpoint 完成
```

#### 2.2.2 Fsync 请求队列

**队列管理**:

```c
// 队列大小
CheckpointerShmem->max_requests = NBuffers;  // 默认等于缓冲区数量

// 后端添加请求
bool ForwardSyncRequest(const FileTag *ftag, SyncRequestType type)
{
    LWLockAcquire(CheckpointerCommLock, LW_EXCLUSIVE);

    // 队列满了？
    if (CheckpointerShmem->num_requests >= CheckpointerShmem->max_requests) {
        // 尝试压缩（去重）
        if (!CompactCheckpointerRequestQueue()) {
            LWLockRelease(CheckpointerCommLock);
            return false;  // 失败，后端需要自己 fsync
        }
    }

    // 添加请求（不检查重复）
    request = &CheckpointerShmem->requests[CheckpointerShmem->num_requests++];
    request->ftag = *ftag;
    request->type = type;

    // 队列超过一半？唤醒 checkpointer
    too_full = (CheckpointerShmem->num_requests >=
                CheckpointerShmem->max_requests / 2);

    LWLockRelease(CheckpointerCommLock);

    if (too_full && ProcGlobal->checkpointerLatch)
        SetLatch(ProcGlobal->checkpointerLatch);

    return true;
}
```

**Checkpointer 吸收请求**:

```c
void AbsorbSyncRequests(void)
{
    CheckpointerRequest *requests = NULL;
    int n;

    LWLockAcquire(CheckpointerCommLock, LW_EXCLUSIVE);

    // 复制请求数组
    n = CheckpointerShmem->num_requests;
    if (n > 0) {
        requests = palloc(n * sizeof(CheckpointerRequest));
        memcpy(requests, CheckpointerShmem->requests,
               n * sizeof(CheckpointerRequest));
    }

    START_CRIT_SECTION();  // 关键：清空后必须处理

    CheckpointerShmem->num_requests = 0;

    LWLockRelease(CheckpointerCommLock);

    // 处理所有请求
    for (request = requests; n > 0; request++, n--)
        RememberSyncRequest(&request->ftag, request->type);

    END_CRIT_SECTION();

    if (requests)
        pfree(requests);
}
```

### 2.3 内存布局

```
CheckpointerShmemStruct 在共享内存中的布局:

偏移     字段                   大小
------------------------------------------
0        checkpointer_pid       8 字节
8        ckpt_lck               1 字节
9        (padding)              3 字节
12       ckpt_started           4 字节
16       ckpt_done              4 字节
20       ckpt_failed            4 字节
24       ckpt_flags             4 字节
28       start_cv               24 字节
52       done_cv                24 字节
76       num_requests           4 字节
80       max_requests           4 字节
84       requests[0]            24 字节
108      requests[1]            24 字节
...
84+24*N  requests[N-1]          24 字节

总大小 = 84 + 24 * NBuffers
       = 84 + 24 * 16384 (默认)
       = 393,300 字节 ≈ 384 KB
```

---

## 3. ControlFileData 结构

**位置**: `src/include/catalog/pg_control.h:104`

### 3.1 完整定义 (精简版)

```c
typedef struct ControlFileData
{
    /* 唯一标识符 */
    uint64    system_identifier;

    /* 版本信息 */
    uint32    pg_control_version;    /* PG_CONTROL_VERSION */
    uint32    catalog_version_no;    /* catversion.h 中定义 */

    /* 系统状态数据 */
    DBState   state;                 /* DB_STARTUP, DB_IN_PRODUCTION 等 */
    pg_time_t time;                  /* 最后更新时间 */
    XLogRecPtr checkPoint;           /* 最新 checkpoint 记录的 LSN */

    CheckPoint checkPointCopy;       /* checkpoint 记录的完整副本 */

    XLogRecPtr unloggedLSN;          /* unlogged 表的伪 LSN */

    /* 恢复相关 */
    XLogRecPtr minRecoveryPoint;     /* 必须恢复到的最小点 */
    TimeLineID minRecoveryPointTLI;
    XLogRecPtr backupStartPoint;
    XLogRecPtr backupEndPoint;
    bool       backupEndRequired;

    /* 参数设置 (用于兼容性检查) */
    int        wal_level;
    bool       wal_log_hints;
    int        MaxConnections;
    int        max_worker_processes;
    int        max_wal_senders;
    int        max_prepared_xacts;
    int        max_locks_per_xact;
    bool       track_commit_timestamp;

    /* 硬件架构兼容性检查 */
    uint32     maxAlign;
    double     floatFormat;

    /* 数据库配置 */
    uint32     blcksz;              /* BLCKSZ */
    uint32     relseg_size;         /* RELSEG_SIZE */
    uint32     xlog_blcksz;         /* XLOG_BLCKSZ */
    uint32     xlog_seg_size;       /* wal_segment_size */
    uint32     nameDataLen;         /* NAMEDATALEN */
    uint32     indexMaxKeys;        /* INDEX_MAX_KEYS */
    uint32     toast_max_chunk_size;
    uint32     loblksize;
    bool       float8ByVal;
    uint32     data_checksum_version;

    /* 认证 nonce */
    char       mock_authentication_nonce[MOCK_AUTH_NONCE_LEN];

    /* CRC 校验和 (必须最后) */
    pg_crc32c  crc;
} ControlFileData;
```

### 3.2 关键字段详解

#### 3.2.1 system_identifier

**作用**: 集群的唯一标识符，防止混用不同集群的文件。

**生成**:
```c
// initdb 时生成
system_identifier = ((uint64) tv.tv_sec) << 32;
system_identifier |= ((uint64) tv.tv_usec) << 12;
system_identifier |= getpid() & 0xFFF;

// 示例: 7280272028926836736
```

**用途**:
- WAL 文件头部包含此 ID，启动时校验
- 防止误用其他集群的 WAL 文件
- 主备复制时校验一致性

#### 3.2.2 state (DBState)

**可能的值**:
```c
typedef enum DBState
{
    DB_STARTUP = 0,           /* 启动中 */
    DB_SHUTDOWNED,            /* 正常关闭 */
    DB_SHUTDOWNED_IN_RECOVERY,/* 恢复过程中关闭 */
    DB_SHUTDOWNING,           /* 正在关闭 */
    DB_IN_CRASH_RECOVERY,     /* 崩溃恢复中 */
    DB_IN_ARCHIVE_RECOVERY,   /* 归档恢复中 */
    DB_IN_PRODUCTION          /* 正常运行 */
} DBState;
```

**状态转换**:
```
1. 正常启动:
   DB_STARTUP -> DB_IN_PRODUCTION

2. 崩溃恢复启动:
   DB_STARTUP -> DB_IN_CRASH_RECOVERY -> DB_IN_PRODUCTION

3. PITR 恢复:
   DB_STARTUP -> DB_IN_ARCHIVE_RECOVERY -> DB_IN_PRODUCTION

4. 正常关闭:
   DB_IN_PRODUCTION -> DB_SHUTDOWNING -> DB_SHUTDOWNED

5. 崩溃:
   DB_IN_PRODUCTION -> (系统崩溃，状态未更新)
```

#### 3.2.3 checkPoint 和 checkPointCopy

**双重保险设计**:

```c
XLogRecPtr checkPoint;      // 最新 checkpoint 记录在 WAL 中的位置
CheckPoint checkPointCopy;  // checkpoint 记录的完整副本

// 启动恢复流程
1. 读取 pg_control，获取 checkPoint LSN
2. 打开 WAL 文件，定位到 checkPoint LSN
3. 读取 checkpoint 记录
4. 与 checkPointCopy 比较，检测损坏
5. 如果 WAL 损坏，使用 checkPointCopy
6. 从 checkpoint.redo 开始重放 WAL
```

**为什么需要副本？**
```
场景: WAL 文件损坏

没有副本:
- 无法读取 checkpoint 记录
- 不知道从哪里开始恢复
- 数据库无法启动

有副本:
- pg_control 中的 checkPointCopy 可用
- 仍能确定 redo 点
- 数据库可以恢复
```

#### 3.2.4 minRecoveryPoint

**作用**: 归档恢复时，必须恢复到此点才能启动。

**使用场景**:
```
场景: 从基础备份恢复

1. 开始恢复:
   minRecoveryPoint = 0/3000000 (备份结束时的 LSN)

2. 恢复过程中，每次刷写数据页:
   if (刷写的页面对应的 WAL LSN > minRecoveryPoint)
       minRecoveryPoint = 该 LSN;

3. 恢复完成检查:
   if (当前 LSN < minRecoveryPoint)
       ERROR("must recover to at least %X/%X", minRecoveryPoint);

原因: 如果刷写了 LSN 0/4000000 的页面到磁盘，
      但恢复只到 0/3500000 就停止，
      数据库会处于不一致状态。
```

### 3.3 文件格式和原子性

**文件大小**:
```c
#define PG_CONTROL_MAX_SAFE_SIZE  512   // 必须 <= 1 个磁盘扇区
#define PG_CONTROL_FILE_SIZE      8192  // 实际文件大小

sizeof(ControlFileData) = 约 280 字节
```

**原子性保证**:
```
1. 单扇区写入:
   - 数据结构 < 512 字节
   - 单次 write() 系统调用
   - 操作系统保证单扇区写入的原子性

2. CRC 校验:
   - 写入前计算 CRC
   - 读取时验证 CRC
   - 检测部分写入或损坏

3. 文件大小填充:
   - 文件总大小 8192 字节
   - 即使 PostgreSQL 升级，文件大小不变
   - 防止读取错误版本时出现读错误
```

**更新流程**:
```c
void UpdateControlFile(void)
{
    // 1. 计算 CRC (除 CRC 字段外的所有内容)
    INIT_CRC32C(ControlFile->crc);
    COMP_CRC32C(ControlFile->crc, ControlFile, offsetof(ControlFileData, crc));
    FIN_CRC32C(ControlFile->crc);

    // 2. 打开文件
    fd = open(XLOG_CONTROL_FILE, O_RDWR | PG_BINARY);

    // 3. 写入 (单次系统调用)
    pgstat_report_wait_start(WAIT_EVENT_CONTROL_FILE_WRITE);
    if (write(fd, ControlFile, PG_CONTROL_FILE_SIZE) != PG_CONTROL_FILE_SIZE)
        ereport(PANIC, "could not write to control file");
    pgstat_report_wait_end();

    // 4. Fsync 确保持久化
    pgstat_report_wait_start(WAIT_EVENT_CONTROL_FILE_SYNC);
    if (fsync(fd) != 0)
        ereport(PANIC, "could not fsync control file");
    pgstat_report_wait_end();

    close(fd);
}
```

---

## 4. XLogCtlData 结构

**位置**: `src/backend/access/transam/xlog.c:451`

### 4.1 核心子结构 XLogCtlInsert

```c
typedef struct XLogCtlInsert
{
    slock_t  insertpos_lck;  /* 保护 CurrBytePos 和 PrevBytePos */

    /* WAL 插入位置 (使用"可用字节位置"而非 XLogRecPtr) */
    uint64   CurrBytePos;    /* 下一条记录插入位置 */
    uint64   PrevBytePos;    /* 前一条记录的起始位置 */

    char     pad[PG_CACHE_LINE_SIZE];  /* 避免伪共享 */

    /* 全页写和 REDO 点 (由所有 WAL 插入锁保护) */
    XLogRecPtr RedoRecPtr;   /* 当前插入的 redo 点 */
    bool       fullPageWrites;

    /* 运行中的备份计数 */
    int        runningBackups;
    XLogRecPtr lastBackupStart;

    /* WAL 插入锁数组 */
    WALInsertLockPadded *WALInsertLocks;
} XLogCtlInsert;
```

### 4.2 完整 XLogCtlData 结构

```c
typedef struct XLogCtlData
{
    XLogCtlInsert Insert;    /* WAL 插入控制 */

    /* 以下由 info_lck 保护 */
    XLogwrtRqst LogwrtRqst;  /* 写请求位置 */
    XLogRecPtr  RedoRecPtr;  /* Insert->RedoRecPtr 的最近副本 */
    FullTransactionId ckptFullXid;  /* 最新 checkpoint 的 nextXid */
    XLogRecPtr  asyncXactLSN;       /* 最新异步提交/中止的 LSN */
    XLogRecPtr  replicationSlotMinLSN;  /* 任何槽需要的最老 LSN */

    XLogSegNo   lastRemovedSegNo;   /* 最新删除/回收的 WAL 段 */

    /* Fake LSN counter (用于 unlogged 表) */
    pg_atomic_uint64 unloggedLSN;

    /* WAL 段切换时间和 LSN (由 WALWriteLock 保护) */
    pg_time_t  lastSegSwitchTime;
    XLogRecPtr lastSegSwitchLSN;

    /* 以下使用原子操作访问 */
    pg_atomic_uint64 logInsertResult;  /* 最后插入到缓冲区的字节+1 */
    pg_atomic_uint64 logWriteResult;   /* 最后写出的字节+1 */
    pg_atomic_uint64 logFlushResult;   /* 最后 flush 的字节+1 */

    /* 已初始化的缓存页 */
    XLogRecPtr InitializedUpTo;

    /* WAL 缓冲区 */
    char      *pages;             /* 未写入的 XLOG 页缓冲区 */
    pg_atomic_uint64 *xlblocks;   /* 第一个字节指针 + XLOG_BLCKSZ */
    int        XLogCacheBlck;     /* 最高分配的 xlog 缓冲区索引 */

    /* 时间线信息 */
    TimeLineID InsertTimeLineID;  /* 插入 WAL 的时间线 */
    TimeLineID PrevTimeLineID;    /* 前一个时间线 */

    /* 恢复状态 */
    RecoveryState SharedRecoveryState;

    /* Checkpoint 相关 */
    XLogRecPtr lastCheckPointRecPtr;
    XLogRecPtr lastCheckPointEndPtr;
    CheckPoint lastCheckPoint;

    slock_t    info_lck;          /* 保护以上部分字段 */

    /* 更多字段... */
} XLogCtlData;
```

### 4.3 WAL 插入锁机制

```c
#define NUM_XLOGINSERT_LOCKS  8  /* 8 个 WAL 插入锁 */

typedef struct WALInsertLock
{
    LWLock     lock;              /* 轻量级锁 */
    pg_atomic_uint64 insertingAt; /* 当前插入到的位置 */
    XLogRecPtr lastImportantAt;   /* 最后重要记录的 LSN */
} WALInsertLock;

/* 缓存行对齐，避免伪共享 */
typedef union WALInsertLockPadded
{
    WALInsertLock l;
    char pad[PG_CACHE_LINE_SIZE];  /* 通常 64 字节 */
} WALInsertLockPadded;
```

**插入锁的作用**:
```
并发插入 WAL:
1. 进程 A 获取锁 0
2. 进程 B 获取锁 1
3. 进程 C 获取锁 2
4. ...

每个进程并发写入 WAL 缓冲区的不同部分。

刷写 WAL 时:
- 需要获取所有 8 个锁
- 确保没有进程正在写入
- 避免刷写未完成的记录
```

---

## 5. CheckpointStatsData 结构

**位置**: 全局变量 `CheckpointStats` (xlog.c:210)

### 5.1 完整定义

```c
typedef struct
{
    TimestampTz ckpt_start_t;      /* Checkpoint 开始时间 */
    TimestampTz ckpt_write_t;      /* 开始写脏页时间 */
    TimestampTz ckpt_sync_t;       /* 开始 fsync 时间 */
    TimestampTz ckpt_sync_end_t;   /* Fsync 结束时间 */
    TimestampTz ckpt_end_t;        /* Checkpoint 结束时间 */

    int         ckpt_bufs_written; /* 写入的缓冲区数量 */

    int         ckpt_segs_added;   /* 新增的 WAL 段 */
    int         ckpt_segs_removed; /* 删除的 WAL 段 */
    int         ckpt_segs_recycled;/* 回收的 WAL 段 */

    int         ckpt_sync_rels;    /* Fsync 的关系数 */
    uint64      ckpt_longest_sync; /* 最长 fsync 时间 (微秒) */
    uint64      ckpt_agg_sync_time;/* 总 fsync 时间 (微秒) */
} CheckpointStatsData;
```

### 5.2 时间统计

```
时间线:
    ckpt_start_t              ckpt_write_t        ckpt_sync_t    ckpt_sync_end_t  ckpt_end_t
        |                          |                    |               |              |
        |<------ 准备阶段 -------->|<--- 写脏页 --->|<--- Fsync --->|<-- 后续清理 -->|
        |                          |                    |               |              |
        |                          |                    |               |              |
        |                          V                    V               V              V
        |                  CheckPointGuts()      ProcessSyncRequests()                |
        |                    - 刷SLRU                  - 批量fsync                     |
        |                    - BufferSync()                                           |
        |                                                                             |
        V                                                                             V
  CreateCheckPoint                                                            LogCheckpointEnd
    开始
```

**时间计算**:
```c
write_time = ckpt_sync_t - ckpt_write_t;     // 写脏页耗时
sync_time = ckpt_sync_end_t - ckpt_sync_t;   // Fsync 耗时
total_time = ckpt_end_t - ckpt_start_t;      // 总耗时
```

### 5.3 日志输出

**log_checkpoints = on 时的输出**:
```
LOG:  checkpoint complete: wrote 1234 buffers (7.5%);
      0 WAL file(s) added, 2 removed, 1 recycled;
      write=25.123 s, sync=1.456 s, total=26.789 s;
      sync files=78, longest=0.234 s, average=0.018 s;
      distance=24567 kB, estimate=25000 kB
```

**字段对应**:
- `1234 buffers (7.5%)`: ckpt_bufs_written / NBuffers
- `0 added, 2 removed, 1 recycled`: ckpt_segs_*
- `write=25.123 s`: (ckpt_sync_t - ckpt_write_t) / 1000
- `sync=1.456 s`: (ckpt_sync_end_t - ckpt_sync_t) / 1000
- `total=26.789 s`: (ckpt_end_t - ckpt_start_t) / 1000
- `sync files=78`: ckpt_sync_rels
- `longest=0.234 s`: ckpt_longest_sync / 1000000
- `average=0.018 s`: (ckpt_agg_sync_time / ckpt_sync_rels) / 1000000

---

## 6. 缓冲区相关结构

### 6.1 BufferDesc (缓冲区描述符)

```c
typedef struct BufferDesc
{
    BufferTag  tag;           /* 缓冲区的文件/块标识 */
    int        buf_id;        /* 缓冲区编号 */

    /* 状态位 (使用原子操作访问) */
    pg_atomic_uint32 state;   /* BM_DIRTY, BM_VALID, BM_CHECKPOINT_NEEDED 等 */

    int        wait_backend_pgprocno;  /* 等待此缓冲区的后端 */

    int        freeNext;      /* 空闲列表中的下一个 */

    LWLock     content_lock;  /* 保护缓冲区内容 */
    LWLock     io_in_progress_lock;  /* I/O 进行中锁 */
} BufferDesc;
```

### 6.2 BufferTag (缓冲区标识)

```c
typedef struct BufferTag
{
    Oid        spcOid;        /* 表空间 OID */
    Oid        dbOid;         /* 数据库 OID */
    RelFileNumber relNumber;  /* 关系文件号 */
    ForkNumber forkNum;       /* Fork 类型: main, fsm, vm, init */
    BlockNumber blockNum;     /* 块号 */
} BufferTag;
```

### 6.3 缓冲区状态位

```c
/* 缓冲区状态标志 (state 字段的位) */
#define BM_LOCKED           (1U << 22)  /* 缓冲区已锁定 */
#define BM_DIRTY            (1U << 23)  /* 数据已修改 */
#define BM_VALID            (1U << 24)  /* 数据有效 */
#define BM_TAG_VALID        (1U << 25)  /* 标签有效 */
#define BM_IO_IN_PROGRESS   (1U << 26)  /* I/O 进行中 */
#define BM_IO_ERROR         (1U << 27)  /* I/O 错误 */
#define BM_JUST_DIRTIED     (1U << 28)  /* 刚被弄脏 */
#define BM_PIN_COUNT_WAITER (1U << 29)  /* 有等待 pin 的进程 */
#define BM_CHECKPOINT_NEEDED (1U << 30) /* 需要 checkpoint */
#define BM_PERMANENT        (1U << 31)  /* 永久关系 */
```

**Checkpoint 相关的状态转换**:
```
正常运行:
state = BM_VALID | BM_DIRTY | BM_PERMANENT

BufferSync 第一阶段 (标记):
state |= BM_CHECKPOINT_NEEDED
state = BM_VALID | BM_DIRTY | BM_PERMANENT | BM_CHECKPOINT_NEEDED

BufferSync 第五阶段 (写入):
SyncOneBuffer() 写入缓冲区
state &= ~BM_CHECKPOINT_NEEDED
state = BM_VALID | BM_DIRTY | BM_PERMANENT

写入完成后可能被后端清理 BM_DIRTY:
state &= ~BM_DIRTY
state = BM_VALID | BM_PERMANENT
```

### 6.4 CkptSortItem (checkpoint 排序项)

```c
typedef struct CkptSortItem
{
    int        buf_id;        /* 缓冲区 ID */
    Oid        tsId;          /* 表空间 OID */
    RelFileNumber relNumber;  /* 关系文件号 */
    ForkNumber forkNum;       /* Fork 类型 */
    BlockNumber blockNum;     /* 块号 */
} CkptSortItem;

/* 全局数组，大小 = NBuffers */
static CkptSortItem *CkptBufferIds;
```

### 6.5 CkptTsStatus (表空间 checkpoint 状态)

```c
typedef struct CkptTsStatus
{
    Oid    tsId;              /* 表空间 OID */
    int    index;             /* CkptBufferIds 中的当前索引 */
    int    num_to_scan;       /* 该表空间的缓冲区总数 */
    int    num_scanned;       /* 已扫描的数量 */
    double progress;          /* 当前进度 (用于堆排序) */
    double progress_slice;    /* 单个缓冲区的进度增量 */
} CkptTsStatus;
```

**负载均衡示例**:
```
假设有 3 个表空间:
ts1: 100 个脏页
ts2: 200 个脏页
ts3: 300 个脏页
总计: 600 个脏页

progress_slice 计算:
ts1: 600 / 100 = 6.0
ts2: 600 / 200 = 3.0
ts3: 600 / 300 = 2.0

写入顺序 (最小堆，选择 progress 最小的):
1. 写 ts3[0]: progress = 0 + 2.0 = 2.0
2. 写 ts3[1]: progress = 2.0 + 2.0 = 4.0
3. 写 ts2[0]: progress = 0 + 3.0 = 3.0
4. 写 ts3[2]: progress = 4.0 + 2.0 = 6.0
5. 写 ts2[1]: progress = 3.0 + 3.0 = 6.0
6. 写 ts1[0]: progress = 0 + 6.0 = 6.0
7. 写 ts3[3], ts2[2], ts1[1] ...

结果: 3 个表空间的写入交替进行，负载均衡
```

---

## 7. 数据结构关系图

```
┌────────────────────────────────────────────────────────────────┐
│                    PostgreSQL 共享内存                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────┐                                   │
│  │ CheckpointerShmemStruct │                                   │
│  ├─────────────────────────┤                                   │
│  │ checkpointer_pid        │                                   │
│  │ ckpt_started  ◄─────────┼──┐ 计数器通信                      │
│  │ ckpt_done               │  │                                │
│  │ ckpt_flags              │  │                                │
│  │ requests[]  ◄───────────┼──┼─┐ fsync 请求队列                │
│  └─────────────────────────┘  │ │                              │
│                               │ │                              │
│  ┌─────────────────────────┐  │ │                              │
│  │      XLogCtlData        │  │ │                              │
│  ├─────────────────────────┤  │ │                              │
│  │ Insert.RedoRecPtr  ─────┼──┼─┼─┐ REDO 点                     │
│  │ pages[]             ─────┼──┼─┼─┼─┐ WAL 缓冲区               │
│  │ xlblocks[]              │  │ │ │ │                          │
│  │ lastCheckPoint          │  │ │ │ │                          │
│  └─────────────────────────┘  │ │ │ │                          │
│                               │ │ │ │                          │
│  ┌─────────────────────────┐  │ │ │ │                          │
│  │   Buffer Descriptors    │  │ │ │ │                          │
│  ├─────────────────────────┤  │ │ │ │                          │
│  │ BufferDesc[0]           │  │ │ │ │                          │
│  │   state (BM_DIRTY)      │  │ │ │ │                          │
│  │   tag (spc/db/rel/blk)  │  │ │ │ │                          │
│  │ BufferDesc[1]           │  │ │ │ │                          │
│  │ ...                     │  │ │ │ │                          │
│  │ BufferDesc[NBuffers-1]  │  │ │ │ │                          │
│  └─────────────────────────┘  │ │ │ │                          │
│                               │ │ │ │                          │
└───────────────────────────────┼─┼─┼─┼──────────────────────────┘
                                │ │ │ │
┌───────────────────────────────┼─┼─┼─┼──────────────────────────┐
│              磁盘文件          │ │ │ │                          │
├───────────────────────────────┼─┼─┼─┼──────────────────────────┤
│                               │ │ │ │                          │
│  ┌─────────────────────────┐  │ │ │ │                          │
│  │  pg_control (8192 字节) │  │ │ │ │                          │
│  ├─────────────────────────┤  │ │ │ │                          │
│  │ checkPoint ◄────────────┼──┘ │ │ │  指向 WAL 中的 LSN        │
│  │ checkPointCopy          │    │ │ │                          │
│  │   redo ◄────────────────┼────┘ │ │  REDO 点                 │
│  │   nextXid               │      │ │                          │
│  │   oldestXid             │      │ │                          │
│  │   ...                   │      │ │                          │
│  │ minRecoveryPoint        │      │ │                          │
│  │ state                   │      │ │                          │
│  └─────────────────────────┘      │ │                          │
│                                    │ │                          │
│  ┌─────────────────────────┐      │ │                          │
│  │  WAL Files (pg_wal/)    │      │ │                          │
│  ├─────────────────────────┤      │ │                          │
│  │ 000000010000000000000001│      │ │                          │
│  │   [WAL Records...]      │      │ │                          │
│  │   [CHECKPOINT_REDO] ◄───┼──────┘ │                          │
│  │   [WAL Records...]      │        │                          │
│  │   [CHECKPOINT_ONLINE] ◄─┼────────┘                          │
│  │ 000000010000000000000002│                                   │
│  │ ...                     │                                   │
│  └─────────────────────────┘                                   │
│                                                                 │
│  ┌─────────────────────────┐                                   │
│  │  Data Files (base/)     │                                   │
│  ├─────────────────────────┤                                   │
│  │ base/12345/16789        │  ◄── 脏页最终写入这里              │
│  │ base/12345/16789.1      │                                   │
│  │ ...                     │                                   │
│  └─────────────────────────┘                                   │
└─────────────────────────────────────────────────────────────────┘

图例:
──► 指针/引用关系
═══► 数据流向
```

---

## 总结

本文档详细介绍了 PostgreSQL Checkpoint 机制的核心数据结构：

1. **CheckPoint**: WAL 中的 checkpoint 记录，包含恢复所需的所有元数据
2. **CheckpointerShmemStruct**: 进程间通信的共享内存结构
3. **ControlFileData**: pg_control 文件，持久化关键元数据
4. **XLogCtlData**: WAL 控制结构，管理 WAL 插入和缓冲
5. **CheckpointStatsData**: 统计信息，用于日志和监控
6. **Buffer 相关结构**: 缓冲区管理和 checkpoint 排序

这些结构协同工作，实现了高效、可靠的 checkpoint 机制。

---

**下一步**: 阅读 `03_implementation_flow.md` 了解这些结构如何在流程中使用。
