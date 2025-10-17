# 性能优化技巧

> 从PostgreSQL学习系统性能优化方法

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-17

---

## 目录

1. [零拷贝技术](#1-零拷贝技术)
2. [批量处理优化](#2-批量处理优化)
3. [内存对齐与缓存优化](#3-内存对齐与缓存优化)
4. [并行查询设计](#4-并行查询设计)
5. [异步I/O优化](#5-异步io优化)
6. [性能优化原则](#6-性能优化原则)

---

## 1. 零拷贝技术

### 1.1 传统拷贝 vs 零拷贝

```
传统方式 (4次拷贝):
┌──────────┐
│  磁盘    │
└────┬─────┘
     │ ① DMA拷贝
     ↓
┌──────────┐
│内核缓冲区 │
└────┬─────┘
     │ ② CPU拷贝
     ↓
┌──────────┐
│用户缓冲区 │
└────┬─────┘
     │ ③ CPU拷贝
     ↓
┌──────────┐
│Socket缓冲│
└────┬─────┘
     │ ④ DMA拷贝
     ↓
┌──────────┐
│  网卡    │
└──────────┘

零拷贝 (2次拷贝):
┌──────────┐
│  磁盘    │
└────┬─────┘
     │ ① DMA拷贝
     ↓
┌──────────┐
│内核缓冲区 │
└────┬─────┘
     │ ② DMA拷贝 (sendfile)
     ↓
┌──────────┐
│  网卡    │
└──────────┘

性能提升: 减少2次CPU拷贝 + 减少上下文切换
```

### 1.2 PostgreSQL中的应用

```c
/* 使用sendfile发送文件 */
#ifdef HAVE_SENDFILE
    sent = sendfile(socket, file_fd, &offset, len);
#else
    /* 传统read+write */
    read(file_fd, buffer, len);
    write(socket, buffer, len);
#endif
```

---

## 2. 批量处理优化

### 2.1 Group Commit

```
单独提交 (慢):
T1: INSERT → WAL → fsync → COMMIT
T2: INSERT → WAL → fsync → COMMIT
T3: INSERT → WAL → fsync → COMMIT

Group Commit (快):
T1: INSERT → WAL ┐
T2: INSERT → WAL ├─→ 一次fsync → All COMMIT
T3: INSERT → WAL ┘

效果: 3次fsync → 1次fsync
TPS提升: 3倍+
```

### 2.2 批量插入

```sql
-- ❌ 慢: 逐条插入
INSERT INTO t VALUES (1);
INSERT INTO t VALUES (2);
INSERT INTO t VALUES (3);

-- ✅ 快: 批量插入
INSERT INTO t VALUES (1), (2), (3);

-- ✅ 更快: COPY
COPY t FROM 'data.csv';

性能对比:
单条INSERT:  1,000 rows/sec
批量INSERT:  10,000 rows/sec
COPY:        100,000 rows/sec
```

---

## 3. 内存对齐与缓存优化

### 3.1 缓存行对齐

```
Cache Line (64 bytes):
┌──────────────────────────────────┐
│ 64 bytes                         │
└──────────────────────────────────┘

不对齐 (False Sharing):
Thread 1                Thread 2
   ↓                       ↓
┌────────┬────────┬────────┬────────┐
│Counter1│Counter2│Counter3│Counter4│ ← 同一Cache Line
└────────┴────────┴────────┴────────┘
         ↑                 ↑
      修改导致整行失效，影响其他线程

对齐 (避免False Sharing):
Thread 1                Thread 2
   ↓                       ↓
┌────────┬───────┐  ┌────────┬───────┐
│Counter1│Padding│  │Counter2│Padding│ ← 不同Cache Line
└────────┴───────┘  └────────┴───────┘
         ↑                 ↑
      修改互不影响
```

### 3.2 数据结构对齐

```c
/* ❌ 不对齐 (占用17字节，实际24字节) */
struct BadAlign {
    char a;      // 1 byte
    long b;      // 8 bytes
    int c;       // 4 bytes
    char d;      // 1 byte
    // 编译器填充3字节
};

/* ✅ 对齐 (占用16字节) */
struct GoodAlign {
    long b;      // 8 bytes
    int c;       // 4 bytes
    char a;      // 1 byte
    char d;      // 1 byte
    // 编译器填充2字节
};

原则: 按大小降序排列成员
```

---

## 4. 并行查询设计

### 4.1 并行扫描

```
顺序扫描:
┌────────────────────────────────┐
│ Worker 1: 扫描整个表            │
│ Time: 100s                     │
└────────────────────────────────┘

并行扫描 (4 workers):
┌────────────────────┐
│ Worker 1: Part 1   │
├────────────────────┤
│ Worker 2: Part 2   │
├────────────────────┤
│ Worker 3: Part 3   │
├────────────────────┤
│ Worker 4: Part 4   │
└────────────────────┘
Time: 25s (理想情况)

实际提升: 3-3.5倍 (因为开销)
```

### 4.2 并行Join

```
Hash Join并行化:
┌──────────────────────────────┐
│ Phase 1: Parallel Hash Build │
│  Worker 1: 构建Hash Part 1   │
│  Worker 2: 构建Hash Part 2   │
└────────┬─────────────────────┘
         ↓
┌──────────────────────────────┐
│ Phase 2: Parallel Probe      │
│  Worker 1: Probe Part 1      │
│  Worker 2: Probe Part 2      │
└──────────────────────────────┘
```

### 4.3 并行参数配置

```sql
-- 全局配置
max_parallel_workers_per_gather = 4  -- 每个查询最大worker数
max_parallel_workers = 8             -- 系统全局最大worker数
min_parallel_table_scan_size = 8MB   -- 并行扫描最小表大小

-- 查询级配置
SET max_parallel_workers_per_gather = 2;
```

---

## 5. 异步I/O优化

### 5.1 预读 (Prefetch)

```
同步I/O:
Read Page 1 → Wait → Read Page 2 → Wait → ...

异步I/O (预读):
Read Page 1 ┐
Read Page 2 ├─→ 并行发起I/O
Read Page 3 ┘
   ↓
Wait所有I/O完成

效果: I/O时间重叠，总时间减少
```

### 5.2 Direct I/O

```
Buffered I/O:
Application
     ↓
OS Page Cache ← 二次缓存
     ↓
Disk

Direct I/O:
Application
     ↓
Disk (绕过OS缓存)

适用场景:
✅ 数据库有自己的缓冲池
✅ 避免双重缓存
✅ 可预测的性能
```

---

## 6. 性能优化原则

### 原则1: 测量优先

```
优化流程:
┌──────────────────────────┐
│ 1. 建立性能基线           │
│    (Baseline)            │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ 2. 识别瓶颈              │
│    (Profiling)           │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ 3. 优化瓶颈              │
│    (Optimization)        │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ 4. 测量效果              │
│    (Measure)             │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ 5. 对比基线              │
│    (Compare)             │
└──────────────────────────┘

警告: 过早优化是万恶之源!
```

### 原则2: 80/20法则

```
Pareto原则:
80%的性能问题来自20%的代码

关注点:
✅ 热点函数 (Hot Path)
✅ 关键循环
✅ I/O密集操作
❌ 不常执行的代码
```

### 原则3: 减少数据移动

```
性能层级:
┌──────────────────────────┐
│ L1 Cache    │ 1ns        │ ← 最快
├──────────────────────────┤
│ L2 Cache    │ 4ns        │
├──────────────────────────┤
│ L3 Cache    │ 40ns       │
├──────────────────────────┤
│ 内存        │ 100ns      │
├──────────────────────────┤
│ SSD         │ 100μs      │
├──────────────────────────┤
│ HDD         │ 10ms       │ ← 最慢
└──────────────────────────┘

优化策略:
✅ 数据紧凑存储
✅ 顺序访问
✅ 缓存友好
✅ 减少拷贝
```

### 原则4: 并行化

```
Amdahl定律:
Speedup = 1 / (s + p/n)

s: 串行比例
p: 并行比例
n: 处理器数

示例:
s=10%, p=90%, n=4
Speedup = 1 / (0.1 + 0.9/4) = 3.08倍

结论: 串行部分是瓶颈!
```

---

## 总结

### 性能优化技巧

1. **零拷贝** - 减少数据拷贝次数
2. **批量处理** - Group Commit, 批量插入
3. **缓存对齐** - 避免False Sharing
4. **并行查询** - 利用多核
5. **异步I/O** - 预读，减少等待

### 优化原则

- ✅ 测量优先，避免盲目优化
- ✅ 关注热点，80/20法则
- ✅ 减少数据移动
- ✅ 充分利用并行
- ✅ 选择合适的数据结构和算法

---

**下一步**: 阅读 [07_software_engineering.md](07_software_engineering.md)

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-17

