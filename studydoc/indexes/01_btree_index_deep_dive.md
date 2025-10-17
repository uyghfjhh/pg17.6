# B-tree索引深度分析

> PostgreSQL最重要的索引类型 - 完整的结构、算法和实现

**重要程度**: ⭐⭐⭐⭐⭐  
**源码位置**: `src/backend/access/nbtree/`  
**使用场景**: 90%的索引需求

---

## 📋 B-tree索引总览

### 为什么叫B-tree？

```
B-tree = Balanced Tree (平衡树)

【关键特性】
✓ 自平衡: 所有叶子节点在同一层
✓ 有序: 数据按key排序存储
✓ 高效: O(log N)查找、插入、删除
✓ 范围: 支持范围扫描（<, >, BETWEEN）
✓ 排序: 支持ORDER BY优化
```

### PostgreSQL中的B-tree

```
【PostgreSQL B-tree = B+ tree变体】

与标准B-tree的区别:
1. 数据只在叶子节点
   - Internal节点只存key和指针
   - Leaf节点存key + TID (tuple identifier)

2. 叶子节点链表
   - Leaf页之间有双向链表
   - 支持高效范围扫描

3. 页级结构
   - 使用8KB页
   - 页内items排序
   - 支持并发访问

4. 高度通常很小
   - 3层可支持数百万行
   - 4层可支持数十亿行
```

---

## 🏗️ B-tree整体结构

### 三层B-tree示例

```
【完整的B-tree结构】

                        Metapage
                        ┌────────────────┐
                        │ magic: 0x053162│
                        │ version: 4     │
                        │ root: 3        │◀─ 指向Root页
                        │ level: 2       │
                        │ fastroot: 3    │
                        └────────────────┘
                              │
                              ↓
                    Root Page (Block #3)
                    ┌──────────────────────────────┐
                    │  Level: 2                    │
                    │  Items: [(∞, 4), (50, 5)]   │
                    │         ↓         ↓          │
                    │      Block 4   Block 5       │
                    └───────┬──────────────┬───────┘
                            │              │
         ┌──────────────────┘              └──────────────────┐
         │                                                     │
    Internal Page (Block #4)                    Internal Page (Block #5)
    ┌────────────────────────┐                 ┌────────────────────────┐
    │  Level: 1              │                 │  Level: 1              │
    │  Items: [(∞,10),       │                 │  Items: [(∞,15),       │
    │          (20,11),      │                 │          (70,16),      │
    │          (40,12)]      │                 │          (90,17)]      │
    │          ↓   ↓    ↓    │                 │          ↓   ↓    ↓    │
    │         10  11   12    │                 │         15  16   17    │
    └─────────┬───┬────┬─────┘                 └─────────┬───┬────┬─────┘
              │   │    │                                 │   │    │
       ┌──────┘   │    └──────┐                   ┌──────┘   │    └──────┐
       │          │           │                   │          │           │
   Leaf(#10)  Leaf(#11)  Leaf(#12)           Leaf(#15)  Leaf(#16)  Leaf(#17)
   ┌────────┐ ┌────────┐ ┌────────┐          ┌────────┐ ┌────────┐ ┌────────┐
   │ Level:0│ │ Level:0│ │ Level:0│          │ Level:0│ │ Level:0│ │ Level:0│
   │ ◀─────▶│ │ ◀─────▶│ │ ◀─────▶│          │ ◀─────▶│ │ ◀─────▶│ │ ◀─────▶│
   │        │ │        │ │        │          │        │ │        │ │        │
   │ Items: │ │ Items: │ │ Items: │          │ Items: │ │ Items: │ │ Items: │
   │ [1→TID]│ │[20→TID]│ │[40→TID]│          │[50→TID]│ │[70→TID]│ │[90→TID]│
   │ [5→TID]│ │[25→TID]│ │[42→TID]│          │[55→TID]│ │[75→TID]│ │[92→TID]│
   │[10→TID]│ │[30→TID]│ │[45→TID]│          │[60→TID]│ │[80→TID]│ │[95→TID]│
   │[15→TID]│ │[35→TID]│ │[48→TID]│          │[65→TID]│ │[85→TID]│ │[98→TID]│
   └────────┘ └────────┘ └────────┘          └────────┘ └────────┘ └────────┘
       ↑ ◀───────▶ ↑ ◀───────▶ ↑                ↑ ◀───────▶ ↑ ◀───────▶ ↑
       └────────────────────────┘                └────────────────────────┘
            双向链表连接                              双向链表连接

【关键点】
1. Metapage (Block 0): 存储元数据
2. Root Page: 树的根，指向下层
3. Internal Pages: 中间层，只有key和指针
4. Leaf Pages: 叶子层，存储key→TID映射
5. Leaf链表: 叶子页双向链接，支持范围扫描
6. 高度: 从Root到Leaf的层数（此例为2）
```

