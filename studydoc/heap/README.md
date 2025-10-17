# HEAP 堆表访问模块

> PostgreSQL 堆表 (Heap) 访问机制的深度分析

**模块状态**: ✅ 完成  
**文档版本**: 1.0  
**最后更新**: 2025-10-16  
**基于版本**: PostgreSQL 17.6

---

## 📚 模块导航

| 文档 | 内容概要 | 重点 |
|------|---------|------|
| [01_overview.md](01_overview.md) | 堆表概述、页结构、TOAST | ⭐⭐⭐⭐⭐ |
| [02_data_structures.md](02_data_structures.md) | PageHeader、HeapTupleHeader | ⭐⭐⭐⭐⭐ |
| [03_implementation_flow.md](03_implementation_flow.md) | INSERT/UPDATE/DELETE流程 | ⭐⭐⭐⭐ |
| [04_key_algorithms.md](04_key_algorithms.md) | HOT Update、剪枝、FSM/VM | ⭐⭐⭐⭐⭐ |
| [05_performance.md](05_performance.md) | 表膨胀、填充因子优化 | ⭐⭐⭐⭐ |
| [06_testcases.md](06_testcases.md) | 堆表测试用例 | ⭐⭐⭐ |
| [07_diagrams.md](07_diagrams.md) | 堆表架构图 | ⭐⭐⭐⭐ |

---

## 🎯 核心概念速览

### 1. 堆表是什么？

**Heap (堆)** 是 PostgreSQL 默认的表存储格式，采用**无序存储**策略：
- ✅ **追加写入**: 新数据直接追加到表末尾或空闲空间
- ✅ **MVCC支持**: 通过元组版本实现多版本并发控制
- ✅ **灵活更新**: 支持HOT Update优化
- ✗ **无序**: 数据按插入顺序存储，不保证物理顺序

### 2. 页面结构（8KB默认）

```
┌─────────────────────────────────────────┐
│         PageHeaderData (24B)            │  页头：LSN、校验和、空闲空间指针
├─────────────────────────────────────────┤
│         ItemIdData[] (4B each)          │  行指针数组（间接索引）
├─────────────────────────────────────────┤
│         Free Space                      │  空闲空间（向上增长）
├─────────────────────────────────────────┤
│         Tuples (from bottom)            │  元组数据（从底部向上）
├─────────────────────────────────────────┤
│         Special Space (optional)        │  特殊空间（索引页使用）
└─────────────────────────────────────────┘
```

### 3. 元组结构

```
HeapTupleHeader (23B fixed + 可变部分)
├── t_xmin (4B)      - 插入事务ID
├── t_xmax (4B)      - 删除/更新事务ID
├── t_cid (4B)       - 命令ID
├── t_ctid (6B)      - 当前/新版本TID
├── t_infomask (2B)  - 标志位
├── t_infomask2 (2B) - 更多标志
├── t_hoff (1B)      - 头部长度
└── t_bits[]         - NULL位图（可选）
```

---

## 🔑 关键特性

### 1. HOT Update (Heap-Only Tuple Update)

**条件**: 更新不改变索引列 + 新版本在同一页

**优势**:
- ✅ 无需更新索引 (巨大性能提升)
- ✅ 减少表膨胀
- ✅ 降低VACUUM压力

```sql
-- 查看HOT Update比例
SELECT relname,
       n_tup_upd,
       n_tup_hot_upd,
       round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 2) AS hot_ratio
FROM pg_stat_user_tables
WHERE n_tup_upd > 0
ORDER BY hot_ratio DESC;
```

### 2. TOAST (The Oversized-Attribute Storage Technique)

超大字段的存储机制：
- **阈值**: 列值 > ~2KB 自动TOAST
- **策略**: PLAIN、EXTENDED、EXTERNAL、MAIN
- **压缩**: 自动LZ压缩
- **外部存储**: 单独的TOAST表

```sql
-- 查看TOAST表
SELECT relname, reltoastrelid::regclass AS toast_table
FROM pg_class
WHERE reltoastrelid != 0 AND relkind = 'r';
```

### 3. 填充因子 (fillfactor)

控制页面填充程度，为HOT Update预留空间：

```sql
-- 默认 fillfactor = 100 (填满)
-- 调整为 70，预留 30% 空间
ALTER TABLE hot_update_table SET (fillfactor = 70);
VACUUM FULL hot_update_table;  -- 重建表

-- 查看表的填充因子
SELECT relname, reloptions
FROM pg_class
WHERE relname = 'hot_update_table';
```

---

## 📊 堆表监控

### 1. 表大小和膨胀

