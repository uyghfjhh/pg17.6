# Planner/Optimizer 核心分析 (精简版)

> PostgreSQL查询优化器的核心原理、代价模型和优化技术

**源码**: `src/backend/optimizer/`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. 优化器架构

### 1.1 核心流程

```
planner(Query *parse)
│
├─→ [1] 预处理
│    └─ preprocess_expression()
│        ├─ 常量折叠: WHERE 1+1=2 → WHERE true
│        ├─ 布尔简化: WHERE true AND x → WHERE x
│        └─ 子查询提升
│
├─→ [2] 生成所有可能的路径
│    └─ query_planner()
│        │
│        ├─ 扫描路径:
│        │   ├─ SeqScan (顺序扫描)
│        │   ├─ IndexScan (索引扫描)
│        │   ├─ IndexOnlyScan (仅索引扫描)
│        │   └─ BitmapScan (位图扫描)
│        │
│        ├─ Join路径:
│        │   ├─ NestedLoop (嵌套循环)
│        │   ├─ HashJoin (哈希连接)
│        │   └─ MergeJoin (归并连接)
│        │
│        └─ 聚合路径:
│            ├─ PlainAgg (普通聚合)
│            ├─ HashAgg (哈希聚合)
│            └─ GroupAgg (分组聚合)
│
├─→ [3] 代价估算
│    └─ cost_*() 系列函数
│        ├─ cost_seqscan()
│        ├─ cost_index()
│        └─ cost_hashjoin()
│
├─→ [4] 选择最优路径
│    └─ 比较所有路径的total_cost
│        └─ 选择成本最低的
│
└─→ [5] 生成执行计划
     └─ create_plan()
         └─ 将Path转换为Plan
```

---

## 2. 代价模型

### 2.1 成本公式

```
Total Cost = Startup Cost + Run Cost

Startup Cost: 开始返回第一行前的成本
Run Cost: 处理所有行的成本

SeqScan Cost:
  = (n_pages * seq_page_cost) +  // I/O成本
    (n_tuples * cpu_tuple_cost)   // CPU成本

IndexScan Cost:
  = (n_index_pages * random_page_cost) +  // 索引I/O
    (n_heap_pages * random_page_cost) +   // 堆表I/O
    (n_tuples * cpu_index_tuple_cost) +   // 索引CPU
    (n_tuples * cpu_tuple_cost)           // 堆表CPU

HashJoin Cost:
  = cost(outer) +                    // 外表成本
    cost(inner) +                    // 内表成本
    (n_outer * cpu_operator_cost) +  // 哈希计算
    (n_inner * cpu_tuple_cost)       // 探测成本
```

### 2.2 关键参数

```sql
-- I/O成本参数
seq_page_cost = 1.0           -- 顺序读一页的成本
random_page_cost = 4.0        -- 随机读一页的成本 (HDD)
                = 1.1         -- SSD建议值

-- CPU成本参数
cpu_tuple_cost = 0.01         -- 处理一个元组的成本
cpu_index_tuple_cost = 0.005  -- 处理一个索引元组
cpu_operator_cost = 0.0025    -- 执行一个操作符

-- 内存参数
work_mem = 4MB                -- 排序/哈希的工作内存
effective_cache_size = 4GB    -- 估算的缓存大小
```

---

## 3. 行数估算

### 3.1 基础统计

```
pg_statistic表存储:
  • n_distinct: 不同值数量
  • most_common_vals: 最常见值
  • most_common_freqs: 最常见值的频率
  • histogram_bounds: 直方图边界

估算公式:

WHERE col = 'value':
  selectivity = 1 / n_distinct
  rows = table_rows * selectivity

WHERE col > 100:
  使用直方图估算在范围内的比例
  rows = table_rows * range_selectivity

WHERE col1 = 1 AND col2 = 2:
  独立性假设:
  selectivity = sel(col1) * sel(col2)
  rows = table_rows * sel1 * sel2
```

### 3.2 更新统计信息

```sql
-- ANALYZE收集统计信息
ANALYZE table_name;

-- 查看统计信息
SELECT * FROM pg_stats WHERE tablename = 'users';

-- 调整统计目标 (更精确但更慢)
ALTER TABLE users ALTER COLUMN name SET STATISTICS 1000;  -- 默认100

-- 多列统计 (相关列)
CREATE STATISTICS stat_city_zip ON city, zipcode FROM addresses;
ANALYZE addresses;
```

---

## 4. Join优化

### 4.1 Join顺序

```
3表Join: A JOIN B JOIN C

可能的Join顺序:
  1. (A JOIN B) JOIN C
  2. (A JOIN C) JOIN B
  3. (B JOIN C) JOIN A

Join顺序选择策略:

• 小表 (≤ 12表): 动态规划 (枚举所有顺序)
• 大表 (> 12表): 遗传算法 (GEQO - Genetic Query Optimizer)
  └─ geqo_threshold = 12
```

