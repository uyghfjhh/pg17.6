# PostgreSQL 调优最佳实践

> 经过生产验证的配置和优化建议

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. 参数配置最佳实践

### 1.1 内存相关参数

#### shared_buffers (共享缓冲区)

```ini
# postgresql.conf

# 推荐配置:
# • 小型服务器 (< 16GB RAM): 25% of RAM
# • 中型服务器 (16-64GB RAM): 25% of RAM  
# • 大型服务器 (> 64GB RAM): 8-16GB (不要太大)

# 示例配置:
shared_buffers = 8GB           # 32GB内存服务器
shared_buffers = 16GB          # 128GB内存服务器

# 原理:
# - 所有进程共享的内存区域
# - 用于缓存表和索引数据
# - 过大会浪费内存，过小会导致频繁I/O
```

#### effective_cache_size (有效缓存大小)

```ini
# 告诉优化器操作系统和PostgreSQL可用的缓存大小
# 不分配实际内存，仅用于查询规划

# 推荐配置:
# • 50-75% of total RAM
effective_cache_size = 24GB    # 32GB内存服务器
effective_cache_size = 96GB    # 128GB内存服务器

# 影响:
# - 影响索引扫描vs顺序扫描的选择
# - 过小会导致优化器低估缓存，错误选择计划
```

#### work_mem (工作内存)

```ini
# 每个查询操作(排序、哈希等)可用的内存

# 计算公式:
# work_mem = (RAM - shared_buffers) / (max_connections * 2-3)

# 示例:
work_mem = 64MB               # 低并发OLAP
work_mem = 16MB               # 中并发混合负载
work_mem = 4MB                # 高并发OLTP

# 注意:
# - 复杂查询可能使用多个work_mem
# - SELECT with multiple sorts可能使用 3-5 * work_mem
# - 可以在会话级别调整
SET work_mem = '256MB';
```

#### maintenance_work_mem (维护操作内存)

```ini
# VACUUM、CREATE INDEX、ALTER TABLE使用的内存

# 推荐配置:
maintenance_work_mem = 2GB      # 一般场景
maintenance_work_mem = 8GB      # 大表维护

# 用途:
# - 加速索引创建
# - 加速VACUUM
# - 加速外键检查
```

### 1.2 检查点和WAL参数

#### checkpoint相关

```ini
# 检查点超时时间
checkpoint_timeout = 15min      # 默认5min，可适当增加

# 最大WAL大小
max_wal_size = 4GB             # 默认1GB，根据写入量调整
min_wal_size = 1GB             # 最小WAL大小

# 检查点完成目标
checkpoint_completion_target = 0.9  # 0-1之间，建议0.9

# 原理:
# - checkpoint_timeout大 → 检查点频率低 → 恢复时间长
# - max_wal_size大 → 允许更多WAL → 平滑I/O写入
# - checkpoint_completion_target接近1 → 分散I/O压力
```

#### WAL配置

```ini
# WAL日志级别
wal_level = replica             # minimal/replica/logical

# WAL缓冲区
wal_buffers = 16MB             # -1表示自动(shared_buffers的1/32)

# WAL写入器延迟
wal_writer_delay = 200ms       # 10-10000ms

# 提交延迟 (组提交)
commit_delay = 10              # 微秒，0-100000
commit_siblings = 5            # 同时活跃事务数阈值

# 异步提交 (根据业务容忍度)
synchronous_commit = on        # on/remote_apply/remote_write/local/off
# off: 最快，但崩溃可能丢失最近的事务
# on: 最安全，但性能最低
```

### 1.3 查询规划参数

