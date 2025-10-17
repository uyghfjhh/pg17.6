# PostgreSQL索引深度分析系列

> PostgreSQL所有索引类型的完整技术文档

**创建日期**: 2025-10-17  
**PostgreSQL版本**: 17.6  
**文档系列**: 索引结构、算法、性能优化

---

## 📚 索引类型总览

PostgreSQL支持多种索引类型，每种都有特定的使用场景：

```
【PostgreSQL索引家族】

1. B-tree (默认) ⭐⭐⭐⭐⭐
   ├─ 最常用的索引类型
   ├─ 支持: =, <, <=, >, >=, BETWEEN, IN, IS NULL
   ├─ 支持: ORDER BY
   └─ 适合: 大部分场景

2. Hash ⭐⭐⭐
   ├─ 仅支持: =
   ├─ 不支持: 范围查询、排序
   └─ 适合: 简单等值查询

3. GiST (Generalized Search Tree) ⭐⭐⭐⭐
   ├─ 通用搜索树框架
   ├─ 支持: 几何、全文、范围等
   ├─ 可扩展
   └─ 适合: 空间数据、复杂类型

4. GIN (Generalized Inverted Index) ⭐⭐⭐⭐⭐
   ├─ 倒排索引
   ├─ 支持: 数组、JSONB、全文搜索
   ├─ 查询快，更新慢
   └─ 适合: 多值列、全文搜索

5. BRIN (Block Range INdex) ⭐⭐⭐⭐
   ├─ 块范围索引
   ├─ 极小的空间占用
   ├─ 适合: 物理有序的大表
   └─ 典型: 时间序列数据

6. SP-GiST (Space-Partitioned GiST) ⭐⭐⭐
   ├─ 空间分区索引
   ├─ 支持: 非平衡结构
   └─ 适合: 特殊数据分布
```

---

## 📖 文档列表

### 1. [B-tree索引深度分析](01_btree_index_deep_dive.md) ⭐⭐⭐⭐⭐
**内容**: B-tree结构、分裂、增删改查、存储  
**重要程度**: 最高（必读）  

**涵盖主题**:
- B-tree基本结构（Root/Internal/Leaf页）
- B-tree节点布局和存储格式
- 插入操作和页分裂
- 删除操作和页合并
- 查找操作详解
- 并发控制（Latch）
- Vacuum处理
- 性能优化

**图表**: 40+ ASCII图表
- 树结构图
- 页分裂过程
- 存储布局
- 操作流程

---

### 2. [Hash索引深度分析](02_hash_index_deep_dive.md) ⭐⭐⭐
**内容**: Hash索引结构和操作  

**涵盖主题**:
- Hash索引结构
- 动态哈希（Linear Hashing）
- Bucket分裂
- 溢出页处理
- 增删改查操作
- 与B-tree对比

---

### 3. [GiST索引深度分析](03_gist_index_deep_dive.md) ⭐⭐⭐⭐
**内容**: 通用搜索树和空间索引  

**涵盖主题**:
- GiST框架设计
- R-tree实现（空间数据）
- Bounding Box
- 空间操作（包含、相交、距离）
- 扩展自定义索引
- PostGIS使用

---

### 4. [GIN索引深度分析](04_gin_index_deep_dive.md) ⭐⭐⭐⭐⭐
**内容**: 倒排索引实现  

**涵盖主题**:
- GIN倒排索引原理
- Posting Tree/List
- Fast Update机制
- 数组索引
- JSONB索引
- 全文搜索
- 性能调优

---

### 5. [BRIN索引深度分析](05_brin_index_deep_dive.md) ⭐⭐⭐⭐
**内容**: 块范围索引  

**涵盖主题**:
- BRIN设计思想
- 范围摘要
- 极小的存储空间
- 时间序列优化
- 适用场景
- 与B-tree对比

---

### 6. [索引选择和性能优化](06_index_selection_and_tuning.md) ⭐⭐⭐⭐⭐
**内容**: 索引选择策略和调优  

**涵盖主题**:
- 如何选择索引类型
- 索引设计最佳实践
- 多列索引 vs 多个单列索引
- 覆盖索引
- 部分索引
- 表达式索引
- 索引维护
- 性能监控

---

## 🎯 学习路径

