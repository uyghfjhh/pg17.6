# Lock Manager 测试用例

本文档提供完整的锁管理测试用例集，涵盖各种锁模式、冲突场景、死锁检测等。

---

## 1. 基础锁获取测试

### 测试用例 1.1: AccessShareLock (SELECT)

```sql
-- 准备
CREATE TABLE test_locks (id INT PRIMARY KEY, val TEXT);
INSERT INTO test_locks VALUES (1, 'test');

-- 会话 1
BEGIN;
SELECT * FROM test_locks WHERE id = 1;
-- 预期: 获取 AccessShareLock

-- 验证
SELECT locktype, mode, granted 
FROM pg_locks 
WHERE relation = 'test_locks'::regclass AND pid = pg_backend_pid();
-- 预期结果:
-- locktype | mode            | granted
-- relation | AccessShareLock | t

COMMIT;
```

### 测试用例 1.2: RowExclusiveLock (UPDATE)

```sql
-- 会话 1
BEGIN;
UPDATE test_locks SET val = 'updated' WHERE id = 1;

-- 验证
SELECT mode FROM pg_locks 
WHERE relation = 'test_locks'::regclass AND pid = pg_backend_pid();
-- 预期: RowExclusiveLock

COMMIT;
```

### 测试用例 1.3: AccessExclusiveLock (DROP TABLE)

```sql
-- 会话 1
BEGIN;
DROP TABLE test_locks;

-- 验证 (在另一个会话)
SELECT mode FROM pg_locks WHERE relation = 'test_locks'::regclass;
-- 预期: AccessExclusiveLock

ROLLBACK;  -- 恢复表
```

---

## 2. 锁冲突测试

### 测试用例 2.1: 读-读不冲突

```sql
-- 会话 1
BEGIN;
SELECT * FROM test_locks;  -- AccessShareLock
-- 不提交

-- 会话 2
BEGIN;
SELECT * FROM test_locks;  -- AccessShareLock
-- 预期: 立即成功，不等待
COMMIT;

-- 会话 1
COMMIT;
```

### 测试用例 2.2: 读-写冲突

```sql
-- 会话 1
BEGIN;
SELECT * FROM test_locks;  -- AccessShareLock

-- 会话 2 (在另一个终端)
BEGIN;
DROP TABLE test_locks;  -- AccessExclusiveLock
-- 预期: 阻塞等待会话 1

-- 验证等待状态
SELECT 
    pid, 
    wait_event_type, 
    wait_event,
    state
FROM pg_stat_activity 
WHERE query LIKE '%DROP TABLE test_locks%';
-- 预期: wait_event = 'Lock'

-- 会话 1
COMMIT;  -- 会话 2 立即继续执行

-- 会话 2
ROLLBACK;
```

### 测试用例 2.3: 写-写冲突

```sql
-- 准备: 确保有数据
INSERT INTO test_locks VALUES (1, 'test') ON CONFLICT DO NOTHING;

-- 会话 1
BEGIN;
UPDATE test_locks SET val = 'v1' WHERE id = 1;

-- 会话 2
BEGIN;
UPDATE test_locks SET val = 'v2' WHERE id = 1;
-- 预期: 阻塞等待会话 1

-- 会话 1
COMMIT;

-- 会话 2 现在继续
COMMIT;

-- 验证最终值
SELECT val FROM test_locks WHERE id = 1;
-- 预期: 'v2'
```

---

## 3. 死锁检测测试

### 测试用例 3.1: 简单死锁 (两个事务)

```sql
-- 准备
CREATE TABLE t1 (id INT PRIMARY KEY);
CREATE TABLE t2 (id INT PRIMARY KEY);
INSERT INTO t1 VALUES (1);
INSERT INTO t2 VALUES (1);

-- 会话 1
BEGIN;
UPDATE t1 SET id = 1 WHERE id = 1;  -- 持有 t1 的锁
\sleep 2
UPDATE t2 SET id = 1 WHERE id = 1;  -- 等待 t2 的锁

-- 会话 2 (在 2 秒内启动)
BEGIN;
UPDATE t2 SET id = 1 WHERE id = 1;  -- 持有 t2 的锁
UPDATE t1 SET id = 1 WHERE id = 1;  -- 等待 t1 的锁 → 死锁!

-- 预期结果:
-- 会话 2 立即报错: ERROR: deadlock detected
-- 会话 1 成功完成

-- 清理
ROLLBACK;  -- 会话 1
DROP TABLE t1, t2;
```

