# Lock Manager 性能优化

本文档详细说明锁性能优化策略、参数调优、监控查询和最佳实践。

---

## 1. 关键性能参数

### 1.1 max_locks_per_transaction

```sql
-- 默认值: 64
max_locks_per_transaction = 64

-- 计算共享内存大小:
-- 锁表大小 = max_locks_per_transaction * (max_connections + max_prepared_transactions)

-- 何时调大:
-- 1. 大事务访问很多表 (如: 分区表)
-- 2. 准备事务较多
-- 3. 出现 "out of shared memory" 错误

-- 示例: 1000 个分区表
max_locks_per_transaction = 256  -- 增加 4 倍
```

**影响**:
- ✅ **调大**: 支持更大事务，但增加共享内存
- ✗ **调小**: 节省内存，但可能导致锁耗尽

### 1.2 deadlock_timeout

```sql
-- 默认值: 1s
deadlock_timeout = 1s

-- 死锁检测延迟: 等待 1 秒后才启动死锁检测器

-- 调优策略:
-- 短事务系统: 500ms ~ 1s (快速检测)
-- 长事务系统: 2s ~ 5s (减少误检)

-- 示例:
deadlock_timeout = 2s  -- OLAP 系统
deadlock_timeout = 500ms  -- OLTP 系统
```

**权衡**:
- ✅ **调小**: 快速检测死锁，但增加 CPU 开销
- ✅ **调大**: 减少误检，但死锁恢复慢

### 1.3 lock_timeout

```sql
-- 默认值: 0 (禁用)
lock_timeout = 0

-- 锁等待超时: 超过时间未获取锁则中止

-- 推荐设置:
lock_timeout = 30s  -- Web 应用
lock_timeout = 5min  -- 批处理作业

-- 会话级设置:
SET lock_timeout = '10s';
```

**使用场景**:
- ✅ 防止长时间等待锁
- ✅ 及时向用户返回错误

---

## 2. 锁监控查询

### 2.1 查看当前锁等待

```sql
-- 查看所有锁等待情况
SELECT 
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocked.query AS blocked_query,
    blocking.query AS blocking_query,
    blocked.wait_event_type,
    blocked.wait_event
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE NOT blocked.granted;
```

### 2.2 查看锁详情

```sql
-- 查看所有持有的锁
SELECT 
    locktype,
    database,
    relation::regclass,
    page,
    tuple,
    transactionid,
    mode,
    granted,
    pid
FROM pg_locks
ORDER BY pid, granted DESC;
```

### 2.3 查看表级锁

```sql
-- 查看特定表的锁
SELECT 
    l.locktype,
    l.mode,
    l.granted,
    l.pid,
    a.usename,
    a.query,
    a.state
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation = 'your_table'::regclass
ORDER BY l.granted, l.mode;
```

### 2.4 死锁历史查询

```sql
-- 查看日志中的死锁记录
-- 需要配置: log_lock_waits = on
SHOW log_lock_waits;

-- 日志示例:
-- LOG:  process 12345 still waiting for ShareLock on relation 16384 after 1000.123 ms
-- DETAIL:  Process holding the lock: 12346. Wait queue: 12345, 12347.
```

---

## 3. 性能优化技巧

### 3.1 使用合适的锁模式

```sql
-- ✗ 不好: 过度使用 LOCK TABLE
BEGIN;
LOCK TABLE users IN ACCESS EXCLUSIVE MODE;  -- 阻塞所有操作
UPDATE users SET last_login = now() WHERE id = 1;
COMMIT;

-- ✓ 好: 使用行级锁
BEGIN;
UPDATE users SET last_login = now() WHERE id = 1;  -- 只锁一行
COMMIT;

-- ✗ 不好: 不必要的 FOR UPDATE
BEGIN;
SELECT * FROM users WHERE id = 1 FOR UPDATE;  -- RowExclusiveLock
-- 只是读取数据...
COMMIT;

-- ✓ 好: 普通 SELECT
BEGIN;
SELECT * FROM users WHERE id = 1;  -- AccessShareLock
COMMIT;
```

### 3.2 减少锁持有时间

```sql
-- ✗ 不好: 在事务中执行耗时操作
BEGIN;
SELECT * FROM big_table;  -- 获取锁
-- 执行复杂业务逻辑 (10 秒)
UPDATE big_table SET status = 'done';
COMMIT;  -- 锁持有 10+ 秒

-- ✓ 好: 分离业务逻辑
-- 1. 读取数据
SELECT * FROM big_table;
-- 2. 在应用层执行业务逻辑 (10 秒)
-- 3. 快速更新
BEGIN;
UPDATE big_table SET status = 'done';
COMMIT;  -- 锁持有 < 1 秒
```

### 3.3 统一锁顺序防止死锁

