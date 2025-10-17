# 从PostgreSQL学习性能优化

> PostgreSQL源码中的性能优化智慧全景

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-17

---

## 目录

1. [I/O优化技巧](#1-io优化技巧)
2. [内存优化技巧](#2-内存优化技巧)
3. [CPU优化技巧](#3-cpu优化技巧)
4. [并发优化技巧](#4-并发优化技巧)
5. [数据结构优化](#5-数据结构优化)
6. [算法优化](#6-算法优化)
7. [系统调用优化](#7-系统调用优化)
8. [性能优化方法论](#8-性能优化方法论)

---

## 1. I/O优化技巧

### 1.1 顺序写优化 (WAL)

**核心思想**: 顺序写比随机写快100倍+

```
随机写数据页:
Page 1 (磁盘位置: 100MB)
Page 2 (磁盘位置: 500MB)  ← 磁头移动！
Page 3 (磁盘位置: 200MB)  ← 磁头移动！

速度: 1-10 MB/s (HDD)

顺序写WAL:
WAL Record 1
WAL Record 2  ← 顺序追加
WAL Record 3  ← 顺序追加

速度: 100+ MB/s (HDD)
```

**应用场景**:
- ✅ 日志系统 (Append-only Log)
- ✅ 消息队列 (Kafka)
- ✅ 时序数据库
- ✅ WAL预写日志

**性能提升**: 10-100倍

---

### 1.2 批量刷盘 (Group Commit)

**问题**: 每次提交都fsync，性能差

```
传统方式:
T1: COMMIT → fsync(1ms)
T2: COMMIT → fsync(1ms)  
T3: COMMIT → fsync(1ms)

总耗时: 3ms
TPS: 333
```

**优化**: 批量提交

```
Group Commit:
T1: COMMIT ┐
T2: COMMIT ├─→ 一次fsync(1ms)
T3: COMMIT ┘

总耗时: 1ms
TPS: 3000 (提升9倍!)
```

**实现要点**:
```c
/* 累积多个事务的WAL记录 */
while (有等待提交的事务) {
    收集WAL记录到缓冲区;
}
/* 一次性刷盘 */
fsync(wal_fd);
/* 通知所有事务提交成功 */
```

**应用场景**:
- ✅ 数据库事务提交
- ✅ 消息队列确认
- ✅ 分布式日志同步

**性能提升**: 3-10倍

---

### 1.3 预读优化 (Prefetch)

**问题**: 同步I/O等待时间长

```
同步读取:
Read Block 1 → Wait 10ms → Process
Read Block 2 → Wait 10ms → Process
Read Block 3 → Wait 10ms → Process

总时间: 30ms
```

**优化**: 异步预读

```
异步预读:
Issue Read Block 1 ┐
Issue Read Block 2 ├─→ 并行I/O
Issue Read Block 3 ┘
     ↓
Wait all (10ms)
     ↓
Process all blocks

总时间: 10ms (提升3倍!)
```

**PostgreSQL实现**:
```c
/* IndexScan预读 */
for (i = 0; i < PREFETCH_SIZE; i++) {
    blocknum = get_next_block();
    PrefetchBuffer(rel, blocknum);  // 异步发起I/O
}

for (i = 0; i < PREFETCH_SIZE; i++) {
    buffer = ReadBuffer(rel, blocknum);  // 很可能已在内存
    ProcessBuffer(buffer);
}
```

**性能提升**: 2-5倍

---

### 1.4 Direct I/O (绕过OS缓存)

**问题**: 双重缓存浪费内存

```
传统Buffered I/O:
┌──────────────┐
│ Application  │
│ Buffer Cache │  ← PostgreSQL Shared Buffers
└──────┬───────┘
       ↓
┌──────────────┐
│   OS Page    │  ← 操作系统页缓存 (重复!)
│    Cache     │
└──────┬───────┘
       ↓
┌──────────────┐
│     Disk     │
└──────────────┘
```

**优化**: Direct I/O

```
Direct I/O:
┌──────────────┐
│ Application  │
│ Buffer Cache │  ← PostgreSQL Shared Buffers
└──────┬───────┘
       ↓ (绕过OS缓存)
┌──────────────┐
│     Disk     │
└──────────────┘
```

**优点**:
- ✅ 避免双重缓存
- ✅ 更可预测的性能
- ✅ 减少内存竞争

**何时使用**:
- ✅ 数据库系统 (有自己的缓存)
- ✅ 大文件顺序读写
- ❌ 小文件随机访问 (OS缓存有帮助)

---

## 2. 内存优化技巧

### 2.1 内存池 (Memory Pool)

**问题**: 频繁malloc/free性能差

```
传统方式:
for (i = 0; i < 1000000; i++) {
    ptr = malloc(100);  // 系统调用，慢!
    // 使用ptr
    free(ptr);          // 系统调用，慢!
}

开销: 系统调用 + 锁竞争
```

**优化**: 内存池

```
内存池:
pool = create_pool(1MB);  // 一次分配大块

for (i = 0; i < 1000000; i++) {
    ptr = pool_alloc(pool, 100);  // O(1)，无系统调用
    // 使用ptr
}

pool_reset(pool);  // 批量释放，O(1)

性能提升: 10-100倍
```

**PostgreSQL MemoryContext**:
```c
/* 创建内存上下文 */
MemoryContext query_context = AllocSetContextCreate(
    CurrentMemoryContext,
    "Query Context",
    ALLOCSET_DEFAULT_SIZES);

/* 在context中分配 */
MemoryContextSwitchTo(query_context);
ptr = palloc(size);  // 快速分配

/* 查询结束，一次性释放所有内存 */
MemoryContextDelete(query_context);
```

**性能提升**: 10-50倍

---

### 2.2 内存对齐 (Alignment)

**问题**: 未对齐访问慢

```
未对齐 (跨Cache Line):
Cache Line 1        Cache Line 2
┌────────┬────────┐┌────────┬────────┐
│  ...   │ int32  ││ (cont) │  ...   │
└────────┴────────┘└────────┴────────┘
         ↑──┼────────┼──↑
            读取需要2次内存访问!

对齐:
Cache Line 1
┌────────┬────────┬────────┬────────┐
│  ...   │ PAD    │ int32  │  ...   │
└────────┴────────┴────────┴────────┘
                   ↑──────↑
                   1次内存访问
```

**结构体对齐优化**:
```c
/* ❌ 未优化 (24 bytes) */
struct Bad {
    char a;      // 1 byte
    int64 b;     // 8 bytes
    char c;      // 1 byte
    // 编译器填充14字节
};

/* ✅ 优化后 (16 bytes) */
struct Good {
    int64 b;     // 8 bytes
    char a;      // 1 byte
    char c;      // 1 byte
    // 编译器填充6字节
};

原则: 按大小降序排列
```

**性能提升**: 20-50%

---

### 2.3 避免False Sharing

**问题**: 多线程修改同一Cache Line导致失效

```
False Sharing:
Thread 1              Thread 2
   ↓                     ↓
┌────────┬────────┬────────┬────────┐
│Counter1│Counter2│Counter3│Counter4│ ← 同一Cache Line (64B)
└────────┴────────┴────────┴────────┘

Thread 1修改Counter1 → 整个Cache Line失效
Thread 2的Counter2缓存失效 → 重新加载
性能下降!
```

**优化**: Cache Line对齐

```c
/* 每个counter占用独立Cache Line */
struct alignas(64) PaddedCounter {
    volatile int64 counter;
    char padding[64 - sizeof(int64)];
};

PaddedCounter counters[4];  // 每个占64字节

性能提升: 2-10倍
```

---

### 2.4 热数据紧凑存储

**原则**: 将频繁访问的数据放在一起

```
❌ 分散存储:
struct Transaction {
    TransactionId xid;          // 8 bytes (热数据)
    char unused[100];           // 不常用
    CommandId cid;              // 4 bytes (热数据)
    char more_unused[200];
};

访问xid和cid需要跨越多个Cache Line

✅ 紧凑存储:
struct Transaction {
    TransactionId xid;          // 8 bytes
    CommandId cid;              // 4 bytes
    // 热数据在同一Cache Line
    
    char unused[100];           // 冷数据
    char more_unused[200];
};

访问热数据只需1个Cache Line
```

---

## 3. CPU优化技巧

### 3.1 分支预测优化

**问题**: 分支预测失败导致流水线停顿

```c
/* ❌ 不可预测的分支 */
for (i = 0; i < n; i++) {
    if (random_condition(data[i])) {  // 50%概率
        // 分支A
    } else {
        // 分支B
    }
}
// CPU分支预测失败率高 → 性能差

/* ✅ 可预测的分支 */
// 先排序，让相同条件聚集
sort_by_condition(data, n);

for (i = 0; i < n; i++) {
    if (data[i].condition) {  // 连续的true或false
        // 分支A
    } else {
        // 分支B
    }
}
// CPU分支预测成功率高 → 性能好
```

**PostgreSQL应用**:
```c
/* MVCC可见性判断优化 */
// 先检查简单条件 (大概率不满足)
if (xmin >= snapshot->xmax)
    return false;  // 大概率在这里返回

// 复杂条件放后面
if (XidInMVCCSnapshot(xmin, snapshot))
    return false;
```

**性能提升**: 10-30%

---

### 3.2 循环展开 (Loop Unrolling)

**原理**: 减少循环控制开销

```c
/* ❌ 普通循环 */
for (i = 0; i < n; i++) {
    sum += array[i];
}
// 每次迭代: 1次加法 + 1次比较 + 1次跳转

/* ✅ 循环展开 */
for (i = 0; i < n; i += 4) {
    sum += array[i];
    sum += array[i+1];
    sum += array[i+2];
    sum += array[i+3];
}
// 每4个元素: 4次加法 + 1次比较 + 1次跳转
// 循环开销减少75%

性能提升: 20-50%
```

**编译器优化**:
```c
// 现代编译器自动展开
// gcc -O3 -funroll-loops
```

---

### 3.3 SIMD优化

**Single Instruction Multiple Data**: 一条指令处理多个数据

```
标量处理:
a[0] + b[0] = c[0]  ← 1条指令
a[1] + b[1] = c[1]  ← 1条指令
a[2] + b[2] = c[2]  ← 1条指令
a[3] + b[3] = c[3]  ← 1条指令
总共: 4条指令

SIMD处理 (AVX2):
┌────┬────┬────┬────┐   ┌────┬────┬────┬────┐
│a[0]│a[1]│a[2]│a[3]│ + │b[0]│b[1]│b[2]│b[3]│
└────┴────┴────┴────┘   └────┴────┴────┴────┘
         ↓ 1条指令
┌────┬────┬────┬────┐
│c[0]│c[1]│c[2]│c[3]│
└────┴────┴────┴────┘
总共: 1条指令 (4倍速度!)
```

**PostgreSQL应用**:
```c
/* 字符串比较 (使用SSE4.2) */
#ifdef USE_SSE42_CRC32C
    // 使用硬件CRC32指令
    crc = _mm_crc32_u64(crc, *buf64);
#else
    // 软件实现
    for (i = 0; i < len; i++)
        crc = crc32_table[(crc ^ buf[i]) & 0xFF] ^ (crc >> 8);
#endif

性能提升: 2-8倍
```

---

## 4. 并发优化技巧

### 4.1 无锁数据结构

**原子操作代替锁**:

```c
/* ❌ 使用锁 */
pthread_mutex_lock(&mutex);
counter++;
pthread_mutex_unlock(&mutex);
// 开销: 系统调用 + 上下文切换

/* ✅ 使用原子操作 */
__atomic_fetch_add(&counter, 1, __ATOMIC_SEQ_CST);
// 开销: 1条CPU指令

性能提升: 10-100倍
```

**PostgreSQL应用**:
```c
/* 原子递增XID */
static inline TransactionId
GetNewTransactionId(void) {
    return pg_atomic_fetch_add_u32(&ShmemVariableCache->nextXid, 1);
}
```

---

### 4.2 读写锁 (RWLock)

**优化读多写少场景**:

```
互斥锁:
Reader 1 ←─┐
Reader 2   ├─→ 互斥，串行执行
Reader 3   │
Writer   ──┘

读写锁:
Reader 1 ┐
Reader 2 ├─→ 并行读取
Reader 3 ┘
    ↕ 互斥
Writer ──→ 独占写入

性能提升: N倍 (N=并发读者数)
```

**PostgreSQL LWLock**:
```c
/* 共享锁 (读) */
LWLockAcquire(lock, LW_SHARED);
// 多个进程可同时持有
ReadData();
LWLockRelease(lock);

/* 排他锁 (写) */
LWLockAcquire(lock, LW_EXCLUSIVE);
// 独占访问
WriteData();
LWLockRelease(lock);
```

---

### 4.3 分区锁 (Lock Partitioning)

**减少锁竞争**:

```
单一锁:
All Threads → [Single Lock] → Hash Table
              ↑ 瓶颈!

分区锁:
Thread 1 → [Lock 0] → Partition 0
Thread 2 → [Lock 1] → Partition 1
Thread 3 → [Lock 2] → Partition 2
Thread 4 → [Lock 3] → Partition 3

并发度提升: N倍 (N=分区数)
```

**PostgreSQL Buffer Mapping Lock**:
```c
/* 16个分区锁 */
#define NUM_BUFFER_PARTITIONS 16

partition = BufTableHashPartition(tag);
LWLockAcquire(BufMappingPartitionLock(partition), LW_EXCLUSIVE);
```

---

### 4.4 乐观并发控制

**MVCC避免读写阻塞**:

```
悲观锁:
Reader → LOCK → Read → UNLOCK
Writer → LOCK → Write → UNLOCK
       ↑ 互斥等待

乐观锁 (MVCC):
Reader → Read Old Version (无锁)
Writer → Write New Version (无锁)
       ↑ 并行执行!

性能提升: 10-100倍 (高并发场景)
```

---

## 5. 数据结构优化

### 5.1 B+树 vs 哈希表

**选择合适的数据结构**:

```
场景: 1000万条记录查找

哈希表:
- 查找: O(1) → 1次
- 范围查询: O(n) → 1000万次
- 内存: 大 (需要预留空间)

B+树:
- 查找: O(log n) → log₁₀₀₀(10000000) ≈ 3次
- 范围查询: O(log n + k) → 3次 + k个结果
- 内存: 小 (紧凑存储)

选择:
✅ 点查询为主 → 哈希表
✅ 范围查询多 → B+树
✅ 内存受限 → B+树
```

---

### 5.2 紧凑数据结构

**减少内存占用**:

```c
/* ❌ 浪费空间 */
struct {
    bool flag;      // 1 byte
    char padding[7];  // 编译器填充
    int64 value;    // 8 bytes
}; // 总共16字节

/* ✅ 位域压缩 */
struct {
    uint64 value : 63;  // 63 bits
    uint64 flag  : 1;   // 1 bit
}; // 总共8字节，节省50%

性能提升:
- 更少内存 → 更好缓存
- 更少内存 → 更少页面
```

---

### 5.3 Cache-Oblivious数据结构

**对任何Cache大小都高效**:

```
B+树参数自适应:
small_cache  → fanout = 16
medium_cache → fanout = 64
large_cache  → fanout = 256

原则: fanout ≈ cache_line_size / pointer_size
```

---

## 6. 算法优化

### 6.1 近似算法

**Clock-Sweep代替精确LRU**:

```
精确LRU:
- 维护双向链表
- 每次访问移动节点 → O(1)但常数大
- 需要锁保护链表 → 并发差

Clock-Sweep:
- 只维护usage_count
- 访问时递增 → O(1)且常数小
- 淘汰时递减 → 无需锁
- 近似LRU效果 → 95%准确度

性能提升: 3-10倍
```

---

### 6.2 动态规划剪枝

**查询优化器剪枝**:

```
完全枚举:
3表: 12种Join顺序
4表: 120种
10表: 17亿种!

动态规划+剪枝:
- 保存最优子计划
- 剪掉明显差的计划
- 10表: 实际评估 < 10000种

性能提升: 1000倍+
```

---

## 7. 系统调用优化

### 7.1 批量系统调用

**减少上下文切换**:

```c
/* ❌ 逐个write */
for (i = 0; i < n; i++) {
    write(fd, buf[i], size);  // n次系统调用
}

/* ✅ writev批量写 */
struct iovec iov[n];
for (i = 0; i < n; i++) {
    iov[i].iov_base = buf[i];
    iov[i].iov_len = size;
}
writev(fd, iov, n);  // 1次系统调用

性能提升: 2-5倍
```

---

### 7.2 零拷贝 (sendfile)

**避免用户态拷贝**:

```
传统: 4次拷贝
Disk → Kernel → User → Kernel → Network

sendfile: 2次拷贝
Disk → Kernel → Network

性能提升: 50-100%
```

---

## 8. 性能优化方法论

### 8.1 优化流程

```
┌──────────────────────────┐
│ 1. 建立性能基线           │
│    Baseline Measurement  │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ 2. 性能分析 (Profiling)  │
│    - CPU: perf/gprof     │
│    - 内存: valgrind      │
│    - I/O: iostat         │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ 3. 识别瓶颈 (Top 20%)    │
│    80/20 原则            │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ 4. 优化瓶颈              │
│    选择合适的优化技巧     │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ 5. 测量效果              │
│    对比基线              │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ 6. 迭代优化              │
│    重复2-5直到达标       │
└──────────────────────────┘
```

---

### 8.2 性能优化层次

```
优化层次 (从上到下):
┌──────────────────────────┐
│ 1. 算法层 (最重要!)       │
│    O(n²) → O(n log n)    │
│    提升: 1000倍+         │
├──────────────────────────┤
│ 2. 数据结构层            │
│    链表 → 哈希表         │
│    提升: 10-100倍        │
├──────────────────────────┤
│ 3. 架构层                │
│    串行 → 并行           │
│    提升: 2-8倍           │
├──────────────────────────┤
│ 4. 代码层                │
│    循环展开、SIMD        │
│    提升: 20-50%          │
├──────────────────────────┤
│ 5. 编译器层              │
│    -O3优化               │
│    提升: 10-30%          │
└──────────────────────────┘

原则: 高层优化优先!
```

---

### 8.3 性能优化原则

**Amdahl定律**:
```
加速比 = 1 / ((1-p) + p/n)

p: 可并行部分比例
n: 处理器数量

示例:
p=90%, n=4
加速比 = 1 / (0.1 + 0.9/4) = 3.08

结论: 串行部分是瓶颈!
优化要点: 减少串行部分
```

**过早优化是万恶之源**:
```
✅ 先保证正确性
✅ 再测量性能
✅ 最后优化瓶颈

❌ 不要盲目优化
❌ 不要优化不重要的代码
```

---

## 总结

### 从PostgreSQL学到的性能优化技巧

**I/O优化**:
1. 顺序写优化 (WAL) - 100倍+
2. 批量刷盘 (Group Commit) - 3-10倍
3. 预读优化 (Prefetch) - 2-5倍
4. Direct I/O - 避免双重缓存

**内存优化**:
5. 内存池 (MemoryContext) - 10-50倍
6. 内存对齐 - 20-50%
7. 避免False Sharing - 2-10倍
8. 热数据紧凑存储 - 更好缓存

**CPU优化**:
9. 分支预测优化 - 10-30%
10. 循环展开 - 20-50%
11. SIMD优化 - 2-8倍

**并发优化**:
12. 无锁数据结构 - 10-100倍
13. 读写锁 - N倍
14. 分区锁 - N倍
15. MVCC乐观并发 - 10-100倍

**数据结构优化**:
16. 选择合适的数据结构
17. 紧凑数据结构
18. Cache-Oblivious设计

**算法优化**:
19. 近似算法 (Clock-Sweep)
20. 动态规划剪枝

**系统调用优化**:
21. 批量系统调用
22. 零拷贝 (sendfile)

### 性能优化方法论

- ✅ 测量优先，避免盲目优化
- ✅ 80/20原则，关注瓶颈
- ✅ 算法 > 数据结构 > 代码
- ✅ 理解硬件 (CPU/内存/I/O)
- ✅ 权衡取舍 (空间换时间)

---

**这些都是经过30年生产验证的性能优化技巧，适用于各类系统！**

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-17