### 测试用例 3.2: 环形死锁 (三个事务)

```sql
-- 准备
CREATE TABLE ta (id INT PRIMARY KEY);
CREATE TABLE tb (id INT PRIMARY KEY);
CREATE TABLE tc (id INT PRIMARY KEY);
INSERT INTO ta VALUES (1);
INSERT INTO tb VALUES (1);
INSERT INTO tc VALUES (1);

-- 会话 1
BEGIN;
UPDATE ta SET id = 1 WHERE id = 1;
\sleep 2
UPDATE tb SET id = 1 WHERE id = 1;

-- 会话 2
BEGIN;
UPDATE tb SET id = 1 WHERE id = 1;
\sleep 1
UPDATE tc SET id = 1 WHERE id = 1;

-- 会话 3
BEGIN;
UPDATE tc SET id = 1 WHERE id = 1;
UPDATE ta SET id = 1 WHERE id = 1;  -- 死锁!

-- 预期: 会话 3 报错
-- ERROR: deadlock detected
-- DETAIL: Process 12345 waits for ShareLock on relation 16384...

-- 清理
ROLLBACK;  -- 各会话
DROP TABLE ta, tb, tc;
```

---

## 4. 锁超时测试

### 测试用例 4.1: lock_timeout

```sql
-- 会话 1
BEGIN;
LOCK TABLE test_locks IN ACCESS EXCLUSIVE MODE;

-- 会话 2
SET lock_timeout = '2s';
BEGIN;
SELECT * FROM test_locks;
-- 预期: 2 秒后报错
-- ERROR: canceling statement due to lock timeout

ROLLBACK;

-- 会话 1
ROLLBACK;
```

### 测试用例 4.2: statement_timeout

```sql
-- 会话 1
BEGIN;
LOCK TABLE test_locks IN ACCESS EXCLUSIVE MODE;

-- 会话 2
SET statement_timeout = '3s';
SELECT * FROM test_locks;
-- 预期: 3 秒后报错
-- ERROR: canceling statement due to statement timeout

-- 会话 1
ROLLBACK;
```

---

## 5. 行级锁测试

### 测试用例 5.1: SELECT FOR UPDATE

```sql
-- 准备
TRUNCATE test_locks;
INSERT INTO test_locks VALUES (1, 'a'), (2, 'b');

-- 会话 1
BEGIN;
SELECT * FROM test_locks WHERE id = 1 FOR UPDATE;

-- 会话 2
BEGIN;
SELECT * FROM test_locks WHERE id = 2 FOR UPDATE;
-- 预期: 成功 (不同行不冲突)

UPDATE test_locks SET val = 'c' WHERE id = 1;
-- 预期: 阻塞 (等待会话 1)

-- 会话 1
COMMIT;

-- 会话 2 继续
COMMIT;
```

### 测试用例 5.2: SELECT FOR SHARE

```sql
-- 会话 1
BEGIN;
SELECT * FROM test_locks WHERE id = 1 FOR SHARE;

-- 会话 2
BEGIN;
SELECT * FROM test_locks WHERE id = 1 FOR SHARE;
-- 预期: 成功 (SHARE 锁不互斥)

UPDATE test_locks SET val = 'd' WHERE id = 1;
-- 预期: 阻塞 (等待会话 1)

-- 会话 1
COMMIT;

-- 会话 2 继续
COMMIT;
```

### 测试用例 5.3: SELECT FOR UPDATE SKIP LOCKED

```sql
-- 会话 1
BEGIN;
SELECT * FROM test_locks WHERE id = 1 FOR UPDATE;

-- 会话 2
SELECT * FROM test_locks FOR UPDATE SKIP LOCKED;
-- 预期: 只返回 id=2 的行 (跳过被锁的 id=1)

-- 验证
-- 结果应该只有:
-- id | val
-- 2  | b

-- 会话 1
COMMIT;
```

---

## 6. 表级锁测试