```ini
# 随机页成本
random_page_cost = 1.1         # SSD: 1.1-1.5
random_page_cost = 4.0         # HDD: 4.0 (默认)

# 顺序页成本
seq_page_cost = 1.0            # 通常保持1.0

# 并行查询
max_parallel_workers_per_gather = 4  # 每个Gather节点的worker数
max_parallel_workers = 8             # 全局最大worker数
parallel_setup_cost = 1000           # 并行启动成本
parallel_tuple_cost = 0.1            # 每个tuple传输成本

# CPU成本
cpu_tuple_cost = 0.01          # 处理每行的成本
cpu_index_tuple_cost = 0.005   # 处理每个索引条目的成本
cpu_operator_cost = 0.0025     # 处理每个操作符的成本

# 有效I/O并发
effective_io_concurrency = 200  # SSD: 200
effective_io_concurrency = 2    # HDD: 2
```

### 1.4 连接和资源参数

```ini
# 最大连接数
max_connections = 200          # 根据实际需求，配合连接池

# 超时设置
statement_timeout = 60000      # 单条SQL超时 (毫秒)
idle_in_transaction_session_timeout = 300000  # 5分钟

# 锁超时
lock_timeout = 30000           # 锁等待超时 (毫秒)
deadlock_timeout = 1000        # 死锁检测间隔 (毫秒)

# 后台worker
max_worker_processes = 8       # 后台worker进程数
```

### 1.5 Autovacuum参数

```ini
# 启用autovacuum
autovacuum = on

# worker数量
autovacuum_max_workers = 6     # 默认3，可根据负载增加

# 触发阈值
autovacuum_vacuum_scale_factor = 0.05   # 默认0.2，大表建议降低
autovacuum_vacuum_threshold = 1000      # 默认50

# 成本限制 (避免影响业务)
autovacuum_vacuum_cost_delay = 2ms      # 默认2ms
autovacuum_vacuum_cost_limit = 2000     # 默认200，可增加

# 针对大表单独配置
# ALTER TABLE large_table SET (
#     autovacuum_vacuum_scale_factor = 0.01,
#     autovacuum_vacuum_threshold = 5000
# );
```

---

## 2. SQL优化最佳实践

### 2.1 SELECT查询优化

#### 避免SELECT *

```sql
-- ❌ 不推荐
SELECT * FROM users WHERE id = 123;

-- ✅ 推荐: 只选择需要的列
SELECT id, username, email FROM users WHERE id = 123;

-- 原因:
-- 1. 减少网络传输
-- 2. 可能利用覆盖索引
-- 3. 提高可维护性
```

#### 使用EXISTS代替IN (大数据集)

```sql
-- ❌ 不推荐 (子查询返回大量数据)
SELECT * FROM orders 
WHERE user_id IN (SELECT id FROM users WHERE created_at > '2025-01-01');

-- ✅ 推荐: 使用EXISTS
SELECT * FROM orders o
WHERE EXISTS (
    SELECT 1 FROM users u 
    WHERE u.id = o.user_id AND u.created_at > '2025-01-01'
);

-- 或者使用JOIN
SELECT o.* FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.created_at > '2025-01-01';
```

#### 避免隐式类型转换

```sql
-- ❌ 不推荐: user_id是INTEGER，但使用字符串
SELECT * FROM orders WHERE user_id = '123';

-- ✅ 推荐: 使用正确的类型
SELECT * FROM orders WHERE user_id = 123;

-- 检查类型转换
EXPLAIN (VERBOSE) SELECT * FROM orders WHERE user_id = '123';
-- 如果看到 "Filter: ((user_id)::text = '123')" 就是有隐式转换
```

#### 分页优化

```sql
-- ❌ 不推荐: OFFSET在大偏移量时很慢
SELECT * FROM orders 
ORDER BY created_at DESC 
OFFSET 100000 LIMIT 20;

-- ✅ 推荐: 使用WHERE条件代替OFFSET
SELECT * FROM orders 
WHERE created_at < '2025-01-15 12:00:00'  -- 上一页的最后时间
ORDER BY created_at DESC 
LIMIT 20;

-- 或使用主键
SELECT * FROM orders 
WHERE id < 1000000  -- 上一页的最小ID
ORDER BY id DESC 
LIMIT 20;
```

