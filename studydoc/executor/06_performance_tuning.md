# Executor性能调优实战

> PostgreSQL查询性能优化的完整指南

**重要程度**: ⭐⭐⭐⭐⭐  
**适用对象**: DBA、开发者  
**核心内容**: 诊断方法、调优技巧、参数配置、实战案例

---

## 📋 性能问题诊断

### EXPLAIN ANALYZE - 核心工具

```sql
-- 基本用法
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)
SELECT * FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.age > 18;

/*
【输出解读】
┌──────────────────────────────────────────────────────────────┐
│ Hash Join  (cost=100.00..500.00 rows=1000 width=64)         │
│   (actual time=5.234..123.456 rows=1234 loops=1)            │
│   Hash Cond: (o.user_id = u.id)                             │
│   Buffers: shared hit=450 read=50                            │
│   ->  Seq Scan on orders o                                   │
│         (cost=0.00..350.00 rows=10000 width=32)             │
│         (actual time=0.010..45.678 rows=10123 loops=1)      │
│         Buffers: shared hit=400 read=45                      │
│   ->  Hash  (cost=75.00..75.00 rows=2000 width=32)          │
│         (actual time=2.345..2.345 rows=2345 loops=1)        │
│         Buckets: 4096  Batches: 1  Memory Usage: 123kB      │
│         Buffers: shared hit=50 read=5                        │
│         ->  Index Scan using users_age_idx on users u       │
│               (cost=0.29..75.00 rows=2000 width=32)         │
│               (actual time=0.025..1.234 rows=2345 loops=1)  │
│               Index Cond: (age > 18)                         │
│               Buffers: shared hit=50 read=5                  │
│ Planning Time: 0.456 ms                                      │
│ Execution Time: 125.678 ms                                   │
└──────────────────────────────────────────────────────────────┘

【关键指标】
1. cost vs actual time
   - cost: 优化器估算
   - actual time: 实际执行时间
   - 差距大 → 统计信息过时，运行ANALYZE

2. rows估算 vs actual rows
   - rows: 优化器估算行数
   - actual rows: 实际行数
   - 差距大 → 统计不准，影响plan选择

3. Buffers
   - shared hit: 从共享缓存读取
   - read: 从磁盘读取
   - temp: 临时文件I/O
   - written: 写入操作

4. loops
   - loops=1: 执行一次
   - loops>1: 嵌套循环，注意!
     actual time和rows要乘以loops才是总开销

5. Memory Usage (HashAgg/HashJoin)
   - 显示内存使用
   - 检查是否超过work_mem
   - Batches>1说明内存不足

6. Execution Time vs Planning Time
   - Planning慢 → 复杂查询，考虑Prepared Statement
   - Execution慢 → 需要优化执行计划
*/
```

### 常见性能问题