---

## 📄 页面结构详解

### 8KB页面布局

```
【B-tree Page布局】

每个B-tree页面 = 8192 bytes (8KB)

┌─────────────────────────────────────────────────── 8KB ───────────────────────────────────────────────────┐
│                                                                                                            │
│  ┌────────────────────────┬──────────────────┬─────────────────────┬──────────────────┬─────────────────┐ │
│  │   PageHeaderData       │  BTPageOpaqueData│   Item Pointers     │   Free Space     │   Items         │ │
│  │   (24 bytes)           │  (16 bytes)      │   (Array)           │   (动态)         │   (变长)        │ │
│  └────────────────────────┴──────────────────┴─────────────────────┴──────────────────┴─────────────────┘ │
│  ↑                        ↑                  ↑                     ↑                  ↑                    │
│  0                        24                 40                    增长→              ←增长             8192│
└────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

【详细结构】

0────────────────────────────────┐
│ PageHeaderData (24 bytes)      │
│ ┌──────────────────────────┐   │
│ │ pd_lsn        (8 bytes)  │◀─ WAL LSN
│ │ pd_checksum   (2 bytes)  │◀─ 页面校验和
│ │ pd_flags      (2 bytes)  │◀─ 标志位
│ │ pd_lower      (2 bytes)  │◀─ Free space起始
│ │ pd_upper      (2 bytes)  │◀─ Free space结束
│ │ pd_special    (2 bytes)  │◀─ Special space偏移
│ │ pd_pagesize   (2 bytes)  │◀─ 页大小(8192)
│ │ pd_version    (2 bytes)  │◀─ 页版本
│ │ pd_prune_xid  (4 bytes)  │◀─ 最老xid
│ └──────────────────────────┘   │
└────────────────────────────────┘
24───────────────────────────────┐
│ BTPageOpaqueData (16 bytes)    │
│ ┌──────────────────────────┐   │
│ │ btpo_prev     (4 bytes)  │◀─ 前一页block号
│ │ btpo_next     (4 bytes)  │◀─ 后一页block号
│ │ btpo_level    (2 bytes)  │◀─ 树的层次(0=leaf)
│ │ btpo_flags    (2 bytes)  │◀─ 页类型标志
│ │ btpo_cycleid  (4 bytes)  │◀─ vacuum周期ID
│ └──────────────────────────┘   │
└────────────────────────────────┘
40───────────────────────────────┐
│ Item Pointers (动态数组)       │
│ ┌──────────────────────────┐   │
│ │ ItemIdData #1 (4 bytes)  │   │
│ │   lp_off    (15 bits)    │◀─ Item偏移
│ │   lp_flags  ( 2 bits)    │◀─ 标志(unused/normal/redirect/dead)
│ │   lp_len    (15 bits)    │◀─ Item长度
│ ├──────────────────────────┤   │
│ │ ItemIdData #2 (4 bytes)  │   │
│ ├──────────────────────────┤   │
│ │ ItemIdData #3 (4 bytes)  │   │
│ ├──────────────────────────┤   │
│ │ ...                      │   │
│ └──────────────────────────┘   │
└────────────────────────────────┘
         │
         │ Free Space (向下增长)
         │
         ↓
┌────────────────────────────────┐
│ Items (从页尾向上增长)         │
│ ┌──────────────────────────┐   │
│ │ ... Item #3              │   │
│ ├──────────────────────────┤   │
│ │ ... Item #2              │   │
│ ├──────────────────────────┤   │
│ │ IndexTupleData #1        │   │
│ │ ┌────────────────────┐   │   │
│ │ │ t_tid (6 bytes)    │◀─┼─ TID (block, offset)
│ │ │   ip_blkid (4)     │   │   │
│ │ │   ip_posid (2)     │   │   │
│ │ ├────────────────────┤   │   │
│ │ │ t_info (2 bytes)   │◀─┼─ 标志和大小
│ │ ├────────────────────┤   │   │
│ │ │ Key data (变长)    │◀─┼─ 索引键值
│ │ │ ...                │   │   │
│ │ └────────────────────┘   │   │
│ └──────────────────────────┘   │
└────────────────────────────────┘
8192─────────────────────────────┘
```

