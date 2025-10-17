# HEAP 堆表概述

本文档介绍PostgreSQL堆表的基本概念、架构和核心特性。

---

## 1. 堆表是什么？

### 1.1 定义

**Heap (堆表)** 是PostgreSQL的默认表存储格式，采用**无序堆存储**结构：

```
特点:
- 📝 无序存储: 数据按插入顺序存储，不保证物理顺序
- ➕ 追加优先: 新数据优先使用FSM记录的空闲空间
- 🔄 MVCC支持: 通过元组版本实现多版本并发
- 🚀 HOT Update: 同页更新优化
```

### 1.2 对比其他存储格式

| 特性 | Heap (堆表) | Index-Organized Table | Columnar |
|------|-------------|----------------------|----------|
| 数据顺序 | 无序 | 按主键排序 | 列存储 |
| 插入速度 | ⚡ 快 | 慢 | 快 |
| 范围查询 | 需索引 | ⚡ 快 | 慢 |
| 更新代价 | 中等 | 高 | 高 |
| 空间效率 | 中等 | 低 | ⚡ 高 |
| MVCC | ✅ 原生支持 | 复杂 | 复杂 |

**PostgreSQL只支持Heap存储**（主表），但通过索引实现排序访问。

---

## 2. 页面结构 (8KB)

### 2.1 完整页面布局

```
┌──────────────────────────────────────────────────────────┐  0
│                   PageHeaderData (24 Bytes)               │
│  ┌────────────────────────────────────────────────────┐  │
│  │ pd_lsn (8B)         - WAL日志序列号                │  │
│  │ pd_checksum (2B)    - 页面校验和                   │  │
│  │ pd_flags (2B)       - 标志位                       │  │
│  │ pd_lower (2B)       - 空闲空间起始偏移             │  │
│  │ pd_upper (2B)       - 空闲空间结束偏移             │  │
│  │ pd_special (2B)     - 特殊空间起始偏移             │  │
│  │ pd_pagesize_version (2B) - 页大小和版本号         │  │
│  │ pd_prune_xid (4B)   - 剪枝事务ID                   │  │
│  └────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────┤  24
│              ItemIdData Array (行指针数组)               │
│  ┌────────────────────────────────────────────────────┐  │
│  │ ItemId[0]: (lp_off, lp_flags, lp_len)  4 Bytes    │  │
│  │ ItemId[1]: ...                          4 Bytes    │  │
│  │ ItemId[2]: ...                          4 Bytes    │  │
│  │ ...                                                 │  │
│  │ ItemId[N]: ...                          4 Bytes    │  │
│  └────────────────────────────────────────────────────┘  │
│                          ↓ pd_lower                       │
├──────────────────────────────────────────────────────────┤
│                                                           │
│                   Free Space (空闲空间)                   │
│                   可以向上或向下增长                       │
│                                                           │
│                          ↑ pd_upper                       │
├──────────────────────────────────────────────────────────┤
│                  Tuples (元组数据，从底部向上)             │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Tuple N: HeapTupleHeader + user data               │  │
│  │ ...                                                 │  │
│  │ Tuple 2: HeapTupleHeader + user data               │  │
│  │ Tuple 1: HeapTupleHeader + user data               │  │
│  │ Tuple 0: HeapTupleHeader + user data               │  │
│  └────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────┤
│             Special Space (特殊空间，可选)                │
│             (索引页使用，堆表页通常为空)                   │
└──────────────────────────────────────────────────────────┘  8192
```

### 2.2 关键字段说明

**pd_lower / pd_upper**: 标记空闲空间边界
```
空闲空间大小 = pd_upper - pd_lower

插入新元组:
  1. 在 ItemIdData 数组末尾添加新的行指针 (pd_lower += 4)
  2. 在页面底部写入元组数据 (pd_upper -= tuple_size)
  3. 如果 pd_lower > pd_upper → 页面满
```

**pd_prune_xid**: 页面剪枝优化
```
记录最小的可能需要剪枝的事务ID
用于快速判断页面是否需要剪枝
```

---

## 3. 元组结构

### 3.1 HeapTupleHeader

```c
// src/include/access/htup_details.h:116
typedef struct HeapTupleHeaderData
{
    union {
        HeapTupleFields t_heap;      // 常规元组字段
        DatumTupleFields t_datum;    // TOAST元组字段
    } t_choice;
    
    ItemPointerData t_ctid;          // 当前或新版本的TID (6B)
    
    uint16 t_infomask2;              // 列数和标志位 (2B)
    uint16 t_infomask;               // 各种标志位 (2B)
    uint8  t_hoff;                   // 头部长度 (1B)
    
    bits8  t_bits[FLEXIBLE_ARRAY_MEMBER];  // NULL位图（可变）
    
    /* 之后是实际的列数据 */
} HeapTupleHeaderData;

// HeapTupleFields (常规字段)
typedef struct HeapTupleFields
{
    TransactionId t_xmin;   // 插入事务ID (4B)
    TransactionId t_xmax;   // 删除/锁定事务ID (4B)
    
    union {
        CommandId t_cid;    // 插入和删除命令ID (4B)
        TransactionId t_xvac; // VACUUM事务ID
    } t_field3;
} HeapTupleFields;
```