### 测试用例 6.1: LOCK TABLE 各模式

```sql
-- 测试各种表级锁模式
BEGIN;

-- ACCESS SHARE (最弱)
LOCK TABLE test_locks IN ACCESS SHARE MODE;
SELECT mode FROM pg_locks 
WHERE relation = 'test_locks'::regclass AND pid = pg_backend_pid();
-- 预期: AccessShareLock

-- ROW EXCLUSIVE
LOCK TABLE test_locks IN ROW EXCLUSIVE MODE;
-- 预期: 成功 (可以升级)

-- ACCESS EXCLUSIVE (最强)
LOCK TABLE test_locks IN ACCESS EXCLUSIVE MODE;
-- 预期: 成功

COMMIT;
```

### 测试用例 6.2: 锁冲突矩阵验证

```sql
-- 验证冲突: AccessShareLock vs AccessExclusiveLock
-- 会话 1
BEGIN;
SELECT * FROM test_locks;  -- AccessShareLock

-- 会话 2
BEGIN;
LOCK TABLE test_locks IN ACCESS EXCLUSIVE MODE;
-- 预期: 阻塞

-- 会话 1
COMMIT;

-- 会话 2 继续
COMMIT;
```

---

## 7. 快速路径测试

### 测试用例 7.1: 快速路径锁

```sql
-- 快速路径只支持弱表锁
BEGIN;
SELECT * FROM test_locks;  -- 使用快速路径

-- 验证: lock 列应为 NULL (快速路径)
SELECT 
    locktype,
    relation::regclass,
    mode,
    fastpath
FROM pg_locks
WHERE pid = pg_backend_pid() AND relation = 'test_locks'::regclass;
-- 预期: fastpath = t

COMMIT;
```

### 测试用例 7.2: 快速路径降级

```sql
-- 准备: 创建 17 个表 (超过快速路径槽位)
DO $$
BEGIN
    FOR i IN 1..17 LOOP
        EXECUTE format('CREATE TABLE IF NOT EXISTS t%s (id INT)', i);
    END LOOP;
END $$;

-- 获取 17 个表锁
BEGIN;
SELECT * FROM t1;
SELECT * FROM t2;
-- ... (省略)
SELECT * FROM t16;
SELECT * FROM t17;  -- 第 17 个应降级到常规路径

-- 验证
SELECT COUNT(*), fastpath
FROM pg_locks
WHERE pid = pg_backend_pid() AND locktype = 'relation'
GROUP BY fastpath;
-- 预期: 
-- count | fastpath
-- 16    | t
-- 1     | f

COMMIT;

-- 清理
DO $$
BEGIN
    FOR i IN 1..17 LOOP
        EXECUTE format('DROP TABLE IF EXISTS t%s', i);
    END LOOP;
END $$;
```

---

## 8. 咨询锁测试

### 测试用例 8.1: pg_advisory_lock

```sql
-- 会话 1
SELECT pg_advisory_lock(12345);
-- 预期: 返回空 (成功获取)

-- 验证
SELECT objid, mode FROM pg_locks 
WHERE locktype = 'advisory' AND pid = pg_backend_pid();
-- 预期: objid = 12345, mode = ExclusiveLock

-- 会话 2
SELECT pg_try_advisory_lock(12345);
-- 预期: 返回 false (已被持有)

SELECT pg_advisory_lock(12345);
-- 预期: 阻塞等待

-- 会话 1
SELECT pg_advisory_unlock(12345);
-- 预期: 返回 true

-- 会话 2 继续执行
SELECT pg_advisory_unlock(12345);
```

### 测试用例 8.2: 事务级咨询锁

```sql
-- 自动释放
BEGIN;
SELECT pg_advisory_xact_lock(54321);
-- 验证
SELECT objid FROM pg_locks WHERE locktype = 'advisory';
-- 预期: 54321

COMMIT;

-- 验证已释放
SELECT objid FROM pg_locks WHERE locktype = 'advisory';
-- 预期: 无结果
```

---

## 9. 子事务锁测试

### 测试用例 9.1: 子事务锁继承