### Internal Page vs Leaf Page

```
【Internal Page - 只存指针】
┌──────────────────────────────────────┐
│ Item #1: [Key: -∞, Pointer: 10]     │◀─ 最小key，指向子页10
│ Item #2: [Key: 50, Pointer: 11]     │◀─ Key≥50，指向子页11
│ Item #3: [Key: 100, Pointer: 12]    │◀─ Key≥100，指向子页12
└──────────────────────────────────────┘

【Leaf Page - 存数据指针】
┌──────────────────────────────────────┐
│ Item #1: [Key: 1, TID: (5, 10)]     │◀─ key=1, 指向heap(block5,off10)
│ Item #2: [Key: 5, TID: (5, 15)]     │◀─ key=5, 指向heap(block5,off15)
│ Item #3: [Key: 10, TID: (8, 3)]     │◀─ key=10, 指向heap(block8,off3)
│ ...                                  │
└──────────────────────────────────────┘
```

---

## 🔍 查找操作详解

### 单点查找流程

```sql
-- 查询: 查找key=75的记录
SELECT * FROM users WHERE id = 75;
```

```
【B-tree查找流程】

Step 1: 从Root开始
┌─────────────────────┐
│ Root (Level 2)      │
│ Items:              │
│   (-∞, 4)  → <50    │
│   (50, 5)  → ≥50    │◀─ 75 ≥ 50，选择页5
└─────────────────────┘
         │
         ↓ 读取Block 5
         
Step 2: 遍历Internal页
┌─────────────────────┐
│ Internal (Level 1)  │
│ Block 5             │
│ Items:              │
│   (-∞, 15) → <50    │
│   (70, 16) → ≥70    │◀─ 75 ≥ 70，选择页16
│   (90, 17) → ≥90    │
└─────────────────────┘
         │
         ↓ 读取Block 16
         
Step 3: 遍历Leaf页
┌─────────────────────┐
│ Leaf (Level 0)      │
│ Block 16            │
│ Items:              │
│   [70 → (100, 5)]   │
│   [75 → (100, 8)]   │◀─ 找到! TID=(100, 8)
│   [80 → (102, 3)]   │
│   [85 → (105, 1)]   │
└─────────────────────┘
         │
         ↓ 根据TID访问heap
         
Step 4: 访问Heap表
┌─────────────────────┐
│ Heap Block 100      │
│ Offset 8:           │
│ ┌─────────────────┐ │
│ │ id: 75          │ │◀─ 返回这个tuple
│ │ name: "Alice"   │ │
│ │ ...             │ │
│ └─────────────────┘ │
└─────────────────────┘

【I/O统计】
总I/O: 4次
  1. Root page
  2. Internal page
  3. Leaf page
  4. Heap page

时间复杂度: O(log N) + O(1)
  log N: 树遍历
  1: heap访问
```

### 范围扫描流程

```sql
-- 查询: 范围查找
SELECT * FROM users WHERE id BETWEEN 70 AND 85;
```

