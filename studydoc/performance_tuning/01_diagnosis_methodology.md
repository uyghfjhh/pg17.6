# PostgreSQL 性能诊断方法论

> 系统化的性能问题诊断方法和工具

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. 性能问题分类

### 1.1 问题类型

```
性能问题分类:

[1] 慢查询问题
    • 单条SQL执行慢
    • 执行计划不优
    • 缺少索引

[2] 高并发问题
    • 锁争用严重
    • 连接数耗尽
    • CPU/内存瓶颈

[3] 存储问题
    • I/O瓶颈
    • 表膨胀严重
    • WAL写入慢

[4] 资源问题
    • 内存不足
    • CPU使用率高
    • 磁盘空间不足

[5] 架构问题
    • 数据模型不合理
    • 分区策略不当
    • 复制延迟高
```

---

## 2. 诊断工具箱

### 2.1 系统级监控

```bash
# CPU使用情况
top -c
htop

# 内存使用
free -h
vmstat 1

# I/O统计
iostat -x 1
iotop

# 网络统计
netstat -antp
ss -antp

# 磁盘空间
df -h
du -sh /var/lib/pgsql/*/data

# 系统调用跟踪
strace -c -p <postgres_pid>
```

### 2.2 PostgreSQL内置视图

```sql
-- [1] 查询统计 (需要pg_stat_statements)
SELECT 
    queryid,
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    stddev_exec_time,
    min_exec_time,
    max_exec_time,
    rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- [2] 活跃查询
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    query_start,
    state_change,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- [3] 长事务
SELECT 
    pid,
    usename,
    application_name,
    xact_start,
    now() - xact_start as duration,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start
LIMIT 10;

-- [4] 锁等待
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement,
    blocked_activity.application_name AS blocked_application
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- [5] 表统计
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- [6] 索引统计
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- [7] 缓冲池命中率
SELECT 
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    round(sum(heap_blks_hit) * 100.0 / 
          nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables;

-- [8] 表膨胀
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio
FROM pg_stat_user_tables
WHERE n_live_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 20;

-- [9] WAL生成速度
SELECT 
    pg_current_wal_lsn(),
    pg_walfile_name(pg_current_wal_lsn());

-- 间隔一段时间后再次查询，计算差值
```

### 2.3 日志分析

```bash
# 慢查询日志配置
# postgresql.conf
log_min_duration_statement = 1000  # 记录超过1秒的查询
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0

# 使用pgBadger分析日志
pgbadger /var/log/postgresql/postgresql-*.log -o report.html
```

---

## 3. 性能指标体系

### 3.1 关键性能指标(KPI)

```
[1] 查询性能
    • QPS (Queries Per Second) - 每秒查询数
    • TPS (Transactions Per Second) - 每秒事务数
    • 平均响应时间
    • P95/P99响应时间

[2] 资源利用率
    • CPU使用率 (< 80%)
    • 内存使用率 (< 80%)
    • 磁盘I/O使用率 (< 80%)
    • 网络带宽使用率

[3] 缓存命中率
    • Buffer Pool命中率 (> 99%)
    • Index命中率 (> 95%)

[4] 锁和等待
    • 锁等待时间
    • 死锁次数
    • 活跃连接数
    • 等待连接数

[5] 数据质量
    • 表膨胀率 (< 20%)
    • 索引膨胀率 (< 20%)
    • Dead Tuple比例 (< 10%)

[6] 可用性
    • 系统可用性 (> 99.9%)
    • 复制延迟 (< 1s)
    • 备份成功率 (100%)
```

### 3.2 监控SQL脚本

```sql
-- 综合性能报告
CREATE OR REPLACE VIEW performance_summary AS
SELECT 
    'Database Size' as metric,
    pg_size_pretty(pg_database_size(current_database())) as value
UNION ALL
SELECT 
    'Active Connections',
    count(*)::text
FROM pg_stat_activity
WHERE state = 'active'
UNION ALL
SELECT 
    'Cache Hit Ratio',
    round(sum(heap_blks_hit) * 100.0 / 
          nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2)::text || '%'
FROM pg_statio_user_tables
UNION ALL
SELECT 
    'TPS (commit+rollback)',
    (xact_commit + xact_rollback)::text
FROM pg_stat_database
WHERE datname = current_database()
UNION ALL
SELECT 
    'Deadlocks',
    deadlocks::text
FROM pg_stat_database
WHERE datname = current_database();

SELECT * FROM performance_summary;
```