### 初级（必读）
1. **B-tree索引** - 最重要！90%的场景都用它
2. **索引选择和优化** - 实用技巧

### 中级
3. **GIN索引** - 数组、JSONB、全文搜索
4. **BRIN索引** - 大表优化

### 高级
5. **GiST索引** - 空间数据
6. **Hash索引** - 特殊场景

---

## 💡 快速参考

### 索引类型选择

```sql
-- 1. 普通列（数字、字符串、日期）
CREATE INDEX ON users(email);           -- B-tree (默认)
CREATE INDEX ON users(created_at);      -- B-tree

-- 2. 等值查询（WHERE col = value）
CREATE INDEX ON users(status);          -- B-tree或Hash
CREATE INDEX USING hash ON users(uid);  -- Hash (仅等值)

-- 3. 数组列
CREATE INDEX ON posts USING gin(tags);  -- GIN倒排索引
-- 查询: WHERE tags @> ARRAY['postgresql']

-- 4. JSONB
CREATE INDEX ON documents USING gin(data);  -- GIN
-- 查询: WHERE data @> '{"type":"article"}'

-- 5. 全文搜索
CREATE INDEX ON articles USING gin(to_tsvector('english', content));
-- 查询: WHERE to_tsvector('english', content) @@ 'postgresql'

-- 6. 空间数据（PostGIS）
CREATE INDEX ON locations USING gist(geom);  -- GiST
-- 查询: WHERE ST_DWithin(geom, point, 1000)

-- 7. 大表时间序列
CREATE INDEX ON logs USING brin(created_at);  -- BRIN
-- 适合: 按时间顺序插入的日志表

-- 8. 多列索引
CREATE INDEX ON orders(user_id, created_at);  -- B-tree
-- 可用: WHERE user_id = 1
-- 可用: WHERE user_id = 1 AND created_at > '2024-01-01'
-- 不可用: WHERE created_at > '2024-01-01' (不是最左列)

-- 9. 覆盖索引（Include columns）
CREATE INDEX ON orders(user_id) INCLUDE (status, amount);
-- Index-Only Scan: SELECT status, amount WHERE user_id = 1

-- 10. 部分索引
CREATE INDEX ON orders(user_id) WHERE status = 'pending';
-- 只索引pending订单，减小索引大小
```

### 性能对比

```
场景: 1000万行表

查询类型              B-tree    Hash      GIN       BRIN
─────────────────────────────────────────────────────────
等值 (=)              0.1ms     0.08ms    -         -
范围 (>, <)           0.5ms     ✗         -         5ms
排序 (ORDER BY)       0.2ms     ✗         -         ✗
数组包含 (@>)         ✗         ✗         0.3ms     -
全文搜索 (@@)         ✗         ✗         2ms       -
空间查询              ✗         ✗         ✗(GiST)   -

索引大小              1GB       800MB     2GB       10MB
构建时间              60s       50s       120s      5s
更新开销              中        中        高        低
```

---

## 🔧 常用命令

```sql
-- 查看表的所有索引
\d table_name

-- 查看索引定义
\d index_name

-- 查看索引大小
SELECT pg_size_pretty(pg_relation_size('index_name'));

-- 查看所有索引大小
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- 查看索引使用情况
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan as scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- 查看未使用的索引
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey';

-- 重建索引
REINDEX INDEX CONCURRENTLY index_name;  -- 不阻塞
REINDEX TABLE CONCURRENTLY table_name;  -- 重建表的所有索引

-- 分析索引bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) as indexes_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

## 📊 索引内部结构预览

### B-tree结构（示例）

```
【B-tree三层结构】
                    Root Page (metapage指向)
                    ┌──────────────────┐
                    │     [50]         │
                    └────┬────────┬────┘
                         │        │
           ┌─────────────┘        └─────────────┐
           │                                    │
    Internal Page                        Internal Page
    ┌──────────────┐                    ┌──────────────┐
    │  [10] [30]   │                    │  [70] [90]   │
    └──┬───┬───┬───┘                    └──┬───┬───┬───┘
       │   │   │                           │   │   │
     ┌─┘   │   └─┐                       ┌─┘   │   └─┐
     │     │     │                       │     │     │
  Leaf   Leaf  Leaf                   Leaf   Leaf  Leaf
  [1,5]  [10,  [30,                   [50,  [70,  [90,
         20,   40,                     60]   80]   95]
         25]   45]