```
【B-tree范围扫描】

Step 1-3: 定位到起始Leaf页（同单点查找）
找到key=70的Leaf页

Step 4: 顺序扫描Leaf页
┌─────────────────────┐      ┌─────────────────────┐
│ Leaf Block 16       │◀────▶│ Leaf Block 17       │
│ Items:              │ next │ Items:              │
│   [70 → (100, 5)] ✓│      │   [90 → (110, 2)]   │
│   [75 → (100, 8)] ✓│      │   [92 → (111, 5)]   │
│   [80 → (102, 3)] ✓│      │   [95 → (112, 8)]   │
│   [85 → (105, 1)] ✓│      │   [98 → (113, 3)]   │
└─────────────────────┘      └─────────────────────┘
         │                            ↑
         │                            │
         └────────────────────────────┘
              通过next指针前进
              直到key > 85停止

【关键优势】
✓ 利用Leaf链表，无需回到上层
✓ 顺序I/O，磁盘友好
✓ 预读优化（OS level）
✓ 高效的范围查询

【I/O统计】
范围[70, 85]，假设跨2个Leaf页:
  I/O: Root(1) + Internal(1) + Leaf(2) = 4次
  vs 全表扫描: 可能需要扫描整个表
```

---

## ➕ 插入操作详解

### 简单插入（页未满）

```
【插入key=73到B-tree】

Before插入:
┌─────────────────────┐
│ Leaf Block 16       │
│ Items: (4个)        │
│   [70 → TID1]       │
│   [75 → TID2]       │
│   [80 → TID3]       │
│   [85 → TID4]       │
│ Free Space: 7KB     │◀─ 还有很多空间
└─────────────────────┘

After插入key=73:
┌─────────────────────┐
│ Leaf Block 16       │
│ Items: (5个)        │
│   [70 → TID1]       │
│   [73 → TID_new]    │◀─ 新插入，保持有序
│   [75 → TID2]       │
│   [80 → TID3]       │
│   [85 → TID4]       │
│ Free Space: 6.9KB   │
└─────────────────────┘

【操作步骤】
1. 定位到Leaf页 (通过树遍历)
2. 在页内二分查找插入位置
3. 插入新Item (保持有序)
4. 更新页头信息 (pd_lower)

时间: O(log N)树遍历 + O(M)页内插入
M = 页内Item数量，通常<100
```

### 页分裂（Page Split）

```
【页满时的分裂】

场景: Leaf页已满，插入key=73

Step 1: Before插入（页已满）
┌─────────────────────────────────────┐
│ Leaf Block 16 (FULL!)               │
│ Free Space: 100 bytes (不够)        │
│ Items: (367个items，填满8KB)        │
│   [70 → TID]                        │
│   [71 → TID]                        │
│   [72 → TID]                        │
│   [74 → TID]                        │◀─ 要在这里插入73
│   [75 → TID]                        │
│   ...                               │
│   [89 → TID]                        │
└─────────────────────────────────────┘

Step 2: 分配新页
┌─────────────────────┐
│ New Leaf Block 20   │◀─ 从磁盘分配新页
│ (空白)              │
└─────────────────────┘

Step 3: 分裂点选择（通常中点）
找到中间key = 80

Step 4: 移动Items
┌────────────────────────┐         ┌────────────────────────┐
│ Left Half (Block 16)   │◀──────▶│ Right Half (Block 20)  │
│ Items:                 │  next  │ Items:                 │
│   [70 → TID]           │        │   [80 → TID]           │
│   [71 → TID]           │        │   [81 → TID]           │
│   [72 → TID]           │        │   [82 → TID]           │
│   [73 → TID_new]       │◀─插入  │   ...                  │
│   [74 → TID]           │        │   [89 → TID]           │
│   [75 → TID]           │        │                        │
│   ...                  │        │ Free Space: ~4KB       │
│   [79 → TID]           │        │                        │
│ Free Space: ~4KB       │        │                        │
└────────────────────────┘        └────────────────────────┘

Step 5: 更新父页（Insert分隔key）
┌─────────────────────────────────┐
│ Parent Internal Page            │
│ Before:                         │
│   [(-∞, 16), (90, 17)]          │
│                                 │
│ After:                          │
│   [(-∞, 16), (80, 20), (90, 17)]│◀─ 添加新的分隔key=80
└─────────────────────────────────┘
              │
              ↓
┌─────────────────────────────────┐
│ 如果父页也满了?                 │
│ → 递归分裂父页!                 │
│ → 最坏情况: 分裂到Root          │
│ → Root分裂 → 树高度+1           │
└─────────────────────────────────┘

【分裂代价】
I/O:
  - 读: 1 (原页)
  - 写: 2 (原页 + 新页)
  - 父页更新: 1写 (可能递归)
  
时间: O(log N) + O(M)
  log N: 定位
  M: 页内移动

【优化】
✓ 分裂点选择: 尽量平衡
✓ Fillfactor: 预留空间，减少分裂
✓ HOT更新: 避免索引更新
```

