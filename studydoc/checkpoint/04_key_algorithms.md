# PostgreSQL Checkpoint - 关键算法分析

## 目录

1. [Checkpoint 调度算法](#1-checkpoint-调度算法)
2. [REDO 点计算算法](#2-redo-点计算算法)
3. [表空间负载均衡算法](#3-表空间负载均衡算法)
4. [Fsync 队列压缩算法](#4-fsync-队列压缩算法)
5. [WAL 文件管理算法](#5-wal-文件管理算法)
6. [缓冲区排序算法](#6-缓冲区排序算法)
7. [算法复杂度分析](#7-算法复杂度分析)

---

## 1. Checkpoint 调度算法

### 1.1 IsCheckpointOnSchedule 算法

**位置**: `src/backend/postmaster/checkpointer.c:780`

**目的**: 判断 checkpoint 进度是否符合预期，决定是否需要限流休眠。

#### 1.1.1 完整实现

```c
static bool
IsCheckpointOnSchedule(double progress)
{
    XLogRecPtr  recptr;
    struct timeval now;
    double      elapsed_xlogs,
                elapsed_time;

    Assert(ckpt_active);

    // 步骤 1: 根据 checkpoint_completion_target 调整进度
    // checkpointer.c:790
    progress *= CheckPointCompletionTarget;
    // 默认 0.9, 即在 90% 的时间内完成

    // 步骤 2: 检查缓存值（优化）
    // checkpointer.c:798-799
    if (progress < ckpt_cached_elapsed)
        return false;  // 进度落后，需要加速

    // 步骤 3: 检查 WAL 生成量进度
    // checkpointer.c:821-826
    if (RecoveryInProgress())
        recptr = GetXLogReplayRecPtr(NULL);  // 恢复模式：重放位置
    else
        recptr = GetInsertRecPtr();          // 正常模式：插入位置

    elapsed_xlogs = (((double) (recptr - ckpt_start_recptr)) /
                     wal_segment_size) / CheckPointSegments;
    // CheckPointSegments = max_wal_size / wal_segment_size
    // 例如: 1024MB / 16MB = 64 个段

    if (progress < elapsed_xlogs)
    {
        ckpt_cached_elapsed = elapsed_xlogs;
        return false;  // WAL 增长过快，需要加速
    }

    // 步骤 4: 检查时间进度
    // checkpointer.c:837-839
    gettimeofday(&now, NULL);
    elapsed_time = ((double) ((pg_time_t) now.tv_sec - ckpt_start_time) +
                    now.tv_usec / 1000000.0) / CheckPointTimeout;

    if (progress < elapsed_time)
    {
        ckpt_cached_elapsed = elapsed_time;
        return false;  // 时间过去太多，需要加速
    }

    // 步骤 5: 进度符合预期
    return true;
}
```

#### 1.1.2 数学模型

**输入参数**:
- `progress`: 当前完成进度 (0.0 ~ 1.0)
- `checkpoint_completion_target`: 目标完成比例 (默认 0.9)
- `CheckPointTimeout`: checkpoint 超时时间 (默认 300 秒)
- `CheckPointSegments`: WAL 段数阈值 (默认 64 个段)

**计算公式**:

```
调整后进度 = progress × checkpoint_completion_target

WAL 进度 = (当前LSN - 开始LSN) / wal_segment_size / CheckPointSegments

时间进度 = (当前时间 - 开始时间) / CheckPointTimeout

判断逻辑:
  IF 调整后进度 < WAL进度 OR 调整后进度 < 时间进度
    THEN 进度落后，加速写入
  ELSE
    进度正常，可以休眠
```

#### 1.1.3 示例计算

**场景**: OLTP 系统，中等负载

**配置**:
```sql
checkpoint_timeout = 300s (5 分钟)
max_wal_size = 1024MB
wal_segment_size = 16MB
checkpoint_completion_target = 0.9
```

**计算 CheckPointSegments**:
```
CheckPointSegments = 1024MB / 16MB = 64 个段
```

**Checkpoint 进行中的检查**:

| 时间点 | 脏页写入进度 | 已用时间 | WAL增长 | 判断 |
|--------|-------------|---------|---------|------|
| T+0s   | 0%          | 0s      | 0 段    | 开始 |
| T+100s | 40%         | 100s    | 10 段   | 计算调整后进度 = 40% × 0.9 = 36%<br>时间进度 = 100/300 = 33%<br>WAL进度 = 10/64 = 16%<br>36% > max(33%, 16%) ✓ 进度超前，休眠 |
| T+150s | 50%         | 150s    | 20 段   | 调整后进度 = 50% × 0.9 = 45%<br>时间进度 = 150/300 = 50%<br>WAL进度 = 20/64 = 31%<br>45% < 50% ✗ 进度落后，加速 |
| T+270s | 100%        | 270s    | 58 段   | 调整后进度 = 100% × 0.9 = 90%<br>时间进度 = 270/300 = 90%<br>WAL进度 = 58/64 = 91%<br>完成 |

**关键观察**:
- 在 T+150s 时，虽然写了 50% 的脏页，但已用去 50% 的时间（应该只用 50%×0.9=45%）
- 系统会停止休眠，全速写入
- 最终在 270 秒完成，符合 0.9 的目标

#### 1.1.4 CheckpointWriteDelay 调用

```c
void
CheckpointWriteDelay(int flags, double progress)
{
    static int absorb_counter = WRITES_PER_ABSORB;

    // 仅 checkpointer 进程执行延迟
    if (!AmCheckpointerProcess())
        return;

    // 检查是否需要休眠
    if (!(flags & CHECKPOINT_IMMEDIATE) &&       // 非立即 checkpoint
        !ShutdownRequestPending &&               // 非关闭中
        !ImmediateCheckpointRequested() &&       // 无新的立即请求
        IsCheckpointOnSchedule(progress))        // 进度符合预期
    {
        // 进度正常，可以休眠

        // 处理配置重载
        if (ConfigReloadPending)
        {
            ConfigReloadPending = false;
            ProcessConfigFile(PGC_SIGHUP);
            UpdateSharedMemoryConfig();
        }

        // 吸收 fsync 请求
        AbsorbSyncRequests();
        absorb_counter = WRITES_PER_ABSORB;

        // 检查归档超时
        CheckArchiveTimeout();

        // 报告统计
        pgstat_report_checkpointer();

        // ★ 休眠 100ms ★
        WaitLatch(MyLatch, WL_LATCH_SET | WL_EXIT_ON_PM_DEATH | WL_TIMEOUT,
                  100,  // 100 毫秒
                  WAIT_EVENT_CHECKPOINT_WRITE_DELAY);
        ResetLatch(MyLatch);
    }
    else if (--absorb_counter <= 0)
    {
        // 即使不休眠，也要定期吸收 fsync 请求
        // 防止队列溢出
        AbsorbSyncRequests();
        absorb_counter = WRITES_PER_ABSORB;
    }

    // 检查 barrier 事件
    if (ProcSignalBarrierPending)
        ProcessProcSignalBarrier();
}
```

**调用频率**:
- BufferSync 中，每写一个缓冲区调用一次
- 假设 10000 个脏页，会调用 10000 次
- 如果进度正常，会有约 9000 次休眠（90%）
- 总休眠时间: 9000 × 100ms = 900s？不对！
- 实际上，休眠会被下一个缓冲区的写入"跳过"，总时间受 completion_target 控制

**实际效果**:
```
假设需要写 10000 个缓冲区:

不限流 (IMMEDIATE):
  写入时间: 10 秒 (假设 1000 页/秒)
  I/O 峰值高，影响查询

渐进式 (completion_target=0.9):
  目标时间: 300s × 0.9 = 270 秒
  每 27ms 写 1 个页面
  I/O 平滑，对查询影响小
```

---

## 2. REDO 点计算算法

### 2.1 算法原理

**REDO 点**: 崩溃恢复时，WAL 重放的起始位置。

**核心约束**: REDO 点之后的所有数据页修改，必须被 checkpoint 持久化。

### 2.2 Online Checkpoint REDO 点

#### 2.2.1 算法步骤

```c
// xlog.c:7048-7062

// 步骤 1: 插入 XLOG_CHECKPOINT_REDO 记录
XLogBeginInsert();
XLogRegisterData((char *) &wal_level, sizeof(wal_level));
recptr = XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT_REDO);

// 步骤 2: REDO 记录的 LSN 成为 redo 点
checkPoint.redo = recptr;

// 步骤 3: 更新共享内存
SpinLockAcquire(&XLogCtl->info_lck);
XLogCtl->RedoRecPtr = checkPoint.redo;
SpinLockRelease(&XLogCtl->info_lck);
```

#### 2.2.2 时间线分析

```
时间: T0         T1              T2              T3
      │          │               │               │
WAL:  [事务A]    [REDO记录]      [CheckPoint]    [事务B]
      修改页P1    LSN=1000       记录开始         修改页P2
                  ↑
              redo点

约束验证:
- 页P1: 在 T0 修改，在 redo 点(T1)之前
  → checkpoint 必须刷写 P1
  → BufferSync 扫描时会标记 P1

- 页P2: 在 T3 修改，在 redo 点之后
  → checkpoint 不需要刷写 P2
  → 即使系统崩溃，恢复时会重放事务B的WAL

恢复流程:
1. 读取 checkpoint 记录，获取 redo=1000
2. 从 LSN 1000 开始重放 WAL
3. 重放 CheckPoint 记录 (可能更新控制信息)
4. 重放事务B的 WAL，恢复页P2
```

### 2.3 Shutdown Checkpoint REDO 点

#### 2.3.1 算法步骤

```c
// xlog.c:6998-7030

// 步骤 1: 计算当前 WAL 插入位置
XLogRecPtr curInsert = XLogBytePosToRecPtr(Insert->CurrBytePos);

// 步骤 2: 计算下一个记录的位置
freespace = INSERT_FREESPACE(curInsert);
if (freespace == 0)
{
    // 当前页已满，跳到下一个页头部
    if (XLogSegmentOffset(curInsert, wal_segment_size) == 0)
        curInsert += SizeOfXLogLongPHD;   // 段开始：28 字节
    else
        curInsert += SizeOfXLogShortPHD;  // 段中间：24 字节
}

// 步骤 3: REDO 点就是下一个记录的位置
checkPoint.redo = curInsert;

// 步骤 4: 更新共享内存
RedoRecPtr = XLogCtl->Insert.RedoRecPtr = checkPoint.redo;
```

#### 2.3.2 为什么不同？

```
Online Checkpoint:
  - 有并发事务，插入位置不断变化
  - 必须先"钉住"一个位置（插入 REDO 记录）
  - 确保该位置之后的修改都被刷写

Shutdown Checkpoint:
  - 等待所有事务完成
  - 没有并发写入
  - 可以直接使用当前位置
  - REDO 点 = Checkpoint 记录的位置
  - 恢复时无需重放任何 WAL
```

### 2.4 边界情况处理

#### 情况 1: WAL 页面边界

```
WAL 布局:
  [页 1: 8192 字节] [页 2: 8192 字节] [页 3: ...]
  [header: 24B][记录1][记录2]...
                              ↑
                         curInsert = 8190

freespace = INSERT_FREESPACE(8190) = 8192 - 8190 = 2 字节

下一个记录至少需要:
  XLogRecord header = 24 字节

2 < 24, 所以 freespace 不足

调整:
  curInsert = 下一页开始 + 页头大小
  curInsert = 8192 + 24 = 8216

checkPoint.redo = 8216
```

#### 情况 2: WAL 段边界

```
WAL 段大小 = 16MB = 16777216 字节

段 1: 0x0000000000000000 - 0x0000000000FFFFFF (16MB)
段 2: 0x0000000001000000 - 0x0000000001FFFFFF (16MB)

如果 curInsert = 0x0000000001000000 (段 2 开始):
  XLogSegmentOffset(curInsert, 16MB) = 0

  调整:
    curInsert += SizeOfXLogLongPHD = 28 字节
    curInsert = 0x000000000100001C

checkPoint.redo = 0x000000000100001C
```

---

## 3. 表空间负载均衡算法

### 3.1 问题描述

**场景**: 有多个表空间，每个表空间有不同数量的脏页。

**问题**: 如果按排序后的顺序写入：
```
表空间 A: 1000 个页
表空间 B: 500 个页
表空间 C: 300 个页

简单顺序写入:
  1. 写完 A 的 1000 个页 (A 的磁盘满负荷)
  2. 写完 B 的 500 个页 (B 的磁盘满负荷)
  3. 写完 C 的 300 个页 (C 的磁盘满负荷)

问题: 各表空间的 I/O 不平衡，可能导致单个磁盘过载
```

**目标**: 各表空间的写入交替进行，I/O 负载均衡。

### 3.2 算法实现

#### 3.2.1 数据结构

```c
typedef struct CkptTsStatus
{
    Oid    tsId;              // 表空间 OID
    int    index;             // 当前在 CkptBufferIds[] 中的索引
    int    num_to_scan;       // 该表空间的缓冲区总数
    int    num_scanned;       // 已写入的缓冲区数
    double progress;          // 当前进度（用于堆排序）
    double progress_slice;    // 单个缓冲区的进度增量
} CkptTsStatus;

// 最小堆比较函数
static int
ts_ckpt_progress_comparator(Datum a, Datum b, void *arg)
{
    CkptTsStatus *sa = (CkptTsStatus *) DatumGetPointer(a);
    CkptTsStatus *sb = (CkptTsStatus *) DatumGetPointer(b);

    // 进度小的排在前面（最小堆）
    if (sa->progress < sb->progress)
        return -1;
    else if (sa->progress > sb->progress)
        return 1;
    else
        return 0;
}
```

#### 3.2.2 算法步骤

```c
// bufmgr.c:3061-3144

// 步骤 1: 计算每个表空间的 progress_slice
for (i = 0; i < num_spaces; i++)
{
    CkptTsStatus *ts_stat = &per_ts_stat[i];

    // 关键公式:
    // progress_slice = 总缓冲区数 / 该表空间缓冲区数
    ts_stat->progress_slice = (float8) num_to_scan / ts_stat->num_to_scan;

    // 加入最小堆
    binaryheap_add_unordered(ts_heap, PointerGetDatum(ts_stat));
}

// 步骤 2: 构建堆
binaryheap_build(ts_heap);

// 步骤 3: 循环写入
while (!binaryheap_empty(ts_heap))
{
    // 3.1 从堆顶取出进度最慢的表空间
    CkptTsStatus *ts_stat = (CkptTsStatus *)
        DatumGetPointer(binaryheap_first(ts_heap));

    // 3.2 写入该表空间的下一个缓冲区
    buf_id = CkptBufferIds[ts_stat->index].buf_id;
    SyncOneBuffer(buf_id, false, &wb_context);

    // 3.3 更新进度
    ts_stat->progress += ts_stat->progress_slice;
    ts_stat->num_scanned++;
    ts_stat->index++;

    // 3.4 调整堆
    if (ts_stat->num_scanned == ts_stat->num_to_scan)
        binaryheap_remove_first(ts_heap);  // 该表空间完成
    else
        binaryheap_replace_first(ts_heap, PointerGetDatum(ts_stat));
        // 重新调整堆，使进度最慢的浮到堆顶

    // 3.5 I/O 限流
    CheckpointWriteDelay(flags, (double) num_processed / num_to_scan);
}
```

### 3.3 数学分析

#### 3.3.1 progress_slice 的含义

```
假设:
  总缓冲区数 N = 1800
  表空间 A: nA = 1000
  表空间 B: nB = 500
  表空间 C: nC = 300

progress_slice 计算:
  sliceA = N / nA = 1800 / 1000 = 1.8
  sliceB = N / nB = 1800 / 500 = 3.6
  sliceC = N / nC = 1800 / 300 = 6.0

含义:
  - 表空间 A 写 1 个页，进度增加 1.8
  - 表空间 B 写 1 个页，进度增加 3.6
  - 表空间 C 写 1 个页，进度增加 6.0

  - 页面少的表空间，单页进度贡献大
  - 确保不会被"遗忘"
```

#### 3.3.2 执行轨迹

```
初始状态:
  Heap: [A(0.0), B(0.0), C(0.0)]

迭代 1: 选择 A (progress=0.0)
  写 A[0], progressA = 0.0 + 1.8 = 1.8
  Heap: [B(0.0), C(0.0), A(1.8)]

迭代 2: 选择 B (progress=0.0)
  写 B[0], progressB = 0.0 + 3.6 = 3.6
  Heap: [C(0.0), A(1.8), B(3.6)]

迭代 3: 选择 C (progress=0.0)
  写 C[0], progressC = 0.0 + 6.0 = 6.0
  Heap: [A(1.8), B(3.6), C(6.0)]

迭代 4: 选择 A (progress=1.8)
  写 A[1], progressA = 1.8 + 1.8 = 3.6
  Heap: [A(3.6), B(3.6), C(6.0)]

迭代 5: 选择 A (progress=3.6, 与 B 并列，堆实现选 A)
  写 A[2], progressA = 3.6 + 1.8 = 5.4
  Heap: [B(3.6), A(5.4), C(6.0)]

迭代 6: 选择 B (progress=3.6)
  写 B[1], progressB = 3.6 + 3.6 = 7.2
  Heap: [A(5.4), C(6.0), B(7.2)]

...继续...

观察:
  - 三个表空间的写入交替进行
  - 进度始终保持接近
  - 实现了负载均衡
```

### 3.4 复杂度分析

**时间复杂度**:
- 构建堆: O(S log S), S = 表空间数
- 每次写入: O(log S)
- 总共写入 N 个缓冲区: O(N log S)
- 通常 S << N (S ≈ 3-10, N ≈ 10000)
- 实际开销很小

**空间复杂度**:
- CkptTsStatus 数组: O(S)
- 堆: O(S)
- CkptBufferIds 数组: O(N)
- 总计: O(N + S) ≈ O(N)

---

## 4. Fsync 队列压缩算法

### 4.1 问题背景

**Fsync 请求队列**:
- 后端进程写脏页后，将 fsync 请求转发给 checkpointer
- 队列大小 = NBuffers (默认 16384)
- 队列满时，后端需要自己执行 fsync (很慢)

**问题**: 队列中可能有重复的请求。

**示例**:
```
后端 1: 写 (tableA, block 100) -> 请求 fsync(tableA)
后端 2: 写 (tableA, block 200) -> 请求 fsync(tableA)
后端 3: 写 (tableB, block 50)  -> 请求 fsync(tableB)
后端 1: 再写 (tableA, block 150) -> 请求 fsync(tableA)

队列: [fsync(tableA), fsync(tableA), fsync(tableB), fsync(tableA)]
      重复!            重复!                          重复!
```

**解决方案**: 压缩队列，去除重复项。

### 4.2 算法实现

```c
// checkpointer.c:1154

static bool
CompactCheckpointerRequestQueue(void)
{
    struct CheckpointerSlotMapping
    {
        CheckpointerRequest request;
        int slot;
    };

    int         n, preserve_count;
    int         num_skipped = 0;
    HASHCTL     ctl;
    HTAB       *htab;
    bool       *skip_slot;

    // 必须持有 CheckpointerCommLock 排它锁
    Assert(LWLockHeldByMe(CheckpointerCommLock));

    // 临界区内不允许内存分配
    if (CritSectionCount > 0)
        return false;

    // 步骤 1: 初始化 skip_slot 数组
    skip_slot = palloc0(sizeof(bool) * CheckpointerShmem->num_requests);

    // 步骤 2: 初始化哈希表
    ctl.keysize = sizeof(CheckpointerRequest);
    ctl.entrysize = sizeof(struct CheckpointerSlotMapping);
    ctl.hcxt = CurrentMemoryContext;

    htab = hash_create("CompactCheckpointerRequestQueue",
                       CheckpointerShmem->num_requests,
                       &ctl,
                       HASH_ELEM | HASH_BLOBS | HASH_CONTEXT);

    // 步骤 3: 扫描队列，使用哈希表检测重复
    for (n = 0; n < CheckpointerShmem->num_requests; n++)
    {
        CheckpointerRequest *request;
        struct CheckpointerSlotMapping *slotmap;
        bool found;

        request = &CheckpointerShmem->requests[n];

        // 查找或插入
        slotmap = hash_search(htab, request, HASH_ENTER, &found);

        if (found)
        {
            // 重复! 标记前一个位置为可跳过
            skip_slot[slotmap->slot] = true;
            num_skipped++;
        }

        // 记住最新出现的位置
        slotmap->slot = n;
    }

    hash_destroy(htab);

    // 步骤 4: 检查是否有去重
    if (!num_skipped)
    {
        pfree(skip_slot);
        return false;  // 没有重复，队列仍然满
    }

    // 步骤 5: 压缩数组
    preserve_count = 0;
    for (n = 0; n < CheckpointerShmem->num_requests; n++)
    {
        if (skip_slot[n])
            continue;  // 跳过重复项

        CheckpointerShmem->requests[preserve_count++] =
            CheckpointerShmem->requests[n];
    }

    ereport(DEBUG1,
            (errmsg_internal("compacted fsync request queue from %d entries to %d entries",
                             CheckpointerShmem->num_requests, preserve_count)));

    CheckpointerShmem->num_requests = preserve_count;

    pfree(skip_slot);
    return true;  // 成功压缩
}
```

### 4.3 算法分析

#### 4.3.1 为什么只保留最后一个？

```
场景: 对同一个文件的多个 fsync 请求

请求序列:
  [1] fsync(tableA) - 块 100 脏
  [2] fsync(tableB) - 块 50 脏
  [3] fsync(tableA) - 块 200 脏  <-- 保留这个
  [4] fsync(tableC) - 块 30 脏

逻辑:
  - 请求 [1] 和 [3] 都是对 tableA 的 fsync
  - Fsync 操作是针对整个文件的，不是单个块
  - 执行 [3] 时会 fsync 整个文件，包括块 100 和 200
  - 所以 [1] 是多余的

压缩后:
  [2] fsync(tableB)
  [3] fsync(tableA)
  [4] fsync(tableC)
```

#### 4.3.2 复杂度

**时间复杂度**:
- 哈希表操作: O(1) 平均
- 扫描队列: O(N)
- 压缩: O(N)
- 总计: O(N)

**空间复杂度**:
- 哈希表: O(U), U = 唯一请求数
- skip_slot: O(N)
- 总计: O(N)

**最坏情况**:
- 所有请求都唯一，无法压缩
- 返回 false，后端必须自己 fsync

**最好情况**:
- 大量重复，压缩后队列有空间
- 后端可以继续转发请求

### 4.4 为什么不总是去重？

```c
// checkpointer.c:1104-1117

// 正常情况: 不检查重复，直接插入
request = &CheckpointerShmem->requests[CheckpointerShmem->num_requests++];
request->ftag = *ftag;
request->type = type;

// 原因:
// 1. 检查重复需要 O(N) 时间或 O(U) 空间的哈希表
// 2. 大多数情况下队列不会满
// 3. 只在队列满时才去重（紧急情况）
// 4. Checkpointer 会定期吸收请求，清空队列
```

---

## 5. WAL 文件管理算法

### 5.1 KeepLogSeg 算法

**目的**: 计算需要保留的最老 WAL 段号。

**位置**: `src/backend/access/transam/xlog.c`

```c
static void
KeepLogSeg(XLogRecPtr recptr, XLogSegNo *logSegNo)
{
    XLogSegNo   currSegNo;
    XLogSegNo   segno;
    XLogRecPtr  keep;

    XLByteToSeg(recptr, currSegNo, wal_segment_size);
    segno = currSegNo;

    // 1. 复制槽要求的最老 WAL
    keep = XLogGetReplicationSlotMinimumLSN();
    if (keep != InvalidXLogRecPtr)
    {
        XLByteToSeg(keep, segno, wal_segment_size);
    }

    // 2. wal_keep_size 参数
    if (wal_keep_size_mb > 0)
    {
        uint64 keep_segs;

        keep_segs = XLogMBVarToSegs(wal_keep_size_mb, wal_segment_size);

        if (currSegNo > keep_segs)
            segno = Max(segno, currSegNo - keep_segs);
    }

    // 3. 旧时间线的段
    // (代码省略，涉及时间线切换的复杂逻辑)

    *logSegNo = segno;
}
```

**算法逻辑**:
```
输入:
  recptr: checkpoint 记录的 LSN
  wal_keep_size_mb: 配置参数 (默认 0)
  复制槽: 各个槽的 restart_lsn

计算:
  currSegNo = recptr 对应的段号

  最老段 = min(
    复制槽要求的最老段,
    currSegNo - (wal_keep_size_mb / 16MB),
    旧时间线的段
  )

输出:
  *logSegNo = 最老段号
  < logSegNo 的段可以删除或回收
```

### 5.2 RemoveOldXlogFiles 算法

**目的**: 删除或回收旧的 WAL 段。

```c
static void
RemoveOldXlogFiles(XLogSegNo segno, XLogRecPtr lastredoptr, XLogRecPtr endptr,
                   TimeLineID insertTLI)
{
    DIR        *xldir;
    struct dirent *xlde;
    char        lastoff[MAXFNAMELEN];
    XLogSegNo   endSegNo;

    // 计算 redo 点和 endpoint 的段号
    XLByteToSeg(lastredoptr, endSegNo, wal_segment_size);

    // 遍历 pg_wal/ 目录
    xldir = AllocateDir(XLOGDIR);
    while ((xlde = ReadDir(xldir, XLOGDIR)) != NULL)
    {
        if (IsXLogFileName(xlde->d_name) ||
            IsPartialXLogFileName(xlde->d_name))
        {
            XLogSegNo seg;
            TimeLineID tli;

            // 解析文件名获取 (timeline, segno)
            XLogFromFileName(xlde->d_name, &tli, &seg, wal_segment_size);

            // 判断是否可以删除
            if (tli == insertTLI && seg <= segno)
            {
                // 在 redo 点之前的段
                XLogArchiveCleanup(xlde->d_name);
                // 如果已归档或不需要归档，删除或回收
            }
        }
    }

    FreeDir(xldir);
}
```

**删除 vs 回收**:
```c
// InstallXLogFileSegment()

// 回收: 重命名为未来的段名
if (可以回收)
{
    // 例如: 000000010000000000000003 -> 00000001000000000000000A
    rename(oldname, newname);
}
else
{
    // 删除
    unlink(oldname);
}

// 回收的优点:
// - 避免重新创建文件（fallocate/posix_fallocate）
// - 某些文件系统，预分配文件可以获得更好的块布局
```

### 5.3 PreallocXlogFiles 算法

**目的**: 预分配未来需要的 WAL 段。

```c
static void
PreallocXlogFiles(XLogRecPtr endptr, TimeLineID tli)
{
    XLogSegNo   _logSegNo;
    int         lf;
    bool        use_existent;
    uint64      offset;

    XLByteToSeg(endptr, _logSegNo, wal_segment_size);

    // 计算需要预分配的段数
    // 公式: (2 + CheckPointSegments) * 2 + 1
    // CheckPointSegments = max_wal_size / wal_segment_size
    // 例如: (2 + 64) * 2 + 1 = 133 个段

    offset = 0;
    for (int i = 0; i < XLOGfileslop; i++)
    {
        XLogSegNo segno = _logSegNo + i + 1;

        use_existent = true;
        lf = XLogFileInit(segno, tli, &use_existent, false);

        if (lf >= 0)
            close(lf);

        if (!use_existent)
            CheckpointStats.ckpt_segs_added++;
    }
}
```

**预分配数量公式**:
```
XLOGfileslop = (2 + CheckPointSegments) * 2 + 1

其中:
  CheckPointSegments = ConvertToXSegs(max_wal_size, wal_segment_size)
                     = max_wal_size / wal_segment_size

示例:
  max_wal_size = 1024MB
  wal_segment_size = 16MB
  CheckPointSegments = 64

  XLOGfileslop = (2 + 64) * 2 + 1 = 133 个段

  预分配的磁盘空间 = 133 × 16MB = 2128MB
```

**为什么预分配这么多？**
```
考虑最坏情况:
  1. Checkpoint 刚完成
  2. 立即开始写入大量 WAL
  3. 在下一次 checkpoint 之前，可能写满 max_wal_size

  预分配数量应该覆盖:
    - 当前 checkpoint 周期: CheckPointSegments
    - 下一个 checkpoint 周期: CheckPointSegments
    - 一些缓冲: 2 * 2 + 1 = 5

  总计: 2 × CheckPointSegments + 5 ≈ (2 + CheckPointSegments) * 2 + 1
```

---

## 6. 缓冲区排序算法

### 6.1 排序键定义

```c
// bufmgr.c:3231

static int
ckpt_buforder_comparator(const CkptSortItem *a, const CkptSortItem *b)
{
    // 第 1 优先级: 表空间 OID
    if (a->tsId < b->tsId)
        return -1;
    else if (a->tsId > b->tsId)
        return 1;

    // 第 2 优先级: 关系文件号
    // 注意: 使用 relNumber 而非 relfilenode
    if (a->relNumber < b->relNumber)
        return -1;
    else if (a->relNumber > b->relNumber)
        return 1;

    // 第 3 优先级: Fork 类型
    if (a->forkNum < b->forkNum)
        return -1;
    else if (a->forkNum > b->forkNum)
        return 1;

    // 第 4 优先级: 块号
    if (a->blockNum < b->blockNum)
        return -1;
    else if (a->blockNum > b->blockNum)
        return 1;

    // 完全相同（不应该发生）
    return 0;
}
```

### 6.2 排序效果

**排序前** (随机分布):
```
Buffer    TsID  Rel   Fork  Block
------    ----  ----  ----  -----
buf[0]    1663  1259  0     50      (pg_class)
buf[1]    1664  16385 0     100     (用户表 A)
buf[2]    1663  1259  0     10      (pg_class)
buf[3]    1664  16385 1     5       (用户表 A 的 FSM)
buf[4]    1663  2608  0     200     (pg_attribute)
...

I/O 模式: 随机，磁头跳来跳去
```

**排序后** (有序):
```
Buffer    TsID  Rel   Fork  Block
------    ----  ----  ----  -----
buf[0]    1663  1259  0     10      (pg_class, block 10)
buf[1]    1663  1259  0     50      (pg_class, block 50)
buf[2]    1663  2608  0     200     (pg_attribute, block 200)
buf[3]    1664  16385 0     100     (用户表 A, main fork, block 100)
buf[4]    1664  16385 1     5       (用户表 A, FSM fork, block 5)
...

I/O 模式: 顺序或近顺序，磁头移动少
```

### 6.3 性能提升

**理论分析**:
```
机械硬盘 (HDD):
  随机 I/O: 100-200 IOPS
  顺序 I/O: 50-100 MB/s ≈ 6000-12000 IOPS (8KB 页)
  提升: 30-120 倍

固态硬盘 (SSD):
  随机 I/O: 10000-50000 IOPS
  顺序 I/O: 50000-500000 IOPS
  提升: 2-10 倍

即使在 SSD 上，排序也有显著效果
```

**实际测试** (以 10000 个脏页为例):
```
HDD (7200 RPM):
  不排序: 10000 / 150 IOPS = 67 秒
  排序后: 10000 / 5000 IOPS = 2 秒
  加速比: 33 倍

SSD (SATA):
  不排序: 10000 / 20000 IOPS = 0.5 秒
  排序后: 10000 / 80000 IOPS = 0.125 秒
  加速比: 4 倍
```

---

## 7. 算法复杂度分析

### 7.1 CreateCheckPoint 总体复杂度

| 操作 | 时间复杂度 | 空间复杂度 | 说明 |
|------|-----------|-----------|------|
| 准备阶段 | O(1) | O(1) | 常数时间 |
| 确定 REDO 点 | O(L) | O(1) | L = WAL 插入锁数量 (8) |
| 收集元数据 | O(1) | O(1) | 常数时间（加锁读取） |
| CheckPointGuts | **O(N log N)** | **O(N)** | **主要开销** |
| 写 WAL 记录 | O(1) | O(1) | 单条记录 |
| 更新 pg_control | O(1) | O(1) | 单个文件 |
| WAL 文件管理 | O(W) | O(1) | W = WAL 文件数 |

### 7.2 CheckPointGuts 详细复杂度

| 操作 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| CheckPointCLOG() 等 SLRU | O(P) | O(1) |
| **BufferSync()** | **O(N log N + N log S)** | **O(N)** |
| ProcessSyncRequests() | O(R log R) | O(R) |
| CheckPointTwoPhase() | O(T) | O(1) |

其中:
- N = NBuffers (缓冲区总数)
- S = 表空间数量
- P = SLRU 页面数
- R = Fsync 请求数
- T = 两阶段事务数

### 7.3 BufferSync 详细复杂度

| 步骤 | 时间复杂度 | 说明 |
|------|-----------|------|
| 1. 扫描标记 | O(N) | 遍历所有缓冲区 |
| 2. 排序 | O(N log N) | qsort |
| 3. 分组 | O(N) | 单次遍历 |
| 4. 构建堆 | O(S log S) | S << N |
| 5. 写入循环 | O(N log S) | N 次，每次 O(log S) |

**总计**: O(N) + O(N log N) + O(N) + O(S log S) + O(N log S)
        = **O(N log N)** (主导项)

**空间复杂度**: O(N) (CkptBufferIds 数组)

### 7.4 实际性能数据

**测试配置**:
- shared_buffers = 1GB (NBuffers = 131072)
- 假设 10000 个脏页 (7.6%)
- 3 个表空间

**操作耗时**:
```
扫描标记:   131072 次 spinlock 操作     = 130ms
排序:      10000 × log2(10000) ≈ 133k 次比较 = 10ms
分组:      10000 次遍历                = 5ms
构建堆:    3 × log2(3) ≈ 5 次操作      = 0.01ms
写入循环:  10000 × log2(3) × I/O 时间  = 主要开销

理论计算时间: 145ms (不含 I/O)
实际 I/O 时间: 30 秒 (受 checkpoint_completion_target 控制)

结论: 算法开销可忽略不计，主要时间在 I/O
```

### 7.5 优化潜力分析

**排序算法**:
- 当前: qsort, O(N log N)
- 优化潜力: 基数排序, O(N × k), k = 键长度
- 实际: 排序耗时很小，优化意义不大

**堆操作**:
- 当前: 二叉堆, O(log S)
- 优化潜力: Fibonacci 堆, O(1) 摊销
- 实际: S 很小 (< 10)，优化无意义

**并行化**:
- 可能性: BufferSync 可以并行扫描
- 挑战: 需要仔细同步，避免数据竞争
- 实际: 当前未实现，未来可能优化

**结论**: 现有算法已经很高效，主要瓶颈在 I/O 而非 CPU。

---

## 总结

本文档深入分析了 PostgreSQL Checkpoint 的关键算法：

1. **Checkpoint 调度算法** - 通过 progress 和双重检查实现 I/O 平滑
2. **REDO 点计算算法** - Online 和 Shutdown 两种策略
3. **表空间负载均衡算法** - 最小堆实现公平调度
4. **Fsync 队列压缩算法** - 哈希去重节省空间
5. **WAL 文件管理算法** - 精确计算保留策略
6. **缓冲区排序算法** - 多级排序减少随机 I/O
7. **复杂度分析** - 证明算法高效性

这些算法共同实现了高效、可靠的 checkpoint 机制。

---

**下一步**: 阅读 `05_performance_optimization.md` 了解性能调优技术。
