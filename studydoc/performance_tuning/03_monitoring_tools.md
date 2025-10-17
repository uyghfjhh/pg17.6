# PostgreSQL 监控和工具

> 全面的性能监控工具和方法

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. PostgreSQL内置监控

### 1.1 系统视图

#### pg_stat_activity - 活动连接监控

```sql
-- 实时查看所有活动
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
    now() - query_start as query_duration,
    left(query, 100) as query_preview
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- 长时间运行的查询
SELECT 
    pid,
    now() - query_start as duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '5 minutes'
ORDER BY duration DESC;

-- Kill长查询
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE pid = 12345;
```

#### pg_stat_statements - 查询统计 (必装扩展)

```sql
-- 安装扩展
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- postgresql.conf 配置
-- shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.track = all
-- pg_stat_statements.max = 10000

-- Top 10 慢查询
SELECT 
    queryid,
    left(query, 80) as query_preview,
    calls,
    round(total_exec_time::numeric, 2) as total_time_ms,
    round(mean_exec_time::numeric, 2) as avg_time_ms,
    round(stddev_exec_time::numeric, 2) as stddev_ms,
    round((100.0 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) as pct_total
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Top 10 最频繁查询
SELECT 
    queryid,
    calls,
    round(total_exec_time::numeric, 2) as total_time_ms,
    round(mean_exec_time::numeric, 2) as avg_time_ms,
    left(query, 80) as query_preview
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- 重置统计
SELECT pg_stat_statements_reset();
```

#### pg_stat_user_tables - 表统计

```sql
-- 表访问统计
SELECT 
    schemaname,
    tablename,
    seq_scan,                    -- 顺序扫描次数
    seq_tup_read,                -- 顺序扫描读取行数
    idx_scan,                    -- 索引扫描次数
    idx_tup_fetch,               -- 索引扫描获取行数
    n_tup_ins,                   -- 插入行数
    n_tup_upd,                   -- 更新行数
    n_tup_del,                   -- 删除行数
    n_live_tup,                  -- 活跃元组数
    n_dead_tup,                  -- 死元组数
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;

-- 需要VACUUM的表
SELECT 
    schemaname || '.' || tablename as table_name,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
  AND round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) > 10
ORDER BY n_dead_tup DESC;
```

#### pg_stat_user_indexes - 索引统计

```sql
-- 索引使用情况
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,                     -- 索引扫描次数
    idx_tup_read,                 -- 索引扫描读取行数
    idx_tup_fetch,                -- 索引扫描获取行数
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- 未使用的索引 (考虑删除)
SELECT 
    schemaname || '.' || tablename as table_name,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'  -- 排除主键
ORDER BY pg_relation_size(indexrelid) DESC;

-- 重复索引检测
SELECT 
    indrelid::regclass as table_name,
    array_agg(indexrelid::regclass) as indexes,
    array_agg(pg_get_indexdef(indexrelid)) as definitions
FROM pg_index
GROUP BY indrelid, indkey
HAVING count(*) > 1;
```

#### pg_statio_* - I/O统计

```sql
-- 表I/O统计
SELECT 
    schemaname,
    tablename,
    heap_blks_read,               -- 磁盘块读取
    heap_blks_hit,                -- 缓存命中
    round(heap_blks_hit * 100.0 / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) as cache_hit_ratio,
    idx_blks_read,
    idx_blks_hit
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC;

-- 缓存命中率 (应该 > 99%)
SELECT 
    'Table Cache' as type,
    sum(heap_blks_hit) as hits,
    sum(heap_blks_read) as reads,
    round(sum(heap_blks_hit) * 100.0 / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) as hit_ratio
FROM pg_statio_user_tables
UNION ALL
SELECT 
    'Index Cache',
    sum(idx_blks_hit),
    sum(idx_blks_read),
    round(sum(idx_blks_hit) * 100.0 / NULLIF(sum(idx_blks_hit) + sum(idx_blks_read), 0), 2)
FROM pg_statio_user_tables;
```

### 1.2 锁监控

```sql
-- 当前锁等待
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_query,
    blocking_activity.query AS blocking_query,
    blocked_activity.application_name
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- 死锁统计
SELECT 
    datname,
    deadlocks,
    conflicts,
    temp_files,
    temp_bytes
FROM pg_stat_database
WHERE datname = current_database();
```