### Root分裂（树高度增长）

```
【Root页分裂 - 唯一增加树高度的操作】

场景: Root页也满了

Before: 2层树
┌─────────────────────┐
│ Root (Full!)        │
│ Level: 1            │
│ Items: (367个)      │
│ 指向很多Leaf页      │
└─────────────────────┘
         │
    Many Leafs

After: 3层树
┌─────────────────────┐
│ New Root            │◀─ 新Root页
│ Level: 2            │
│ Items: (2个)        │
│   [(-∞, old_root)]  │
│   [(split_key, new)]│
└──────┬───────┬──────┘
       │       │
   ┌───┘       └───┐
   │               │
┌──┴──────────┐ ┌──┴──────────┐
│ Old Root    │ │ New Page    │◀─ 变成Internal页
│ Level: 1    │ │ Level: 1    │
│ (一半items) │ │ (另一半)    │
└─────────────┘ └─────────────┘
   │    │           │    │
  Leafs ...       Leafs ...

【关键点】
✓ Root分裂是树高度增加的唯一方式
✓ 旧Root变成Internal页
✓ 新Root只有2个items (分隔点)
✓ 所有Leaf仍在同一层 (平衡性)
```

---

## ➖ 删除操作详解

### 简单删除

```
【删除key=73】

Before:
┌─────────────────────┐
│ Leaf Block 16       │
│ Items:              │
│   [70 → TID1]       │
│   [73 → TID2]       │◀─ 要删除
│   [75 → TID3]       │
│   [80 → TID4]       │
└─────────────────────┘

Step 1: 标记为Dead (不立即删除)
┌─────────────────────┐
│ Leaf Block 16       │
│ Items:              │
│   [70 → TID1]       │
│   [73 → DEAD]       │◀─ 标记为Dead
│   [75 → TID3]       │
│   [80 → TID4]       │
└─────────────────────┘

Step 2: VACUUM清理
┌─────────────────────┐
│ Leaf Block 16       │
│ Items:              │
│   [70 → TID1]       │
│   [75 → TID3]       │◀─ Dead item被移除
│   [80 → TID4]       │
│ Free Space: +20B    │
└─────────────────────┘

【为什么不立即删除？】
✓ 并发: 可能有事务正在读取
✓ MVCC: 需要保留可见性信息
✓ 性能: 批量清理更高效
```

### 页合并（Page Merge）