```sql
-- ✗ 不好: 不同顺序访问表
-- 事务 1:
BEGIN;
UPDATE table_a SET ... WHERE id = 1;
UPDATE table_b SET ... WHERE id = 1;
COMMIT;

-- 事务 2:
BEGIN;
UPDATE table_b SET ... WHERE id = 1;  -- 死锁风险!
UPDATE table_a SET ... WHERE id = 1;
COMMIT;

-- ✓ 好: 统一访问顺序 (按表名或OID排序)
-- 所有事务都按 table_a → table_b 顺序访问
BEGIN;
UPDATE table_a SET ... WHERE id = 1;
UPDATE table_b SET ... WHERE id = 1;
COMMIT;
```

### 3.4 使用咨询锁避免冲突

```sql
-- 场景: 并发导入数据到同一表
-- ✗ 不好: 直接并发 INSERT (可能死锁)
BEGIN;
INSERT INTO data_table VALUES (...);
COMMIT;

-- ✓ 好: 使用咨询锁序列化
BEGIN;
SELECT pg_advisory_xact_lock(12345);  -- 获取咨询锁
INSERT INTO data_table VALUES (...);
COMMIT;  -- 自动释放咨询锁
```

---

## 4. 分区表锁优化

### 4.1 问题: 分区表锁开销

```sql
-- 假设有 1000 个分区:
CREATE TABLE measurements (
    time TIMESTAMP,
    value NUMERIC
) PARTITION BY RANGE (time);

-- 查询所有分区:
SELECT * FROM measurements WHERE time > '2024-01-01';
-- 需要获取 1000+ 个 AccessShareLock!

-- 如果 max_locks_per_transaction = 64:
-- ERROR: out of shared memory
-- HINT: You might need to increase max_locks_per_transaction.
```

### 4.2 解决方案

```sql
-- 方案 1: 增加 max_locks_per_transaction
max_locks_per_transaction = 256

-- 方案 2: 使用分区裁剪
SELECT * FROM measurements 
WHERE time BETWEEN '2024-01-01' AND '2024-01-31';
-- 只访问 31 个分区 (按天分区)

-- 方案 3: 查询特定分区
SELECT * FROM measurements_2024_01 WHERE ...;
```

---

## 5. 长事务锁问题

### 5.1 识别长事务

```sql
-- 查找运行超过 5 分钟的事务
SELECT 
    pid,
    usename,
    state,
    query_start,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state != 'idle'
  AND (now() - query_start) > interval '5 minutes'
ORDER BY duration DESC;
```

### 5.2 长事务的影响

```
长事务持有锁 → 阻塞其他事务 → 级联阻塞

示例:
  T1: 长事务 (持有 AccessShareLock on table_a, 30 分钟)
  T2: 等待 AccessExclusiveLock on table_a (ALTER TABLE)
  T3-T100: 等待 AccessShareLock (被 T2 阻塞)

结果: 98 个事务被阻塞!
```

### 5.3 终止长事务

```sql
-- 优雅终止 (发送 SIGTERM)
SELECT pg_cancel_backend(pid);

-- 强制终止 (发送 SIGQUIT)
SELECT pg_terminate_backend(pid);

-- 批量终止空闲事务
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND (now() - state_change) > interval '10 minutes';
```

---

## 6. DDL 操作锁优化

### 6.1 ADD COLUMN 优化

```sql
-- ✗ 不好: 添加带默认值的列 (PG < 11)
ALTER TABLE big_table ADD COLUMN status TEXT DEFAULT 'pending';
-- 需要重写整表 (获取 AccessExclusiveLock, 可能几小时)

-- ✓ 好: 分两步添加 (PG 11+)
ALTER TABLE big_table ADD COLUMN status TEXT;  -- 快速
ALTER TABLE big_table ALTER COLUMN status SET DEFAULT 'pending';  -- 不锁表
-- 然后批量更新已有行
UPDATE big_table SET status = 'pending' WHERE status IS NULL;
```

### 6.2 CREATE INDEX CONCURRENTLY

```sql
-- ✗ 不好: 普通创建索引
CREATE INDEX idx_users_email ON users(email);
-- 持有 ShareLock, 阻塞 INSERT/UPDATE/DELETE

-- ✓ 好: 并发创建索引
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
-- 只持有 ShareUpdateExclusiveLock, 不阻塞 DML
```

### 6.3 VACUUM 和 ANALYZE

```sql
-- VACUUM 不阻塞读写 (只需 ShareUpdateExclusiveLock)
VACUUM users;

-- VACUUM FULL 阻塞所有操作 (需要 AccessExclusiveLock)
VACUUM FULL users;  -- 避免在高峰期执行!

-- 推荐: 使用 pg_repack 替代 VACUUM FULL
-- pg_repack 在线重建表, 无需长时间锁表
```

---