---

## 4. 问题定位流程

### 4.1 慢查询诊断流程

```
[步骤1] 识别慢查询
    ↓
    • 查看pg_stat_statements
    • 检查日志中的慢查询
    • 用户反馈
    
[步骤2] 获取执行计划
    ↓
    • EXPLAIN ANALYZE
    • 查看实际执行时间
    • 检查行数估算
    
[步骤3] 分析瓶颈
    ↓
    • Seq Scan (顺序扫描) → 缺少索引?
    • Nested Loop → 数据量不匹配?
    • Sort → work_mem不足?
    • Hash → 内存不足?
    
[步骤4] 制定优化方案
    ↓
    • 创建索引
    • 调整参数
    • 重写查询
    • 分区表
    
[步骤5] 验证效果
    ↓
    • 再次EXPLAIN ANALYZE
    • 对比执行时间
    • 监控生产环境
```

### 4.2 EXPLAIN输出分析

```sql
-- 示例查询
EXPLAIN (ANALYZE, BUFFERS, TIMING, VERBOSE)
SELECT u.*, o.* 
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2025-01-01'
  AND o.status = 'completed';

/*
关键指标解读:

1. cost=0.00..1000.00
   • 第一个数字: 启动成本
   • 第二个数字: 总成本

2. rows=1000
   • 估算返回行数
   • 与actual rows对比

3. actual time=0.123..45.678 rows=950 loops=1
   • actual time: 实际执行时间
   • rows: 实际返回行数
   • loops: 执行次数

4. Buffers: shared hit=1000 read=100
   • shared hit: 缓存命中
   • read: 磁盘读取

5. Planning Time: 0.123 ms
   Execution Time: 45.678 ms
   • 计划时间 vs 执行时间
*/
```

### 4.3 常见问题模式

```sql
-- [问题1] 顺序扫描大表
/*
Seq Scan on large_table (cost=0.00..100000.00 rows=1000000 width=50)
  Filter: (status = 'active')
  Rows Removed by Filter: 999000

解决方案: 创建索引
*/
CREATE INDEX idx_large_table_status ON large_table(status);

-- [问题2] 索引未被使用
/*
Seq Scan on users (cost=0.00..1000.00 rows=100 width=50)
  Filter: (lower(email) = 'test@example.com')

原因: 使用了函数，普通索引无效
解决方案: 函数索引
*/
CREATE INDEX idx_users_email_lower ON users(lower(email));

-- [问题3] Nested Loop性能差
/*
Nested Loop (cost=0.00..100000.00 rows=100)
  -> Seq Scan on large_table (rows=1000000)
  -> Index Scan on small_table (rows=1)

原因: 外表太大
解决方案: 添加WHERE条件缩小外表
*/

-- [问题4] Sort内存不足
/*
Sort (cost=10000.00..11000.00 rows=100000 width=50)
  Sort Key: created_at
  Sort Method: external merge Disk: 12345kB

原因: work_mem不足，使用磁盘排序
解决方案: 增加work_mem
*/
SET work_mem = '256MB';

-- [问题5] Hash Join内存不足
/*
Hash Join (cost=10000.00..50000.00 rows=100000 width=100)
  Hash Cond: (o.user_id = u.id)
  -> Seq Scan on orders o
  -> Hash
        Buckets: 1024 Batches: 10

原因: Batches > 1 说明内存不足
解决方案: 增加work_mem
*/
```

---

## 5. 性能基线建立

### 5.1 收集基线数据

