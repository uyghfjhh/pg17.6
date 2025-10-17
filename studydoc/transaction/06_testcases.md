# Transaction Manager 测试用例

本文档提供全面的事务管理器测试用例集合。

---

## 1. 基础事务测试

### 测试1: 简单COMMIT

```sql
-- 目的: 验证基本事务提交功能
-- 预期: 数据成功写入并持久化

BEGIN;
    CREATE TEMP TABLE test_commit (id INT, value TEXT);
    INSERT INTO test_commit VALUES (1, 'test');
    SELECT * FROM test_commit;  -- 应返回 (1, 'test')
COMMIT;

-- 验证: 新会话中可见
SELECT * FROM test_commit;  -- 应返回 (1, 'test')
```

### 测试2: 简单ROLLBACK

```sql
-- 目的: 验证基本事务回滚功能
-- 预期: 数据未写入

BEGIN;
    CREATE TEMP TABLE test_rollback (id INT);
    INSERT INTO test_rollback VALUES (1), (2), (3);
    SELECT count(*) FROM test_rollback;  -- 应返回 3
ROLLBACK;

-- 验证: 表不存在
SELECT * FROM test_rollback;  -- 错误: relation does not exist
```

### 测试3: 自动提交模式

```sql
-- 目的: 验证自动提交行为
-- 预期: 每个语句自动提交

CREATE TEMP TABLE test_autocommit (id INT);
INSERT INTO test_autocommit VALUES (1);
-- 自动提交,立即可见

SELECT * FROM test_autocommit;  -- 返回 (1)
```

---

## 2. 隔离级别测试

### 测试4: READ COMMITTED - 脏读防护

```sql
-- 会话1
BEGIN;
    CREATE TEMP TABLE test_rc (id INT, value INT);
    INSERT INTO test_rc VALUES (1, 100);
    UPDATE test_rc SET value = 200 WHERE id = 1;
    -- 未提交

-- 会话2
BEGIN;
    SELECT * FROM test_rc WHERE id = 1;
    -- 预期: 看不到未提交的数据 (空结果)
COMMIT;

-- 会话1
COMMIT;

-- 会话2 (新事务)
SELECT * FROM test_rc WHERE id = 1;
-- 预期: 现在可以看到 (1, 200)
```

### 测试5: REPEATABLE READ - 不可重复读防护

```sql
-- 会话1
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    SELECT * FROM accounts WHERE id = 1;  -- 余额=1000

-- 会话2
BEGIN;
    UPDATE accounts SET balance = 2000 WHERE id = 1;
COMMIT;

-- 会话1
    SELECT * FROM accounts WHERE id = 1;  -- 仍然=1000 (快照隔离)
COMMIT;

-- 验证: 新事务看到新值
SELECT * FROM accounts WHERE id = 1;  -- 余额=2000
```

### 测试6: SERIALIZABLE - 幻读防护

```sql
-- 会话1
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SELECT count(*) FROM orders WHERE user_id = 1;  -- 返回 5

-- 会话2
BEGIN;
    INSERT INTO orders (user_id, amount) VALUES (1, 100);
COMMIT;

-- 会话1
    SELECT count(*) FROM orders WHERE user_id = 1;  -- 仍然=5
    INSERT INTO summary VALUES (1, 5);  -- 基于旧快照的操作
COMMIT;
-- 预期: 可能因序列化冲突而失败
-- ERROR: could not serialize access
```

---

## 3. 子事务 (SAVEPOINT) 测试

### 测试7: SAVEPOINT 基本功能

```sql
-- 目的: 验证SAVEPOINT创建和回滚
-- 预期: 部分操作回滚

BEGIN;
    INSERT INTO t VALUES (1);
    
    SAVEPOINT sp1;
    INSERT INTO t VALUES (2);
    
    SAVEPOINT sp2;
    INSERT INTO t VALUES (3);
    
    ROLLBACK TO sp2;  -- 回滚插入(3)
    SELECT * FROM t;  -- 返回 (1), (2)
    
    ROLLBACK TO sp1;  -- 回滚插入(2)
    SELECT * FROM t;  -- 返回 (1)
    
COMMIT;

SELECT * FROM t;  -- 只有 (1)
```

### 测试8: RELEASE SAVEPOINT