### 2.2 索引设计最佳实践

#### 单列 vs 多列索引

```sql
-- 场景: 经常查询 WHERE a = ? AND b = ?

-- ❌ 不推荐: 创建两个单列索引
CREATE INDEX idx_a ON table(a);
CREATE INDEX idx_b ON table(b);

-- ✅ 推荐: 创建复合索引
CREATE INDEX idx_a_b ON table(a, b);

-- 复合索引顺序原则:
-- 1. 等值条件在前
-- 2. 过滤性强的在前
-- 3. 范围条件在后
-- 4. 排序字段在最后
```

#### 部分索引

```sql
-- 场景: 只查询status='active'的记录

-- ❌ 普通索引 (包含所有status)
CREATE INDEX idx_status ON orders(status);

-- ✅ 部分索引 (只索引active)
CREATE INDEX idx_status_active ON orders(status) 
WHERE status = 'active';

-- 优点:
-- • 索引更小
-- • 维护成本更低
-- • 查询更快
```

#### 表达式索引

```sql
-- 场景: 经常使用lower(email)查询

-- ❌ 普通索引无法使用
CREATE INDEX idx_email ON users(email);
SELECT * FROM users WHERE lower(email) = 'test@example.com';
-- 不会使用索引！

-- ✅ 表达式索引
CREATE INDEX idx_email_lower ON users(lower(email));
SELECT * FROM users WHERE lower(email) = 'test@example.com';
-- 会使用索引！
```

#### 覆盖索引 (Include)

```sql
-- 场景: SELECT id, name WHERE email = ?

-- ❌ 普通索引需要回表
CREATE INDEX idx_email ON users(email);

-- ✅ 覆盖索引避免回表
CREATE INDEX idx_email_include ON users(email) INCLUDE (id, name);

-- 或者使用复合索引
CREATE INDEX idx_email_id_name ON users(email, id, name);
```

### 2.3 JOIN优化

```sql
-- JOIN顺序优化原则:
-- 1. 小表在前，大表在后
-- 2. 过滤条件强的表在前
-- 3. 确保JOIN键上有索引

-- ❌ 不推荐: 大表先JOIN
SELECT * 
FROM huge_table1 h1
JOIN huge_table2 h2 ON h1.id = h2.id
JOIN small_table s ON h2.category = s.id
WHERE s.type = 'special';  -- 过滤性很强

-- ✅ 推荐: 先过滤小表
SELECT * 
FROM small_table s
JOIN huge_table2 h2 ON h2.category = s.id AND s.type = 'special'
JOIN huge_table1 h1 ON h1.id = h2.id;

-- 确保索引存在
CREATE INDEX idx_h2_category ON huge_table2(category);
CREATE INDEX idx_h1_id ON huge_table1(id);
CREATE INDEX idx_s_type ON small_table(type);
```

### 2.4 子查询优化

```sql
-- ❌ 不推荐: 相关子查询(每行执行一次)
SELECT 
    u.id,
    u.name,
    (SELECT count(*) FROM orders o WHERE o.user_id = u.id) as order_count
FROM users u;

-- ✅ 推荐: 使用JOIN
SELECT 
    u.id,
    u.name,
    COALESCE(o.order_count, 0) as order_count
FROM users u
LEFT JOIN (
    SELECT user_id, count(*) as order_count
    FROM orders
    GROUP BY user_id
) o ON u.id = o.user_id;

-- 或使用窗口函数
SELECT 
    u.id,
    u.name,
    count(o.id) OVER (PARTITION BY u.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

---

## 3. 表设计最佳实践

### 3.1 数据类型选择

```sql
-- [1] 使用合适大小的整数类型
-- ❌ 不推荐
CREATE TABLE users (
    id BIGINT,          -- 最大值9万亿，通常用不到
    age INTEGER         -- 最大值21亿，年龄用不了这么大
);