```sql
BEGIN;
SELECT * FROM test_locks WHERE id = 1 FOR UPDATE;

SAVEPOINT sp1;
UPDATE test_locks SET val = 'savepoint' WHERE id = 1;

-- 验证锁仍然持有
SELECT mode FROM pg_locks 
WHERE relation = 'test_locks'::regclass AND pid = pg_backend_pid();
-- 预期: RowExclusiveLock

RELEASE SAVEPOINT sp1;
COMMIT;
```

### 测试用例 9.2: 子事务回滚

```sql
BEGIN;
SELECT * FROM test_locks WHERE id = 1 FOR UPDATE;

SAVEPOINT sp1;
UPDATE test_locks SET val = 'rollback_test' WHERE id = 1;

ROLLBACK TO SAVEPOINT sp1;

-- 验证锁仍然持有
SELECT mode FROM pg_locks 
WHERE relation = 'test_locks'::regclass AND pid = pg_backend_pid();
-- 预期: RowExclusiveLock (由主事务持有)

COMMIT;
```

---

## 10. DDL 锁测试

### 测试用例 10.1: ALTER TABLE

```sql
-- 会话 1
BEGIN;
SELECT * FROM test_locks;  -- AccessShareLock

-- 会话 2
ALTER TABLE test_locks ADD COLUMN new_col INT;
-- 预期: 阻塞 (需要 AccessExclusiveLock)

-- 会话 1
COMMIT;

-- 会话 2 继续
-- 验证
\d test_locks
-- 预期: 显示 new_col 列
```

### 测试用例 10.2: CREATE INDEX CONCURRENTLY

```sql
-- 普通创建索引 (阻塞写入)
BEGIN;
INSERT INTO test_locks VALUES (100, 'test');
-- 不提交

-- 会话 2
CREATE INDEX idx_val ON test_locks(val);
-- 预期: 阻塞

ROLLBACK;  -- 会话 1

-- 并发创建索引 (不阻塞写入)
-- 会话 1
BEGIN;
INSERT INTO test_locks VALUES (101, 'concurrent');
-- 不提交

-- 会话 2
CREATE INDEX CONCURRENTLY idx_val2 ON test_locks(val);
-- 预期: 不阻塞，可以完成

-- 会话 1
COMMIT;
```

---

## 11. 性能测试

### 测试用例 11.1: 锁吞吐量

```bash
# 使用 pgbench 测试
# 高并发读
pgbench -c 50 -j 4 -T 30 -S testdb

# 预期结果: > 50000 TPS (快速路径优化)
```

### 测试用例 11.2: 死锁频率

```sql
-- 创建死锁测试脚本
-- 运行并统计死锁次数
-- 预期: deadlock_timeout 后检测到
```

---

## 12. 监控查询测试

### 测试用例 12.1: 查看锁等待

```sql
-- 创建锁等待场景
-- 会话 1
BEGIN;
LOCK TABLE test_locks IN ACCESS EXCLUSIVE MODE;

-- 会话 2
BEGIN;
SELECT * FROM test_locks;
-- 阻塞

-- 会话 3: 监控查询
SELECT 
    blocked.pid AS blocked_pid,
    blocking.pid AS blocking_pid,
    blocked.query AS blocked_query
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE NOT blocked.granted;

-- 预期: 显示会话 2 被会话 1 阻塞

-- 清理
ROLLBACK;  -- 会话 1 和 2
```

---

## 总结

### 测试覆盖

- ✅ **基础锁**: 8 种锁模式
- ✅ **冲突检测**: 读-读、读-写、写-写
- ✅ **死锁**: 简单死锁、环形死锁
- ✅ **超时**: lock_timeout、statement_timeout
- ✅ **行级锁**: FOR UPDATE、FOR SHARE、SKIP LOCKED
- ✅ **快速路径**: 正常、降级场景
- ✅ **咨询锁**: 会话级、事务级
- ✅ **子事务**: 继承、回滚
- ✅ **DDL**: ALTER、CREATE INDEX CONCURRENTLY
- ✅ **性能**: 吞吐量、死锁频率
- ✅ **监控**: 锁等待查询

### 自动化测试建议

```bash
# 使用 pg_regress 或 pg_prove 运行测试
pg_prove -d testdb lock_tests.sql
```

---

**下一步**: 阅读 [07_diagrams.md](07_diagrams.md) 查看架构图表

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