```sql
-- 问题1: Sequential Scan on 大表
EXPLAIN ANALYZE
SELECT * FROM large_table WHERE id = 1;

/*
Seq Scan on large_table  (cost=0.00..1000000 rows=1 width=...)
  (actual time=5000..10000 rows=1 loops=1)
  Filter: (id = 1)
  Rows Removed by Filter: 9999999

【问题】
❌ 全表扫描1000万行，找1行
❌ 10秒执行时间

【解决】
✅ 创建索引:
*/
CREATE INDEX ON large_table(id);

-- 再次执行
EXPLAIN ANALYZE
SELECT * FROM large_table WHERE id = 1;

/*
Index Scan using large_table_id_idx on large_table
  (cost=0.43..8.45 rows=1 width=...)
  (actual time=0.025..0.026 rows=1 loops=1)
  Index Cond: (id = 1)

【结果】
✅ 0.026ms (从10秒降到0.026ms!)
✅ 400,000倍提速!
*/

-- 问题2: Nested Loop with 高loops
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.city = 'NYC';

/*
Nested Loop  (cost=... rows=...)
  (actual time=...1000 loops=1)
  ->  Seq Scan on users u  (cost=... rows=10000)
        (actual time=... rows=10000 loops=1)
        Filter: (city = 'NYC')
  ->  Index Scan using orders_user_id_idx on orders o
        (cost=... rows=10)
        (actual time=...0.5 rows=10 loops=10000)  ← 注意loops!
        Index Cond: (user_id = u.id)

【问题】
❌ Inner loop执行10000次
❌ 每次0.5ms × 10000 = 5000ms
❌ 太慢!

【解决】
✅ 在users.city上创建索引:
*/
CREATE INDEX ON users(city);

-- 或者调整Join算法
SET work_mem = '256MB';  -- 允许Hash Join

/*
Hash Join  (cost=... rows=...)
  (actual time=...100 loops=1)
  Hash Cond: (o.user_id = u.id)
  ->  Seq Scan on orders  (actual time=...50 loops=1)
  ->  Hash  (cost=... rows=10000)
        (actual time=...10 loops=1)
        ->  Index Scan using users_city_idx on users
              (actual time=...5 loops=1)
              Index Cond: (city = 'NYC')

【结果】
✅ 100ms (从5000ms降到100ms)
✅ 50倍提速!
*/

-- 问题3: Hash Aggregate with Batches
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, COUNT(*), SUM(amount)
FROM large_orders
GROUP BY user_id;

/*
HashAggregate  (cost=... rows=...)
  (actual time=...5000 loops=1)
  Group Key: user_id
  Batches: 16  ← 问题! 需要16批
  Memory Usage: 4096kB
  Disk Usage: 102400kB  ← 使用了100MB磁盘
  ->  Seq Scan on large_orders
        (actual time=...4000 loops=1)
Buffers: shared read=100000, temp read=12800 temp written=12800

【问题】
❌ work_mem太小，无法hold所有groups
❌ 磁盘I/O慢

【解决】
✅ 增大work_mem:
*/
SET work_mem = '128MB';  -- 从4MB增加到128MB

/*
HashAggregate  (cost=... rows=...)
  (actual time=...1000 loops=1)
  Group Key: user_id
  Batches: 1  ← 现在只需1批!
  Memory Usage: 65536kB
  ->  Seq Scan on large_orders
Buffers: shared read=100000  ← 无temp I/O

【结果】
✅ 1000ms (从5000ms降到1000ms)
✅ 5倍提速!
✅ 无磁盘I/O
*/

-- 问题4: Sort Spilling to Disk
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table
ORDER BY created_at
LIMIT 10;

/*
Limit  (cost=... rows=10)
  (actual time=...8000 loops=1)
  ->  Sort  (cost=... rows=10000000)
        (actual time=...8000 loops=1)
        Sort Key: created_at
        Sort Method: external merge  ← 问题! 外部归并
        Disk: 307200kB  ← 使用了300MB磁盘
        ->  Seq Scan on large_table
              (actual time=...2000 loops=1)
Buffers: shared read=100000, temp read=38400 temp written=38400

【问题】
❌ 需要排序1000万行
❌ 但只需要前10行
❌ 排序溢出到磁盘

【解决方案1】
✅ 增大work_mem:
*/
SET work_mem = '512MB';

/*
Sort Method: quicksort  ← 内存排序!
Memory: 524288kB
【结果】
✅ 3000ms (从8000ms降到3000ms)
*/

/*
【解决方案2 - 更好】
✅ 创建索引:
*/
CREATE INDEX ON large_table(created_at);

/*
Limit  (cost=... rows=10)
  (actual time=...0.5 loops=1)
  ->  Index Scan using large_table_created_at_idx
        on large_table
        (actual time=...0.5 rows=10 loops=1)

【结果】
✅ 0.5ms (从8000ms降到0.5ms!)
✅ 16,000倍提速!
✅ 无需排序!
*/
```

---

## 核心调优参数