-- ✅ 推荐
CREATE TABLE users (
    id INTEGER,         -- 21亿够用了
    age SMALLINT        -- 年龄用SMALLINT (最大32767)
);

-- [2] 字符串类型选择
-- VARCHAR(n) vs TEXT
-- • VARCHAR(n): 有长度限制，适合已知最大长度
-- • TEXT: 无长度限制，适合长度不确定的文本
-- • CHAR(n): 定长，空格填充，很少使用

CREATE TABLE products (
    sku VARCHAR(50),      -- SKU通常不超过50字符
    description TEXT,     -- 描述长度不确定
    status VARCHAR(20)    -- 状态值固定几种
);

-- [3] 时间类型选择
-- TIMESTAMP vs TIMESTAMPTZ
-- • TIMESTAMPTZ: 带时区，推荐使用
-- • TIMESTAMP: 不带时区，容易出问题

CREATE TABLE orders (
    created_at TIMESTAMPTZ DEFAULT now(),  -- ✅ 推荐
    updated_at TIMESTAMPTZ
);

-- [4] 布尔类型
-- 不要用INTEGER或VARCHAR存储布尔值
CREATE TABLE users (
    is_active BOOLEAN DEFAULT true,  -- ✅ 正确
    is_verified BOOLEAN DEFAULT false
);
```

### 3.2 主键设计

```sql
-- [1] 使用SERIAL或BIGSERIAL
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,       -- 小表
    id BIGSERIAL PRIMARY KEY     -- 大表或高增长表
);

-- [2] 复合主键 (必要时)
CREATE TABLE order_items (
    order_id INTEGER,
    product_id INTEGER,
    quantity INTEGER,
    PRIMARY KEY (order_id, product_id)
);

-- [3] UUID主键 (分布式系统)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE TABLE distributed_table (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    data TEXT
);

-- 注意: UUID主键的缺点
-- • 占用空间大 (16字节 vs 4/8字节)
-- • 索引性能较差 (无序)
-- • 考虑使用UUID v7 (有序UUID)
```

### 3.3 分区表设计

```sql
-- [1] 范围分区 (时间序列数据)
CREATE TABLE measurements (
    id BIGSERIAL,
    sensor_id INTEGER,
    value NUMERIC,
    measured_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, measured_at)
) PARTITION BY RANGE (measured_at);

