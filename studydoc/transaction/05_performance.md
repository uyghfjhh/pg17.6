# Transaction Manager 性能优化

本文档介绍事务管理器的性能优化技术和最佳实践。

---

## 1. 关键性能参数

### 1.1 同步提交控制

```ini
# postgresql.conf

# 同步提交模式 (影响持久性和性能)
synchronous_commit = on  # on | off | local | remote_write | remote_apply

# on (默认): COMMIT等待WAL刷盘 → 最安全，性能最慢
# off: COMMIT不等待WAL刷盘 → 可能丢失最后几个事务，性能最快
# local: 等待本地WAL刷盘
# remote_write: 等待WAL传输到standby
# remote_apply: 等待standby应用WAL

# 性能对比 (TPS):
# synchronous_commit=on:  1,000 TPS
# synchronous_commit=off: 10,000+ TPS (提升10倍!)
```

**使用场景**:
- ✅ `synchronous_commit=off` 适合: 日志、统计数据、可重建的数据
- ✅ `synchronous_commit=on` 适合: 金融数据、关键业务数据

### 1.2 WAL相关参数

```ini
# WAL缓冲区大小
wal_buffers = 16MB  # 默认=shared_buffers/32，最小64KB

# WAL写入器延迟
wal_writer_delay = 200ms  # 10ms-10000ms，默认200ms

# 提交延迟 (组提交优化)
commit_delay = 0           # 0-100000微秒
commit_siblings = 5        # 并发事务数阈值

# commit_delay>0 时:
# 如果有 >= commit_siblings 个并发事务，
# 延迟 commit_delay 微秒再刷WAL，
# 让更多事务加入组提交
```

### 1.3 检查点参数

```ini
# Checkpoint间隔
checkpoint_timeout = 5min      # 默认5分钟

# WAL大小触发checkpoint
max_wal_size = 1GB             # 默认1GB
min_wal_size = 80MB

# Checkpoint完成目标 (平滑写入)
checkpoint_completion_target = 0.9  # 0.0-1.0，默认0.9
```

### 1.4 2PC参数

```ini
# 最大prepared事务数
max_prepared_transactions = 0  # 默认0（禁用2PC）

# 启用2PC:
max_prepared_transactions = 100  # 根据需求调整
```

---

## 2. 性能优化技术

### 2.1 组提交 (Group Commit)

**原理**: 多个并发COMMIT共享一次WAL刷盘。

```
传统方式:
T1 COMMIT → XLogFlush → 10ms
T2 COMMIT → XLogFlush → 10ms
T3 COMMIT → XLogFlush → 10ms
总计: 30ms, 3次I/O

组提交:
T1 COMMIT ┐
T2 COMMIT ├→ XLogFlush (一次) → 10ms
T3 COMMIT ┘
总计: 10ms, 1次I/O
提升: 3倍!
```

**配置**:

```sql
-- 启用组提交
SET commit_delay = 10;      -- 10微秒延迟
SET commit_siblings = 5;    -- 至少5个并发事务

-- 测试效果
pgbench -c 10 -j 4 -T 60 -M prepared
```

### 2.2 异步提交

**原理**: COMMIT不等待WAL刷盘，后台异步刷盘。

```sql
-- 会话级别
SET synchronous_commit = off;

-- 事务级别
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    SET LOCAL synchronous_commit = off;
COMMIT;  -- 立即返回，不等待WAL刷盘
```

**风险**: 崩溃时最多丢失约3 * `wal_writer_delay` 的事务（默认600ms）。

**最佳实践**:

```sql
-- 关键事务: 同步提交
BEGIN;
    UPDATE critical_data SET ...;
COMMIT;  -- 使用默认 synchronous_commit=on

-- 非关键事务: 异步提交
BEGIN;
    SET LOCAL synchronous_commit = off;
    INSERT INTO logs (...) VALUES (...);
COMMIT;  -- 快速返回
```

### 2.3 减少子事务开销

**问题**: 过多子事务导致性能下降。

```sql
-- 不好: 每次循环一个子事务
DO $$
BEGIN
    FOR i IN 1..1000 LOOP
        BEGIN
            INSERT INTO t VALUES (i);
        EXCEPTION WHEN unique_violation THEN
            -- 忽略错误
        END;
    END LOOP;
END $$;
-- 性能: 1000个子事务，缓存溢出

-- 更好: 减少子事务数量
DO $$
BEGIN
    BEGIN
        FOR i IN 1..1000 LOOP
            INSERT INTO t VALUES (i);
        END LOOP;
    EXCEPTION WHEN unique_violation THEN
        -- 批量处理错误
    END;
END $$;
-- 性能: 只有1个子事务
```