### 1.3 复制监控

```sql
-- 复制状态 (主库执行)
SELECT 
    client_addr,
    application_name,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) as replay_lag_bytes,
    now() - pg_last_xact_replay_timestamp() as replay_lag_time
FROM pg_stat_replication;

-- 复制延迟 (从库执行)
SELECT 
    now() - pg_last_xact_replay_timestamp() as replication_lag,
    pg_is_in_recovery() as is_standby;
```

---

## 2. 系统监控命令

### 2.1 CPU和内存

```bash
# 实时CPU监控
top -c -p $(pgrep -d',' postgres)

# htop (更友好)
htop -p $(pgrep -d',' postgres)

# 内存使用
free -h
ps aux | grep postgres | awk '{sum+=$6} END {print sum/1024 " MB"}'

# vmstat - 虚拟内存统计
vmstat 1 10
```

### 2.2 磁盘I/O

```bash
# iostat - I/O统计
iostat -x 1 10

# 关注指标:
# %util - 磁盘使用率 (>80%需要注意)
# await - 平均等待时间
# r/s, w/s - 读写操作数

# iotop - 进程级I/O监控
iotop -o

# 磁盘空间
df -h /var/lib/pgsql/
du -sh /var/lib/pgsql/17/data/base/*
```

### 2.3 网络

```bash
# 连接数统计
netstat -anp | grep :5432 | wc -l
ss -anp | grep :5432 | wc -l

# 网络流量
iftop -i eth0

# 连接详情
netstat -antp | grep :5432
```

---

## 3. 日志分析

### 3.1 配置日志

```ini
# postgresql.conf
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB

# 记录慢查询
log_min_duration_statement = 1000  # 1秒

# 详细日志前缀
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '

# 记录检查点、连接等
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
```

### 3.2 pgBadger - 日志分析工具

```bash
# 安装pgBadger
yum install pgbadger
# 或
cpan install pgBadger

# 分析日志
pgbadger /var/lib/pgsql/17/data/log/postgresql-*.log -o /tmp/report.html

# 增量分析
pgbadger --incremental --outdir /var/www/html/pgbadger /var/lib/pgsql/17/data/log/*.log

# 定时任务
cat > /etc/cron.daily/pgbadger <<'EOF'
#!/bin/bash
pgbadger --incremental \
  --outdir /var/www/html/pgbadger \
  /var/lib/pgsql/17/data/log/postgresql-$(date +%Y-%m-%d)*.log
EOF
chmod +x /etc/cron.daily/pgbadger
```

---

## 4. 第三方监控工具

### 4.1 Prometheus + Grafana

#### 安装postgres_exporter

```bash
# 下载
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.15.0/postgres_exporter-0.15.0.linux-amd64.tar.gz
tar xvf postgres_exporter-0.15.0.linux-amd64.tar.gz
cd postgres_exporter-0.15.0.linux-amd64

# 配置
export DATA_SOURCE_NAME="postgresql://monitor:password@localhost:5432/postgres?sslmode=disable"

# 启动
./postgres_exporter &

# 验证
curl http://localhost:9187/metrics
```

#### Prometheus配置

```yaml
# /etc/prometheus/prometheus.yml
scrape_configs:
  - job_name: 'postgresql'
    static_configs:
      - targets: ['localhost:9187']
        labels:
          instance: 'pg-prod-01'
```

#### Grafana Dashboard

```
推荐Dashboard:
- PostgreSQL Database (ID: 9628)
- PostgreSQL Overview (ID: 455)
- PostgreSQL Exporter Quickstart (ID: 14114)

导入方法:
Grafana UI -> Dashboards -> Import -> 输入ID
```

### 4.2 pgAdmin 4

```bash
# 安装 (CentOS/RHEL)
yum install https://ftp.postgresql.org/pub/pgadmin/pgadmin4/yum/pgadmin4-redhat-repo-2-1.noarch.rpm
yum install pgadmin4

# 初始化
/usr/pgadmin4/bin/setup-web.sh

# 访问
http://your-server/pgadmin4
```