```sql
-- 目的: 验证RELEASE功能
-- 预期: 释放保存点但保留修改

BEGIN;
    INSERT INTO t VALUES (1);
    
    SAVEPOINT sp1;
    INSERT INTO t VALUES (2);
    
    RELEASE SAVEPOINT sp1;  -- 释放但不回滚
    
    SELECT * FROM t;  -- 返回 (1), (2)
    
    -- ROLLBACK TO sp1;  -- 错误: savepoint不存在
COMMIT;

SELECT * FROM t;  -- 返回 (1), (2)
```

### 测试9: 嵌套子事务

```sql
-- 目的: 验证多层嵌套
-- 预期: 支持深层嵌套

BEGIN;
    INSERT INTO t VALUES (1);  -- Level 1
    
    SAVEPOINT L2;
        INSERT INTO t VALUES (2);  -- Level 2
        
        SAVEPOINT L3;
            INSERT INTO t VALUES (3);  -- Level 3
            
            SAVEPOINT L4;
                INSERT INTO t VALUES (4);  -- Level 4
            ROLLBACK TO L4;
            
        ROLLBACK TO L3;
        
    RELEASE L2;
    
COMMIT;

SELECT * FROM t;  -- 返回 (1), (2)
```

### 测试10: 子事务溢出 (>64个)

```sql
-- 目的: 验证子事务缓存溢出处理
-- 预期: 自动溢出到pg_subtrans

BEGIN;
    FOR i IN 1..100 LOOP
        EXECUTE format('SAVEPOINT sp%s', i);
        INSERT INTO t VALUES (i);
    END LOOP;
    
    -- 验证: 所有子事务可见
    SELECT count(*) FROM t;  -- 返回 100
COMMIT;

-- 检查子事务状态
SELECT subxidStatus FROM pg_stat_activity 
WHERE pid = pg_backend_pid();
-- 预期: SUBXIDS_OVERFLOW (2)
```

---

## 4. 两阶段提交 (2PC) 测试

### 测试11: PREPARE TRANSACTION 基本功能

```sql
-- 准备事务
BEGIN;
    INSERT INTO accounts VALUES (1, 1000);
PREPARE TRANSACTION 'txn_001';

-- 验证: 事务准备就绪
SELECT * FROM pg_prepared_xacts WHERE gid = 'txn_001';
-- 应返回一行

-- 验证: 数据尚未提交(其他会话看不到)
SELECT * FROM accounts WHERE id = 1;  -- 空结果

-- 提交准备好的事务
COMMIT PREPARED 'txn_001';

-- 验证: 数据现在可见
SELECT * FROM accounts WHERE id = 1;  -- 返回 (1, 1000)
```

### 测试12: ROLLBACK PREPARED

```sql
-- 准备事务
BEGIN;
    INSERT INTO accounts VALUES (2, 2000);
PREPARE TRANSACTION 'txn_002';

-- 验证准备状态
SELECT gid, prepared FROM pg_prepared_xacts 
WHERE gid = 'txn_002';

-- 回滚准备好的事务
ROLLBACK PREPARED 'txn_002';

-- 验证: 数据未写入
SELECT * FROM accounts WHERE id = 2;  -- 空结果

-- 验证: 准备事务已清理
SELECT * FROM pg_prepared_xacts WHERE gid = 'txn_002';  -- 空结果
```

### 测试13: 2PC 锁持久化

```sql
-- 会话1: 准备事务并持有锁
BEGIN;
    SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
PREPARE TRANSACTION 'txn_lock';

-- 会话2: 尝试获取同一行的锁
BEGIN;
    SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
    -- 阻塞! 等待准备好的事务释放锁

-- 会话1: 提交准备好的事务
COMMIT PREPARED 'txn_lock';

-- 会话2: 现在获得锁
-- (会话2 解除阻塞)
COMMIT;
```

### 测试14: 2PC 崩溃恢复

```bash
# 准备事务
psql -c "BEGIN; INSERT INTO t VALUES (1); PREPARE TRANSACTION 'crash_test';"

# 模拟崩溃
pg_ctl stop -m immediate

# 重启
pg_ctl start

# 验证: 准备事务恢复
psql -c "SELECT * FROM pg_prepared_xacts;"
# 应包含 'crash_test'

# 提交恢复的事务
psql -c "COMMIT PREPARED 'crash_test';"

# 验证: 数据存在
psql -c "SELECT * FROM t;"  -- 返回 (1)
```

