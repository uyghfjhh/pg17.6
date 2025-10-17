# B-Tree 索引模块

> PostgreSQL B-Tree索引的深度分析

**模块状态**: ✅ 完成  
**文档版本**: 1.0  
**最后更新**: 2025-10-16  
**基于版本**: PostgreSQL 17.6

---

## 📚 模块导航

| 文档 | 内容概要 | 重点 |
|------|---------|------|
| [01_overview.md](01_overview.md) | B+树概述、特性 | ⭐⭐⭐⭐⭐ |
| [02_data_structures.md](02_data_structures.md) | 页结构、索引元组 | ⭐⭐⭐⭐⭐ |
| [03_implementation_flow.md](03_implementation_flow.md) | 搜索/插入/删除流程 | ⭐⭐⭐⭐ |
| [04_key_algorithms.md](04_key_algorithms.md) | 页分裂、合并、唯一性 | ⭐⭐⭐⭐⭐ |
| [05_performance.md](05_performance.md) | 索引优化、膨胀处理 | ⭐⭐⭐⭐ |
| [06_testcases.md](06_testcases.md) | 索引测试用例 | ⭐⭐⭐ |
| [07_diagrams.md](07_diagrams.md) | B树架构图 | ⭐⭐⭐⭐ |

---

## 🎯 核心概念

### 1. B+树特性

- ✅ **平衡树**: 所有叶子节点在同一层
- ✅ **高扇出**: 每页存储数百个键 (高度低)
- ✅ **有序存储**: 支持范围查询
- ✅ **右链**: 叶子节点右链接，支持顺序扫描

### 2. 页面类型

```
Meta Page (page 0): 元数据 (根页号、层级等)
Root Page: 根节点
Internal Pages: 内部节点 (指向下层)
Leaf Pages: 叶子节点 (存储TID)
```

### 3. 关键操作

```sql
-- 创建索引
CREATE INDEX idx_name ON table(column);

-- 唯一索引
CREATE UNIQUE INDEX idx_unique ON table(column);

-- 多列索引
CREATE INDEX idx_multi ON table(col1, col2);

-- 部分索引
CREATE INDEX idx_partial ON table(column) WHERE condition;

-- 并发创建
CREATE INDEX CONCURRENTLY idx_name ON table(column);
```

---

## 🚀 快速监控

```sql
-- 查看索引大小
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS idx_size,
    idx_scan,  -- 使用次数
    idx_tup_read,  -- 读取行数
    idx_tup_fetch  -- 返回行数
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- 查看索引膨胀
SELECT 
    schemaname || '.' || tablename AS table_name,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    round(100 * pg_relation_size(indexrelid)::numeric / 
          NULLIF(pg_relation_size(tablename::regclass), 0), 2) AS index_ratio
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- 未使用的索引
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND schemaname NOT IN ('pg_catalog', 'information_schema');
```

---

## 📖 学习路径

### 基础 (1-2天)
1. 阅读 [01_overview.md](01_overview.md) - B+树基础
2. 理解索引扫描 vs 顺序扫描

### 进阶 (3-5天)
3. 阅读 [04_key_algorithms.md](04_key_algorithms.md) - 页分裂算法
4. 阅读 [05_performance.md](05_performance.md) - 性能优化

### 专家 (1-2周)
5. 源码阅读: `src/backend/access/nbtree/`
6. 实践: pgstattuple分析索引健康度

---

**下一步**: 开始阅读 [01_overview.md](01_overview.md)

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-10-16

