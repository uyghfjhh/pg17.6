# WAL 关键算法与复杂度分析

> 深入分析 WAL 系统中的关键算法，包括时间复杂度、空间复杂度和性能优化技巧。

---

## 目录

1. [LSN 计算和转换算法](#一lsn-计算和转换算法)
2. [WAL 缓冲区管理算法](#二wal-缓冲区管理算法)
3. [压缩和优化算法](#三压缩和优化算法)
4. [恢复算法分析](#四恢复算法分析)
5. [文件管理算法](#五文件管理算法)
6. [性能优化算法](#六性能优化算法)
7. [复杂度总结](#七复杂度总结)

---

## 一、LSN 计算和转换算法

### 1.1 LSN 编解码算法

**LSN 结构**:
```c
// 64位 LSN 编码
// 高32位: 段内偏移 (segment offset)
// 低32位: 页内偏移 (page offset)
typedef uint64 XLogRecPtr;
```

#### 1.1.1 LSN 到文件路径转换

**时间复杂度**: O(1) - 纯计算

```c
// src/backend/access/transam/xlog.c:4183
void
XLogFilePath(char *path, TimeLineID tli, XLogRecPtr segno, int seg_size)
{
    uint32 log, seg;
    
    // 分解 segno 为 log 和 seg 部分 - O(1)
    XLByteToSeg(segno, log, seg, seg_size);
    
    // 格式化为文件名 - O(1)
    snprintf(path, MAXPGPATH, "%s/%08X%08X%08X",
             XLOGDIR, tli, log, seg);
}

// 分解宏 - 编译时计算
#define XLByteToSeg(segno, log, seg, segsize) \
    do { \
        (log) = (segno) / ((segsize) / XLog_SEG_SIZE); \
        (seg) = (segno) % ((segsize) / XLOG_SEG_SIZE); \
    } while (0)
```

**性能分析**:
- 查表: `log = segno / (2048)` (除法可优化为移位)
- 取模: `seg = segno % 2048` (取模可优化为位运算)
- 路径构造: 格式化字符串 (固定长度)

#### 1.1.2 LSN 距离计算

**时间复杂度**: O(1) - 数学运算

```c
// src/include/access/xlogdefs.h
static inline uint64
XLogRecPtrDiff(XLogRecPtr r1, XLogRecPtr r2)
{
    if (r1 >= r2)
        return r1 - r2;
    else
        return ((uint64) 1 << 63) * 2 - (r2 - r1);
}
```

**实际应用**:
```sql
-- SQL 接口，C 实现为 O(1)
SELECT pg_wal_lsn_diff('1/2B000000', '1/2A000000');
-- 返回: 1048576 (1MB)

-- 大批量计算示例
SELECT 
    pg_wal_lsn_diff(current_wal_flush_lsn(), 
                    pg_last_wal_replay_lsn()) AS lag_bytes,
    pg_size_pretty(
        pg_wal_lsn_diff(current_wal_flush_lsn(),
                       pg_last_wal_replay_lsn())
    ) AS lag_size;
```

### 1.2 LSN 定位算法

#### 1.2.1 二分查找定位 WAL 记录

**问题**: 在 WAL 段文件中快速定位特定 LSN

**时间复杂度**: O(log N) - N = 段内页数

```c
// 伪代码实现，实际在 xlogreader.c 中
static XLogRecPtr
FindSegmentStartPoint(XLogRecPtr lsn)
{
    // 1. 计算段文件位置 - O(1)
    XLogSegNo segno;
    XLByteToSeg(lsn, segno, dump_segno, WalSegSz);
    
    // 2. 打开段文件 - O(1) (可能缓存命中)
    int fd = XLogFileOpen(segno);
    
    // 3. 二分查找定位页面 - O(log N)
    // WalSegSz/XLogBlcksz = 2048 页 (N=2048)
    uint32 low = 0, high = WalSegSz / XLogBlcksz;
    uint32 target_page = (lsn % XLogSegSz) / XLogBlcksz;
    
    while (low <= high) {
        uint32 mid = (low + high) / 2;
        XLogPageHeader pagehdr = ReadPage(fd, mid);
        
        if (pagehdr->xlp_pageaddr <= lsn) {
            if (mid == high || 
                ReadPage(fd, mid + 1)->xlp_pageaddr > lsn) {
                return GetPageStart(mid);
            }
            low = mid + 1;
        } else {
            high = mid - 1;
        }
    }
    
    return 0; // 未找到
}
```

**优化**: 由于 WAL 是顺序写入，通常只需要顺序检查几页

---

## 二、WAL 缓冲区管理算法

### 2.1 循环缓冲区算法

**结构**: 环形缓冲区，支持并发读写

**时间复杂度**:
- 插入: O(1) - 原子操作
- 读取: O(1) - 直接访问
- 查询: O(k) - k = 记录数

```c
// WAL 缓冲区状态管理
typedef struct WALBufferState {
    pg_atomic_uint64 curr_pos;     // 当前写入位置
    pg_atomic_uint64 flush_pos;    // 已刷写位置
    uint32         buffer_size;    // 缓冲区总大小
    char          *buffer;         // 缓冲区起始地址
} WALBufferState;

// 插入算法 - O(1)
static XLogRecPtr
ReserveWALBuffer(uint32 size)
{
    XLogRecPtr old_pos, new_pos;
    
    // 原子获取旧位置
    old_pos = pg_atomic_fetch_add_u64(&state.curr_pos, size);
    new_pos = old_pos + size;
    
    // 检查缓冲区是否足够
    if (new_pos - pg_atomic_read_u64(&state.flush_pos) > state.buffer_size) {
        // 缓冲区满，需要等待
        WaitForFlush(old_pos);
    }
    
    return old_pos;
}
```

### 2.2 并发写入优化

#### 2.2.1 槽位预留算法

**问题**: 多个 Backend 同时写入时的并发控制

**解决方案**: 原子预留 + 批量写入

```c
// 批量预留槽位 - 减少锁竞争
static XLogRecPtr *
ReserveMultipleSlots(int count, uint32 *sizes)
{
    XLogRecPtr *slots = palloc(count * sizeof(XLogRecPtr));
    XLogRecPtr curr_pos;
    
    // 原子获取起始位置 - O(1)
    curr_pos = pg_atomic_fetch_add_u64(&state.curr_pos,
                                       sum_sizes(sizes, count));
    
    // 计算每个槽位位置 - O(count)
    for (int i = 0; i < count; i++) {
        slots[i] = curr_pos;
        curr_pos += sizes[i];
    }
    
    return slots;
}
```

#### 2.2.2 无锁队列算法

**应用场景**: WAL Writer 的任务队列

**时间复杂度**: O(1) 入队，O(1) 出队

```c
// Michael-Scott 无锁队列
typedef struct WALTa无菌头
{
    _Atomic(struct WALTask *) next;
} WALTa无菌头;

bool
EnqueueWALTa无菌头(WALTa无菌头 *head, struct WALTask *task)
{
    struct WALTask *last, *next;
    
    // 1. 定位队尾
    while (true) {
        last = atomic_load(&head->next);
        next = atomic_load(&last->next);
        
        if (last == atomic_load(&head->next)) {
            if (next == NULL) {
                // 尝试添加到队尾
                if (atomic_compare_exchange_weak(&last->next, &next, task))
                    return true;
            } else {
                // 帮助推进 tail 指针
                atomic_compare_exchange_weak(&head->next, &last, next);
            }
        }
    }
}
```

---

## 三、压缩和优化算法

### 3.1 全页镜像压缩算法

#### 3.1.1 Hole Detection 算法

**时间复杂度**: O(B) - B = 页面大小 (8KB)

```c
// src/backend/access/transam/xloginsert.c:压缩函数
static char *
compress_page(Page page, uint32 *compressed_size)
{
    char      *compressed;
    uint32     hole_offset = 0;
    uint32     hole_length = 0;
    uint32     i;
    
    // 1. 查找最大的连续空白区域 - O(B)
    uint32     max_hole = 0;
    uint32     max_hole_start = 0;
    uint32     current_hole_start = 0;
    uint32     current_hole_len = 0;
    
    for (i = 0; i < BLCKSZ; i++) 
    {
        if (((char *) page)[i] == 0)
        {
            if (current_hole_len == 0)
                current_hole_start = i;
            current_hole_len++;
            
            // 寻找最大洞
            if (current_hole_len > max_hole)
            {
                max_hole = current_hole_len;
                max_hole_start = current_hole_start;
            }
        }
        else
        {
            current_hole_len = 0;
        }
    }
    
    // 2. 判断是否值得压缩
    if (max_hole < HOLE_THRESHOLD) {
        *compressed_size = 0;
        return NULL;
    }
    
    // 3. 构造压缩图像 - O(B)
    *compressed_size = BLCKSZ - max_hole;
    compressed = palloc(*compressed_size);
    
    // 复制前面部分
    if (max_hole_start > 0)
        memcpy(compressed, page, max_hole_start);
    
    // 复制后面部分
    uint32 tail_start = max_hole_start + max_hole;
    uint32 tail_len = BLCKSZ - tail_start;
    if (tail_len > 0)
        memcpy(compressed + max_hole_start,
               (char *) page + tail_start,
               tail_len);
    
    return compressed;
}
```

**压缩率分析**:
- 新页面: 通常 50-80% 空白 → 压缩率 50-80%
- 页面填满后: 压缩率 < 10% → 不值得压缩

### 3.2 组提交优化算法

#### 3.2.1 延迟搜集算法

**目标**: 将多个事务的 WAL 一起刷写

**时间窗口**: T = commit_delay 链接

**算法流程**:
```c
// src/backend/access/transam/xlog.c:2700
static void
group_commit_delay(void)
{
    TimestampTz now;
    TimestampTz target_time;
    long       delay_usec;
    
    // 1. 检查组提交条件
    if (ProcGlobal->nCommitters < commit_siblings)
        return;
    
    // 2. 计算等待时间
    now = GetCurrentTimestamp();
    delay_usec = commit_delay + (random() % commitDelayRange);
    target_time = TimestampTzPlusMilliseconds(now, delay_usec / 1000);
    
    // 3. 等待时间段
    while (now < target_time)
    {
        // 检查是否有足够的同伴
        if (ProcGlobal->nCommitters >= commit_siblings * 2)
        {
            break; // 同伴足够，提前退出
        }
        
        // 短暂休眠
        pg_usleep(TimeToSleep);
        now = GetCurrentTimestamp();
    }
}
```

**性能提升**:
```
单事务 fsync 延迟: 2-5ms
组提交 fsync 延迟: 0.5-1ms (摊销)
吞吐量提升: 3-5x
```

---

## 四、恢复算法分析

### 4.1 恢复点查找算法

#### 4.1.1 启发式查找

**问题**: 快速找到恢复起始点

**策略**:
1. 优先从控制文件读取 checkpoint
2. 检查 backup_label
3. 退回到段边界

**时间复杂度**: O(1) - 最多读取 3 个位置

```c
// src/backend/access/transam/xlogrecovery.c
static XLogRecPtr
FindRestartPoint()
{
    XLogRecPtr restart_point;
    
    // 1. 尝试从 backup_label - O(1)
    if (ReadBackupLabel(&restart_point))
        return restart_point;
    
    // 2. 从控制文件读取 - O(1)
    if (ReadCheckpointRecord(ControlFile->checkPoint, &checkpoint))
        return checkpoint.redo;
    
    // 3. 退到段边界 - O(1)
    return AlignToSegmentBoundary(ControlFile->checkPoint);
}
```

### 4.2 WAL 重放算法

#### 4.2.1 流式重放

**特点**: 内存高效，支持大 WAL

**时间复杂度**: O(N) - N = WAL 记录数

```c
// src/backend/access/transam/xlogrecovery.c
static void
ReplayWALStream()
{
    XLogReaderState *reader;
    XLogRecord *record;
    
    reader = XLogReaderAllocate(wal_segment_size, NULL);
    
    // 主循环 - O(N)
    while ((record = XLogReadRecord(reader, NULL)) != NULL)
    {
        // 调用资源管理器的 redo 函数
        RmgrTable[record->xl_rmid].rm_redo(reader);
        
        // 更新最小恢复点
        UpdateMinRecoveryPoint(reader->currRecPtr);
        
        // 符合查询条件
        if ( reached recovery target )
            break;
    }
}
```

#### 4.2.2 并行重放优化

**最终口标**: 利用多核加速恢复

**算法**: 按关系编号并行重放

```c
// 伪代码 - PostgreSQL 17+ 可能的实现
typedef struct ParallelRedoTask {
    Oid relid;
    List *records;
} ParallelRedoTask;

void
ParallelRecoverWAL()
{
    HashTbl *task_by_relation;
    
    // 1. 扫描 WAL，按关系分组 - O(N)
    while ((record = ReadRecord()) != NULL)
    {
        if (IsRelationRecord(record))
        {
            Oid relid = GetRelationOid(record);
            AppendToRelationTask(task_by_relation, relid, record);
        }
    }
    
    // 2. 并行重放 - O(R*M) R=关系数, M=平均记录数
    ParallelExecuteTasks(task_by_relation);
}
```

---

## 五、文件管理算法

### 5.1 段文件管理

#### 5.1.1 LRU 缓存算法

**问题**: 减少文件打开/关闭开销

**实现**: 最近最少使用缓存

**时间复杂度**: O(1) 命中，O(1) 替换

```c
// 段文件缓存管理
#define MAX_SEG_CACHE 16

typedef struct SegCacheEntry {
    XLogSegNo   segno;              // 段号
    int         fd;                 // 文件描述符
    uint32      ref_count;          // 引用计数
    TimestampTz last_use;           // 最后使用时间
} SegCacheEntry;

static SegCacheEntry seg_cache[MAX_SEG_CACHE];
static int cache_pos = 0;

// 查找缓存 - O(1)
int
GetSegmentFile(XLogSegNo segno)
{
    // 1. 哈希查找缓存
    for (int i = 0; i < MAX_SEG_CACHE; i++)
    {
        if (seg_cache[i].segno == segno && seg_cache[i].fd >= 0)
        {
            seg_cache[i].ref_count++;
            seg_cache[i].last_use = GetCurrentTimestamp();
            return seg_cache[i].fd;
        }
    }
    
    // 2. 缓存未命中，打开新文件
    int fd = open_segment_file(segno);
    
    // 3. LRU 替换
    cache_pos = (cache_pos + 1) % MAX_SEG_CACHE;
    if (seg_cache[cache_pos].fd >= 0)
        close(seg_cache[cache_pos].fd);
    
    seg_cache[cache_pos].segno = segno;
    seg_cache[cache_pos].fd = fd;
    seg_cache[cache_pos].ref_count = 1;
    seg_cache[cache_pos].last_use = GetCurrentTimestamp();
    
    return fd;
}
```

### 5.2 存储回收算法

#### 5.2.1 基于需求的回收

**算法计算**:
```c
// src/backend/access/transam/xlog.c
static void
RecycleWALSegment(XLogRecPtr keep_lsn)
{
    XLogSegNo recycle_segno;
    XLogSegNo keep_segno;
    XLogSegNo cur_segno;
    
    // 计算保留段号
    keep_segno = keep_lsn / XLog_SEG_SIZE;
    
    // 获取当前段
    cur_segno = GetCurrentSegment();
    
    // 识别可回收段 - O(R) R=段文件数
    for (XLogSegNo segno = recycle_segno; segno < keep_segno; segno++)
    {
        // 检查是否在需要保留
        if (ShouldKeepSegment(segno))
            continue;
        
        // 回收重命名
        RenameToRecycled(segno);
    }
    
    // 重命名到当前段位置
    RenameRecycledToCurrent(cur_segno);
}
```

#### 5.2.2 预分配算法

**目标**: 减少文件分配开销

```c
// 预分配策略 - 维护 2-3 个预分配段
static void
PreallocateSegments()
{
    XLogSegNo cur_seg = GetCurrentSegment();
    int preallocated = 0;
    
    // 检查并预分配跨段空间
    while (preallocated < 3)
    {
        XLogSegNo target_seg = cur_seg + preallocated + 1;
        
        if (!FileExists(target_seg))
        {
            // 使用 fallocate 预分配空间
            PreallocateSegment(target_seg);
            break;
        }
        
        preallocated++;
    }
}
```

---

## 六、性能优化算法

### 6.1 批量 I/O 算法

#### 6.1.1 向量化写入

**目标**: 减少系统调用次数

**实现**: 收集多个记录后批量写入

```c
#define BATCH_SIZE_LIMIT   64    // 最大批大小
#define BATCH_TIME_LIMIT   10    // 最大延迟 ms

typedef struct WriteBatch {
    XLogRecPtr positions[BATCH_SIZE_LIMIT];
    uint32    lengths[BATCH_SIZE_LIMIT];
    char     *datas[BATCH_SIZE_LIMIT];
    int       count;
    TimestampTz start_time;
    Size      total_bytes;
} WriteBatch;

static WriteBatch current_batch;

add_to_batch(XLogRecPtr pos, uint32 len, char *data)
{
    if (current_batch.count >= BATCH_SIZE_LIMIT ||
        (TimestampTzDifferenceMilliseconds(GetCurrentTimestamp(),
                                          current_batch.start_time) > BATCH_TIME_LIMIT))
    {
        // 批量写入 - 调用 pwritev
        flush_batch(&current_batch);
        reset_batch(&current_batch);
    }
    
    // 添加到当前批次
    current_batch.positions[current_batch.count] = pos;
    current_batch.lengths[current_batch.count] = len;
    current_batch.datas[current_batch.count] = data;
    current_batch.count++;
    current_batch.total_bytes += len;
}

flush_batch(WriteBatch *batch)
{
    // 使用 pwritev 向量化写入
    struct iovec iov[BATCH_SIZE_LIMIT];
    for (int i = 0; i < batch->count; i++)
    {
        iov[i].iov_base = batch->datas[i];
        iov[i].iov_len = batch->lengths[i];
    }
    
    pwritev(fd, iov, batch->count, offset);
}
```

**性能提升**:
- 系统调用减少: N → 1
- I/O 延迟降低: 30-50%
- 吞吐量提升: 2-3x

### 6.2 布隆过滤器优化

#### 6.2.1 重复记录检测

**问题**: 避免重复记录相同的数据页修改

**实现**: 使用布隆过滤器快速判断

```c
// 源码中实际没有实现，但可作为优化
typedef struct PageModificationFilter {
    uint64_t bitmap[BITMAP_SIZE];
    uint32_t hash_count;
} PageModificationFilter;

bool
MaybeModifiedPage(PageModificationFilter *filter, RelFileNode rnode,
                 ForkNumber fork, BlockNumber blkno)
{
    uint64_t hash = CityHash64(&rnode, sizeof(rnode));
    hash = CityHash64(&hash, sizeof(hash), &fork, sizeof(fork));
    hash = CityHash64(&hash, sizeof(hash), &blkno, sizeof(blkno));
    
    for (uint32 i = 0; i < filter->hash_count; i++)
    {
        uint64_t pos = (hash + i * 0x9e3779b9) % (BITMAP_SIZE * 64);
        if (!(filter->bitmap[pos / 64] & (1ULL << (pos % 64))))
            return false;
    }
    
    return true; // 可能修改过
}
```

---

## 七、复杂度总结

### 7.1 核心操作复杂度

| 操作                 | 时间复杂度      | 空间复杂度      | 优化点                     |
|----------------------|----------------|----------------|---------------------------|
| **LSN 转换**         | O(1)          | O(1)          | 移位/位运算               |
| **记录插入**         | O(1)          | O(record_size) | 原子预留                  |
| **记录读取**         | O(1)          | O(record_size) | 页缓存                    |
| **缓冲区写入**       | O(n)          | O(n)          | 向量化 I/O                |
| **恢复扫描**         | O(N)          | O(1)          | 并行处理                  |
| **文件管理**         | O(1)          | O(cache_size) | LRU 缓存                  |
| **空间回收**         | O(R)          | O(1)          | 批量操作                  |

### 7.2 性能影响分析

#### 7.2.1 关键路径中断点分析

```text
事务提交流程中的中断点:

XLogInsert() {
    预留空间: ~2-5μs     (原子操作)
    复制数据: ~5-10μs    (内存拷贝)  
    释放锁: ~1-2μs       (锁操作)
}

XLogFlush() {
    检查状态: ~0.5μs     (内存读取)
    唤醒 Writer: ~10μs    (信号)
    等待完成: ~2-5ms   (fsync 延迟)
}

总延迟: 组提交时 200-500μs，非组提交时 2-5ms
```

#### 7.2.2 吞吐量瓶颈分析

| 组件             | 瓶颈原因                  | 解决方案                    |
|------------------|--------------------------|----------------------------|
| **WAL Writer**    | fsync 延迟              | 组提交、批量刷写            |
| **磁盘 I/O**      | 顺序 I/O 饱和            | 分离 WAL 磁盘、SSD          |
| **网络复制**      | 网络带宽限制            | 压缩、并行复制              |
| **恢复速度**      | 单线程重放              | 并行重放、增量恢复          |

### 7.3 内存使用分析

```c
// 内存开销估算
shared_buffers = 128MB      // 数据缓冲
wal_buffers = 16MB          // WAL 缓冲  
clog_buffers = 1MB          // 事务日志
locks_total = 5MB           // 锁表
proc_array = 1MB            // 进程数组

总计: ~151MB
```

**优化指南**:
1. 高并发写入场景:
   - 增加 `wal_buffers` 到 32MB
   - 启用 `wal_compression`
   - 调整 `commit_delay`

---

## 总结

WAL 系统的算法设计体现了 PostgreSQL 的几个核心优化原则：

1. **最小化同步点**: 通过原子操作减少锁竞争
2. **批量化处理**: 减少 I/O 和系统调用次数  
3. **缓存友好**: 利用局部性原理
4. **可扩展性**: 支持并行和分布式优化

理解这些算法有助于：
- 性能调优
- 故障诊断
- 系统设计
- 代码贡献

**下一步**: 性能优化技术详解 → [05_performance_optimization.md](05_performance_optimization.md)