【每个页8KB】
- Root: 指向Internal Pages
- Internal: 指向下层（Internal或Leaf）
- Leaf: 存储实际数据（key + TID）
- Leaf页之间双向链表（支持范围扫描）
```

### GIN倒排索引（示例）

```
【GIN索引结构】
数组列: tags

表数据:
  id | tags
  ---+------------------
  1  | {postgresql, database}
  2  | {postgresql, index}
  3  | {database, sql}

GIN索引:
  Entry Tree (B-tree)
  ┌────────────────────┐
  │ "database"    ────→ Posting List: [1, 3]
  │ "index"       ────→ Posting List: [2]
  │ "postgresql"  ────→ Posting List: [1, 2]
  │ "sql"         ────→ Posting List: [3]
  └────────────────────┘

查询: WHERE tags @> ARRAY['postgresql']
  → 查找Entry "postgresql"
  → 返回Posting List [1, 2]
  → 访问tuple 1和2
```

---

## ⚡ 性能调优提示

### 1. B-tree索引优化
```sql
-- ✅ 好: 选择性高的列
CREATE INDEX ON users(email);  -- 几乎唯一

-- ❌ 差: 选择性低的列
CREATE INDEX ON users(gender);  -- 只有2-3个值

-- ✅ 好: 多列索引（查询常用组合）
CREATE INDEX ON orders(user_id, status, created_at);

-- ✅ 好: 覆盖索引
CREATE INDEX ON orders(user_id) INCLUDE (amount, status);
```

### 2. GIN索引优化
```sql
-- 调整fastupdate和gin_pending_list_limit
ALTER INDEX idx_tags SET (fastupdate = on);
ALTER INDEX idx_tags SET (gin_pending_list_limit = 4096);  -- 4MB

-- 定期清理pending list
VACUUM table_name;
```

### 3. BRIN索引优化
```sql
-- 调整pages_per_range
CREATE INDEX ON logs USING brin(created_at) 
WITH (pages_per_range = 128);  -- 默认128

-- 更小的范围 = 更精确但更大的索引
-- 更大的范围 = 更小的索引但精度降低
```

---

## 📈 监控和维护

```sql
-- 1. 定期ANALYZE（更新统计信息）
ANALYZE table_name;

-- 2. 定期VACUUM（清理死元组）
VACUUM table_name;

-- 3. 定期REINDEX（重建bloated索引）
REINDEX INDEX CONCURRENTLY index_name;

-- 4. 监控索引使用
SELECT * FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;

-- 5. 删除未使用的索引
-- DROP INDEX CONCURRENTLY unused_index;
```

---

## 🎓 推荐阅读顺序

### 必读（所有人）
1. ✅ **B-tree索引深度分析** - 理解90%的索引场景
2. ✅ **索引选择和性能优化** - 实用技巧

### 进阶（开发者）
3. ✅ **GIN索引深度分析** - 数组、JSONB
4. ✅ **BRIN索引深度分析** - 大表优化

### 专家（DBA、架构师）
5. ✅ **GiST索引深度分析** - 空间数据
6. ✅ **Hash索引深度分析** - 了解所有选项

---

## 📚 参考资源

### PostgreSQL官方文档
- [Chapter 11: Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [Chapter 64: Index Access Method Interface](https://www.postgresql.org/docs/current/indexam.html)
- [Chapter 67: B-Tree Indexes](https://www.postgresql.org/docs/current/btree.html)

### 源码位置
- **B-tree**: `src/backend/access/nbtree/`
- **Hash**: `src/backend/access/hash/`
- **GiST**: `src/backend/access/gist/`
- **GIN**: `src/backend/access/gin/`
- **BRIN**: `src/backend/access/brin/`

### 经典论文
- **B-tree**: "Organization and Maintenance of Large Ordered Indexes" (Bayer & McCreight, 1972)
- **R-tree**: "R-trees: A Dynamic Index Structure for Spatial Searching" (Guttman, 1984)
- **GiST**: "Generalized Search Trees for Database Systems" (Hellerstein et al., 1995)

---

**文档版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-17  
**状态**: 🚧 创建中...

---

**下一步**: 开始创建B-tree索引深度分析！这是最重要的索引类型，值得详细研究！