### 2.4 避免长事务

**危害**:

```
长事务 (运行2小时)
    ↓
阻碍 VACUUM 清理 (xmin=长事务XID)
    ↓
死元组积累 → 表膨胀 → 查询变慢
    ↓
索引膨胀 → 更新变慢
    ↓
磁盘空间不足
```

**监控长事务**:

```sql
-- 查找长事务
SELECT pid, 
       now() - xact_start AS duration,
       state,
       query
FROM pg_stat_activity
WHERE xact_start < now() - interval '5 minutes'
  AND state != 'idle'
ORDER BY xact_start;

-- 终止长事务
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE xact_start < now() - interval '1 hour';
```

**最佳实践**:

```sql
-- 不好: 长事务处理大量数据
BEGIN;
    DELETE FROM large_table WHERE created_at < '2020-01-01';  -- 删除1000万行
COMMIT;
-- 问题: 事务可能运行数小时

-- 更好: 分批处理
DO $$
DECLARE
    rows_affected INTEGER;
BEGIN
    LOOP
        DELETE FROM large_table 
        WHERE ctid IN (
            SELECT ctid FROM large_table
            WHERE created_at < '2020-01-01'
            LIMIT 1000
        );
        
        GET DIAGNOSTICS rows_affected = ROW_COUNT;
        EXIT WHEN rows_affected = 0;
        
        COMMIT;  -- 每批提交一次
        pg_sleep(0.1);  -- 避免过载
    END LOOP;
END $$;
```

---

## 3. 性能监控

### 3.1 事务统计

```sql
-- 数据库级事务统计
SELECT datname,
       xact_commit,
       xact_rollback,
       round(xact_rollback::numeric / NULLIF(xact_commit + xact_rollback, 0) * 100, 2) AS rollback_pct,
       blks_read,
       blks_hit,
       round(blks_hit::numeric / NULLIF(blks_hit + blks_read, 0) * 100, 2) AS cache_hit_ratio
FROM pg_stat_database
ORDER BY xact_commit DESC;

-- 表级事务统计
SELECT schemaname, relname,
       n_tup_ins AS inserts,
       n_tup_upd AS updates,
       n_tup_del AS deletes,
       n_tup_hot_upd AS hot_updates,
       round(n_tup_hot_upd::numeric / NULLIF(n_tup_upd, 0) * 100, 2) AS hot_update_ratio
FROM pg_stat_user_tables
ORDER BY n_tup_upd DESC
LIMIT 20;
```

### 3.2 活跃事务监控

```sql
-- 当前活跃事务
SELECT pid,
       usename,
       datname,
       state,
       now() - xact_start AS xact_duration,
       now() - query_start AS query_duration,
       wait_event_type,
       wait_event,
       query
FROM pg_stat_activity
WHERE state != 'idle'
  AND pid != pg_backend_pid()
ORDER BY xact_start NULLS LAST;
```

### 3.3 2PC监控

```sql
-- 准备好的事务
SELECT gid,
       prepared,
       owner,
       database,
       transaction AS xid
FROM pg_prepared_xacts;

-- 清理长时间未完成的prepared事务
SELECT gid, now() - prepared AS age
FROM pg_prepared_xacts
WHERE prepared < now() - interval '1 hour';

-- 手动回滚
ROLLBACK PREPARED 'gid_001';
```

### 3.4 XID使用情况

```sql
-- 检查XID年龄 (防止回卷)
SELECT datname,
       age(datfrozenxid) AS xid_age,
       2^31 - 1000000 - age(datfrozenxid) AS xids_remaining,
       round((age(datfrozenxid)::float / (2^31 - 1000000)) * 100, 2) AS usage_pct
FROM pg_database
ORDER BY xid_age DESC;

-- 警告: xid_age > 2亿时需要VACUUM
```

---

## 4. 性能基准测试

### 4.1 pgbench 事务性能测试

```bash
# 初始化测试数据
pgbench -i -s 100 testdb

# 测试1: 默认配置 (synchronous_commit=on)
pgbench -c 10 -j 4 -T 60 -M prepared testdb
# 结果: ~1,000 TPS

# 测试2: 异步提交 (synchronous_commit=off)
psql testdb -c "ALTER SYSTEM SET synchronous_commit = off"
psql testdb -c "SELECT pg_reload_conf()"
pgbench -c 10 -j 4 -T 60 -M prepared testdb
# 结果: ~10,000 TPS (提升10倍!)

# 测试3: 组提交优化
psql testdb -c "ALTER SYSTEM SET commit_delay = 10"
psql testdb -c "ALTER SYSTEM SET commit_siblings = 5"
psql testdb -c "SELECT pg_reload_conf()"
pgbench -c 20 -j 4 -T 60 -M prepared testdb
# 结果: ~12,000 TPS (再提升20%)
```