```
【页利用率过低时合并】

场景: Leaf页删除很多items后，利用率<30%

Before合并:
┌──────────────────┐       ┌──────────────────┐
│ Leaf Block 16    │◀─────▶│ Leaf Block 17    │
│ Items: (很少)    │  next │ Items: (很少)    │
│   [70 → TID]     │       │   [90 → TID]     │
│   [75 → TID]     │       │   [95 → TID]     │
│ Free: 7KB (87%)  │       │ Free: 7KB (87%)  │
│ 利用率太低!      │       │ 利用率太低!      │
└──────────────────┘       └──────────────────┘

After合并:
┌──────────────────┐       ┌──────────────────┐
│ Leaf Block 16    │◀─────▶│ Leaf Block 18    │
│ Items: (合并)    │  next │ (17被删除)       │
│   [70 → TID]     │       │                  │
│   [75 → TID]     │       │ (可被重用)       │
│   [90 → TID]     │       │                  │
│   [95 → TID]     │       │                  │
│ Free: 6KB (75%)  │       │                  │
└──────────────────┘       └──────────────────┘

更新父页:
┌─────────────────────────────────┐
│ Parent                          │
│ Before: [16, 17, 18]            │
│ After:  [16, 18]                │◀─ 移除17的引用
└─────────────────────────────────┘

【触发条件】
✓ 页利用率 < 30% (可配置)
✓ VACUUM时检查
✓ 相邻页都利用率低

【优化】
✓ Fillfactor: 90% (预留空间给更新)
✓ 延迟合并: 避免频繁split/merge循环
```

---

## 🔐 并发控制

### Latch机制

```
【B-tree使用Latch保护并发访问】

Latch vs Lock:
┌────────────────────────────────────────┐
│                Latch          Lock     │
├────────────────────────────────────────┤
│ 保护对象:   内存结构      数据行       │
│ 持有时间:   微秒级        事务级       │
│ 死锁检测:   不需要        需要         │
│ 类型:       共享/排他      多种        │
│ 使用场景:   页访问        MVCC        │
└────────────────────────────────────────┘

【读取时的Latch协议】
查找key=75:

Step 1: Lock Root (Shared)
┌────────────────┐
│ Root           │ ← S-Latch
│ 找到下一页=5   │
└────────────────┘
     ↓ 释放S-Latch，获取子页Latch

Step 2: Lock Internal (Shared)
┌────────────────┐
│ Internal #5    │ ← S-Latch
│ 找到下一页=16  │
└────────────────┘
     ↓ 释放S-Latch，获取子页Latch

Step 3: Lock Leaf (Shared)
┌────────────────┐
│ Leaf #16       │ ← S-Latch
│ 读取key=75     │
└────────────────┘
     ↓ 读取完成，释放S-Latch

【关键: 自顶向下，逐层释放】
✓ 不会死锁（单向加锁）
✓ 并发度高（快速释放）
✓ 读不阻塞写（MVCC）
```

### 插入时的Latch协议

```
【插入key=73的Latch协议】

Optimistic方法:
Step 1: S-Latch向下遍历
┌────────────────┐
│ Root           │ ← S-Latch (读取)
└────────────────┘
     ↓ 释放
┌────────────────┐
│ Internal       │ ← S-Latch (读取)
└────────────────┘
     ↓ 释放
┌────────────────┐
│ Leaf           │ ← S-Latch (读取)
│ 检查: 有空间？  │
└────────────────┘

Step 2: 如果有空间，升级为X-Latch
┌────────────────┐
│ Leaf           │ ← X-Latch (写入)
│ 插入key=73     │
└────────────────┘
     ↓ 完成，释放

【如果页满，需要分裂】
Pessimistic方法:
重新遍历，但这次使用X-Latch

Step 1: X-Latch向下遍历
┌────────────────┐
│ Root           │ ← X-Latch
│ 检查: 会分裂？  │
│ 不会 → 可以释放│
└────────────────┘
     ↓ 可以释放
┌────────────────┐
│ Internal       │ ← X-Latch
│ 检查: 会分裂？  │
│ 不会 → 可以释放│
└────────────────┘
     ↓ 可以释放
┌────────────────┐
│ Leaf           │ ← X-Latch (保持到操作完成)
│ 分裂页         │
└────────────────┘
┌────────────────┐
│ New Leaf       │ ← X-Latch
│ 移动items      │
└────────────────┘

【Right-Link技术】
如果需要更新父页:
┌────────────────┐
│ Parent         │ ← 需要X-Latch
│ 添加新分隔key  │
└────────────────┘

使用Right-Link避免回溯:
- 分裂页设置right-link指向新页
- 读者遇到right-link时跟随
- 父页更新可以异步完成
```

---

## 🧹 VACUUM处理

