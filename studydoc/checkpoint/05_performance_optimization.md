# PostgreSQL Checkpoint - 性能优化技术

## 目录

1. [渐进式 Checkpoint](#1-渐进式-checkpoint)
2. [I/O 优化技术](#2-io-优化技术)
3. [锁优化技术](#3-锁优化技术)
4. [配置参数调优](#4-配置参数调优)
5. [监控和诊断](#5-监控和诊断)
6. [常见问题和解决方案](#6-常见问题和解决方案)

---

## 1. 渐进式 Checkpoint

### 1.1 原理

**问题**: 一次性刷写所有脏页会导致 I/O 突发，影响正常查询。

**解决方案**: 分散 checkpoint 的 I/O，在指定时间内平滑完成。

### 1.2 实现机制

```c
// checkpointer.c:711
void CheckpointWriteDelay(int flags, double progress)
{
    if (!(flags & CHECKPOINT_IMMEDIATE) &&
        IsCheckpointOnSchedule(progress))
    {
        // 进度正常，休眠 100ms
        WaitLatch(MyLatch, WL_LATCH_SET | WL_TIMEOUT, 100,
                  WAIT_EVENT_CHECKPOINT_WRITE_DELAY);
    }
}

// checkpointer.c:780
static bool IsCheckpointOnSchedule(double progress)
{
    // 调整进度
    progress *= CheckPointCompletionTarget;  // 默认 0.9

    // 检查 WAL 进度
    elapsed_xlogs = (recptr - ckpt_start_recptr) / wal_segment_size / CheckPointSegments;
    if (progress < elapsed_xlogs)
        return false;  // 落后，加速

    // 检查时间进度
    elapsed_time = (now - ckpt_start_time) / CheckPointTimeout;
    if (progress < elapsed_time)
        return false;  // 落后，加速

    return true;  // 进度正常
}
```

### 1.3 效果对比

**场景**: 10000 个脏页需要刷写

#### 不使用渐进式 (CHECKPOINT_IMMEDIATE)

```
时间线: |--------|
        0s      10s

I/O 曲线:
  ▲
  │    ████████
  │    ████████
  │    ████████
  │____████████_________________
  0s            10s

特点:
- 完成快 (10 秒)
- I/O 峰值高 (1000 IOPS)
- 严重影响并发查询
```

#### 使用渐进式 (completion_target=0.9)

```
时间线: |--------------------------|
        0s                        270s

I/O 曲线:
  ▲
  │ ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
  │ ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
  │ ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
  │_▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
  0s                           270s

特点:
- 完成慢 (270 秒)
- I/O 平滑 (37 IOPS)
- 对并发查询影响小
```

### 1.4 调优建议

**OLTP 系统** (高并发读写):
```sql
-- 拉长 checkpoint 时间，减少影响
checkpoint_timeout = 15min          -- 增加到 15 分钟
max_wal_size = 4GB                  -- 增加 WAL 容量
checkpoint_completion_target = 0.9  -- 保持默认
```

**OLAP 系统** (大批量操作):
```sql
-- 加快 checkpoint 完成，腾出更多时间给批处理
checkpoint_timeout = 30min          -- 更长间隔
max_wal_size = 16GB                 -- 大容量
checkpoint_completion_target = 0.5  -- 加快完成
```

**混合负载**:
```sql
-- 平衡策略
checkpoint_timeout = 10min
max_wal_size = 2GB
checkpoint_completion_target = 0.7  -- 稍微加快
```

---

## 2. I/O 优化技术

### 2.1 排序优化

**原理**: 将随机 I/O 转换为顺序 I/O。

#### 2.1.1 未排序的问题

```
脏页分布 (逻辑):
  TableA Block 100
  TableB Block 50
  TableA Block 10
  TableC Block 200
  TableA Block 150

磁盘布局:
  [TableA: blocks 1-200] [TableB: blocks 1-100] [TableC: blocks 1-300]
  |                     ||                     ||                        |
  Sector 0-1000         Sector 1001-1500      Sector 1501-3000

写入顺序 (未排序):
  1. Seek to TableA block 100 (sector 500)
  2. Seek to TableB block 50 (sector 1050)  <- 长距离寻道
  3. Seek to TableA block 10 (sector 50)    <- 回退
  4. Seek to TableC block 200 (sector 2200) <- 跳跃
  5. Seek to TableA block 150 (sector 750)  <- 再次回退

总寻道距离: 500 + 550 + 1000 + 2150 + 1450 = 5650 sectors
```

#### 2.1.2 排序后的优化

```
排序后的写入顺序:
  1. TableA block 10 (sector 50)
  2. TableA block 100 (sector 500)   <- 顺序
  3. TableA block 150 (sector 750)   <- 顺序
  4. TableB block 50 (sector 1050)   <- 短距离
  5. TableC block 200 (sector 2200)  <- 顺序切换

总寻道距离: 50 + 450 + 250 + 300 + 1150 = 2200 sectors

优化效果: 5650 -> 2200 = 60% 减少
```

#### 2.1.3 实现代码

```c
// bufmgr.c:2987
sort_checkpoint_bufferids(CkptBufferIds, num_to_scan);

// 排序键: (表空间, 关系, fork, 块号)
static int
ckpt_buforder_comparator(const CkptSortItem *a, const CkptSortItem *b)
{
    if (a->tsId != b->tsId)
        return (a->tsId < b->tsId) ? -1 : 1;
    if (a->relNumber != b->relNumber)
        return (a->relNumber < b->relNumber) ? -1 : 1;
    if (a->forkNum != b->forkNum)
        return (a->forkNum < b->forkNum) ? -1 : 1;
    if (a->blockNum != b->blockNum)
        return (a->blockNum < b->blockNum) ? -1 : 1;
    return 0;
}
```

### 2.2 表空间负载均衡

**原理**: 多个表空间并行写入，避免单磁盘过载。

#### 2.2.1 问题场景

```
假设 3 个表空间在不同的磁盘:
  /dev/sda: 表空间 ts1 (1000 个脏页)
  /dev/sdb: 表空间 ts2 (500 个脏页)
  /dev/sdc: 表空间 ts3 (300 个脏页)

简单顺序写入:
  0-10s:   /dev/sda 全负荷 (1000 页) ████████████
  10-15s:  /dev/sdb 全负荷 (500 页)  ██████
  15-18s:  /dev/sdc 全负荷 (300 页)  ███

问题:
  - 单磁盘顺序满负荷
  - 其他磁盘空闲
  - 无法利用并行 I/O
```

#### 2.2.2 负载均衡效果

```
使用最小堆负载均衡:
  0-1s:    /dev/sda █  /dev/sdb █  /dev/sdc █
  1-2s:    /dev/sda █  /dev/sdb █  /dev/sdc █
  ...
  17-18s:  /dev/sda █  /dev/sdb █  /dev/sdc █

效果:
  - 三个磁盘并行工作
  - 总时间相同，但峰值负载降低
  - 利用了多磁盘并行带宽
```

#### 2.2.3 实现关键

```c
// bufmgr.c:3069
ts_stat->progress_slice = (float8) num_to_scan / ts_stat->num_to_scan;

// ts1: 1800 / 1000 = 1.8  (页少，单页贡献大)
// ts2: 1800 / 500 = 3.6
// ts3: 1800 / 300 = 6.0

// 最小堆确保进度最慢的优先写入
while (!binaryheap_empty(ts_heap))
{
    ts_stat = binaryheap_first(ts_heap);  // 进度最小的
    SyncOneBuffer(ts_stat->next_buffer);
    ts_stat->progress += ts_stat->progress_slice;
    binaryheap_replace_first(ts_heap, ts_stat);  // 重新调整
}
```

### 2.3 Writeback 机制

**原理**: 提示操作系统提前刷写页缓存，避免后续 fsync 阻塞。

```c
// bufmgr.c:3111
ScheduleBufferTagForWriteback(&wb_context, IOOBJECT_RELATION, &tag);

// 定期发出 writeback
// bufmgr.c:3150
IssuePendingWritebacks(&wb_context, IOCONTEXT_NORMAL);
```

**工作原理**:
```
正常流程 (无 writeback):
  1. write() 写入页缓存                  <- 快
  2. 继续写其他页...
  3. fsync() 刷写所有页缓存到磁盘        <- 慢，阻塞

使用 writeback:
  1. write() 写入页缓存                  <- 快
  2. posix_fadvise(POSIX_FADV_DONTNEED)  <- 提示 OS
  3. OS 开始异步刷写                     <- 后台
  4. 继续写其他页...
  5. fsync() 只需等待少量未完成的 I/O    <- 快
```

**配置参数**:
```sql
-- 每写多少个缓冲区发出一次 writeback
checkpoint_flush_after = 256kB  -- 默认 256kB / 8kB = 32 个页
```

---

## 3. 锁优化技术

### 3.1 短锁持有时间

**原则**: 在锁内只做必要的操作，I/O 在锁外。

#### 3.1.1 反例 (不好的实现)

```c
// 假设的不好的实现
LWLockAcquire(BufferContentLock, LW_EXCLUSIVE);
memcpy(temp_buffer, bufHdr->data, BLCKSZ);      // 8KB 内存拷贝
write(fd, temp_buffer, BLCKSZ);                  // 磁盘 I/O，可能几毫秒
fsync(fd);                                       // 更慢！
LWLockRelease(BufferContentLock);

// 锁持有时间: 几毫秒到几十毫秒
// 其他进程访问此缓冲区: 阻塞
```

#### 3.1.2 正确实现

```c
// bufmgr.c:3262 SyncOneBuffer
LockBufHdr(bufHdr);                              // 获取 spinlock
buf_state = bufHdr->state;
if (!(buf_state & BM_CHECKPOINT_NEEDED)) {
    UnlockBufHdr(bufHdr);
    return;
}
PinBuffer_Locked(bufHdr);                        // Pin 住
tag = bufHdr->tag;                               // 复制 tag
buf_state &= ~BM_CHECKPOINT_NEEDED;              // 清除标志
UnlockBufHdr(bufHdr, buf_state);                 // 释放 spinlock

// ★ 锁外执行 I/O ★
if (buf_state & BM_DIRTY) {
    FlushBuffer(bufHdr, NULL);                   // 磁盘 I/O
}

UnpinBuffer(bufHdr);

// Spinlock 持有时间: 几微秒
// I/O 在锁外: 不阻塞其他进程
```

### 3.2 避免伪共享

**伪共享问题**:
```
CPU 缓存行大小: 64 字节

不好的布局:
struct {
    volatile int counter1;  // 0-3 字节
    volatile int counter2;  // 4-7 字节
    // ... 同一缓存行
};

CPU1 修改 counter1 -> 使 counter2 所在缓存行失效
CPU2 修改 counter2 -> 使 counter1 所在缓存行失效
-> 缓存行在 CPU 间来回传递 (cache ping-pong)
```

**PostgreSQL 的解决**:
```c
// xlog.c:382
typedef union WALInsertLockPadded
{
    WALInsertLock l;
    char pad[PG_CACHE_LINE_SIZE];  // 填充到 64 字节
} WALInsertLockPadded;

// 数组布局:
// [Lock0: 64B] [Lock1: 64B] [Lock2: 64B] ...
// 每个锁独占一个缓存行，无伪共享
```

### 3.3 读写锁分离

**场景**: 允许多个读者并发访问。

```c
// LWLock 支持共享模式和排它模式

// 读取配置
LWLockAcquire(SomeLock, LW_SHARED);  // 多个进程可同时持有
value = SharedData->config;
LWLockRelease(SomeLock);

// 更新配置
LWLockAcquire(SomeLock, LW_EXCLUSIVE);  // 独占
SharedData->config = new_value;
LWLockRelease(SomeLock);
```

---

## 4. 配置参数调优

### 4.1 参数矩阵

| 参数 | OLTP | OLAP | 混合 | 说明 |
|------|------|------|------|------|
| **shared_buffers** | 2-4GB | 8-16GB | 4-8GB | 缓冲池大小 |
| **checkpoint_timeout** | 15-30min | 30-60min | 15-30min | checkpoint 间隔 |
| **max_wal_size** | 4-8GB | 16-32GB | 8-16GB | WAL 触发阈值 |
| **checkpoint_completion_target** | 0.9 | 0.5-0.7 | 0.7-0.9 | 完成目标 |
| **checkpoint_flush_after** | 256kB | 1MB | 256kB | writeback 阈值 |
| **wal_buffers** | 16MB | 64MB | 32MB | WAL 缓冲区 |

### 4.2 调优决策树

```
┌─────────────────────────────┐
│ 观察到 checkpoint 频繁警告？ │
└─────────┬──────────────────┘
          │
      是  │  否 (正常)
          ▼
┌─────────────────────────────┐
│ 增大 max_wal_size            │
│ 例如: 1GB -> 4GB             │
└─────────┬──────────────────┘
          │
          ▼
┌─────────────────────────────┐
│ 问题解决？                   │
└─────────┬──────────────────┘
          │
      否  │  是 (结束)
          ▼
┌─────────────────────────────┐
│ checkpoint 期间查询变慢？    │
└─────────┬──────────────────┘
          │
      是  │  否
          ▼
┌─────────────────────────────┐
│ 增大 checkpoint_timeout      │
│ 降低 completion_target       │
│ 例如: 0.9 -> 0.7             │
└─────────┬──────────────────┘
          │
          ▼
┌─────────────────────────────┐
│ 恢复时间可接受？             │
└─────────┬──────────────────┘
          │
      否  │  是 (结束)
          ▼
┌─────────────────────────────┐
│ 恢复时间过长                 │
│ 需要在性能和恢复间平衡       │
│ 考虑:                        │
│ - 增大 completion_target     │
│ - 减小 checkpoint_timeout    │
│ - 使用流复制减少依赖恢复     │
└─────────────────────────────┘
```

### 4.3 实战案例

#### 案例 1: 电商 OLTP 系统

**问题**:
```
日志显示:
  LOG: checkpoints are occurring too frequently (30 seconds apart)
  HINT: Consider increasing max_wal_size.

分析:
  - 高峰期每秒几千个事务
  - WAL 生成速度: ~35MB/s
  - max_wal_size = 1GB
  - 触发时间: 1024MB / 35MB/s ≈ 29 秒

影响:
  - Checkpoint 过于频繁
  - I/O 峰值影响查询
```

**解决方案**:
```sql
-- 调整前
max_wal_size = 1GB
checkpoint_timeout = 5min
checkpoint_completion_target = 0.9

-- 调整后
max_wal_size = 4GB               -- 4 倍容量
checkpoint_timeout = 15min       -- 3 倍间隔
checkpoint_completion_target = 0.9  -- 保持

-- 效果
- Checkpoint 间隔: 29s -> 4-5 分钟
- WAL 使用: 峰值 2.5GB, 低谷 1GB
- 查询延迟: P99 降低 30%
```

#### 案例 2: 数据仓库 OLAP 系统

**问题**:
```
场景:
  - 夜间 ETL 批量加载
  - 白天只读查询

观察:
  - Checkpoint 耗时 5-10 分钟
  - ETL 期间 I/O 竞争
  - 加载速度慢
```

**解决方案**:
```sql
-- ETL 期间配置
ALTER SYSTEM SET checkpoint_timeout = '60min';
ALTER SYSTEM SET max_wal_size = '32GB';
ALTER SYSTEM SET checkpoint_completion_target = 0.5;
SELECT pg_reload_conf();

-- 加载后手动 checkpoint
CHECKPOINT;

-- 白天恢复正常配置
ALTER SYSTEM RESET checkpoint_timeout;
ALTER SYSTEM RESET max_wal_size;
ALTER SYSTEM RESET checkpoint_completion_target;
SELECT pg_reload_conf();

-- 效果
- ETL 加载速度: 提升 40%
- Checkpoint 次数: 10次 -> 2次
- 恢复时间仍可控 (< 5 分钟)
```

---

## 5. 监控和诊断

### 5.1 关键指标

#### 5.1.1 pg_stat_bgwriter

```sql
SELECT
    checkpoints_timed,                  -- 时间触发的 checkpoint 数
    checkpoints_req,                    -- 请求触发的 checkpoint 数
    checkpoint_write_time,              -- 写脏页总时间 (ms)
    checkpoint_sync_time,               -- fsync 总时间 (ms)
    buffers_checkpoint,                 -- checkpoint 写入的缓冲区数
    buffers_clean,                      -- bgwriter 写入的缓冲区数
    maxwritten_clean,                   -- bgwriter 因超过限制而停止的次数
    buffers_backend,                    -- 后端自己写入的缓冲区数
    buffers_backend_fsync,              -- 后端自己 fsync 的次数
    buffers_alloc,                      -- 分配的缓冲区数
    stats_reset                         -- 统计重置时间
FROM pg_stat_bgwriter;
```

**健康指标**:
```
checkpoints_timed / (checkpoints_timed + checkpoints_req) > 0.9
  -> 大部分 checkpoint 由时间触发，WAL 大小设置合理

buffers_backend_fsync = 0
  -> 后端从不需要自己 fsync，checkpointer 队列足够

checkpoint_write_time / (checkpoint_write_time + checkpoint_sync_time) > 0.9
  -> 写入时间占主导，fsync 时间短，writeback 生效
```

#### 5.1.2 日志分析

```bash
# 提取 checkpoint 日志
grep "checkpoint complete" postgresql.log | tail -20

# 示例输出
LOG:  checkpoint complete: wrote 12043 buffers (73.5%);
      0 WAL file(s) added, 3 removed, 2 recycled;
      write=45.231 s, sync=1.204 s, total=46.587 s;
      sync files=287, longest=0.156 s, average=0.004 s;
      distance=193562 kB, estimate=195123 kB

# 关键指标分析
# 1. 写入比例: 73.5% -> 合理 (50-90%)
# 2. 写入时间: 45.2s, 同步时间: 1.2s -> 比例正常
# 3. 同步文件数: 287 -> 较多，考虑增加 max_files_per_process
# 4. 最长同步: 0.156s -> 正常 (< 1s)
# 5. 距离 vs 估计: 193MB vs 195MB -> 估计准确
```

### 5.2 监控脚本

```bash
#!/bin/bash
# checkpoint_monitor.sh

# 数据库连接参数
PGHOST=localhost
PGPORT=5432
PGDATABASE=postgres
PGUSER=postgres

# 监控间隔 (秒)
INTERVAL=60

echo "Checkpoint Monitor - $(date)"
echo "=================================================="

while true; do
    psql -h $PGHOST -p $PGPORT -d $PGDATABASE -U $PGUSER <<EOF
    \x
    SELECT
        now() AS sample_time,
        checkpoints_timed,
        checkpoints_req,
        round(checkpoint_write_time::numeric / 1000, 2) AS write_time_sec,
        round(checkpoint_sync_time::numeric / 1000, 2) AS sync_time_sec,
        buffers_checkpoint,
        buffers_backend_fsync,
        round(buffers_checkpoint::numeric * 100 /
              NULLIF(buffers_checkpoint + buffers_clean + buffers_backend, 0), 2)
              AS checkpoint_write_pct
    FROM pg_stat_bgwriter;
EOF

    sleep $INTERVAL
done
```

### 5.3 系统级监控

```bash
# 监控磁盘 I/O
iostat -x 5

# 关注指标:
# - %util: 磁盘利用率 (< 80% 健康)
# - await: 平均等待时间 (< 10ms 健康)
# - svctm: 平均服务时间

# 监控 checkpoint 进程
ps aux | grep checkpointer

# 监控 I/O
pidstat -d 5 -p $(pgrep -f checkpointer)

# 磁盘写入速率
vmstat 5
```

---

## 6. 常见问题和解决方案

### 6.1 问题 1: Checkpoint 过于频繁

**症状**:
```
LOG: checkpoints are occurring too frequently (45 seconds apart)
HINT: Consider increasing max_wal_size.
```

**诊断**:
```sql
-- 查看 WAL 生成速率
SELECT
    pg_current_wal_lsn(),
    pg_walfile_name(pg_current_wal_lsn());

-- 等待 60 秒

SELECT
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), '上次的LSN')
    ) AS wal_generated_per_minute;

-- 输出: 例如 "560 MB"
```

**解决方案**:
```sql
-- 方案 1: 增大 max_wal_size
ALTER SYSTEM SET max_wal_size = '4GB';
SELECT pg_reload_conf();

-- 方案 2: 如果磁盘空间受限，增大 checkpoint_timeout
ALTER SYSTEM SET checkpoint_timeout = '15min';
SELECT pg_reload_conf();

-- 方案 3: 优化应用减少 WAL 生成
-- - 使用 COPY 而非 INSERT
-- - 批量提交
-- - 考虑 unlogged 表 (临时数据)
```

### 6.2 问题 2: Checkpoint 耗时过长

**症状**:
```
-- 日志显示
checkpoint complete: total=450.123 s

-- 查询在 checkpoint 期间变慢
```

**诊断**:
```sql
-- 检查脏页比例
SELECT
    pg_size_pretty(
        (SELECT setting FROM pg_settings WHERE name = 'shared_buffers')::bigint * 8192
    ) AS shared_buffers,
    pg_size_pretty(
        buffers_checkpoint * 8192::bigint
    ) AS checkpoint_written
FROM pg_stat_bgwriter;

-- 如果 checkpoint_written 接近 shared_buffers
-- 说明几乎所有缓冲区都是脏的
```

**解决方案**:
```sql
-- 方案 1: 增大缓冲池，减少脏页比例
ALTER SYSTEM SET shared_buffers = '8GB';
-- 需要重启

-- 方案 2: 调整 completion_target
ALTER SYSTEM SET checkpoint_completion_target = 0.7;
-- 加快完成，但 I/O 峰值会增大

-- 方案 3: 启用 huge pages (减少 TLB miss)
ALTER SYSTEM SET huge_pages = 'on';
-- 需要重启并配置 OS
```

### 6.3 问题 3: Fsync 时间过长

**症状**:
```
checkpoint complete: write=50.123 s, sync=45.234 s
                                         ^^^^^^^^ 占比过高
```

**诊断**:
```bash
# 检查磁盘队列
iostat -x 1

# 如果 avgqu-sz 很大 (> 10)
# 说明磁盘队列积压严重
```

**解决方案**:
```sql
-- 方案 1: 增大 checkpoint_flush_after，更频繁 writeback
ALTER SYSTEM SET checkpoint_flush_after = '128kB';
SELECT pg_reload_conf();

-- 方案 2: 使用更快的磁盘
-- - 从 HDD 升级到 SSD
-- - 使用 RAID 10 增加并行度

-- 方案 3: 文件系统调优
-- - 使用 XFS 而非 ext4 (更好的 fsync 性能)
-- - 挂载选项: noatime, nobarrier (仔细评估风险)
```

### 6.4 问题 4: 恢复时间过长

**症状**:
```
-- 崩溃后启动
LOG: database system was interrupted; last known up at 2025-01-15 10:00:00 CST
LOG: starting point-in-time recovery to WAL location (redo point)
...
(20 分钟后)
LOG: redo done at 0/A0000000
```

**诊断**:
```sql
-- 检查 checkpoint 间隔
SELECT
    age(now(), stats_reset) AS uptime,
    checkpoints_timed,
    checkpoints_req,
    checkpoints_timed + checkpoints_req AS total_checkpoints,
    age(now(), stats_reset) / (checkpoints_timed + checkpoints_req) AS avg_interval
FROM pg_stat_bgwriter;
```

**解决方案**:
```sql
-- 方案 1: 减小 checkpoint 间隔
ALTER SYSTEM SET checkpoint_timeout = '5min';
ALTER SYSTEM SET max_wal_size = '1GB';

-- 方案 2: 使用流复制 + 故障转移
-- 而非依赖崩溃恢复

-- 方案 3: 更快的 WAL 重放
-- - 使用 SSD 存储 WAL
-- - 增大 wal_buffers
```

### 6.5 问题 5: 后端被迫 fsync

**症状**:
```sql
SELECT buffers_backend_fsync FROM pg_stat_bgwriter;
-- 输出: 非零值

-- 意味着 fsync 请求队列满了
-- 后端不得不自己执行 fsync (很慢)
```

**诊断**:
```sql
-- 检查缓冲区数量
SHOW shared_buffers;
-- 输出: 例如 1GB = 131072 个 8KB 页

-- fsync 队列大小 = NBuffers
```

**解决方案**:
```sql
-- 方案 1: 增大 shared_buffers
ALTER SYSTEM SET shared_buffers = '2GB';
-- 队列大小翻倍

-- 方案 2: 降低写入负载
-- - 分散写入高峰
-- - 使用连接池减少并发

-- 方案 3: 更频繁的 checkpoint
ALTER SYSTEM SET checkpoint_timeout = '3min';
-- 更频繁清空队列
```

---

## 总结

本文档介绍了 PostgreSQL Checkpoint 的性能优化技术：

1. **渐进式 Checkpoint** - 平滑 I/O，减少峰值
2. **I/O 优化** - 排序、负载均衡、writeback
3. **锁优化** - 短锁、避免伪共享
4. **配置调优** - 针对不同场景的参数矩阵
5. **监控诊断** - 关键指标和监控脚本
6. **问题解决** - 5 个常见问题的诊断和解决方案

通过合理配置和优化，可以显著提升 checkpoint 性能和系统稳定性。

---

**下一步**: 阅读 `06_verification_testcases.md` 进行实践验证。