-- 创建月度分区
CREATE TABLE measurements_2025_01 PARTITION OF measurements
FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- [2] 列表分区 (状态、地区等)
CREATE TABLE orders (
    id BIGSERIAL,
    user_id INTEGER,
    region VARCHAR(10) NOT NULL,
    created_at TIMESTAMPTZ,
    PRIMARY KEY (id, region)
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('US');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('EU');
CREATE TABLE orders_asia PARTITION OF orders FOR VALUES IN ('ASIA');

-- [3] 哈希分区 (均匀分布)
CREATE TABLE user_actions (
    id BIGSERIAL,
    user_id INTEGER NOT NULL,
    action TEXT,
    created_at TIMESTAMPTZ,
    PRIMARY KEY (id, user_id)
) PARTITION BY HASH (user_id);

CREATE TABLE user_actions_p0 PARTITION OF user_actions
FOR VALUES WITH (MODULUS 4, REMAINDER 0);
-- 创建4个分区...
```

### 3.4 约束和默认值

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    
    -- NOT NULL约束 (必填字段)
    user_id INTEGER NOT NULL,
    order_no VARCHAR(50) NOT NULL,
    
    -- 默认值
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT now(),
    
    -- CHECK约束
    total_amount NUMERIC(10,2) CHECK (total_amount >= 0),
    quantity INTEGER CHECK (quantity > 0),
    
    -- UNIQUE约束
    UNIQUE (order_no),
    
    -- 外键约束 (谨慎使用，影响性能)
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- 部分唯一约束
CREATE UNIQUE INDEX idx_email_unique ON users(email) 
WHERE deleted_at IS NULL;
```

---

## 4. 架构设计最佳实践

### 4.1 读写分离

```
架构:
[应用层]
   ↓ 写操作
[主库 (Master)]
   ↓ 流复制
[从库1 (Standby)]  [从库2 (Standby)]
   ↑                     ↑
   └─────── 读操作 ──────┘

实现:
1. 应用层路由 (推荐)
   - 写操作 → 主库
   - 读操作 → 从库(负载均衡)

2. 连接池路由 (PgBouncer)
   
3. 中间件路由 (Pgpool-II)
```

```python
# 应用层实现示例
class DatabaseRouter:
    def db_for_read(self, model):
        return random.choice(['standby1', 'standby2'])
    
    def db_for_write(self, model):
        return 'primary'
```

### 4.2 连接池

```ini
# PgBouncer配置
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction        # session/transaction/statement
max_client_conn = 1000         # 最大客户端连接
default_pool_size = 25         # 每DB的连接池大小
reserve_pool_size = 5          # 预留池
reserve_pool_timeout = 3
server_idle_timeout = 600      # 服务器连接空闲超时

# 效果:
# 1000个应用连接 → 25个数据库连接
# 减少数据库连接压力
```

### 4.3 分库分表策略

```sql
-- 垂直分库: 按业务模块
database_user     -- 用户相关
database_order    -- 订单相关  
database_product  -- 商品相关

-- 水平分表: 按分片键
-- 用户表按user_id哈希分4个表
users_0, users_1, users_2, users_3

-- 分片规则
shard_id = user_id % 4

-- 注意:
-- • 跨分片查询很复杂
-- • 分布式事务需要特殊处理
-- • 优先考虑PostgreSQL原生分区表
```

### 4.4 缓存策略

```
[应用层]
    ↓
[Redis缓存]  ← 热数据
    ↓ Cache Miss
[PostgreSQL]

缓存场景:
1. 查询缓存
   - 热点数据
   - 计算结果
   
2. 会话缓存
   - 用户登录状态
   - 购物车数据
   
3. 计数器缓存
   - 点赞数、浏览数
   - 定期同步到PostgreSQL

注意:
• 缓存一致性问题
• 缓存穿透/击穿/雪崩
• 设置合理的TTL
```

---

## 5. 运维最佳实践

### 5.1 备份策略

```bash
# [1] 每日全量备份
pg_basebackup -D /backup/full_$(date +%Y%m%d) -Ft -z -P

# [2] WAL归档 (持续备份)
# postgresql.conf
archive_mode = on
archive_command = 'cp %p /backup/wal_archive/%f'

# [3] 逻辑备份 (重要表)
pg_dump -Fc -t important_table mydb > important_table_$(date +%Y%m%d).dump

# [4] 备份验证
# 定期恢复测试，确保备份可用

# [5] 备份保留策略
# • 全量备份: 保留30天
# • WAL归档: 保留30天
# • 月度备份: 保留12个月
```

### 5.2 巡检清单

```sql
-- 每日巡检SQL
-- [1] 数据库大小增长
SELECT 
    datname,
    pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- [2] 表膨胀检查
SELECT 
    schemaname || '.' || tablename as table_name,
    n_dead_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC
LIMIT 10;

-- [3] 长事务检查
SELECT 
    pid,
    now() - xact_start as duration,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND now() - xact_start > interval '10 minutes'
ORDER BY xact_start;

-- [4] 锁等待检查
SELECT count(*) FROM pg_stat_activity WHERE wait_event_type = 'Lock';

-- [5] 复制延迟检查
SELECT 
    application_name,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) / 1024 / 1024 as lag_mb
FROM pg_stat_replication;
```

### 5.3 升级策略

```bash
# PostgreSQL大版本升级

# 方案1: pg_upgrade (最快，需停机)
pg_upgrade \
  -b /usr/pgsql-16/bin \
  -B /usr/pgsql-17/bin \
  -d /var/lib/pgsql/16/data \
  -D /var/lib/pgsql/17/data \
  --check  # 先检查

# 方案2: 逻辑复制 (在线升级)
# 1. 搭建新版本实例
# 2. 配置逻辑复制
# 3. 等待同步完成
# 4. 切换应用连接

# 方案3: pg_dump/pg_restore (最慢但最安全)
pg_dumpall > full_backup.sql
# 安装新版本
psql -f full_backup.sql

# 升级前:
# • 详细阅读Release Notes
# • 测试环境先升级
# • 准备回滚方案
# • 通知业务方停机窗口
```

### 5.4 容量规划

```
规划维度:

[1] 存储容量
    • 当前使用: 100GB
    • 月增长率: 10GB
    • 规划周期: 12个月
    • 需求: 100 + 10 * 12 = 220GB
    • 冗余系数: 1.5
    • 最终: 330GB

[2] 连接数
    • 当前峰值: 150
    • 业务增长: 50%
    • 需求: 150 * 1.5 = 225
    • max_connections = 300 (留余量)

[3] IOPS
    • 当前峰值: 5000 IOPS
    • 业务增长: 50%
    • 需求: 7500 IOPS
    • 选择: SSD (20000+ IOPS)

[4] 内存
    • shared_buffers: 8GB
    • work_mem: 4MB * 300 = 1.2GB
    • OS Cache: 16GB
    • 应用: 8GB
    • 总需求: 33GB → 选择64GB服务器
```

---

## 6. 安全最佳实践

### 6.1 访问控制

```ini
# pg_hba.conf

# 本地连接使用peer认证
local   all   postgres   peer

# 应用连接使用md5/scram-sha-256
host    mydb  myapp  10.0.0.0/8  scram-sha-256

# 禁止所有其他连接
host    all   all    0.0.0.0/0   reject
```

```sql
-- 权限管理
-- [1] 创建只读用户
CREATE USER readonly_user WITH PASSWORD 'strong_password';
GRANT CONNECT ON DATABASE mydb TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
    GRANT SELECT ON TABLES TO readonly_user;

-- [2] 创建应用用户 (最小权限)
CREATE USER app_user WITH PASSWORD 'strong_password';
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- [3] 撤销public schema的public权限
REVOKE ALL ON SCHEMA public FROM PUBLIC;
```

### 6.2 SSL连接

```ini
# postgresql.conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'

# 强制SSL
# pg_hba.conf
hostssl  mydb  myapp  10.0.0.0/8  scram-sha-256
```

### 6.3 审计日志

```ini
# postgresql.conf
log_statement = 'ddl'            # none/ddl/mod/all
log_connections = on
log_disconnections = on
log_duration = on
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '

# 敏感操作审计
log_statement = 'all'            # 记录所有SQL (性能影响大)

# 使用pgAudit扩展 (更精细的审计)
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'
pgaudit.log_relation = on
```

---

## 总结

### 配置优化检查清单

- [ ] shared_buffers = 25% RAM
- [ ] effective_cache_size = 50-75% RAM  
- [ ] work_mem 根据并发调整
- [ ] maintenance_work_mem = 1-2GB
- [ ] max_wal_size = 2-4GB
- [ ] checkpoint_completion_target = 0.9
- [ ] random_page_cost = 1.1 (SSD)
- [ ] autovacuum参数优化
- [ ] 启用pg_stat_statements
- [ ] 配置连接池

### SQL优化检查清单

- [ ] 避免SELECT *
- [ ] 使用EXPLAIN分析
- [ ] 创建合适的索引
- [ ] 避免隐式类型转换
- [ ] 优化分页查询
- [ ] JOIN使用索引
- [ ] 避免相关子查询

### 架构优化检查清单

- [ ] 实现读写分离
- [ ] 部署连接池
- [ ] 配置缓存层
- [ ] 考虑分区表
- [ ] 定期VACUUM
- [ ] 建立监控体系
- [ ] 制定备份策略

---

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

