# MVCC 性能优化技术

> MVCC 性能调优、表膨胀处理、VACUUM 优化等关键技术。

---

## 目录

1. [表膨胀监控与处理](#一表膨胀监控与处理)
2. [VACUUM 调优](#二vacuum-调优)
3. [长事务问题](#三长事务问题)
4. [事务 ID 回绕处理](#四事务-id-回绕处理)
5. [HOT Update 优化](#五hot-update-优化)

---

## 一、表膨胀监控与处理

### 1.1 表膨胀诊断

```sql
-- 查看死元组比例
SELECT 
    schemaname || '.' || tablename AS table_name,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    n_live_tup AS live_tuples,
    n_dead_tup AS dead_tuples,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;

-- 表膨胀率估算 (需要 pgstattuple 扩展)
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT 
    table_name,
    pg_size_pretty(table_bytes) AS table_size,
    round(100.0 * dead_tuple_percent, 2) AS bloat_pct
FROM (
    SELECT 
        'users'::regclass AS table_name,
        pg_relation_size('users') AS table_bytes,
        (pgstattuple('users')).dead_tuple_percent
) t
WHERE dead_tuple_percent > 10;
```

### 1.2 膨胀处理策略

```
膨胀等级分类:
────────────────────────────────────────
死元组比例    等级    处理方式
< 10%        健康    无需处理
10-20%       轻度    监控
20-40%       中度    手动 VACUUM
40-60%       重度    VACUUM FULL (谨慎)
> 60%        严重    VACUUM FULL + 重建索引

处理流程:
────────────────────────────────────────
轻度膨胀 (10-20%):
  1. 检查 autovacuum 是否正常运行
  2. 调整 autovacuum 参数
  3. 继续监控

中度膨胀 (20-40%):
  1. 手动 VACUUM
     VACUUM VERBOSE table_name;
  2. 如果无效,检查长事务
  3. 终止长事务后重试

重度膨胀 (40%+):
  1. 业务低峰期执行 VACUUM FULL
     VACUUM FULL table_name;  -- 需要 AccessExclusiveLock
  2. 或使用 pg_repack (在线重组)
     pg_repack -t table_name
```

---

## 二、VACUUM 调优

### 2.1 Autovacuum 参数优化

```ini
# postgresql.conf

# === 全局 Autovacuum 配置 ===

# 启用 autovacuum
autovacuum = on

# 并发 worker 数量
autovacuum_max_workers = 3  # 默认, 建议 3-8

# 检查间隔
autovacuum_naptime = 1min   # 默认, 可降至 30s

# 触发阈值 (全局)
autovacuum_vacuum_threshold = 50         # 最少 50 个死元组
autovacuum_vacuum_scale_factor = 0.2     # 20% 死元组触发
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.1    # 10% 变化触发

# 示例计算:
# 表有 1M 行
# 触发 VACUUM 需要: 50 + 1M * 0.2 = 200,050 个死元组
# 触发 ANALYZE 需要: 50 + 1M * 0.1 = 100,050 次变化

# FREEZE 相关
autovacuum_freeze_max_age = 200000000    # 2 亿事务
vacuum_freeze_table_age = 150000000      # 1.5 亿事务
vacuum_freeze_min_age = 50000000         # 5000 万事务

# 性能相关
autovacuum_vacuum_cost_delay = 2ms       # 降低影响查询
autovacuum_vacuum_cost_limit = 200       # 每轮操作数
```

### 2.2 表级 Autovacuum 调优

```sql
-- 高频更新表 (调整更激进)
ALTER TABLE high_update_table SET (
    autovacuum_vacuum_scale_factor = 0.05,   -- 5% 触发
    autovacuum_analyze_scale_factor = 0.02,  -- 2% 触发
    autovacuum_vacuum_cost_delay = 0,        -- 最快速度
    autovacuum_vacuum_cost_limit = -1        -- 无限制
);

-- 低频更新表 (调整更保守)
ALTER TABLE low_update_table SET (
    autovacuum_vacuum_scale_factor = 0.5,    -- 50% 触发
    autovacuum_analyze_scale_factor = 0.2    -- 20% 触发
);

-- 超大表 (需要特殊处理)
ALTER TABLE huge_table SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- 1% (绝对值大)
    autovacuum_vacuum_cost_delay = 0,
    toast.autovacuum_vacuum_scale_factor = 0.05
);

-- 禁用 autovacuum (极少情况)
ALTER TABLE temp_staging_table SET (
    autovacuum_enabled = false
);
-- 手动控制 VACUUM 时机
```

### 2.3 VACUUM 监控

```sql
-- 创建监控视图
CREATE VIEW vacuum_monitor AS
SELECT 
    schemaname || '.' || relname AS table_name,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) AS dead_ratio,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    age(relfrozenxid) AS xid_age
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- 定期检查
SELECT * FROM vacuum_monitor WHERE dead_ratio > 10;

-- 查看正在运行的 VACUUM
SELECT 
    pid,
    datname,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    round(100.0 * heap_blks_scanned / NULLIF(heap_blks_total, 0), 2) AS progress
FROM pg_stat_progress_vacuum;
```

---

## 三、长事务问题

### 3.1 长事务的影响

```
长事务危害:
────────────────────────────────────────
1. 阻止 VACUUM 清理死元组
   ├─ 长事务创建快照
   ├─ 快照阻止删除"对其可见"的元组
   └─ 导致表膨胀

2. 事务 ID 消耗
   ├─ 加速事务 ID 回绕
   └─ 可能触发紧急 VACUUM

3. 锁持有时间长
   ├─ 阻塞其他事务
   └─ 影响并发性能

4. 复制延迟
   ├─ 备库需要等待长事务完成
   └─ 影响故障切换

示例影响:
────────────────────────────────────────
时间线:
T0: 事务 A 开始 (XID=1000)
T1: 大量 UPDATE/DELETE (产生死元组)
T2: Autovacuum 启动
    检查: 事务 A 的快照 xmin=1000
    结果: 死元组对快照 1000 可见,无法清理
T3: 表持续膨胀...
T10: 事务 A 提交 (10 小时后!)
T11: Autovacuum 再次运行
     现在可以清理积累的死元组
```

### 3.2 长事务诊断

```sql
-- 查找长事务
SELECT 
    pid,
    usename,
    datname,
    state,
    now() - xact_start AS duration,
    now() - state_change AS idle_in_transaction_time,
    left(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND xact_start < now() - interval '5 minutes'
ORDER BY xact_start
LIMIT 20;

-- 查找 idle in transaction (空闲事务)
SELECT 
    pid,
    usename,
    now() - state_change AS idle_time,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND state_change < now() - interval '10 minutes';

-- 查找阻塞 VACUUM 的事务
SELECT 
    datname,
    age(datfrozenxid) AS xid_age,
    (SELECT xact_start FROM pg_stat_activity 
     WHERE backend_xmin IS NOT NULL 
     ORDER BY xact_start LIMIT 1) AS oldest_xact
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

### 3.3 长事务处理

```sql
-- 方案 1: 终止长事务 (最直接)
SELECT pg_terminate_backend(pid);

-- 方案 2: 取消查询 (不回滚事务)
SELECT pg_cancel_backend(pid);

-- 方案 3: 设置超时 (预防)
-- 会话级别
SET statement_timeout = '5min';
SET idle_in_transaction_session_timeout = '10min';

-- 全局配置
ALTER DATABASE mydb SET statement_timeout = '30min';
ALTER DATABASE mydb SET idle_in_transaction_session_timeout = '30min';

-- 方案 4: 应用层控制
-- 使用连接池 (PgBouncer)
-- 设置 server_idle_timeout
```

---

## 四、事务 ID 回绕处理

### 4.1 回绕监控

```sql
-- 查看各数据库的 XID age
SELECT 
    datname,
    age(datfrozenxid) AS xid_age,
    2^31 - age(datfrozenxid) AS xids_until_wraparound,
    round(100.0 * age(datfrozenxid) / (2^31), 2) AS pct_used
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- 查看各表的 XID age
SELECT 
    schemaname || '.' || relname AS table_name,
    age(relfrozenxid) AS xid_age,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size
FROM pg_stat_user_tables
WHERE age(relfrozenxid) > 100000000  -- 1 亿
ORDER BY age(relfrozenxid) DESC
LIMIT 20;

-- 告警阈值
-- 1.5 亿: 警告
-- 1.8 亿: 严重
-- 2.0 亿: 强制 VACUUM (数据库拒绝写入)
```

### 4.2 预防措施

```sql
-- 配置合理的 FREEZE 参数
ALTER SYSTEM SET autovacuum_freeze_max_age = 200000000;  -- 2 亿
ALTER SYSTEM SET vacuum_freeze_table_age = 150000000;    -- 1.5 亿
ALTER SYSTEM SET vacuum_freeze_min_age = 50000000;       -- 5000 万

-- 定期 VACUUM FREEZE
-- 方案 1: 定期对大表执行
VACUUM FREEZE large_table;

-- 方案 2: 定期对整个数据库执行 (低峰期)
VACUUM FREEZE;

-- 方案 3: cron 作业
-- 每周末执行
0 2 * * 0 psql -d mydb -c "VACUUM FREEZE;"
```

### 4.3 紧急处理

```sql
-- 情况: age(datfrozenxid) > 2 亿,即将停止接受命令

-- 步骤 1: 查找问题表
SELECT relname, age(relfrozenxid)
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 10;

-- 步骤 2: 对问题表执行 VACUUM FREEZE
VACUUM (FREEZE, VERBOSE) problem_table;

-- 步骤 3: 如果表太大,分批处理
DO $$
DECLARE
    start_id INT := 0;
    batch_size INT := 100000;
BEGIN
    WHILE start_id < 10000000 LOOP
        DELETE FROM problem_table 
        WHERE id >= start_id AND id < start_id + batch_size
          AND is_old_enough(xmin);
        
        VACUUM problem_table;
        
        start_id := start_id + batch_size;
        RAISE NOTICE 'Processed up to %', start_id;
    END LOOP;
END $$;

-- 步骤 4: 监控进度
SELECT datname, age(datfrozenxid) FROM pg_database;
```

---

## 五、HOT Update 优化

### 5.1 表设计优化

```sql
-- 优化 1: 分离频繁更新的列
-- 不好的设计
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT,
    status VARCHAR(20),      -- 频繁更新
    total NUMERIC,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,    -- 频繁更新
    details JSONB            -- 大字段
);
CREATE INDEX idx_user ON orders(user_id);
CREATE INDEX idx_created ON orders(created_at);

-- 好的设计
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT,
    total NUMERIC,
    created_at TIMESTAMP,
    details JSONB
);
CREATE INDEX idx_user ON orders(user_id);
CREATE INDEX idx_created ON orders(created_at);

-- 频繁更新的列放到单独的表
CREATE TABLE order_status (
    order_id INT PRIMARY KEY REFERENCES orders(id),
    status VARCHAR(20),
    updated_at TIMESTAMP
);

-- 效果:
-- orders 表: HOT Update 概率高
-- order_status: 表小,膨胀可控
```

### 5.2 填充因子调整

```sql
-- 调整 fillfactor 为 HOT Update 预留空间
ALTER TABLE high_update_table SET (fillfactor = 80);

-- 重建表使配置生效
VACUUM FULL high_update_table;

-- 或使用 pg_repack (不锁表)
pg_repack -t high_update_table

/*
fillfactor 说明:
- 默认: 100 (页面完全填满)
- 80: 预留 20% 空间给 HOT Update
- 70: 预留 30% (高频更新表)

权衡:
- 更小的 fillfactor → 更多 HOT → 但表更大
- 建议: 70-80 (高频更新), 90-95 (中频), 100 (只读)
*/
```

### 5.3 HOT Update 监控

```sql
-- 查看 HOT Update 比例
SELECT 
    schemaname || '.' || relname AS table_name,
    n_tup_upd AS total_updates,
    n_tup_hot_upd AS hot_updates,
    round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 2) AS hot_ratio
FROM pg_stat_user_tables
WHERE n_tup_upd > 1000
ORDER BY n_tup_upd DESC;

-- HOT Update 比例解读:
-- > 90%: 优秀
-- 70-90%: 良好
-- 50-70%: 一般
-- < 50%: 需要优化
```

---

## 总结

### MVCC 性能优化要点

1. **表膨胀**: 监控死元组比例,及时 VACUUM
2. **Autovacuum**: 合理配置触发阈值和 worker 数量
3. **长事务**: 设置超时,及时终止
4. **事务 ID**: 监控 age,定期 FREEZE
5. **HOT Update**: 优化表设计,调整 fillfactor

### 性能提升总结

```
优化项                  性能提升
────────────────────────────────────
提示位                  50x
HOT Update              10x
Visibility Map          20x
Autovacuum 并发         3x
fillfactor 调整         2-3x
```

---

**文档版本**: 1.0
**相关源码**: PostgreSQL 17.5
**创建日期**: 2025-01-16