### work_mem - 工作内存

```sql
/*
【work_mem作用】
控制每个查询操作可使用的内存:
  - Sort (ORDER BY)
  - Hash (Hash Join, Hash Agg)
  - Bitmap operations

默认: 4MB (太小!)
推荐: 64-256MB (取决于业务)

【注意】
❌ 每个操作都会分配work_mem
❌ 复杂查询可能有多个操作
❌ 多个并发查询 × work_mem = 总内存

计算公式:
  max_work_mem = (RAM * 0.25) / max_connections / avg_operations_per_query
  
  示例: 64GB RAM, 100 connections, 平均3个操作
  work_mem = (64GB * 0.25) / 100 / 3 ≈ 50MB
*/

-- 全局设置 (postgresql.conf)
work_mem = '64MB'

-- 会话级别设置
SET work_mem = '256MB';  -- 对当前session有效

-- 事务级别设置
BEGIN;
SET LOCAL work_mem = '512MB';  -- 仅对当前事务
SELECT ...;
COMMIT;

-- 查询级别设置
SET work_mem = '1GB';
SELECT ...;  -- 这个查询使用1GB
RESET work_mem;  -- 恢复默认

/*
【监控work_mem使用】
*/
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;

-- 查看输出:
-- Sort Method: quicksort  Memory: 123456kB  ← 内存排序
-- Sort Method: external merge  Disk: 307200kB  ← 磁盘排序 (需要增大work_mem!)
-- Batches: 1  Memory Usage: 456kB  ← Hash在内存
-- Batches: 8  Disk Usage: 1024kB  ← Hash溢出 (需要增大work_mem!)
```

### shared_buffers - 共享缓存

```sql
/*
【shared_buffers作用】
PostgreSQL的主缓存
存储经常访问的数据pages

默认: 128MB (太小!)
推荐: RAM的25% (不超过32GB)

示例:
  16GB RAM → 4GB shared_buffers
  64GB RAM → 16GB shared_buffers
  256GB RAM → 32GB shared_buffers (上限)

【为什么有上限?】
PostgreSQL还依赖OS cache
shared_buffers太大会挤占OS cache
最优: shared_buffers + OS cache = 75% RAM
*/

-- 设置 (postgresql.conf, 需要重启)
shared_buffers = '16GB'

/*
【监控命中率】
*/
SELECT 
    blks_hit,
    blks_read,
    round(blks_hit::numeric / (blks_hit + blks_read) * 100, 2) as hit_ratio
FROM pg_stat_database
WHERE datname = current_database();

/*
  blks_hit  | blks_read | hit_ratio
 -----------+-----------+-----------
  10000000  |  100000   |  99.01

理想: hit_ratio > 99%
如果 < 95%: 考虑增大shared_buffers
*/
```

### effective_cache_size - 缓存大小提示

```sql
/*
【effective_cache_size作用】
告诉优化器可用的总缓存
包括: shared_buffers + OS cache

不分配内存，仅影响cost估算
越大 → 优化器越倾向IndexScan

推荐: RAM的75%

示例:
  16GB RAM → 12GB effective_cache_size
  64GB RAM → 48GB effective_cache_size
*/

-- 设置 (postgresql.conf, reload即可)
effective_cache_size = '48GB'

/*
【影响】
影响IndexScan vs SeqScan的选择
effective_cache_size大 → IndexScan cost低
*/
```

### random_page_cost - 随机I/O代价