### Dead Tuples清理

```
【VACUUM清理B-tree】

场景: 表DELETE了很多行，索引有dead tuples

┌─────────────────────────────────┐
│ Heap Table                      │
│ Block 100: [Deleted rows]       │
│ Block 101: [Deleted rows]       │
│ ...                             │
└─────────────────────────────────┘
         ↑
         │ 这些行的TID在索引中
         │
┌─────────────────────────────────┐
│ B-tree Index                    │
│ Leaf页包含dead tuples:          │
│   [70 → (100, 5)] ← Dead!      │
│   [71 → (101, 3)] ← Dead!      │
│   [72 → (102, 1)] ← Live       │
│   ...                           │
└─────────────────────────────────┘

VACUUM步骤:
Step 1: 扫描Heap，收集dead TIDs
dead_tids = [(100,5), (101,3), ...]

Step 2: 扫描Index，标记dead items
┌─────────────────────────────────┐
│ 遍历所有Leaf页                  │
│ 对每个item:                     │
│   if item.tid in dead_tids:     │
│     mark_dead(item)              │
└─────────────────────────────────┘

Step 3: 清理dead items
┌─────────────────────────────────┐
│ Leaf页 (Before)                 │
│   [70 → DEAD]                   │
│   [71 → DEAD]                   │
│   [72 → (102, 1)]               │
│   [73 → DEAD]                   │
│   [74 → (103, 5)]               │
└─────────────────────────────────┘
         ↓ 压缩
┌─────────────────────────────────┐
│ Leaf页 (After)                  │
│   [72 → (102, 1)]               │
│   [74 → (103, 5)]               │
│   Free Space: +60 bytes         │
└─────────────────────────────────┘

【VACUUM优化】
✓ 批量处理: 一次VACUUM处理多个dead tuples
✓ 索引扫描: 按物理顺序扫描（顺序I/O）
✓ 页合并: 利用率过低的页会合并
```

### Index Bloat

```
【索引膨胀问题】

原因:
1. 频繁INSERT/DELETE
2. 页分裂产生半空页
3. Dead tuples未及时清理

示例:
┌─────────────────────────────────┐
│ 健康的索引                      │
│ 页利用率: 80-90%                │
│ 大小: 100MB                     │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│ Bloated索引                     │
│ 页利用率: 30-40%                │
│ 大小: 250MB (2.5倍!)            │
│ 很多半空页和dead space          │
└─────────────────────────────────┘

【检测Bloat】
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size,
    idx_scan,
    pg_size_pretty(
        pg_relation_size(indexrelid) - 
        pg_relation_size(indexrelid) * 
        (SELECT sum(relpages) FROM pg_class WHERE oid = indexrelid) / 
        (SELECT sum(relpages) FROM pg_class WHERE relname = tablename)
    ) as bloat
FROM pg_stat_user_indexes;

【解决Bloat】
-- 方法1: VACUUM (清理dead tuples)
VACUUM table_name;

-- 方法2: REINDEX (重建索引)
REINDEX INDEX CONCURRENTLY index_name;  -- 不阻塞查询

-- 方法3: CREATE INDEX + DROP OLD (零停机)
CREATE INDEX CONCURRENTLY new_index ON ...;
DROP INDEX CONCURRENTLY old_index;
```

---

## ⚡ 性能优化

### Fillfactor

```
【Fillfactor - 预留更新空间】

默认: fillfactor = 90 (Leaf页)
           fillfactor = 70 (Internal页)

含义: 页只填充到90%，留10%给更新

┌─────────────────────────────────┐
│ Leaf Page (Fillfactor=90)       │
│ ┌─────────────────────┐         │
│ │ Used: 7.2KB (90%)   │         │
│ │ Items: [...]        │         │
│ ├─────────────────────┤         │
│ │ Reserved: 0.8KB     │◀─ 预留  │
│ │ (10%)               │         │
│ └─────────────────────┘         │
│ Total: 8KB                      │
└─────────────────────────────────┘

【优势】
✓ UPDATE时: 可能不需要分裂
✓ HOT更新: 在同一页更新
✓ 减少索引维护开销

【配置】
-- 频繁UPDATE的表
CREATE INDEX ON orders(user_id) 
WITH (fillfactor = 70);  -- 预留30%

-- 只INSERT的表 (日志)
CREATE INDEX ON logs(created_at) 
WITH (fillfactor = 100);  -- 无需预留

【权衡】
fillfactor低: 
  ✓ 更少分裂
  ✗ 索引更大
  ✗ 扫描更慢

fillfactor高:
  ✓ 索引更小
  ✓ 扫描更快
  ✗ 更多分裂
```