```sql
-- 查看表大小
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - 
                   pg_relation_size(schemaname||'.'||tablename)) AS index_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- 估算表膨胀
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS size,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_ratio DESC;
```

### 2. 死元组统计

```sql
-- 查看死元组最多的表
SELECT 
    schemaname,
    relname,
    n_dead_tup,
    n_live_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 20;
```

---

## 🚀 快速开始

### 1. 创建优化的堆表

```sql
-- 创建带优化选项的表
CREATE TABLE optimized_table (
    id SERIAL PRIMARY KEY,
    status TEXT,           -- 频繁更新
    data JSONB,            -- 不索引
    large_text TEXT,       -- 自动TOAST
    created_at TIMESTAMP DEFAULT now()
) WITH (
    fillfactor = 70,       -- 预留30%空间给HOT Update
    autovacuum_enabled = true,
    autovacuum_vacuum_scale_factor = 0.1,  -- 10%死元组触发
    autovacuum_analyze_scale_factor = 0.05
);

-- 只对需要的列创建索引
CREATE INDEX idx_status ON optimized_table(status) WHERE status != 'completed';
```

### 2. 检查页面利用率

```sql
-- 安装 pageinspect 扩展
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- 检查页面头信息
SELECT * FROM page_header(get_raw_page('your_table', 0));

-- 检查页面内容
SELECT lp, lp_off, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('your_table', 0))
LIMIT 10;
```

---

## 📖 学习路径

### 基础路径 (1-2天)
1. 阅读 [01_overview.md](01_overview.md) - 理解堆表基本概念
2. 阅读 [02_data_structures.md](02_data_structures.md) - 掌握页和元组结构
3. 实践监控查询 - 熟悉pg_stat_user_tables

### 进阶路径 (3-5天)
4. 阅读 [04_key_algorithms.md](04_key_algorithms.md) - 深入HOT Update
5. 阅读 [05_performance.md](05_performance.md) - 学习性能优化
6. 实践填充因子调优

### 专家路径 (1-2周)
7. 阅读 [03_implementation_flow.md](03_implementation_flow.md) - 理解实现细节
8. 使用 pageinspect 分析页面结构
9. 源码阅读: `src/backend/access/heap/`

---

## 🔗 相关模块

- **Buffer Manager**: 堆表页面的缓存管理
- **MVCC**: 元组版本控制和可见性判断
- **VACUUM**: 死元组清理和空间回收
- **FSM/VM**: 空闲空间映射和可见性映射
- **TOAST**: 大字段的外部存储

---

## 📝 常见问题

### Q1: 为什么我的表很大，但查询很慢？
**A**: 可能是表膨胀导致的。检查 `n_dead_tup`，运行 `VACUUM` 或 `VACUUM FULL`。

### Q2: HOT Update 为什么没生效？
**A**: 检查：
1. 更新是否改变了索引列
2. 新版本是否在同一页 (检查 fillfactor)
3. 页面是否有足够空闲空间

### Q3: 什么时候应该调整 fillfactor？
**A**: 
- 频繁 UPDATE 的表：fillfactor = 70-80
- 只读或追加的表：fillfactor = 100 (默认)
- 大量小更新：fillfactor = 60-70

### Q4: TOAST 表占用太多空间怎么办？
**A**:
```sql
-- VACUUM TOAST表
VACUUM FULL pg_toast.pg_toast_16384;

-- 调整TOAST策略
ALTER TABLE your_table ALTER COLUMN large_col SET STORAGE EXTERNAL;
```

---

## 🎯 性能指标

| 指标 | 目标 | 监控方法 |
|------|------|----------|
| HOT Update 比例 | > 80% | `pg_stat_user_tables.n_tup_hot_upd` |
| 死元组比例 | < 10% | `n_dead_tup / (n_live_tup + n_dead_tup)` |
| 表膨胀率 | < 20% | pgstattuple 扩展 |
| 页面利用率 | > 70% | pageinspect 扩展 |

---

## 📚 源码文件

| 文件 | 功能 |
|------|------|
| `src/include/storage/bufpage.h` | 页面结构定义 |
| `src/include/access/htup_details.h` | 元组结构定义 |
| `src/backend/access/heap/heapam.c` | 堆表访问方法 |
| `src/backend/access/heap/heapam_handler.c` | 表访问方法接口 |
| `src/backend/access/heap/hio.c` | 堆I/O操作 |
| `src/backend/access/heap/pruneheap.c` | 页面剪枝 |

---

**下一步**: 开始阅读 [01_overview.md](01_overview.md)

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-10-16