### 3.2 元组大小计算

```
元组总大小 = HeapTupleHeader + NULL位图 + 用户数据 + 对齐

示例: CREATE TABLE t (a INT, b TEXT, c TIMESTAMP);

HeapTupleHeader: 23B (固定部分)
NULL位图: CEIL(3列 / 8) = 1B
对齐: 至少对齐到4字节边界

实际数据:
  a (INT4): 4B
  b (TEXT): varlena头(4B) + 实际字符串
  c (TIMESTAMP): 8B

最小元组: 23 + 1 + 对齐 + 4 + 4 + 8 ≈ 48B (含对齐)
```

---

## 4. TID (Tuple Identifier)

### 4.1 结构

```c
typedef struct ItemPointerData
{
    BlockIdData ip_blkid;   // 页号 (4B)
    OffsetNumber ip_posid;  // 页内偏移 (2B)
} ItemPointerData;

// 总共 6 字节
// 格式: (blockNumber, offsetNumber)
// 示例: (0, 1) 表示第0页的第1个元组
```

### 4.2 作用

```
1. 唯一标识: 每个元组的物理位置
2. 索引引用: 索引存储TID指向堆表元组
3. t_ctid链: 元组更新链
   - 旧版本的 t_ctid 指向新版本
   - 最新版本的 t_ctid 指向自己

示例更新链:
  (0,1) [t_ctid=(0,5)] → (0,5) [t_ctid=(1,3)] → (1,3) [t_ctid=(1,3)]
  旧版本                  旧版本                   最新版本
```

---

## 5. HOT Update (关键优化)

### 5.1 原理

**HOT (Heap-Only Tuple)** 是同页更新优化：

```
传统UPDATE:
  1. 创建新元组 (可能在不同页)
  2. 更新所有索引指向新元组
  3. 标记旧元组为删除
  
  代价: N个索引 × 索引更新代价

HOT UPDATE:
  1. 在同一页创建新元组
  2. 旧元组 t_ctid 指向新元组
  3. 索引不变 (仍指向旧元组)
  4. 查询时沿着 HOT 链找到最新版本
  
  代价: 0 × 索引更新 (巨大节省!)
```

### 5.2 HOT Update 条件

```
✅ 必须满足:
  1. 更新不改变任何索引列
  2. 新元组在同一页有空间
  3. 页面未满 (需要 fillfactor < 100)

示例:
  表: CREATE TABLE users (
        id SERIAL PRIMARY KEY,  -- 索引列
        status TEXT,            -- 非索引列
        last_login TIMESTAMP
      );
  
  HOT Update ✅:
    UPDATE users SET last_login = now() WHERE id = 1;
    (不改变索引列 id)
  
  非HOT ✗:
    UPDATE users SET id = 999 WHERE id = 1;
    (改变了索引列，必须更新索引)
```

### 5.3 监控HOT Update

```sql
-- 查看HOT Update比例
SELECT 
    schemaname,
    relname,
    n_tup_upd AS total_updates,
    n_tup_hot_upd AS hot_updates,
    CASE 
        WHEN n_tup_upd > 0 
        THEN round(100.0 * n_tup_hot_upd / n_tup_upd, 2)
        ELSE 0
    END AS hot_ratio
FROM pg_stat_user_tables
WHERE n_tup_upd > 0
ORDER BY n_tup_upd DESC;

-- 目标: hot_ratio > 80%
```

---

## 6. TOAST (超大字段存储)

### 6.1 触发条件

```
TOAST 触发阈值: 
  - 元组大小 > TOAST_TUPLE_THRESHOLD (~2KB)
  - 或单个字段 > TOAST_TUPLE_TARGET (~2KB)

自动行为:
  1. 尝试压缩字段 (EXTENDED策略)
  2. 如果仍太大，移到TOAST表 (pg_toast.pg_toast_xxxxx)
  3. 主表只保留指针
```

### 6.2 TOAST 策略

```c
#define TYPSTORAGE_PLAIN    'p'  // 不压缩，不TOAST
#define TYPSTORAGE_EXTERNAL 'e'  // 不压缩，外部存储
#define TYPSTORAGE_EXTENDED 'x'  // 压缩，外部存储 (默认)
#define TYPSTORAGE_MAIN     'm'  // 压缩，优先主表
```

```sql
-- 查看列的TOAST策略
SELECT attname, atttypid::regtype, attstorage
FROM pg_attribute
WHERE attrelid = 'your_table'::regclass AND attnum > 0;

-- 修改TOAST策略
ALTER TABLE your_table ALTER COLUMN large_col SET STORAGE EXTERNAL;
```

### 6.3 TOAST表结构

```sql
-- TOAST表自动创建
-- pg_toast.pg_toast_<oid>

-- TOAST记录结构:
-- (chunk_id, chunk_seq, chunk_data)
--   chunk_id: 原行的OID
--   chunk_seq: 块序号 (0, 1, 2, ...)
--   chunk_data: 2KB数据块
```

---

## 7. 填充因子 (Fillfactor)