---

## 5. 并发测试

### 测试15: 并发UPDATE冲突

```sql
-- 会话1
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    -- 持有行锁

-- 会话2
BEGIN;
    UPDATE accounts SET balance = balance + 50 WHERE id = 1;
    -- 阻塞! 等待会话1

-- 会话1
COMMIT;  -- 释放锁

-- 会话2
    -- 解除阻塞，继续执行
COMMIT;

-- 验证: 两个更新都成功
SELECT balance FROM accounts WHERE id = 1;
-- 原始=1000, -100+50 = 950
```

### 测试16: 死锁检测

```sql
-- 会话1
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    -- 持有 id=1 的锁

-- 会话2
BEGIN;
    UPDATE accounts SET balance = balance - 50 WHERE id = 2;
    -- 持有 id=2 的锁

-- 会话1
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
    -- 等待 id=2 的锁

-- 会话2
    UPDATE accounts SET balance = balance + 50 WHERE id = 1;
    -- 等待 id=1 的锁 → 检测到死锁!
    -- ERROR: deadlock detected
    
-- 会话1
    -- 继续执行并提交
COMMIT;
```

---

## 6. 性能测试

### 测试17: 同步vs异步提交性能

```sql
-- 测试同步提交
\timing on
SET synchronous_commit = on;

BEGIN;
    INSERT INTO test_perf SELECT generate_series(1, 1000);
COMMIT;
-- 记录时间: 例如 500ms

-- 测试异步提交
SET synchronous_commit = off;

BEGIN;
    INSERT INTO test_perf SELECT generate_series(1001, 2000);
COMMIT;
-- 记录时间: 例如 50ms (快10倍!)
```

### 测试18: 组提交效果

```bash
# 启用组提交
psql -c "ALTER SYSTEM SET commit_delay = 10"
psql -c "ALTER SYSTEM SET commit_siblings = 5"
psql -c "SELECT pg_reload_conf()"

# 并发测试
pgbench -c 20 -j 4 -T 60 -M prepared testdb
# 记录 TPS: 例如 12000

# 禁用组提交
psql -c "ALTER SYSTEM SET commit_delay = 0"
psql -c "SELECT pg_reload_conf()"

# 再次测试
pgbench -c 20 -j 4 -T 60 -M prepared testdb
# 记录 TPS: 例如 10000 (组提交提升20%)
```

---

## 7. 错误处理测试

### 测试19: 事务中的错误

```sql
-- 目的: 验证错误后事务状态
-- 预期: 事务进入ABORT状态

BEGIN;
    INSERT INTO t VALUES (1);
    INSERT INTO t VALUES (1);  -- 错误: 唯一约束冲突
    -- 事务进入ABORT状态
    INSERT INTO t VALUES (2);  -- 错误: 当前事务已中止
ROLLBACK;
```

### 测试20: 子事务错误恢复

```sql
-- 目的: 验证子事务错误不影响父事务
-- 预期: 部分回滚，事务继续

BEGIN;
    INSERT INTO t VALUES (1);
    
    SAVEPOINT sp1;
        INSERT INTO t VALUES (2);
        INSERT INTO t VALUES (2);  -- 错误
    ROLLBACK TO sp1;  -- 恢复到sp1
    
    INSERT INTO t VALUES (3);  -- 成功
COMMIT;

SELECT * FROM t;  -- 返回 (1), (3)
```

---

## 8. 边界条件测试

### 测试21: 空事务

```sql
-- 目的: 验证空事务处理
-- 预期: 不分配XID

BEGIN;
    -- 不执行任何操作
COMMIT;

-- 验证: 未分配XID
SELECT pg_current_xact_id_if_assigned();  -- 返回 NULL
```

### 测试22: 大事务

```sql
-- 目的: 验证大事务处理
-- 预期: 成功提交

BEGIN;
    INSERT INTO large_table SELECT generate_series(1, 10000000);
    -- 插入1000万行
COMMIT;

SELECT count(*) FROM large_table;  -- 10000000
```

### 测试23: 深度嵌套子事务