```sql
-- 创建性能基线表
CREATE TABLE performance_baseline (
    collected_at timestamp DEFAULT now(),
    metric_name text,
    metric_value numeric,
    metric_unit text
);

-- 收集基线脚本
INSERT INTO performance_baseline (metric_name, metric_value, metric_unit)
SELECT 'tps', xact_commit + xact_rollback, 'transactions/sec'
FROM pg_stat_database WHERE datname = current_database()
UNION ALL
SELECT 'connections', count(*), 'count'
FROM pg_stat_activity
UNION ALL
SELECT 'cache_hit_ratio', 
       round(sum(heap_blks_hit) * 100.0 / 
             nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2),
       'percent'
FROM pg_statio_user_tables
UNION ALL
SELECT 'avg_query_time',
       avg(mean_exec_time),
       'ms'
FROM pg_stat_statements;

-- 定期执行(cron)，建立历史趋势
```

### 5.2 性能测试

```bash
# pgbench初始化
pgbench -i -s 100 testdb

# 只读测试
pgbench -c 10 -j 4 -T 60 -S testdb

# 读写测试
pgbench -c 10 -j 4 -T 60 testdb

# 自定义测试
pgbench -c 10 -j 4 -T 60 -f custom_test.sql testdb
```

---

## 6. 诊断工具脚本

### 6.1 一键诊断脚本

```sql
-- pg_diagnose.sql
\echo '=== PostgreSQL Performance Diagnosis ==='
\echo ''

\echo '[1] Database Overview'
SELECT 
    current_database() as database,
    pg_size_pretty(pg_database_size(current_database())) as size,
    (SELECT setting FROM pg_settings WHERE name = 'version') as version;
\echo ''

\echo '[2] Connection Status'
SELECT 
    state,
    count(*) as connections
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
\echo ''

\echo '[3] Top 10 Slowest Queries'
SELECT 
    substring(query, 1, 60) as query_preview,
    calls,
    round(mean_exec_time::numeric, 2) as avg_ms,
    round(total_exec_time::numeric, 2) as total_ms
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
\echo ''

\echo '[4] Cache Hit Ratio (should > 99%)'
SELECT 
    round(sum(heap_blks_hit) * 100.0 / 
          nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables;
\echo ''

\echo '[5] Table Bloat Top 10'
SELECT 
    schemaname || '.' || tablename as table_name,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    n_dead_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio
FROM pg_stat_user_tables
WHERE n_live_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 10;
\echo ''

\echo '[6] Unused Indexes'
SELECT 
    schemaname || '.' || indexname as index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 10;
\echo ''

\echo '[7] Lock Waits'
SELECT 
    count(*) as lock_waits
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
\echo ''

\echo '=== Diagnosis Complete ==='
```

运行诊断:
```bash
psql -d mydb -f pg_diagnose.sql
```

---

## 7. 告警阈值设置

### 7.1 推荐阈值

```yaml
# 告警配置示例
alerts:
  # 连接数
  - name: high_connections
    threshold: max_connections * 0.8
    severity: warning
    
  - name: critical_connections
    threshold: max_connections * 0.95
    severity: critical

  # 缓存命中率
  - name: low_cache_hit
    threshold: < 95%
    severity: warning

  # 查询响应时间
  - name: slow_query
    threshold: > 1000ms
    severity: warning
    
  - name: very_slow_query
    threshold: > 5000ms
    severity: critical

  # 表膨胀
  - name: table_bloat
    threshold: dead_ratio > 20%
    severity: warning

  # 复制延迟
  - name: replication_lag
    threshold: > 10s
    severity: critical

  # 磁盘空间
  - name: disk_space
    threshold: > 80%
    severity: warning
    
  - name: disk_space_critical
    threshold: > 90%
    severity: critical
```

---

## 总结

### 诊断方法论核心

1. **系统化** - 使用系统化的方法，不要盲目尝试
2. **数据驱动** - 基于监控数据和指标做决策
3. **基线对比** - 建立性能基线，对比历史数据
4. **逐步排查** - 从宏观到微观，逐步缩小范围
5. **验证效果** - 每次优化都要验证效果

### 关键工具

- ✅ pg_stat_statements - 查询统计必备
- ✅ EXPLAIN ANALYZE - 执行计划分析
- ✅ pgBadger - 日志分析
- ✅ 系统监控 - top/iostat/vmstat
- ✅ 自定义监控脚本

### 下一步

阅读 [02_real_world_cases.md](02_real_world_cases.md) 学习实战案例！

---

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