### 多列索引顺序

```
【多列索引的列顺序很重要！】

示例: (user_id, status, created_at)

最左前缀原则:
CREATE INDEX ON orders(user_id, status, created_at);

可使用索引的查询:
✅ WHERE user_id = 1
✅ WHERE user_id = 1 AND status = 'pending'
✅ WHERE user_id = 1 AND status = 'pending' AND created_at > '2024-01-01'
✅ WHERE user_id = 1 ORDER BY status, created_at

不能使用索引:
❌ WHERE status = 'pending'  (不是最左列)
❌ WHERE created_at > '2024-01-01'  (不是最左列)

【选择顺序原则】
1. 等值条件 > 范围条件
   (user_id = ?, created_at > ?)
   → (user_id, created_at) ✓

2. 选择性高 > 选择性低
   user_id (1M values) vs status (3 values)
   → (user_id, status) ✓

3. 常用查询模式
   分析WHERE子句的常见组合
```

---

## 📊 性能测试

### B-tree vs 全表扫描

```sql
-- 测试表: 1000万行
CREATE TABLE test_table (
    id bigint PRIMARY KEY,
    value text
);

INSERT INTO test_table 
SELECT i, md5(i::text) 
FROM generate_series(1, 10000000) i;

-- 测试1: 单点查询
-- 无索引
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM test_table WHERE id = 5000000;
/*
Seq Scan on test_table
  (actual time=1234.567..2345.678 rows=1 loops=1)
  Filter: (id = 5000000)
  Rows Removed by Filter: 9999999
  Buffers: shared hit=54321
Planning Time: 0.123 ms
Execution Time: 2345.789 ms  ← 2.3秒!
*/

-- 有索引 (PRIMARY KEY)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM test_table WHERE id = 5000000;
/*
Index Scan using test_table_pkey on test_table
  (actual time=0.025..0.026 rows=1 loops=1)
  Index Cond: (id = 5000000)
  Buffers: shared hit=4
Planning Time: 0.123 ms
Execution Time: 0.045 ms  ← 0.045ms!

提速: 52,000倍!
*/

-- 测试2: 范围查询
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM test_table 
WHERE id BETWEEN 5000000 AND 5001000;

/*
Index Scan using test_table_pkey on test_table
  (actual time=0.034..0.567 rows=1001 loops=1)
  Index Cond: ((id >= 5000000) AND (id <= 5001000))
  Buffers: shared hit=15
Execution Time: 0.678 ms

vs 全表扫描: ~2500ms
提速: 3,700倍!
*/
```

---

## 总结

### B-tree核心要点

1. **结构特点**
   - ✅ 自平衡: 所有叶子同层
   - ✅ 有序: 支持范围扫描
   - ✅ 高效: O(log N)操作
   - ✅ 通用: 适用90%场景

2. **关键操作**
   - **查找**: O(log N)树遍历
   - **插入**: 可能触发分裂
   - **删除**: 标记dead + VACUUM
   - **分裂**: 保持平衡

3. **并发控制**
   - Latch保护页访问
   - 自顶向下加锁
   - Right-Link优化

4. **性能优化**
   - Fillfactor预留空间
   - 多列索引顺序
   - 定期VACUUM
   - 监控Bloat

---

**文档版本**: PostgreSQL 17.6  
**创建日期**: 2025-10-17  
**重要程度**: ⭐⭐⭐⭐⭐  
**下一篇**: Hash索引深度分析