```sql
/*
【random_page_cost作用】
随机I/O的代价估算
相对于seq_page_cost=1.0

默认: 4.0 (假设HDD)
SSD推荐: 1.1-1.5
NVMe推荐: 1.0-1.1

【影响】
random_page_cost高 → 优化器倾向SeqScan
random_page_cost低 → 优化器倾向IndexScan

【测试你的磁盘】
*/
-- HDD:
--   顺序: 100-200 MB/s
--   随机: 1-2 MB/s
--   比率: ~100倍 → random_page_cost = 4.0

-- SATA SSD:
--   顺序: 500 MB/s
--   随机: 300 MB/s
--   比率: ~1.7倍 → random_page_cost = 1.5

-- NVMe SSD:
--   顺序: 3000 MB/s
--   随机: 2500 MB/s
--   比率: ~1.2倍 → random_page_cost = 1.1

-- 设置
random_page_cost = 1.1  -- SSD

/*
【验证效果】
*/
-- 设置前
EXPLAIN SELECT * FROM large_table WHERE id = 1;
-- Seq Scan ...

-- 设置后
SET random_page_cost = 1.1;
EXPLAIN SELECT * FROM large_table WHERE id = 1;
-- Index Scan ...  ← 优化器改选择IndexScan了!
```

### 并行查询参数

```sql
/*
【并行查询配置】
*/
-- 最大并行worker数
max_parallel_workers_per_gather = 4  -- 每个查询最多4个worker

-- 系统总worker数
max_parallel_workers = 8  -- 系统级别

-- 并行的最小表大小
min_parallel_table_scan_size = '8MB'  -- 小于8MB不并行

-- 并行的最小索引大小
min_parallel_index_scan_size = '512kB'

/*
【监控并行查询】
*/
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FROM large_table;

/*
Finalize Aggregate  (cost=... rows=1)
  (actual time=...500 loops=1)
  ->  Gather  (cost=... rows=4)  ← 并行gather
        (actual time=...500 loops=1)
        Workers Planned: 4  ← 计划4个worker
        Workers Launched: 4  ← 实际启动4个
        ->  Partial Aggregate
              (actual time=...120 loops=5)  ← 5个进程(1主+4worker)
              ->  Parallel Seq Scan on large_table
                    (actual time=...100 loops=5)

【效果】
单线程: 500ms × 5 = 2500ms
并行: 500ms (5倍提速!)
*/

-- 强制并行 (调试用)
SET force_parallel_mode = on;
SET max_parallel_workers_per_gather = 0;  -- 禁用并行
```

---

## 索引优化

### 索引选择策略

```sql
-- 1. B-tree索引 (最常用)
CREATE INDEX ON users(email);  -- 等值查询
CREATE INDEX ON users(created_at);  -- 范围查询
CREATE INDEX ON users(age);  -- 排序

-- 2. 复合索引
CREATE INDEX ON orders(user_id, created_at);  -- 多列查询

-- 使用规则:
-- ✅ WHERE user_id = 1  (使用索引)
-- ✅ WHERE user_id = 1 AND created_at > '2024-01-01'  (使用索引)
-- ✅ WHERE user_id = 1 ORDER BY created_at  (使用索引)
-- ❌ WHERE created_at > '2024-01-01'  (不使用索引! 不是最左列)

-- 3. 覆盖索引 (Index-Only Scan)
CREATE INDEX ON orders(user_id) INCLUDE (amount, status);

-- ✅ Index-Only Scan:
SELECT amount, status FROM orders WHERE user_id = 1;
-- 不需要访问表! 最快!

-- 4. 部分索引
CREATE INDEX ON orders(user_id) 
WHERE status = 'pending';  -- 只索引pending订单

-- 适用场景:
-- ✅ 只查询某些状态的数据
-- ✅ 减少索引大小
-- ✅ 加快更新速度

-- 5. 表达式索引
CREATE INDEX ON users(lower(email));  -- 大小写不敏感

-- 使用:
SELECT * FROM users WHERE lower(email) = 'alice@example.com';
-- ✅ 使用索引

-- 6. GIN索引 (全文搜索、数组)
CREATE INDEX ON documents USING gin(to_tsvector('english', content));
CREATE INDEX ON tags USING gin(tag_array);

-- 7. GiST索引 (地理空间)
CREATE INDEX ON locations USING gist(geom);

-- 8. BRIN索引 (大表，顺序数据)
CREATE INDEX ON logs USING brin(created_at);
-- 适合: 时间序列数据，只占用很小空间
```