### 4.2 Join算法选择

```
NestedLoop:
  • 小表 × 小表
  • 内表有索引
  • 成本: O(M × N) 或 O(M × log N)

HashJoin:
  • 大表 × 大表
  • 等值连接
  • 内表可放入内存 (work_mem)
  • 成本: O(M + N)

MergeJoin:
  • 两表都已排序
  • 非等值连接也支持
  • 成本: O(M + N)
```

---

## 5. 索引选择

### 5.1 索引使用条件

```
使用索引的条件:
  ✅ WHERE子句有索引列
  ✅ 估算行数 < 总行数的 5-10%
  ✅ random_page_cost不能太高
  ✗ 查询返回大部分行 → SeqScan更快

示例:
  SELECT * FROM users WHERE id = 1;
  → IndexScan (只返回1行)

  SELECT * FROM users WHERE age > 0;
  → SeqScan (返回所有行，顺序扫描更快)
```

### 5.2 多列索引

```
CREATE INDEX idx_multi ON users(city, age);

可以使用的查询:
  ✅ WHERE city = 'NYC'
  ✅ WHERE city = 'NYC' AND age > 18
  ✗ WHERE age > 18  (不能单独使用age)

顺序很重要:
  • 高选择性列放前面
  • 等值条件列放前面
```

---

## 6. 优化技巧

### 6.1 EXPLAIN分析

```sql
-- 查看执行计划
EXPLAIN SELECT * FROM users WHERE age > 18;

-- 实际执行并显示统计
EXPLAIN ANALYZE SELECT * FROM users WHERE age > 18;

-- 显示更多细节
EXPLAIN (ANALYZE, BUFFERS, TIMING) SELECT ...;

-- 关键指标:
--   cost: 估算成本
--   rows: 估算行数
--   actual time: 实际时间
--   actual rows: 实际行数
```

### 6.2 强制执行计划

```sql
-- 禁用某种扫描
SET enable_seqscan = off;     -- 禁用顺序扫描
SET enable_indexscan = off;   -- 禁用索引扫描
SET enable_hashjoin = off;    -- 禁用哈希连接

-- 调整成本参数
SET random_page_cost = 1.1;   -- SSD优化
SET work_mem = '256MB';       -- 增加工作内存

-- 会话结束后恢复
RESET enable_seqscan;
RESET random_page_cost;
```

---

## 7. 常见问题

### 7.1 执行计划不准确

```sql
-- 问题: 估算行数与实际相差太大
-- 原因: 统计信息过期

-- 解决:
ANALYZE table_name;  -- 更新统计

-- 检查统计信息新鲜度:
SELECT 
    schemaname,
    relname,
    last_analyze,
    last_autovacuum,
    n_mod_since_analyze  -- 修改行数
FROM pg_stat_user_tables;
```

### 7.2 索引不被使用

```sql
-- 问题: 有索引但用SeqScan
-- 可能原因:

-- 1. 数据类型不匹配
WHERE age = '18'  -- age是INT，'18'是TEXT
-- 解决: WHERE age = 18

-- 2. 使用了函数
WHERE LOWER(name) = 'alice'
-- 解决: 创建函数索引
CREATE INDEX idx_name_lower ON users(LOWER(name));

-- 3. 返回行数太多
WHERE age > 0  -- 返回所有行
-- 无需索引，SeqScan更快

-- 4. random_page_cost太高
SHOW random_page_cost;  -- 默认4.0 (HDD)
SET random_page_cost = 1.1;  -- SSD
```

---

## 8. 并行查询

### 8.1 并行扫描

```sql
-- 参数配置
max_parallel_workers_per_gather = 4  -- 每个查询最多4个worker
max_parallel_workers = 8             -- 全局最多8个worker

-- 触发条件:
--   • 表大小 > min_parallel_table_scan_size (8MB)
--   • 估算行数足够多
--   • 查询足够复杂

EXPLAIN SELECT COUNT(*) FROM large_table;
/*
Finalize Aggregate  (cost=... rows=1)
  ->  Gather  (cost=... rows=4)
        Workers Planned: 4
        ->  Partial Aggregate
              ->  Parallel Seq Scan on large_table
*/
```

---

## 总结

### 优化器核心

1. **路径生成**: 枚举所有可能的执行方式
2. **代价估算**: 基于统计信息估算成本
3. **路径选择**: 选择成本最低的路径
4. **动态规划**: 小表Join顺序优化
5. **遗传算法**: 大表Join顺序优化

### 最佳实践

- ✅ 定期ANALYZE更新统计
- ✅ 创建合适的索引
- ✅ SSD设置random_page_cost=1.1
- ✅ 增加work_mem (排序/哈希)
- ✅ 使用EXPLAIN ANALYZE分析

---

**完成**: Planner/Optimizer核心文档完成！

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