**主要功能**:
- 可视化查询执行计划
- 实时服务器监控
- 查询历史和统计
- 数据库管理

### 4.3 pg_top

```bash
# 安装
yum install pg_top

# 使用
pg_top -U postgres -d mydb

# 快捷键
# M - 按内存排序
# T - 按时间排序
# i - 切换idle进程显示
```

### 4.4 pg_activity

```bash
# 安装
pip install pg_activity

# 使用
pg_activity -U postgres -h localhost

# 特性:
# - 实时查询监控
# - CPU/内存使用
# - I/O统计
# - 锁等待
```

---

## 5. 性能分析工具

### 5.1 EXPLAIN 可视化工具

#### explain.depesz.com

```sql
-- 1. 获取执行计划
EXPLAIN (ANALYZE, BUFFERS, TIMING, VERBOSE, FORMAT JSON)
SELECT ...;

-- 2. 访问 https://explain.depesz.com
-- 3. 粘贴JSON输出
-- 4. 点击Explain查看可视化结果
```

#### explain.dalibo.com

```
类似功能，支持:
- 执行计划可视化
- 节点耗时分析
- 缓冲区统计
- 性能建议
```

### 5.2 pg_stat_kcache - 系统调用统计

```sql
-- 安装扩展
CREATE EXTENSION pg_stat_kcache;

-- 查看系统调用统计
SELECT 
    query,
    user_time,
    system_time,
    reads,
    writes
FROM pg_stat_kcache
JOIN pg_stat_statements USING (queryid)
ORDER BY system_time DESC
LIMIT 10;
```

### 5.3 auto_explain - 自动记录慢查询执行计划

```ini
# postgresql.conf
shared_preload_libraries = 'auto_explain,pg_stat_statements'
auto_explain.log_min_duration = 1000  # 1秒
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_timing = on
auto_explain.log_triggers = on
auto_explain.log_verbose = on
auto_explain.log_nested_statements = on
```

---

## 6. 监控脚本集

### 6.1 综合性能监控脚本

```sql
-- pg_monitor_dashboard.sql
\echo '========================================='
\echo 'PostgreSQL Performance Monitor Dashboard'
\echo '========================================='
\echo ''

\echo '[1] Database Overview'
\echo '-------------------------------------'
SELECT 
    current_database() as database,
    pg_size_pretty(pg_database_size(current_database())) as size,
    (SELECT count(*) FROM pg_stat_activity) as total_connections,
    (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') as active_connections,
    version() as version;
\echo ''

\echo '[2] Top 5 Slowest Queries (avg time)'
\echo '-------------------------------------'
SELECT 
    left(query, 60) as query_preview,
    calls,
    round(mean_exec_time::numeric, 2) as avg_ms
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 5;
\echo ''

\echo '[3] Cache Hit Ratio (should > 99%)'
\echo '-------------------------------------'
SELECT 
    round(sum(heap_blks_hit) * 100.0 / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) 
    as cache_hit_ratio_pct
FROM pg_statio_user_tables;
\echo ''

\echo '[4] Table Bloat Top 5'
\echo '-------------------------------------'
SELECT 
    tablename,
    n_dead_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio_pct
FROM pg_stat_user_tables
WHERE n_live_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 5;
\echo ''

\echo '[5] Long Running Queries (> 1 min)'
\echo '-------------------------------------'
SELECT 
    pid,
    now() - query_start as duration,
    state,
    left(query, 60) as query_preview
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - query_start > interval '1 minute'
ORDER BY query_start;
\echo ''

\echo '[6] Lock Waits'
\echo '-------------------------------------'
SELECT count(*) as lock_wait_count
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
\echo ''

\echo '[7] Replication Lag (if applicable)'
\echo '-------------------------------------'
SELECT 
    client_addr,
    application_name,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) / 1024 / 1024 as lag_mb
FROM pg_stat_replication;
\echo ''

\echo '========================================='
\echo 'Monitor Complete'
\echo '========================================='
```

### 6.2 自动化监控脚本