### 索引维护

```sql
-- 1. 查看索引使用情况
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,  -- 索引扫描次数
    idx_tup_read,  -- 读取的tuple数
    idx_tup_fetch  -- 实际获取的tuple数
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- 找出未使用的索引
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- 从未使用
  AND indexrelname NOT LIKE '%_pkey'  -- 排除主键
ORDER BY pg_relation_size(indexrelid) DESC;

-- 删除未使用的索引
-- DROP INDEX unused_index;

-- 2. 重建bloated索引
-- 查找膨胀的索引
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size,
    idx_scan
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- 重建索引
REINDEX INDEX CONCURRENTLY index_name;  -- 不阻塞查询
-- 或
DROP INDEX CONCURRENTLY old_index;
CREATE INDEX CONCURRENTLY new_index ON ...;

-- 3. 定期ANALYZE
-- 更新统计信息，帮助优化器选择正确的索引
ANALYZE users;
ANALYZE;  -- 所有表

-- 自动ANALYZE (推荐开启)
-- autovacuum = on  (默认)
```

---

## 查询优化技巧

### 1. 避免SELECT *

```sql
-- ❌ 慢
SELECT * FROM large_table WHERE id = 1;
-- 读取所有列，传输大量数据

-- ✅ 快
SELECT id, name, email FROM large_table WHERE id = 1;
-- 只读取需要的列

-- ✅ 更快 (覆盖索引)
CREATE INDEX ON large_table(id) INCLUDE (name, email);
SELECT id, name, email FROM large_table WHERE id = 1;
-- Index-Only Scan! 不访问表!
```

### 2. 优化JOIN顺序

```sql
-- PostgreSQL会自动优化，但有时需要手动干预

-- ❌ 可能慢
SELECT * FROM large_table l
JOIN small_table s ON l.id = s.id;
-- 如果优化器选择以large_table驱动

-- ✅ 提示优化器
SET join_collapse_limit = 1;  -- 按写的顺序Join
SELECT * FROM small_table s  -- 小表在前
JOIN large_table l ON s.id = l.id;
RESET join_collapse_limit;

-- 或者强制使用特定Join算法
SET enable_nestloop = off;  -- 禁用Nested Loop
SET enable_hashjoin = on;   -- 强制Hash Join
```

### 3. 使用EXISTS代替IN (子查询)

```sql
-- ❌ 可能慢 (如果子查询结果很大)
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders);

-- ✅ 更快 (短路优化)
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
-- 找到第一个匹配就停止

-- ✅ 最快 (如果有索引)
SELECT DISTINCT u.* FROM users u
JOIN orders o ON u.id = o.user_id;
-- 使用Hash Join或Merge Join
```

### 4. 分页优化

```sql
-- ❌ 慢 (深分页)
SELECT * FROM users
ORDER BY created_at
LIMIT 100 OFFSET 1000000;
-- 需要skip前100万行

-- ✅ 快 (Keyset Pagination)
SELECT * FROM users
WHERE created_at > '2024-01-01 12:00:00'  -- 上一页的最后值
ORDER BY created_at
LIMIT 100;
-- 直接定位，不需要skip

-- 需要索引:
CREATE INDEX ON users(created_at);
```

### 5. 批量操作

```sql
-- ❌ 慢 (逐行插入)
FOR i IN 1..10000 LOOP
    INSERT INTO users VALUES (...);
END LOOP;
-- 10000次网络往返

-- ✅ 快 (批量插入)
INSERT INTO users VALUES
    (1, ...),
    (2, ...),
    (3, ...),
    ...
    (10000, ...);
-- 1次网络往返

-- ✅ 更快 (COPY)
COPY users FROM '/path/to/data.csv' WITH CSV;
-- 最快的批量加载方式
```