## 7. 锁争用诊断

### 7.1 识别锁争用热点

```sql
-- 查看锁等待最多的表
SELECT 
    relation::regclass AS table_name,
    mode,
    COUNT(*) AS wait_count
FROM pg_locks
WHERE NOT granted
GROUP BY relation, mode
ORDER BY wait_count DESC
LIMIT 10;
```

### 7.2 pg_stat_statements 分析

```sql
-- 启用扩展
CREATE EXTENSION pg_stat_statements;

-- 查找导致锁等待的查询
SELECT 
    queryid,
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements
WHERE query LIKE '%UPDATE%' OR query LIKE '%DELETE%'
ORDER BY total_exec_time DESC
LIMIT 20;
```

---

## 8. 性能基准测试

### 8.1 锁吞吐量测试

```bash
# pgbench 测试锁性能
pgbench -i -s 10 testdb  # 初始化

# 测试 1: 高并发读 (AccessShareLock)
pgbench -c 50 -j 4 -T 60 -S testdb
# 结果: ~50000 TPS (快速路径优化)

# 测试 2: 高并发写 (RowExclusiveLock)
pgbench -c 50 -j 4 -T 60 testdb
# 结果: ~5000 TPS (锁争用)

# 测试 3: 混合负载
pgbench -c 50 -j 4 -T 60 -f custom_script.sql testdb
```

### 8.2 死锁频率测试

```sql
-- 创建测试脚本模拟死锁
-- script1.sql
BEGIN;
UPDATE table_a SET val = val + 1 WHERE id = 1;
\sleep 0.1
UPDATE table_b SET val = val + 1 WHERE id = 1;
COMMIT;

-- script2.sql
BEGIN;
UPDATE table_b SET val = val + 1 WHERE id = 1;
\sleep 0.1
UPDATE table_a SET val = val + 1 WHERE id = 1;
COMMIT;

-- 并发执行
pgbench -c 10 -j 2 -T 60 -f script1.sql -f script2.sql testdb
-- 统计 ERROR: deadlock detected 次数
```

---

## 9. 最佳实践总结

### 9.1 设计阶段

- ✅ **统一锁顺序**: 避免循环等待
- ✅ **分区策略**: 减少单表锁争用
- ✅ **索引优化**: 减少锁持有时间

### 9.2 开发阶段

- ✅ **使用弱锁**: 优先 SELECT 而非 SELECT FOR UPDATE
- ✅ **短事务**: 及时 COMMIT
- ✅ **批量操作**: 减少锁获取次数

### 9.3 运维阶段

- ✅ **监控锁等待**: 定期检查 pg_locks
- ✅ **调优参数**: 根据负载调整 max_locks_per_transaction
- ✅ **维护窗口**: DDL 操作放在低峰期

### 9.4 问题排查

- ✅ **识别瓶颈**: 使用 pg_stat_activity
- ✅ **分析死锁**: 查看日志详情
- ✅ **终止僵尸事务**: pg_terminate_backend

---

## 10. 监控脚本示例

### 10.1 锁等待告警

```sql
-- 创建监控函数
CREATE OR REPLACE FUNCTION check_lock_waits()
RETURNS TABLE(
    blocked_pid INT,
    blocked_query TEXT,
    blocking_pid INT,
    blocking_query TEXT,
    wait_duration INTERVAL
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        b.pid,
        b.query,
        a.pid,
        a.query,
        now() - b.query_start
    FROM pg_stat_activity b
    JOIN pg_stat_activity a ON a.pid = ANY(pg_blocking_pids(b.pid))
    WHERE NOT b.granted
      AND (now() - b.query_start) > interval '30 seconds';
END;
$$ LANGUAGE plpgsql;

-- 定期检查 (cron)
SELECT * FROM check_lock_waits();
```

### 10.2 死锁统计

```sql
-- 从日志统计死锁次数
-- 需要配置: log_destination = 'csvlog'
SELECT 
    DATE(log_time) AS date,
    COUNT(*) AS deadlock_count
FROM postgres_log
WHERE message LIKE 'deadlock detected%'
GROUP BY DATE(log_time)
ORDER BY date DESC;
```

---

## 总结

### 关键性能指标

| 指标 | 目标 | 监控方法 |
|------|------|----------|
| 锁等待时间 | < 1s | pg_stat_activity |
| 死锁频率 | < 1/分钟 | postgres_log |
| 锁争用率 | < 5% | pg_locks |
| 长事务数 | 0 | query_start |

### 优化优先级

1. **P0**: 消除死锁、终止长事务
2. **P1**: 优化锁模式、缩短持有时间
3. **P2**: 调优参数、分区优化
4. **P3**: 监控告警、性能测试

---

**下一步**: 阅读 [06_testcases.md](06_testcases.md) 了解锁测试用例

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