```bash
#!/bin/bash
# pg_monitor.sh - PostgreSQL监控脚本

PGHOST="localhost"
PGPORT="5432"
PGUSER="postgres"
PGDB="mydb"
ALERT_EMAIL="dba@example.com"

# 连接数检查
check_connections() {
    CURRENT=$(psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDB -t -c \
        "SELECT count(*) FROM pg_stat_activity;")
    MAX=$(psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDB -t -c \
        "SELECT setting::int FROM pg_settings WHERE name='max_connections';")
    
    RATIO=$(echo "scale=2; $CURRENT * 100 / $MAX" | bc)
    
    if (( $(echo "$RATIO > 80" | bc -l) )); then
        echo "ALERT: Connection usage at ${RATIO}% ($CURRENT/$MAX)"
        # 发送告警邮件
        echo "Connection alert: ${RATIO}%" | mail -s "PostgreSQL Alert" $ALERT_EMAIL
    fi
}

# 复制延迟检查
check_replication_lag() {
    LAG=$(psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDB -t -c \
        "SELECT COALESCE(EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())), 0);")
    
    if (( $(echo "$LAG > 10" | bc -l) )); then
        echo "ALERT: Replication lag is ${LAG} seconds"
        echo "Replication lag: ${LAG}s" | mail -s "PostgreSQL Alert" $ALERT_EMAIL
    fi
}

# 表膨胀检查
check_bloat() {
    psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDB -t -c \
        "SELECT tablename, n_dead_tup, 
         round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio
         FROM pg_stat_user_tables
         WHERE n_dead_tup > 100000 
           AND round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) > 20;" \
    | while read line; do
        echo "ALERT: Table bloat detected: $line"
        echo "Bloat: $line" | mail -s "PostgreSQL Alert" $ALERT_EMAIL
    done
}

# 执行检查
check_connections
check_replication_lag
check_bloat

# 定时任务
# */5 * * * * /path/to/pg_monitor.sh >> /var/log/pg_monitor.log 2>&1
```

---

## 7. 监控最佳实践

### 7.1 监控指标优先级

**P0 (核心指标 - 必须监控)**
- 数据库可用性
- 连接数和连接状态
- 复制延迟
- 磁盘空间使用率
- 缓存命中率

**P1 (重要指标)**
- 慢查询统计
- TPS/QPS
- 表膨胀率
- 锁等待
- I/O使用率

**P2 (辅助指标)**
- CPU和内存使用
- WAL生成速度
- 临时文件使用
- Autovacuum执行情况
- 索引使用统计

### 7.2 告警阈值建议

```yaml
alerts:
  - name: database_down
    threshold: 连接失败
    severity: critical
    action: 立即处理
    
  - name: connection_usage
    threshold: > 90%
    severity: critical
    
  - name: replication_lag
    threshold: > 60s
    severity: critical
    
  - name: disk_space
    threshold: > 85%
    severity: warning
    
  - name: cache_hit_ratio
    threshold: < 95%
    severity: warning
    
  - name: table_bloat
    threshold: dead_ratio > 30%
    severity: warning
    
  - name: slow_query
    threshold: > 5s
    severity: info
```

### 7.3 监控架构

```
[PostgreSQL 服务器]
        ↓
   [Exporters]
   - postgres_exporter
   - node_exporter
        ↓
   [Prometheus]
   - 数据采集
   - 存储
        ↓
    [Grafana]
    - 可视化
    - 告警
        ↓
   [AlertManager]
   - 告警路由
   - 邮件/短信/Slack
```

---

## 总结

### 监控工具选择建议

**小型环境 (<10个实例)**
- pg_stat_statements
- pgBadger
- pgAdmin 4
- 自定义监控脚本

**中型环境 (10-100个实例)**
- Prometheus + Grafana
- pg_stat_statements
- pgBadger
- 自动化告警

**大型环境 (>100个实例)**
- Prometheus + Grafana + Thanos
- 专业监控平台 (DataDog, New Relic)
- 集中日志管理 (ELK)
- 自动化运维平台

### 关键原则

1. **全面覆盖** - 监控所有关键指标
2. **及时告警** - 问题发生时第一时间知道
3. **历史追踪** - 保留历史数据用于分析
4. **可视化** - 图表比数字更直观
5. **自动化** - 减少人工巡检

---

**下一步**: 学习 [04_best_practices.md](04_best_practices.md) 了解调优最佳实践！

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