---

## 实战案例

### 案例1: 慢查询优化

```sql
-- 问题查询 (执行8秒)
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    u.name,
    COUNT(*) as order_count,
    SUM(o.amount) as total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.created_at > '2024-01-01'
GROUP BY u.name;

/*
HashAggregate  (actual time=8000...8100)
  Group Key: u.name
  Batches: 16  ← 问题1: 内存不足
  ->  Hash Join  (actual time=1000...7000)
        Hash Cond: (o.user_id = u.id)
        ->  Seq Scan on orders o  ← 问题2: 全表扫描
              (actual time=0...6000)
              Filter: (created_at > '2024-01-01')
              Rows Removed by Filter: 5000000  ← 问题3: 过滤大量行
        ->  Hash  (actual time=500...500)
              ->  Seq Scan on users u

【问题诊断】
1. HashAggregate使用16批 → work_mem太小
2. Orders全表扫描 → 缺少索引
3. 过滤500万行 → 索引不生效
*/

-- 优化步骤1: 创建索引
CREATE INDEX ON orders(created_at, user_id) INCLUDE (amount);

-- 优化步骤2: 增大work_mem
SET work_mem = '256MB';

-- 再次执行
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    u.name,
    COUNT(*) as order_count,
    SUM(o.amount) as total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.created_at > '2024-01-01'
GROUP BY u.name;

/*
HashAggregate  (actual time=200...250)
  Group Key: u.name
  Batches: 1  ← ✅ 只需1批
  Memory Usage: 123456kB
  ->  Hash Join  (actual time=50...180)
        ->  Index Only Scan using orders_created_at_user_id_idx
              (actual time=0...100)  ← ✅ 使用索引
              Index Cond: (created_at > '2024-01-01')
        ->  Hash  (actual time=30...30)
              ->  Seq Scan on users u

【结果】
✅ 250ms (从8000ms降到250ms)
✅ 32倍提速!
✅ 无磁盘I/O
✅ 使用覆盖索引
*/
```

### 案例2: N+1查询问题

```sql
-- ❌ N+1问题 (应用代码)
users = query("SELECT * FROM users LIMIT 100")
for user in users:
    orders = query("SELECT * FROM orders WHERE user_id = ?", user.id)
    # 处理orders...

-- 总查询: 1 + 100 = 101次
-- 执行时间: 101 × 10ms = 1010ms

-- ✅ 解决方案1: JOIN
SELECT 
    u.*,
    o.order_id,
    o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.id IN (SELECT id FROM users LIMIT 100);

-- 总查询: 1次
-- 执行时间: ~50ms

-- ✅ 解决方案2: 批量查询
users = query("SELECT * FROM users LIMIT 100")
user_ids = [u.id for u in users]
orders = query("SELECT * FROM orders WHERE user_id = ANY(?)", user_ids)
# 在应用层组装

-- 总查询: 2次
-- 执行时间: ~30ms
```

### 案例3: 复杂统计查询

```sql
-- 需求: 计算每个用户的订单统计
-- 订单表: 1亿行
-- 用户表: 100万行

-- ❌ 慢查询 (60秒)
SELECT 
    u.id,
    u.name,
    COUNT(o.id) as order_count,
    SUM(o.amount) as total_amount,
    AVG(o.amount) as avg_amount,
    MAX(o.created_at) as last_order
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- 问题: JOIN + GROUP BY 1亿行

-- ✅ 优化方案1: 预聚合
CREATE TABLE user_order_stats AS
SELECT 
    user_id,
    COUNT(*) as order_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount,
    MAX(created_at) as last_order
FROM orders
GROUP BY user_id;

CREATE INDEX ON user_order_stats(user_id);

-- 查询变为:
SELECT 
    u.id,
    u.name,
    COALESCE(s.order_count, 0),
    COALESCE(s.total_amount, 0),
    s.avg_amount,
    s.last_order
FROM users u
LEFT JOIN user_order_stats s ON u.id = s.user_id;

-- 执行时间: ~500ms (120倍提速!)

-- ✅ 优化方案2: 物化视图 (自动更新)
CREATE MATERIALIZED VIEW user_order_stats_mv AS
SELECT 
    user_id,
    COUNT(*) as order_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount,
    MAX(created_at) as last_order
FROM orders
GROUP BY user_id;

CREATE INDEX ON user_order_stats_mv(user_id);

-- 定期刷新:
REFRESH MATERIALIZED VIEW CONCURRENTLY user_order_stats_mv;

-- 查询:
SELECT u.*, s.*
FROM users u
LEFT JOIN user_order_stats_mv s ON u.id = s.user_id;
```