### 4.2 子事务性能测试

```sql
-- 测试1: 无子事务
\timing on
BEGIN;
    INSERT INTO test SELECT generate_series(1, 10000);
COMMIT;
-- 时间: 100ms

-- 测试2: 1000个子事务
BEGIN;
    FOR i IN 1..1000 LOOP
        SAVEPOINT sp;
        INSERT INTO test SELECT generate_series(1, 10);
        RELEASE sp;
    END LOOP;
COMMIT;
-- 时间: 500ms (慢5倍)

-- 测试3: 子事务溢出 (>64个)
BEGIN;
    FOR i IN 1..100 LOOP
        SAVEPOINT sp;
        INSERT INTO test VALUES (i);
        RELEASE sp;
    END LOOP;
COMMIT;
-- 时间: 800ms (溢出到pg_subtrans，更慢)
```

---

## 5. 性能调优建议

### 5.1 通用建议

| 场景 | 推荐配置 | 理由 |
|------|----------|------|
| OLTP高并发 | `synchronous_commit=off` | 提升10倍TPS |
| 批量导入 | `synchronous_commit=off` | 加速数据导入 |
| 金融系统 | `synchronous_commit=on` | 保证持久性 |
| 大事务 | 分批处理 | 避免表膨胀 |
| 频繁子事务 | 减少SAVEPOINT | 避免缓存溢出 |

### 5.2 硬件优化

```
CPU: 多核心有助于并发事务处理
内存: 增大shared_buffers减少I/O
磁盘: 
  - SSD > HDD (WAL刷盘速度提升10倍+)
  - 独立WAL磁盘 (避免与数据文件竞争)
  - RAID10 > RAID5 (写入性能)
```

### 5.3 应用层优化

```python
# 不好: 每行一个事务 (频繁COMMIT)
for row in data:
    conn.execute("INSERT INTO t VALUES (?)", row)
    conn.commit()
# 性能: 1000行 → 10秒

# 更好: 批量事务
conn.begin()
for row in data:
    conn.execute("INSERT INTO t VALUES (?)", row)
conn.commit()
# 性能: 1000行 → 0.1秒 (提升100倍!)

# 最好: COPY命令
conn.copy_from(data_file, 't')
# 性能: 1000行 → 0.01秒
```

---

## 6. 性能问题诊断

### 6.1 COMMIT慢

**症状**: `COMMIT` 花费数秒。

**原因**:
1. WAL刷盘慢 (磁盘I/O瓶颈)
2. 持有大量锁
3. 触发器执行慢

**诊断**:

```sql
-- 检查WAL写入速度
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 
       extract(epoch from now() - pg_postmaster_start_time()) / 1024 / 1024 
       AS wal_mb_per_sec;
-- 期望: > 10 MB/s

-- 检查锁
SELECT pid, mode, granted
FROM pg_locks
WHERE pid = <your_pid>
  AND NOT granted;
```

**解决方案**:
- 启用异步提交: `SET synchronous_commit = off`
- 升级到SSD
- 优化触发器逻辑

### 6.2 表膨胀

**症状**: 表大小不断增长，查询变慢。

**原因**: 长事务阻碍VACUUM。

**诊断**:

```sql
-- 检查表膨胀
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
       n_live_tup,
       n_dead_tup,
       round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;
```

**解决方案**:
- 终止长事务
- 调整`autovacuum_vacuum_scale_factor`
- 手动VACUUM: `VACUUM FULL <table>`

---

## 总结

### 核心优化点

1. **异步提交**: 性能提升10倍
2. **组提交**: 并发场景下再提升20-50%
3. **避免长事务**: 防止表膨胀
4. **减少子事务**: 避免缓存溢出
5. **SSD硬盘**: WAL刷盘速度提升10倍+

### 监控指标

- ✅ TPS (每秒事务数)
- ✅ 回滚率 (<5%)
- ✅ HOT更新率 (>80%)
- ✅ XID年龄 (<2亿)
- ✅ 长事务数量 (=0)

---

**下一步**: 阅读 [06_testcases.md](06_testcases.md) 查看完整测试用例

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-01-16