### 7.1 作用

```
fillfactor: 控制页面填充程度，为HOT Update预留空间

默认: 100 (填满整页)
推荐:
  - 频繁UPDATE: 70-80 (预留20-30%空间)
  - 只读/追加: 100 (节省空间)
```

### 7.2 设置方法

```sql
-- 创建表时设置
CREATE TABLE hot_table (
    id INT PRIMARY KEY,
    status TEXT
) WITH (fillfactor = 70);

-- 修改现有表
ALTER TABLE hot_table SET (fillfactor = 70);

-- 重建表使设置生效
VACUUM FULL hot_table;  -- 或 REINDEX + CLUSTER

-- 查看设置
SELECT relname, reloptions
FROM pg_class
WHERE relname = 'hot_table';
```

---

## 8. FSM 和 VM

### 8.1 FSM (Free Space Map)

```
作用: 记录每个页面的空闲空间大小
文件: <table_oid>_fsm

结构: 3层B树
  - 根节点: 记录各分支的最大空闲空间
  - 中间节点: 记录各叶子的最大空闲空间
  - 叶子节点: 记录每个页面的空闲空间 (1字节精度 ~32B)

INSERT流程:
  1. 查询FSM找到有足够空间的页
  2. 在该页插入元组
  3. 更新FSM记录
```

### 8.2 VM (Visibility Map)

```
作用: 记录哪些页面的所有元组对所有事务可见
文件: <table_oid>_vm

结构: 位图 (每页2位)
  - Bit 0: all-visible (所有元组可见)
  - Bit 1: all-frozen (所有元组已冻结)

VACUUM优化:
  - 跳过 all-visible 页面 (Index-Only Scan也依赖VM)
  - 只扫描可能有死元组的页面
```

```sql
-- 查看VM统计
SELECT 
    relname,
    pg_size_pretty(pg_relation_size(oid)) AS size,
    pg_stat_get_blocks_fetched(oid) - pg_stat_get_blocks_hit(oid) AS disk_reads
FROM pg_class
WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace;
```

---

## 9. 页面剪枝 (Page Pruning)

### 9.1 触发时机

```
页面剪枝 vs VACUUM:
  - 页面剪枝: 单页局部清理 (轻量级)
  - VACUUM: 全表扫描清理 (重量级)

触发条件:
  1. UPDATE/SELECT 访问页面
  2. 页面有死元组 (检查 pd_prune_xid)
  3. 事务 >= pd_prune_xid (死元组可能可见)

操作:
  - 清理死元组占用的空间
  - 不回收空间给操作系统
  - 空间可供同页的新INSERT使用
```

### 9.2 HOT链剪枝

```
场景: HOT Update链中的旧版本已对所有事务不可见

剪枝前:
  ItemId[1] → Tuple (v1, t_ctid=(0,5))  [dead]
  ItemId[5] → Tuple (v2, t_ctid=(0,9))  [dead]
  ItemId[9] → Tuple (v3, t_ctid=(0,9))  [current]

剪枝后:
  ItemId[1] → (redirect to 9)
  ItemId[5] → (dead)
  ItemId[9] → Tuple (v3)  [current]

效果:
  - 回收v1, v2占用的空间
  - ItemId[1] 重定向到最新版本 (索引仍有效)
```

---

## 10. 性能监控

### 10.1 关键指标

```sql
-- 表统计信息
SELECT 
    schemaname,
    relname,
    n_tup_ins,      -- 插入数
    n_tup_upd,      -- 更新数
    n_tup_del,      -- 删除数
    n_tup_hot_upd,  -- HOT更新数
    n_live_tup,     -- 活元组数
    n_dead_tup,     -- 死元组数
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables;
```

### 10.2 页面统计

```sql
-- 使用 pgstattuple 扩展
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- 查看表统计
SELECT * FROM pgstattuple('your_table');
-- table_len: 表大小
-- tuple_count: 元组数
-- tuple_len: 元组数据总大小
-- dead_tuple_count: 死元组数
-- free_space: 空闲空间
-- avg_leaf_density: 平均填充密度

-- 快速估算 (不精确但快)
SELECT * FROM pgstattuple_approx('your_table');
```

---

## 总结

### 核心概念

1. **堆表**: 无序追加存储
2. **页面**: 8KB基本单位，包含PageHeader + ItemId数组 + 元组
3. **元组**: HeapTupleHeader + 用户数据
4. **TID**: (blockNumber, offsetNumber) 物理位置标识
5. **HOT Update**: 同页更新优化，避免索引更新
6. **TOAST**: 超大字段外部存储
7. **FSM/VM**: 空间管理和可见性优化
8. **页面剪枝**: 轻量级局部清理

### 性能要点

- ✅ 合理设置 fillfactor (频繁UPDATE的表设为70-80)
- ✅ 监控 HOT Update 比例 (目标 > 80%)
- ✅ 定期 VACUUM 清理死元组
- ✅ 避免更新索引列 (以利用HOT Update)
- ✅ 合理设计TOAST策略

---

**下一步**: 阅读 [02_data_structures.md](02_data_structures.md) 深入了解数据结构

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16
