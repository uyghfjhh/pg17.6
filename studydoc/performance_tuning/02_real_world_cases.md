# PostgreSQL 性能调优实战案例

> 真实生产环境的性能问题和解决方案

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 案例索引

1. [案例1: 慢查询优化 - 从30秒到0.1秒](#案例1-慢查询优化)
2. [案例2: 高并发下的锁争用](#案例2-高并发锁争用)
3. [案例3: 表膨胀导致性能下降](#案例3-表膨胀问题)
4. [案例4: 连接池耗尽](#案例4-连接池耗尽)
5. [案例5: 分区表性能优化](#案例5-分区表优化)
6. [案例6: 索引选择错误](#案例6-索引选择错误)
7. [案例7: JOIN顺序导致性能差](#案例7-join顺序问题)
8. [案例8: 内存配置不当](#案例8-内存配置问题)
9. [案例9: I/O瓶颈](#案例9-io瓶颈)
10. [案例10: 统计信息过期](#案例10-统计信息问题)

---

## 案例1: 慢查询优化

### 问题描述

某电商系统订单查询接口响应缓慢，平均响应时间30秒，严重影响用户体验。

```sql
-- 原始查询
SELECT o.*, u.username, u.email
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at > '2025-01-01'
  AND o.status = 'pending'
ORDER BY o.created_at DESC
LIMIT 20;
```

### 诊断过程

#### Step 1: 查看执行计划

```sql
EXPLAIN (ANALYZE, BUFFERS) 
SELECT o.*, u.username, u.email
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at > '2025-01-01'
  AND o.status = 'pending'
ORDER BY o.created_at DESC
LIMIT 20;

/*
输出分析:
Limit (cost=150000.00..150000.05 rows=20 width=100) (actual time=30245..30246 rows=20 loops=1)
  -> Sort (cost=150000.00..152500.00 rows=1000000) (actual time=30245..30245 rows=20 loops=1)
        Sort Key: o.created_at DESC
        Sort Method: top-N heapsort Memory: 28kB
        -> Hash Join (cost=50000.00..120000.00 rows=1000000) (actual time=1234..29876 rows=980000)
              Hash Cond: (o.user_id = u.id)
              -> Seq Scan on orders o (cost=0.00..50000.00 rows=1000000) (actual time=0.123..15234 rows=980000)
                    Filter: ((created_at > '2025-01-01') AND (status = 'pending'))
                    Rows Removed by Filter: 1020000
              -> Hash (cost=25000.00..25000.00 rows=1000000) (actual time=1234..1234 rows=1000000)
                    Buckets: 131072 Batches: 16 (originally 8) Memory Usage: 45678kB
                    -> Seq Scan on users u (cost=0.00..25000.00 rows=1000000)

Planning Time: 2.345 ms
Execution Time: 30246.789 ms
Buffers: shared hit=12345 read=67890
*/
```

**问题点识别**:
1. ❌ Seq Scan on orders - 顺序扫描200万行
2. ❌ Rows Removed by Filter: 1020000 - 过滤掉大量数据
3. ❌ Hash Batches: 16 - 内存不足，分批处理
4. ❌ Sort before Limit - 先排序再LIMIT效率低

#### Step 2: 查看索引情况

```sql
SELECT 
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE tablename = 'orders';

/*
现有索引:
- orders_pkey (id)
- idx_orders_user_id (user_id)  -- 存在但未使用
*/
```

### 优化方案

#### 优化1: 创建复合索引

```sql
-- 创建复合索引，覆盖WHERE和ORDER BY
CREATE INDEX idx_orders_status_created ON orders(status, created_at DESC);

-- 或者创建部分索引(更高效)
CREATE INDEX idx_orders_pending_created ON orders(created_at DESC) 
WHERE status = 'pending';
```

#### 优化2: 调整work_mem

```sql
-- 增加work_mem避免Hash分批
SET work_mem = '128MB';
```

#### 优化3: 查询重写

```sql
-- 优化后的查询
SELECT o.*, u.username, u.email
FROM (
    SELECT * 
    FROM orders
    WHERE created_at > '2025-01-01'
      AND status = 'pending'
    ORDER BY created_at DESC
    LIMIT 20
) o
JOIN users u ON o.user_id = u.id;
```

### 优化后效果

```sql
EXPLAIN (ANALYZE, BUFFERS)
-- 执行优化后的查询...

/*
输出:
Nested Loop (cost=0.56..145.67 rows=20 width=100) (actual time=0.234..0.567 rows=20 loops=1)
  -> Limit (cost=0.42..23.45 rows=20 width=80) (actual time=0.123..0.234 rows=20 loops=1)
        -> Index Scan using idx_orders_pending_created on orders
              Index Cond: ((status = 'pending') AND (created_at > '2025-01-01'))
  -> Index Scan using users_pkey on users u (cost=0.14..8.16 rows=1 width=40) (actual time=0.012..0.013 rows=1 loops=20)
        Index Cond: (id = o.user_id)

Planning Time: 1.234 ms
Execution Time: 0.678 ms
Buffers: shared hit=45
*/
```

### 效果对比

| 指标 | 优化前 | 优化后 | 提升 |
|-----|-------|-------|------|
| 执行时间 | 30246 ms | 0.678 ms | **44,600倍** |
| 缓冲区读取 | 80,235 | 45 | 99.94%减少 |
| 扫描行数 | 2,000,000 | 20 | 99.999%减少 |

### 经验总结

✅ **关键点**:
1. 复合索引设计要覆盖WHERE + ORDER BY
2. 部分索引可以显著减小索引大小
3. 先LIMIT再JOIN比先JOIN再LIMIT高效
4. work_mem要根据查询复杂度调整

---

## 案例2: 高并发锁争用

### 问题描述

秒杀系统在高并发场景下，库存扣减接口TPS急剧下降，大量请求超时。

```sql
-- 库存扣减逻辑
BEGIN;
SELECT stock FROM products WHERE id = 123 FOR UPDATE;
-- 应用层判断库存
UPDATE products SET stock = stock - 1 WHERE id = 123;
COMMIT;
```

### 诊断过程

#### Step 1: 查看锁等待

```sql
SELECT 
    pid,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';

/*
发现大量进程在等待Lock:
pid  | wait_event_type | wait_event | state  | query
-----|-----------------|------------|--------|-------
1234 | Lock           | tuple      | active | UPDATE products...
1235 | Lock           | tuple      | active | UPDATE products...
1236 | Lock           | tuple      | active | UPDATE products...
... (上百个)
*/
```

#### Step 2: 查看锁争用情况

```sql
SELECT 
    locktype,
    relation::regclass,
    mode,
    granted,
    count(*)
FROM pg_locks
WHERE NOT granted
GROUP BY locktype, relation, mode, granted;

/*
locktype | relation | mode              | granted | count
---------|----------|-------------------|---------|------
tuple    | products | ExclusiveLock     | f       | 234
*/
```

#### Step 3: 分析问题根源

```
问题分析:
1. FOR UPDATE导致行锁
2. 事务串行化执行
3. 高并发下锁等待链很长
4. UPDATE等待时间过长
```

### 优化方案

#### 优化1: 使用乐观锁

```sql
-- 方案A: 使用版本号
ALTER TABLE products ADD COLUMN version INTEGER DEFAULT 0;

-- 更新时带版本检查
UPDATE products 
SET stock = stock - 1, 
    version = version + 1
WHERE id = 123 
  AND stock > 0
  AND version = #{current_version};
  
-- 检查affected_rows，如果为0则重试
```

#### 优化2: 使用CAS (Compare-And-Swap)

```sql
-- 方案B: 直接CAS更新
UPDATE products 
SET stock = stock - 1
WHERE id = 123 
  AND stock > 0;

-- 返回affected_rows，判断是否成功
```

#### 优化3: 使用SKIP LOCKED

```sql
-- 方案C: 对于可以跳过的场景
BEGIN;
SELECT * FROM products 
WHERE id = 123 
  AND stock > 0
FOR UPDATE SKIP LOCKED;

-- 如果返回空，说明被锁定，直接返回"商品已售罄"
UPDATE products SET stock = stock - 1 WHERE id = 123;
COMMIT;
```

#### 优化4: 分段库存

```sql
-- 方案D: 将库存分段，减少单行锁争用
CREATE TABLE product_stock_segments (
    product_id INTEGER,
    segment_id INTEGER,
    stock INTEGER,
    PRIMARY KEY (product_id, segment_id)
);

-- 插入10个段
INSERT INTO product_stock_segments 
SELECT 123, generate_series(1, 10), 1000;

-- 扣减时随机选择段
UPDATE product_stock_segments
SET stock = stock - 1
WHERE product_id = 123
  AND segment_id = (random() * 9 + 1)::int
  AND stock > 0;
```

### 优化后效果

| 方案 | TPS | 平均响应时间 | 锁等待 |
|-----|-----|------------|--------|
| 原方案(FOR UPDATE) | 50 | 2000ms | 严重 |
| 乐观锁(版本号) | 3000 | 20ms | 无 |
| CAS更新 | 3500 | 15ms | 无 |
| SKIP LOCKED | 4000 | 12ms | 无 |
| 分段库存 | 8000 | 5ms | 极少 |

### 经验总结

✅ **关键点**:
1. 避免长事务持有行锁
2. 高并发场景优先使用乐观锁
3. SKIP LOCKED适用于可跳过的场景
4. 分段存储可以显著减少锁争用
5. 考虑使用Redis等缓存层承担高并发

---

## 案例3: 表膨胀问题

### 问题描述

某日志表插入正常，但查询性能持续下降，磁盘空间占用异常。

### 诊断过程

#### Step 1: 检查表大小

```sql
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) as index_size
FROM pg_stat_user_tables
WHERE tablename = 'access_logs'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

/*
输出:
tablename    | total_size | table_size | index_size
-------------|------------|------------|------------
access_logs  | 250 GB     | 200 GB     | 50 GB
*/
```

#### Step 2: 检查表膨胀

```sql
SELECT 
    schemaname,
    tablename,
    n_live_tup,
    n_dead_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE tablename = 'access_logs';

/*
输出:
n_live_tup | n_dead_tup | dead_ratio | last_vacuum | last_autovacuum
-----------|------------|------------|-------------|----------------
10,000,000 | 8,000,000  | 44.44%     | NULL        | 3 days ago

分析: 死元组比例高达44%，严重膨胀！
*/
```

#### Step 3: 检查VACUUM配置

```sql
SELECT name, setting, unit
FROM pg_settings
WHERE name LIKE '%vacuum%' OR name LIKE '%autovacuum%';

/*
关键配置:
autovacuum                        | on
autovacuum_max_workers            | 3
autovacuum_naptime                | 1min
autovacuum_vacuum_threshold       | 50
autovacuum_vacuum_scale_factor    | 0.2    -- 问题！太大了
*/
```

### 优化方案

#### 优化1: 调整autovacuum参数

```sql
-- 全局配置 (postgresql.conf)
autovacuum_vacuum_scale_factor = 0.05     -- 从0.2降低到0.05
autovacuum_vacuum_threshold = 1000        -- 从50提高到1000
autovacuum_max_workers = 6                -- 从3提高到6
autovacuum_vacuum_cost_limit = 2000       -- 从200提高到2000

-- 针对大表单独配置
ALTER TABLE access_logs SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_vacuum_threshold = 5000
);
```

#### 优化2: 立即执行VACUUM

```sql
-- 先执行VACUUM回收空间
VACUUM (VERBOSE, ANALYZE) access_logs;

/*
输出示例:
INFO: vacuuming "public.access_logs"
INFO: table "access_logs": found 8000000 dead rows in 156250 pages
INFO: "access_logs": removed 8000000 dead rows
INFO: table "access_logs": truncated to 78125 pages
DETAIL: CPU: user: 45.23 s, system: 12.34 s, elapsed: 234.56 s
*/
```

#### 优化3: 对于极度膨胀的表，使用VACUUM FULL

```sql
-- 注意: VACUUM FULL会锁表，需要停机窗口
VACUUM FULL access_logs;

-- 或者使用pg_repack (不锁表)
pg_repack -t access_logs -d mydb
```

#### 优化4: 分区表改造

```sql
-- 将历史数据表改为分区表
CREATE TABLE access_logs_new (
    id BIGSERIAL,
    user_id INTEGER,
    action TEXT,
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- 创建月度分区
CREATE TABLE access_logs_2025_01 PARTITION OF access_logs_new
FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE access_logs_2025_02 PARTITION OF access_logs_new
FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- 迁移数据后，可以直接DROP老分区
DROP TABLE access_logs_2025_01;  -- 瞬间完成！
```

#### 优化5: 设置合理的fillfactor

```sql
-- 对于频繁UPDATE的表，降低fillfactor
ALTER TABLE access_logs SET (fillfactor = 70);

-- 重建表使其生效
VACUUM FULL access_logs;
```

### 优化后效果

| 指标 | 优化前 | 优化后 | 改善 |
|-----|-------|-------|------|
| 表大小 | 200 GB | 110 GB | 减少45% |
| Dead Tuple比例 | 44% | 2% | 减少95% |
| 查询时间 | 5s | 0.8s | 提升6倍 |
| VACUUM频率 | 3天/次 | 2小时/次 | 36倍 |

### 经验总结

✅ **关键点**:
1. 监控表膨胀率，设置告警阈值(如>20%)
2. 大表要调整autovacuum参数
3. 频繁UPDATE的表要降低fillfactor
4. 历史数据表建议使用分区表
5. VACUUM FULL需要停机，优先考虑pg_repack

---

## 案例4: 连接池耗尽

### 问题描述

应用高峰期频繁报错 "FATAL: too many connections"，数据库连接数达到上限。

### 诊断过程

#### Step 1: 查看连接数配置

```sql
SELECT name, setting 
FROM pg_settings 
WHERE name = 'max_connections';

/*
max_connections = 100  -- 设置太小
*/
```

#### Step 2: 查看当前连接情况

```sql
SELECT 
    state,
    application_name,
    count(*) as connections
FROM pg_stat_activity
GROUP BY state, application_name
ORDER BY count(*) DESC;

/*
state              | application_name | connections
-------------------|------------------|------------
idle               | app_server       | 65
active             | app_server       | 30
idle in transaction| app_server       | 15  -- 问题！
*/
```

#### Step 3: 查看长时间空闲的连接

```sql
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    state_change,
    now() - state_change as idle_duration,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - state_change > interval '5 minutes'
ORDER BY state_change;

/*
发现大量"idle in transaction"连接
原因: 应用层事务未提交
*/
```

### 优化方案

#### 优化1: 配置连接超时

```sql
-- postgresql.conf
idle_in_transaction_session_timeout = 300000  -- 5分钟超时
statement_timeout = 60000                      -- 单条SQL 60秒超时
```

#### 优化2: 使用连接池(PgBouncer)

```bash
# 安装PgBouncer
yum install pgbouncer

# 配置 /etc/pgbouncer/pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction         # 事务级连接池
max_client_conn = 1000          # 最大客户端连接
default_pool_size = 25          # 每个数据库的连接池大小
reserve_pool_size = 5           # 预留连接
reserve_pool_timeout = 3        # 预留连接超时

# 启动
systemctl start pgbouncer
```

#### 优化3: 调整max_connections

```sql
-- postgresql.conf
max_connections = 200  -- 从100提高到200

-- 重启PostgreSQL
systemctl restart postgresql-17
```

#### 优化4: 应用层优化

```python
# 错误示例
def bad_example():
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("BEGIN")
    cursor.execute("SELECT * FROM users")
    # 忘记COMMIT，连接一直处于"idle in transaction"
    
# 正确示例
def good_example():
    try:
        with get_connection() as conn:
            with conn.cursor() as cursor:
                cursor.execute("SELECT * FROM users")
                conn.commit()  # 明确提交
    except Exception as e:
        conn.rollback()  # 异常回滚
    finally:
        conn.close()  # 确保关闭
```

### 架构改进

```
优化前:
[App Servers (100个)]
         ↓
  [PostgreSQL]
  max_connections=100
  问题: 连接数不够

优化后:
[App Servers (1000个)]
         ↓
    [PgBouncer]
    max_client_conn=1000
    default_pool_size=25
         ↓
  [PostgreSQL]
  max_connections=100
  效果: 1000个应用连接 -> 25个数据库连接
```

### 优化后效果

| 指标 | 优化前 | 优化后 | 改善 |
|-----|-------|-------|------|
| 最大支持连接数 | 100 | 1000 | 10倍 |
| 数据库实际连接数 | 100 | 30 | 减少70% |
| 连接建立时间 | 50ms | 2ms | 25倍 |
| "连接耗尽"错误 | 频繁 | 0 | 完全解决 |

### 经验总结

✅ **关键点**:
1. 使用PgBouncer等连接池是标配
2. 设置合理的超时参数
3. 应用层必须正确管理连接生命周期
4. 监控"idle in transaction"连接
5. max_connections不是越大越好，要考虑内存

---

## 案例5: 分区表优化

### 问题描述

订单表数据量达到5亿行，单表查询和维护都很慢，需要分区优化。

### 原始表结构

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id INTEGER,
    order_no VARCHAR(50),
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP
);

-- 数据量: 500,000,000 行
-- 表大小: 180 GB
```

### 查询性能问题

```sql
-- 查询最近一周订单
SELECT * FROM orders 
WHERE created_at > now() - interval '7 days';

-- 执行时间: 45秒
-- 问题: 扫描整个500M行的表
```

### 优化方案: 分区表改造

#### Step 1: 创建分区表

```sql
-- 创建新的分区表
CREATE TABLE orders_partitioned (
    id BIGSERIAL,
    user_id INTEGER,
    order_no VARCHAR(50),
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
```

#### Step 2: 创建分区

```sql
-- 创建月度分区
CREATE TABLE orders_2024_01 PARTITION OF orders_partitioned
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders_partitioned
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- ... 创建所有历史分区

CREATE TABLE orders_2025_10 PARTITION OF orders_partitioned
FOR VALUES FROM ('2025-10-01') TO ('2025-11-01');

-- 创建未来分区
CREATE TABLE orders_2025_11 PARTITION OF orders_partitioned
FOR VALUES FROM ('2025-11-01') TO ('2025-12-01');
```

#### Step 3: 自动化分区创建

```sql
-- 创建分区维护函数
CREATE OR REPLACE FUNCTION create_monthly_partitions()
RETURNS void AS $$
DECLARE
    start_date DATE;
    end_date DATE;
    partition_name TEXT;
BEGIN
    -- 创建未来3个月的分区
    FOR i IN 0..2 LOOP
        start_date := date_trunc('month', now() + (i || ' months')::interval);
        end_date := start_date + interval '1 month';
        partition_name := 'orders_' || to_char(start_date, 'YYYY_MM');
        
        -- 检查分区是否存在
        IF NOT EXISTS (
            SELECT 1 FROM pg_class WHERE relname = partition_name
        ) THEN
            EXECUTE format(
                'CREATE TABLE %I PARTITION OF orders_partitioned FOR VALUES FROM (%L) TO (%L)',
                partition_name, start_date, end_date
            );
            RAISE NOTICE 'Created partition: %', partition_name;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- 设置cron定时任务（需要pg_cron扩展）
CREATE EXTENSION pg_cron;
SELECT cron.schedule('create-partitions', '0 0 1 * *', 'SELECT create_monthly_partitions()');
```

#### Step 4: 迁移数据

```sql
-- 方案A: INSERT SELECT (简单但锁表)
INSERT INTO orders_partitioned SELECT * FROM orders;

-- 方案B: pg_dump + pg_restore (停机迁移)
pg_dump -t orders -Fc mydb > orders.dump
pg_restore -t orders_partitioned -d mydb orders.dump

-- 方案C: 分批迁移(推荐, 不停机)
DO $$
DECLARE
    min_id BIGINT;
    max_id BIGINT;
    batch_size INTEGER := 100000;
BEGIN
    SELECT min(id), max(id) INTO min_id, max_id FROM orders;
    
    FOR i IN min_id..max_id BY batch_size LOOP
        INSERT INTO orders_partitioned
        SELECT * FROM orders
        WHERE id >= i AND id < i + batch_size;
        
        COMMIT;
        PERFORM pg_sleep(0.1);  -- 降低负载
    END LOOP;
END $$;
```

#### Step 5: 重命名表

```sql
-- 切换表名
BEGIN;
ALTER TABLE orders RENAME TO orders_old;
ALTER TABLE orders_partitioned RENAME TO orders;
COMMIT;

-- 验证后删除老表
DROP TABLE orders_old;
```

### 优化后效果

```sql
-- 同样的查询
EXPLAIN ANALYZE
SELECT * FROM orders 
WHERE created_at > now() - interval '7 days';

/*
优化前:
Seq Scan on orders (cost=0.00..9500000.00 rows=500000000)
Execution Time: 45000 ms

优化后:
Append (cost=0.00..5000.00 rows=50000)
  -> Seq Scan on orders_2025_10 (cost=0.00..2500.00 rows=25000)
  -> Seq Scan on orders_2025_11 (cost=0.00..2500.00 rows=25000)
Execution Time: 120 ms

提升: 375倍！
*/
```

### 分区表最佳实践

```sql
-- [1] 在分区键上创建索引
CREATE INDEX ON orders_partitioned(user_id);
-- 自动在所有分区上创建

-- [2] 设置分区剪枝
SET enable_partition_pruning = on;
SET constraint_exclusion = partition;

-- [3] 老分区可以设置为只读
ALTER TABLE orders_2024_01 SET (fillfactor = 100);
VACUUM FULL orders_2024_01;

-- [4] 归档老数据
-- 直接DETACH分区，迁移到归档库
ALTER TABLE orders_partitioned DETACH PARTITION orders_2023_12;
```

### 经验总结

✅ **关键点**:
1. 选择合适的分区键（通常是时间字段）
2. 分区数量不宜过多（建议<1000个）
3. 自动化分区创建和维护
4. 查询必须包含分区键才能剪枝
5. 老分区可以归档或删除

---

## 案例6: 索引选择错误

### 问题描述

查询明明有索引，但执行计划却选择了错误的索引或顺序扫描。

### 案例场景

```sql
-- 表结构
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100),
    created_at TIMESTAMP,
    status VARCHAR(20)
);

-- 现有索引
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created ON users(created_at);
CREATE INDEX idx_users_status ON users(status);

-- 问题查询
SELECT * FROM users 
WHERE status = 'active' 
  AND created_at > '2025-01-01'
ORDER BY created_at DESC
LIMIT 100;
```

### 诊断: 执行计划分析

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users 
WHERE status = 'active' 
  AND created_at > '2025-01-01'
ORDER BY created_at DESC
LIMIT 100;

/*
问题执行计划:
Limit (cost=0.42..1234.56 rows=100)
  -> Index Scan Backward using idx_users_created on users
        Filter: (status = 'active')
        Rows Removed by Filter: 990000  -- 过滤掉99%的数据！
        Buffers: shared hit=25000

分析: 
- 选择了created_at索引
- 但status过滤性很强(99%的数据是inactive)
- 导致扫描大量不需要的数据
*/
```

### 优化方案

#### 优化1: 创建复合索引

```sql
-- 方案A: 联合索引（顺序很重要！）
CREATE INDEX idx_users_status_created ON users(status, created_at);

-- 为什么这个顺序?
-- status过滤性强，先过滤可以大幅减少数据量
-- created_at用于排序

DROP INDEX idx_users_status;  -- 删除冗余索引
DROP INDEX idx_users_created; -- 删除冗余索引（如果其他查询不需要）
```

#### 优化2: 部分索引

```sql
-- 方案B: 如果只查询active用户，使用部分索引
CREATE INDEX idx_users_active_created ON users(created_at)
WHERE status = 'active';

-- 优点: 索引更小，效率更高
-- 缺点: 只能用于status='active'的查询
```

#### 优化3: 调整统计信息

```sql
-- 确保统计信息是最新的
ANALYZE users;

-- 或者增加统计信息采样
ALTER TABLE users ALTER COLUMN status SET STATISTICS 1000;
ANALYZE users;
```

#### 优化4: 强制使用索引提示（最后手段）

```sql
-- PostgreSQL不直接支持索引提示，但可以禁用不想要的索引
SET enable_seqscan = off;
SET enable_indexscan = off;  -- 禁用特定类型

-- 或者临时删除索引
DROP INDEX idx_users_created;
-- 执行查询
-- 再重建
```

### 优化后效果

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users 
WHERE status = 'active' 
  AND created_at > '2025-01-01'
ORDER BY created_at DESC
LIMIT 100;

/*
优化后执行计划:
Limit (cost=0.42..45.67 rows=100)
  -> Index Scan Backward using idx_users_status_created on users
        Index Cond: ((status = 'active') AND (created_at > '2025-01-01'))
        Buffers: shared hit=15

对比:
- 优化前: Buffers hit=25000, Time=5678ms
- 优化后: Buffers hit=15, Time=1.2ms
- 提升: 4700倍！
*/
```

### 索引设计原则

```
多列索引顺序决策:

[1] 等值条件 > 范围条件
    WHERE a = 1 AND b > 10
    索引: (a, b) ✅  (b, a) ❌

[2] 过滤性强 > 过滤性弱
    WHERE status = 'active' AND created_at > ...
    status过滤99% → 索引: (status, created_at) ✅

[3] 排序字段放最后
    WHERE a = 1 ORDER BY b
    索引: (a, b) ✅

[4] 考虑查询频率
    高频查询优先满足

[5] 索引不是越多越好
    • 每个索引都会降低INSERT/UPDATE性能
    • 每个索引都占用存储空间
    • 维护成本增加
```

### 经验总结

✅ **关键点**:
1. 多列索引顺序非常重要
2. 使用EXPLAIN ANALYZE验证索引选择
3. 定期ANALYZE更新统计信息
4. 删除未使用的索引
5. 考虑使用部分索引优化特定查询

---

## 案例7-10: 简要案例

### 案例7: JOIN顺序问题

**问题**: 多表JOIN时顺序不当导致中间结果集过大

**解决**:
```sql
-- 优化前: 大表JOIN大表
SELECT * FROM big_table1 a
JOIN big_table2 b ON a.id = b.id
JOIN small_table c ON b.category = c.id
WHERE c.type = 'special';

-- 优化后: 先过滤小表，再JOIN大表
SELECT * FROM small_table c
JOIN big_table2 b ON b.category = c.id AND c.type = 'special'
JOIN big_table1 a ON a.id = b.id;
```

**效果**: 执行时间从120秒降低到3秒

---

### 案例8: 内存配置问题

**问题**: shared_buffers和work_mem配置不当

**诊断**:
```sql
-- 检查缓存命中率
SELECT 
    sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100 
    as cache_hit_ratio
FROM pg_statio_user_tables;
-- 结果: 75% (太低了！)
```

**优化**:
```ini
# postgresql.conf
shared_buffers = 8GB          # 原来1GB
effective_cache_size = 24GB   # 原来4GB
work_mem = 64MB              # 原来4MB
maintenance_work_mem = 2GB    # 原来256MB
```

**效果**: 缓存命中率提升到99%，查询性能提升3-5倍

---

### 案例9: I/O瓶颈

**问题**: WAL和数据文件在同一磁盘，I/O成为瓶颈

**诊断**:
```bash
iostat -x 1
# sda (数据+WAL): %util = 95%
```

**优化**:
```bash
# 1. WAL分离到独立SSD
initdb -D /data/pgdata -X /ssd/pg_wal

# 2. 调整I/O调度器
echo deadline > /sys/block/sda/queue/scheduler

# 3. 启用异步提交（根据业务容忍度）
ALTER DATABASE mydb SET synchronous_commit = off;
```

**效果**: TPS从2000提升到8000

---

### 案例10: 统计信息过期

**问题**: 大批量数据导入后执行计划异常

**诊断**:
```sql
-- 查看统计信息更新时间
SELECT 
    schemaname,
    tablename,
    last_analyze,
    last_autoanalyze,
    n_mod_since_analyze
FROM pg_stat_user_tables
WHERE tablename = 'large_table';

/*
last_analyze: 7 days ago
n_mod_since_analyze: 5000000  -- 修改了500万行但未分析
*/
```

**优化**:
```sql
-- 立即分析
ANALYZE large_table;

-- 或者增加autovacuum频率
ALTER TABLE large_table SET (
    autovacuum_analyze_scale_factor = 0.05,
    autovacuum_analyze_threshold = 1000
);
```

**效果**: 执行计划恢复正常，查询时间从30秒降到2秒

---

## 总结

### 性能优化通用方法论

```
1. 测量现状
   ├─ 收集性能指标
   ├─ 建立基线
   └─ 识别瓶颈

2. 定位问题
   ├─ EXPLAIN ANALYZE
   ├─ pg_stat_*视图
   └─ 系统监控

3. 制定方案
   ├─ 索引优化
   ├─ 查询重写
   ├─ 参数调整
   └─ 架构改进

4. 实施验证
   ├─ 测试环境验证
   ├─ 灰度发布
   ├─ 监控指标
   └─ 回滚方案

5. 持续改进
   ├─ 定期Review
   ├─ 性能测试
   └─ 知识沉淀
```

### 快速检查清单

- [ ] EXPLAIN ANALYZE检查执行计划
- [ ] pg_stat_statements找慢查询
- [ ] 检查表膨胀和索引膨胀
- [ ] 验证统计信息是否最新
- [ ] 检查锁等待情况
- [ ] 监控缓存命中率
- [ ] 查看连接数和连接状态
- [ ] 检查WAL生成速度
- [ ] 验证参数配置合理性
- [ ] 查看系统资源使用率

---

**下一步**: 学习 [03_monitoring_tools.md](03_monitoring_tools.md) 了解监控工具！

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