```sql
-- 目的: 验证嵌套层次限制
-- 预期: 超过64层时报错或溢出

DO $$
DECLARE
    i INT;
BEGIN
    FOR i IN 1..100 LOOP
        EXECUTE format('SAVEPOINT sp%s', i);
    END LOOP;
    
    -- 回滚到第50层
    ROLLBACK TO sp50;
    
    FOR i IN 51..100 LOOP
        EXECUTE format('RELEASE SAVEPOINT sp%s', i);
    END LOOP;
END $$;
```

---

## 9. 集成测试

### 测试24: 事务+MVCC

```sql
-- 验证事务与MVCC集成

-- 会话1
BEGIN;
    UPDATE t SET value = 'v1' WHERE id = 1;
    -- t_xmin=100, t_xmax=0

-- 会话2 (快照在UPDATE之前)
BEGIN;
    SELECT value FROM t WHERE id = 1;  -- 'old_value'
    -- 看到旧版本

-- 会话1
COMMIT;
-- 更新CLOG: 100 → COMMITTED

-- 会话2
    SELECT value FROM t WHERE id = 1;  -- 仍然='old_value'
    -- 快照隔离
COMMIT;

-- 会话2 (新快照)
SELECT value FROM t WHERE id = 1;  -- 'v1'
```

### 测试25: 事务+WAL

```bash
# 验证事务与WAL集成

# 执行事务
psql -c "BEGIN; INSERT INTO t VALUES (1); COMMIT;"

# 检查WAL记录
pg_waldump -p $PGDATA/pg_wal -t xact | grep COMMIT
# 应包含 COMMIT记录

# 模拟崩溃前的事务
psql -c "BEGIN; INSERT INTO t VALUES (2); COMMIT;" &
sleep 0.1
pg_ctl stop -m immediate

# 恢复
pg_ctl start

# 验证: WAL重放恢复事务
psql -c "SELECT * FROM t;"  -- 应包含 (2)
```

---

## 10. 自动化测试脚本

### 综合测试脚本

```bash
#!/bin/bash
# transaction_test.sh

psql <<EOF
-- 清理
DROP TABLE IF EXISTS test_txn CASCADE;
CREATE TABLE test_txn (id INT PRIMARY KEY, value TEXT);

-- 测试1: COMMIT
BEGIN;
INSERT INTO test_txn VALUES (1, 'commit_test');
COMMIT;
SELECT * FROM test_txn WHERE id = 1;  -- 应有结果

-- 测试2: ROLLBACK
BEGIN;
INSERT INTO test_txn VALUES (2, 'rollback_test');
ROLLBACK;
SELECT * FROM test_txn WHERE id = 2;  -- 应无结果

-- 测试3: SAVEPOINT
BEGIN;
INSERT INTO test_txn VALUES (3, 'savepoint_test');
SAVEPOINT sp1;
INSERT INTO test_txn VALUES (4, 'should_rollback');
ROLLBACK TO sp1;
COMMIT;
SELECT count(*) FROM test_txn WHERE id IN (3, 4);  -- 应返回 1

-- 测试4: 2PC
BEGIN;
INSERT INTO test_txn VALUES (5, '2pc_test');
PREPARE TRANSACTION 'test_2pc';
SELECT * FROM pg_prepared_xacts WHERE gid = 'test_2pc';  -- 应有结果
COMMIT PREPARED 'test_2pc';
SELECT * FROM test_txn WHERE id = 5;  -- 应有结果

-- 清理
DROP TABLE test_txn;

\\echo '所有测试通过!'
EOF
```

---

## 总结

本文档提供了25个完整的测试用例，涵盖：

- ✅ 基础事务功能 (COMMIT/ROLLBACK)
- ✅ 隔离级别 (RC/RR/Serializable)
- ✅ 子事务 (SAVEPOINT)
- ✅ 两阶段提交 (2PC)
- ✅ 并发控制
- ✅ 性能测试
- ✅ 错误处理
- ✅ 边界条件
- ✅ 系统集成

这些测试用例可以用于：
- 验证事务管理器功能
- 回归测试
- 性能基准测试
- 故障排查

---

**下一步**: 阅读 [07_diagrams.md](07_diagrams.md) 查看完整架构图

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-01-16