---

## 监控和分析工具

### 1. pg_stat_statements

```sql
-- 启用 (postgresql.conf)
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all

-- 重启PostgreSQL

-- 创建扩展
CREATE EXTENSION pg_stat_statements;

-- 查询最慢的查询
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time,
    stddev_exec_time,
    rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- 查询调用最多的查询
SELECT 
    query,
    calls,
    mean_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- 重置统计
SELECT pg_stat_statements_reset();
```

### 2. auto_explain

```sql
-- 启用 (postgresql.conf)
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 1000  -- 记录>1秒的查询
auto_explain.log_analyze = true
auto_explain.log_buffers = true
auto_explain.log_timing = true
auto_explain.log_triggers = true

-- 重启PostgreSQL

-- 慢查询会自动记录EXPLAIN ANALYZE到日志
-- 查看: tail -f /var/log/postgresql/postgresql.log
```

### 3. pgBadger (日志分析)

```bash
# 安装
sudo apt-get install pgbadger  # Ubuntu/Debian
sudo yum install pgbadger      # CentOS/RHEL

# 配置PostgreSQL记录慢查询 (postgresql.conf)
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_min_duration_statement = 1000  # 记录>1秒的查询
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0

# 生成报告
pgbadger /var/lib/postgresql/data/log/postgresql-*.log -o /tmp/pgbadger.html

# 打开报告
firefox /tmp/pgbadger.html
```

---

## 性能调优Checklist

```
【数据库级别】
☐ 1. 统计信息最新?
   ANALYZE;

☐ 2. shared_buffers合理? (RAM的25%)
   SHOW shared_buffers;

☐ 3. work_mem足够? (64-256MB)
   SHOW work_mem;

☐ 4. effective_cache_size正确? (RAM的75%)
   SHOW effective_cache_size;

☐ 5. random_page_cost适合存储类型? (SSD=1.1)
   SHOW random_page_cost;

☐ 6. 启用并行查询?
   SHOW max_parallel_workers_per_gather;

【查询级别】
☐ 7. 使用EXPLAIN ANALYZE分析?
   EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

☐ 8. 有合适的索引?
   \d table_name

☐ 9. 索引被使用?
   检查EXPLAIN输出

☐ 10. 避免SELECT *?
   只选择需要的列

☐ 11. JOIN顺序合理?
   小表驱动大表

☐ 12. 使用EXISTS代替IN?
   子查询优化

☐ 13. 分页使用Keyset?
   避免大OFFSET

☐ 14. 批量操作?
   避免N+1查询

【监控】
☐ 15. 启用pg_stat_statements?
   CREATE EXTENSION pg_stat_statements;

☐ 16. 启用auto_explain?
   记录慢查询

☐ 17. 定期分析日志?
   使用pgBadger

☐ 18. 监控Cache命中率? (>99%)
   SELECT * FROM pg_stat_database;

☐ 19. 检查未使用的索引?
   SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;

☐ 20. 定期VACUUM?
   防止表膨胀
```

---

**文档版本**: PostgreSQL 17.6  
**创建日期**: 2025-10-17  
**重要程度**: ⭐⭐⭐⭐⭐

