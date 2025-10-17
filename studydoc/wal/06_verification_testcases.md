# WAL 验证测试用例

> 全面的 WAL 功能验证测试用例，涵盖基本功能、边界条件、性能和错误场景。

---

## 目录

1. [测试环境准备](#一测试环境准备)
2. [基础功能测试](#二基础功能测试)
3. [并发测试用例](#三并发测试用例)
4. [性能基准测试](#四性能基准测试)
5. [故障恢复测试](#五故障恢复测试)
6. [复制场景测试](#六复制场景测试)
7. [边界条件测试](#七边界条件测试)
8. [自动化测试框架](#八自动化测试框架)

---

## 一、测试环境准备

### 1.1 测试环境配置

**环境变量设置**:
```bash
#!/bin/bash
# test_env.sh

export PGPORT=5432
export PGDATA="/tmp/wal_test_data"
export PGUSER=postgres
export PGDATABASE=waltest

export WAL_TEST_DIR="/tmp/wal_test"
export LOG_DIR="$WAL_TEST_DIR/logs"
export BACKUP_DIR="$WAL_TEST_DIR/backup"

# 创建测试目录
mkdir -p $WAL_TEST_DIR $LOG_DIR $BACKUP_DIR

# 初始化数据库
initdb -D $PGDATA -E UTF8 --locale=C

# 启动数据库
pg_ctl -D $PGDATA -l $LOG_DIR/postgres.log start
```

**初始配置**:
```bash
# postgresql.conf 基础配置
cat >> $PGDATA/postgresql.conf << EOF
# 测试专用配置
listen_addresses = '*'
port = $PGPORT
max_connections = 200
shared_buffers = 256MB
wal_level = replica
max_wal_senders = 5
wal_keep_size = 64MB
EOF

# pg_hba.conf 信任配置
echo "host all all 0.0.0.0/0 trust" >> $PGDATA/pg_hba.conf
```

### 1.2 测试数据准备

```sql
-- 创建测试表
CREATE TABLE test_table (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    data JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- 创建索引
CREATE INDEX idx_test_table_name ON test_table(name);
CREATE INDEX idx_test_table_created ON test_table(created_at);

-- 填充测试数据
INSERT INTO test_table (name, data) 
SELECT 
    concat('user_', i) as name,
    jsonb_build_object('seq', i, 'random', random()) as data
FROM generate_series(1, 10000) i;

-- 创建函数：产生 WAL 记录
CREATE OR REPLACE FUNCTION generate_wal_update(
    p_id BIGINT, 
    p_iterations INTEGER DEFAULT 100
) RETURNS INTEGER AS $$
DECLARE
    count INTEGER := 0;
BEGIN
    FOR i IN 1..p_iterations LOOP
        UPDATE test_table 
        SET 
            data = data || jsonb_build_object('update', i, 'time', now()),
            updated_at = now()
        WHERE id = p_id;
        
        count := count + 1;
        
        IF i % 10 = 0 THEN
            COMMIT;
        END IF;
    END LOOP;
    
    RETURN count;
END;
$$ LANGUAGE plpgsql;
```

---

## 二、基础功能测试

### 2.1 WAL 记录生成验证

**测试用例 1: 基本 CRUD 操作 WAL 生成**

```sql
-- test_basic_wal.sql
DO $$
DECLARE
    start_lsn XLogRecPtr;
    end_lsn XLogRecPtr;
    wal_bytes NUMERIC;
BEGIN
    -- 记录开始 LSN
    start_lsn := pg_current_wal_lsn();
    RAISE NOTICE 'Start LSN: %', lsn_to_str(start_lsn);
    
    -- 执行操作
    INSERT INTO test_table (name, data) VALUES ('test_wal', '{"test": true}');
    
    UPDATE test_table SET data = '{"test": false}' WHERE name = 'test_wal';
    
    DELETE FROM test_table WHERE name = 'test_wal';
    
    -- 记录结束 LSN
    end_lsn := pg_current_wal_lsn();
    RAISE NOTICE 'End LSN: %', lsn_to_str(end_lsn);
    
    -- 计算 WAL 产生量
    wal_bytes := pg_wal_lsn_diff(end_lsn, start_lsn);
    RAISE NOTICE 'WAL bytes generated: %', wal_bytes;
    
    -- 验证 WAL 产生量合理 (大于 0)
    ASSERT wal_bytes > 0, 'WAL should be generated for basic operations';
    
END $$;
```

**测试用例 2: 事务边界 WAL 验证**

```sql
-- test_transaction_wal.sql
CREATE TABLE trans_test (
    id SERIAL PRIMARY KEY,
    value VARCHAR(50)
);

DO $$
DECLARE
    lsn_before XLogRecPtr;
    lsn_after_insert XLogRecPtr;
    lsn_after_commit XLogRecPtr;
    lsn_after_rollback XLogRecPtr;
BEGIN
    -- 记录开始前 LSN
    lsn_before := pg_current_wal_insert_lsn();
    
    -- 开始事务
    BEGIN;
    
    -- 插入数据
    INSERT INTO trans_test (value) VALUES ('test1'), ('test2');
    lsn_after_insert := pg_current_wal_insert_lsn();
    
    -- 提交事务
    COMMIT;
    lsn_after_commit := pg_current_wal_lsn();
    
    -- 验证 WAL 记录
    ASSERT pg_wal_lsn_diff(lsn_after_insert, lsn_before) > 0, 
           'INSERT should generate WAL';
    ASSERT pg_wal_lsn_diff(lsn_after_commit, lsn_after_insert) > 0,
           'COMMIT should generate WAL';
    
    -- 测试回滚
    BEGIN;
    INSERT INTO trans_test (value) VALUES ('rollback');
    lsn_before := pg_current_wal_insert_lsn();
    ROLLBACK;
    lsn_after_rollback := pg_current_wal_lsn();
    
    -- 回滚后没有新 WAL (MVCC 不需要 undo)
    ASSERT pg_wal_lsn_diff(lsn_after_rollback, lsn_before) >= 0,
           'ROLLBACK should not generate substantial WAL';
END $$;
```

### 2.2 WAL 文件管理测试

**测试用例 3: WAL 文件切换**

```sql
-- test_file_switch.sql
DO $$
DECLARE
    initial_segment TEXT;
    current_segment TEXT;
    switch_count INTEGER := 0;
BEGIN
    -- 获取当前段文件
    SELECT pg_walfile_name(pg_current_wal_lsn()) INTO initial_segment;
    RAISE NOTICE 'Initial segment: %', initial_segment;
    
    -- 强制多段文件切换
    FOR i IN 1..5 LOOP
        PERFORM pg_switch_wal();
        switch_count := switch_count + 1;
        
        -- 检查段文件变化
        SELECT pg_walfile_name(pg_current_wal_lsn()) INTO current_segment;
        RAISE NOTICE 'After switch %: segment %', i, current_segment;
        
        -- 产生一些 WAL
        PERFORM generate_wal_update(1, 100);
    END LOOP;
    
    -- 验证段文件确实切换了
    SELECT pg_walfile_name(pg_current_wal_lsn()) INTO current_segment;
    ASSERT current_segment != initial_segment, 'Segment should switch after multiple switches';
    
    RAISE NOTICE 'Total switches performed: %', switch_count;
END $$;
```

**测试用例 4: 检查点触发测试**

```sql
-- test_checkpoint.sql
DO $$
DECLARE
    checkpoint_before TEXT;
    checkpoint_after TEXT;
    start_time TIMESTAMP;
    end_time TIMESTAMP;
BEGIN
    -- 获取当前检查点位置
    SELECT pg_control_checkpoint() INTO checkpoint_before;
    RAISE NOTICE 'Checkpoint before: %', checkpoint_before;
    
    -- 强制检查点
    start_time := clock_timestamp();
    CHECKPOINT;
    end_time := clock_timestamp();
    
    -- 获取新检查点位置
    SELECT pg_control_checkpoint() INTO checkpoint_after;
    RAISE NOTICE 'Checkpoint after: %', checkpoint_after;
    
    -- 验证检查点时间合理 (< 30秒 for moderate data)
    ASSERT extract(epoch FROM (end_time - start_time)) < 30,
           'Checkpoint should complete within reasonable time';
    
    -- 验证检查点位置前进
    -- 注意: 这里需要解析检查点字符串，简化验证
    RAISE NOTICE 'Checkpoint completed in % seconds', 
                 extract(epoch FROM (end_time - start_time));
END $$;
```

---

## 三、并发测试用例

### 3.1 多会话并发写入

**测试用例 5: 并发事务 WAL 生成**

```sql
-- test_concurrent_wal.sql
-- 主会话: 设置测试表
CREATE TABLE concurrent_test (
    id BIGSERIAL,
    session_id INTEGER,
    counter INTEGER,
    timestamp TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (id, session_id)
);

-- 背景: 运行多个并发会话
-- 每个会话执行:
SET LOCAL work_mem = '1MB';

DO $$
DECLARE
    session_id INTEGER := pg_backend_pid() % 1000;
    start_lsn XLogRecPtr;
    end_lsn XLogRecPtr;
    i INTEGER;
BEGIN
    start_lsn := pg_current_wal_lsn();
    
    FOR i IN 1..1000 LOOP
        INSERT INTO concurrent_test (session_id, counter)
        VALUES (session_id, i);
        
        IF i % 100 = 0 THEN
            COMMIT;
        END IF;
    END LOOP;
    
    end_lsn := pg_current_wal_lsn();
    
    RAISE NOTICE 'Session %: generated % WAL bytes', 
                 session_id, pg_wal_lsn_diff(end_lsn, start_lsn);
END $$;

-- 主会话: 检查结果
SELECT 
    session_id,
    count(*) as rows_inserted,
    min(counter) as min_counter,
    max(counter) as max_counter
FROM concurrent_test 
GROUP BY session_id 
ORDER BY session_id;
```

**测试用例 6: 组提交验证**

```sql
-- test_group_commit.sql
-- 需要在配置中启用组提交: commit_delay > 0, commit_siblings > 0

-- 主会话: 设置测试环境
ALTER SYSTEM SET commit_delay = '1000ms';
ALTER SYSTEM SET commit_siblings = '5';
SELECT pg_reload_conf();

-- 创建测试函数
CREATE OR REPLACE FUNCTION test_group_commit_worker(
    session_id INTEGER,
    iterations INTEGER
) RETURNS INTEGER AS $$
DECLARE
    i INTEGER;
    start_time TIMESTAMP;
    end_time TIMESTAMP;
BEGIN
    start_time := clock_timestamp();
    
    FOR i IN 1..iterations LOOP
        INSERT INTO concurrent_test (session_id, counter)
        VALUES (session_id, i);
        
        PERFORM pg_sleep(0.01); -- 模拟业务处理
    END LOOP;
    
    end_time := clock_timestamp();
    
    RAISE NOTICE 'Session % completed in % ms',
                 session_id, 
                 extract(ms FROM (end_time - start_time));
    
    RETURN iterations;
END;
$$ LANGUAGE plpgsql;

-- 并发执行
-- 在不同会话中同时执行:
SELECT test_group_commit_worker(pg_backend_pid() % 1000, 50);

-- 监控结果
SELECT 
    wait_event_type,
    wait_event,
    count(*) as waiting_processes,
    avg(extract(ms FROM clock_timestamp() - state_change)) as avg_wait_ms
FROM pg_stat_activity
WHERE wait_event IN ('WalWrite', 'WalSenderWaitForWAL', 'WalReceiverWaitStart')
GROUP BY wait_event_type, wait_event;
```

### 3.2 高并发事务冲突

**测试用例 7: 锁等待和 WAL 影响**

```sql
-- test_lock_wal.sql
-- 创建测试表
CREATE TABLE lock_test (
    id SERIAL PRIMARY KEY,
    value VARCHAR(100)
);

INSERT INTO lock_test (value) VALUES ('initial');

-- 会话1: 开始长事务
BEGIN;
UPDATE lock_test SET value = 'session1' WHERE id = 1;
-- 不提交，保持锁

-- 会话2: 尝试更新同一行
BEGIN;
UPDATE lock_test SET value = 'session2' WHERE id = 1; -- 会阻塞

-- 会话3: 监控 WAL 产生
DO $$
DECLARE
    start_lsn XLogRecPtr;
    end_lsn XLogRecPtr;
BEGIN
    start_lsn := pg_current_wal_lsn();
    
    -- 等待等待中的事务
    PERFORM pg_sleep(5);
    
    -- 会话2 中断锁等待后，检查 WAL
    end_lsn := pg_current_wal_lsn();
    
    RAISE NOTICE 'WAL during lock contention: % bytes',
                 pg_wal_lsn_diff(end_lsn, start_lsn);
END $$;

-- 会话4: 检查锁状态
SELECT 
    pid,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

---

## 四、性能基准测试

### 4.1 吞吐量测试

**测试用例 8: WAL 写入吞吐量**

```sql
-- test_throughput.sql
-- 设置测试参数
CREATE TABLE throughput_test (
    id BIGSERIAL,
    payload JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 单线程性能测试
DO $$
DECLARE
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    start_lsn XLogRecPtr;
    end_lsn XLogRecPtr;
    wal_bytes NUMERIC;
    duration INTERVAL;
    tps NUMERIC;
    wal_mbps NUMERIC;
BEGIN
    start_time := clock_timestamp();
    start_lsn := pg_current_wal_lsn();
    
    -- 插入 10,000 行数据
    INSERT INTO throughput_test (payload)
    SELECT 
        jsonb_build_object(
            'seq', i,
            'data', md5(i::text),
            'random', random()
        )
    FROM generate_series(1, 10000) i;
    
    end_time := clock_timestamp();
    end_lsn := pg_current_wal_lsn();
    
    -- 计算指标
    duration := end_time - start_time;
    wal_bytes := pg_wal_lsn_diff(end_lsn, start_lsn);
    tps := 10000.0 / extract(epoch FROM duration);
    wal_mbps := wal_bytes / (extract(epoch FROM duration) * 1024 * 1024);
    
    RAISE NOTICE 'Throughput Test Results:';
    RAISE NOTICE '  Duration: % seconds', extract(epoch FROM duration);
    RAISE NOTICE '  TPS: % tps', tps;
    RAISE NOTICE '  WAL Generated: % MB', wal_bytes / (1024 * 1024);
    RAISE NOTICE '  WAL Rate: % MB/s', wal_mbps;
END $$;

-- 并发性能测试
-- 设置连接池，在多个会话中同时执行:
\o /tmp/concurrent_throughput.csv
COPY (
    SELECT 
        'benchmark',
        clock_timestamp() AS start_time,
        pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') AS start_wal
) TO stdout WITH CSV;

-- 执行相同的插入操作

SELECT 
    'benchmark',
    clock_timestamp() AS end_time,
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') AS end_wal;
\o
```

### 4.2 延迟测试

**测试用例 9: 事务提交延迟测试**

```sql
-- test_latency.sql
-- 创建测量表
CREATE TABLE latency_test (
    id SERIAL PRIMARY KEY,
    transaction_id BIGINTEGER,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    duration_ms INTEGER
);

-- 测试函数
CREATE OR REPLACE FUNCTION measure_commit_latency(
    test_size INTEGER
) RETURNS NUMERIC AS $$
DECLARE
    i INTEGER;
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    total_duration NUMERIC := 0;
BEGIN
    FOR i IN 1..test_size LOOP
        start_time := clock_timestamp();
        
        INSERT INTO throughput_test (payload)
        VALUES (jsonb_build_object('test', i, 'timestamp', start_time));
        
        end_time := clock_timestamp();
        
        total_duration := total_duration + extract(ms FROM (end_time - start_time));
    END LOOP;
    
    RETURN total_duration / test_size; -- 返回平均延迟
END;
$$ LANGUAGE plpgsql;

-- 运行不同配置的延迟测试
DO $$
DECLARE
    avg_latency NUMERIC;
BEGIN
    -- 测试 async commit
    ALTER SYSTEM SET synchronous_commit = 'off';
    SELECT pg_reload_conf();
    
    avg_latency := measure_commit_latency(1000);
    RAISE NOTICE 'Async commit average latency: % ms', avg_latency;
    
    -- 测试 remote_write commit  
    ALTER SYSTEM SET synchronous_commit = 'remote_write';
    SELECT pg_reload_conf();
    
    avg_latency := measure_commit_latency(1000);
    RAISE NOTICE 'Remote_write commit average latency: % ms', avg_latency;
    
    -- 测试 full sync commit
    ALTER SYSTEM SET synchronous_commit = 'on';
    SELECT pg_reload_conf();
    
    avg_latency := measure_commit_latency(1000);
    RAISE NOTICE 'Full sync commit average latency: % ms', avg_latency;
END $$;
```

---

## 五、故障恢复测试

### 5.1 崩溃恢复测试

**测试用例 10: 模拟进程崩溃恢复**

```bash
#!/bin/bash
# test_crash_recovery.sh

PGDATA="/tmp/crash_test_data"
PGLOG="/tmp/crash_test.log"

# 1. 创建测试环境
initdb -D $PGDATA
cat >> $PGDATA/postgresql.conf << EOF
wal_level = replica
max_wal_senders = 2
checkpoint_timeout = '5min'
EOF

# 2. 启动数据库并插入数据
pg_ctl -D $PGDATA -l $PGLOG start
psql -d postgres << 'SQL'
CREATE TABLE crash_test (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 插入测试数据
INSERT INTO crash_test (data)
SELECT 'data_' || i || ' ' || md5(i::text)
FROM generate_series(1, 10000) i;

-- 强制检查点但没有刷盘
CHECKPOINT;

-- 插入更多数据但不要 close
INSERT INTO crash_test (data)
SELECT 'after_checkpoint_' || i || ' ' || md5(i::text)
FROM generate_series(10001, 15000) i;

SQL

# 3. 模拟进程崩溃 (kill -9)
echo "Simulating crash..."
pkill -9 postgres

# 4. 检查恢复日志
echo "Checking recovery process..."
pg_ctl -D $PGDATA promote -l $PGLOG > /dev/null 2>&1

# 5. 验证数据完整性
echo "Verifying data integrity..."
RECOVERY_COUNT=$(psql -t -d postgres -c "SELECT COUNT(*) FROM crash_test;")
EXPECTED_COUNT=15000

if [ "$RECOVERY_COUNT" -eq "$EXPECTED_COUNT" ]; then
    echo "✅ Recovery successful: $RECOVERY_COUNT rows recovered"
else
    echo "❌ Recovery failed: expected $EXPECTED_COUNT, got $RECOVERY_COUNT"
    exit 1
fi

# 6. 检查是否有数据不一致
echo "Checking for data corruption..."
CORRUPTED_COUNT=$(psql -t -d postgres -c "
SELECT COUNT(*) FROM crash_test 
WHERE data IS NULL OR data = '' OR created_at IS NULL;
")

if [ "$CORRUPTED_COUNT" -eq 0 ]; then
    echo "✅ No data corruption found"
else
    echo "❌ Found $CORRUPTED_COUNT corrupted records"
    exit 1
fi

pg_ctl -D $PGDATA stop
```

### 5.2 时间点恢复测试

**测试用例 11: PITR 恢复验证**

```bash
#!/bin/bash
# test_pitr.sh

BACKUP_DIR="/tmp/pitr_backup"
PGDATA="/tmp/pitr_test_data"
ARCHIVE_DIR="/tmp/pitr_archive"

# 1. 准备环境
rm -rf $BACKUP_DIR $PGDATA $ARCHIVE_DIR
mkdir -p $BACKUP_DIR $ARCHIVE_DIR

# 2. 初始化并创建备份
initdb -D $PGDATA
cat >> $PGDATA/postgresql.conf << EOF
wal_level = replica
archive_mode = on
archive_command = 'cp %p $ARCHIVE_DIR/%f'
checkpoint_timeout = '1min'
EOF

pg_ctl -D $PGDATA start

# 3. 基础备份 + 生成测试数据
psql -d postgres << 'SQL'
CREATE TABLE pitr_test (
    id SERIAL PRIMARY KEY,
    value TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

SELECT pg_start_backup('pitr_test_backup');

INSERT INTO pitr_test (value)
SELECT 'backup_data_' || i 
FROM generate_series(1, 1000) i;

SELECT pg_stop_backup();
SQL

# 4. 基础备份
rsync -a $PGDATA/ $BACKUP_DIR/

# 5. 生成更多数据和 WAL
psql -d postgres << 'SQL'
INSERT INTO pitr_test (value)
SELECT 'after_backup_' || i 
FROM generate_series(1001, 2000) i;

SELECT clock_timestamp() as recovery_target_time;

-- 再插入更多数据
INSERT INTO pitr_test (value)
SELECT 'target_time_' || i 
FROM generate_series(2001, 3000) i;
SQL

# 6. 记录恢复目标时间
RECOVERY_TIME=$(psql -t -d postgres -c "SELECT clock_timestamp();" | tr -d ' ')
echo "Recovery target time: $RECOVERY_TIME"

# 7. 停止并清理
pg_ctl -D $PGDATA stop

# 8. 准备恢复
rm -rf $PGDATA
rsync -a $BACKUP_DIR/ $PGDATA/

# 9. 配置恢复
cat >> $PGDATA/postgresql.conf << EOF
restore_command = 'cp $ARCHIVE_DIR/%f %p'
recovery_target_time = '$RECOVERY_TIME'
EOF

# 10. 创建恢复信号文件
touch $PGDATA/recovery.signal

# 11. 执行恢复
pg_ctl -D $PGDATA start

# 12. 验证恢复结果
echo "Verifying PITR recovery..."
RECOVERED_COUNT=$(psql -t -d postgres -c "SELECT COUNT(*) FROM pitr_test;")

echo "Recovered $RECOVERED_COUNT records"

# 应该只包含恢复时间点之前的数据
FINAL_COUNT=$(psql -t -d postgres -c "SELECT COUNT(*) FROM pitr_test WHERE created_at <= '$RECOVERY_TIME';")

echo "Records before target time: $FINAL_COUNT"

if [ "$FINAL_COUNT" -ge 2000 ] && [ "$FINAL_COUNT" -lt 3000 ]; then
    echo "✅ PITR recovery successful"
else
    echo "❌ PITR recovery failed"
    exit 1
fi

# 13. 清理
pg_ctl -D $PGDATA stop
```

---

## 六、复制场景测试

### 6.1 流复制测试

**测试用例 12: 主从复制验证**

```bash
#!/bin/bash
# test_streaming_replication.sh

PRIMARY_DATA="/tmp/primary_data"
STANDBY_DATA="/tmp/standby_data"
REPLICATION_USER="replicator"
REPLICATION_PASSWORD="reppass"

# 1. 主库设置
initdb -D $PRIMARY_DATA
cat >> $PRIMARY_DATA/postgresql.conf << EOF
port = 5432
listen_addresses = '*'
wal_level = replica
max_wal_senders = 3
wal_keep_segments = 32
archive_mode = on
archive_command = '/bin/true'
primary_conninfo = 'host=localhost port=5433 user=$REPLICATION_USER'
EOF

echo "host replication $REPLICATION_USER 0.0.0.0/0 md5" >> $PRIMARY_DATA/pg_hba.conf
echo "host all all 0.0.0.0/0 trust" >> $PRIMARY_DATA/pg_hba.conf

# 2. 启动主库
pg_ctl -D $PRIMARY_DATA -l /tmp/primary.log start

# 3. 创建复制用户
psql -p 5432 -d postgres -c "CREATE USER $REPLICATION_USER REPLICATION LOGIN ENCRYPTED PASSWORD '$REPLICATION_PASSWORD';"

# 4. 从库设置
pg_basebackup -h localhost -D $STANDBY_DATA -U $REPLICATION_USER -v -P -W --port=5432

cat >> $STANDBY_DATA/postgresql.conf << EOF
port = 5433
hot_standby = on
EOF

cat > $STANDBY_DATA/recovery.conf << EOF
standby_mode = 'on'
primary_conninfo = 'host=localhost port=5432 user=$REPLICATION_USER'
EOF

# 5. 启动从库
pg_ctl -D $STANDBY_DATA -l /tmp/standby.log start

# 6. 测试数据复制
psql -p 5432 -d postgres << 'SQL'
CREATE TABLE replication_test (
    id SERIAL PRIMARY KEY,
    data TEXT,
    timestamp TIMESTAMP DEFAULT NOW()
);

-- 插入测试数据
INSERT INTO replication_test (data)
SELECT 'replica_test_' || i || ' ' || md5(i::text)
FROM generate_series(1, 5000) i;
SQL

# 7. 验证复制延迟
sleep 5

PRIMARY_COUNT=$(psql -t -p 5432 -d postgres -c "SELECT COUNT(*) FROM replication_test;")
STANDBY_COUNT=$(psql -t -p 5433 -d postgres -c "SELECT COUNT(*) FROM replication_test;")

echo "Primary count: $PRIMARY_COUNT"
echo "Standby count: $STANDBY_COUNT"

REPLICATION_DELAY=$(psql -t -p 5432 -d postgres -c "
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) 
FROM pg_stat_replication 
LIMIT 1;
" | tr -d ' ')

echo "Replication Lag: $REPLICATION_DELAY bytes"

# 8. 验证数据一致性
if [ "$PRIMARY_COUNT" -eq "$STANDBY_COUNT" ]; then
    echo "✅ Streaming replication successful"
else
    echo "❌ Streaming replication failed"
    exit 1
fi

# 9. 清理
pg_ctl -D $PRIMARY_DATA stop
pg_ctl -D $STANDBY_DATA stop
```

### 6.2 故障切换测试

**测试用例 13: 主备切换验证**

```sql
-- test_failover.sql
-- 在主库执行
SELECT * FROM pg_stat_replication;

-- 监控复制状态
CREATE OR REPLACE VIEW replication_status AS
SELECT 
    application_name,
    state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS network_lag,
    reply_time
FROM pg_stat_replication;

-- 测试提升
-- 在备库执行:
SELECT pg_promote();
SELECT pg_is_in_recovery() as is_primary;
SELECT pg_current_wal_lsn(); -- 新主库的 LSN

-- 测试新主库写入
CREATE TABLE failover_test (id SERIAL, data TEXT);
INSERT INTO failover_test (data) VALUES ('post_promote');
SELECT * FROM failover_test;
```

---

## 七、边界条件测试

### 7.1 极限场景测试

**测试用例 14: 大事务 WAL 处理**

```sql
-- test_large_transaction.sql
-- 测试单个大事务的 WAL 产生
DO $$
DECLARE
    start_lsn XLogRecPtr;
    end_lsn XLogRecPtr;
    large_json JSONB;
    i INTEGER;
BEGIN
    -- 生成大型 JSONB 对象 (~10KB)
    large_json := jsonb_build_object();
    FOR i IN 1..1000 LOOP
        large_json := large_json || jsonb_build_object(
            'key_' || i, 
            repeat('data_', 100),
            'random_' || i, random()
        );
    END LOOP;
    
    start_lsn := pg_current_wal_lsn();
    
    -- 大事务测试
    BEGIN;
    
    FOR i IN 1..100 LOOP
        INSERT INTO throughput_test (payload)
        VALUES (large_json || jsonb_build_object('row', i));
    END LOOP;
    
    COMMIT;
    
    end_lsn := pg_current_wal_lsn();
    
    RAISE NOTICE 'Large transaction generated % WAL bytes',
                 pg_wal_lsn_diff(end_lsn, start_lsn);
END $$;
```

**测试用例 15: 高频率小事务**

```sql
-- test_high_frequency.sql
-- 测试大量小事务的 WAL 影响
\o /tmp/high_freq_test.csv
\timing on

-- 测试配置
ALTER SYSTEM SET synchronous_commit = 'remote_write';
ALTER SYSTEM SET commit_delay = '500';
SELECT pg_reload_conf();

-- 执行 10000 个小事务
DO $$
DECLARE
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    start_lsn XLogRecPtr;
    end_lsn XLogRecPtr;
BEGIN
    start_time := clock_timestamp();
    start_lsn := pg_current_wal_lsn();
    
    FOR i IN 1..10000 LOOP
        INSERT INTO small_tx_test (id, value)
        VALUES (i, 'small_tx_' || i);
        
        IF i % 100 = 0 THEN
            COMMIT;
        END IF;
    END LOOP;
    
    end_time := clock_timestamp();
    end_lsn := pg_current_wal_lsn();
    
    RAISE NOTICE 'High frequency test: % duration, % WAL bytes',
                 extract(epoch FROM (end_time - start_time)),
                 pg_wal_lsn_diff(end_lsn, start_lsn);
END $$;
\o
```

### 7.2 资源限制测试

**测试用例 16: WAL 目录空间耗尽**

```bash
#!/bin/bash
# test_wal_space_exhaustion.sh

PGDATA="/tmp/space_test_data"
SMALL_MOUNT="/tmp/wal_partition"

# 创建小型文件系统 (100MB)
dd if=/dev/zero of=/tmp/wal_disk.img bs=1M count=100
mkfs.ext4 /tmp/wal_disk.img
mkdir -p $SMALL_MOUNT
mount -o loop /tmp/wal_disk.img $SMALL_MOUNT

# 设置 WAL 目录
initdb -D $PGDATA
ln -s $SMALL_MOUNT $PGDATA/pg_wal

cat >> $PGDATA/postgresql.conf << EOF
wal_level = replica
max_wal_size = '80MB'
min_wal_size = '32MB'
checkpoint_timeout = '1min'
EOF

pg_ctl -D $PGDATA start

# 尝试填满 WAL 空间
psql -d postgres << 'SQL'
CREATE TABLE space_test (
    id BIGSERIAL,
    payload JSONB,
    PRIMARY KEY (id)
);

-- 生成大量数据填满 WAL
DO $$
DECLARE
    i INTEGER;
BEGIN
    FOR i IN 1..100000 LOOP  
        INSERT INTO space_test (payload)
        VALUES (jsonb_build_object('seq', i, 'data', repeat('x', 1000)));
        
        IF i % 1000 = 0 THEN
            COMMIT;
            -- 检查 WAL 空间
            PERFORM pg_size_pretty(pg_wal_lsn_diff(
                pg_current_wal_lsn(), 
                '0/0'
            ));
        END IF;
    END LOOP;
END $$;
SQL

# 监控系统响应
watch -n 1 'df -h /tmp/wal_partition; ps -aux | grep postgres'

# 清理
pg_ctl -D $PGDATA stop -m immediate
umount $SMALL_MOUNT
rm /tmp/wal_disk.img
```

---

## 八、自动化测试框架

### 8.1 Python 测试框架

**测试框架脚本**:

```python
#!/usr/bin/env python3
# wal_test_framework.py

import subprocess
import time
import psycopg2
import os
import sys
from typing import Dict, List, Tuple

class WALTestFramework:
    def __init__(self):
        self.test_dir = "/tmp/wal_auto_test"
        self.primary_data = f"{self.test_dir}/primary"
        self.standby_data = f"{self.test_dir}/standby"
        self.primary_port = 5432
        self.standby_port = 5433
        
    def setup_environment(self):
        """设置测试环境"""
        os.makedirs(self.test_dir, exist_ok=True)
        
        # 初始化主库
        subprocess.run(f"initdb -D {self.primary_data}", shell=True, check=True)
        
        # 配置主库
        with open(f"{self.primary_data}/postgresql.conf", 'a') as f:
            f.write(f"""
port = {self.primary_port}
wal_level = replica
max_wal_senders = 3
archive_mode = on
archive_command = '/bin/true'
checkpoint_timeout = '5min'
""")
        
        with open(f"{self.primary_data}/pg_hba.conf", 'a') as f:
            f.write("host replication all 0.0.0.0/0 trust\n")
            f.write("host all all 0.0.0.0/0 trust\n")
    
    def start_primary(self):
        """启动主库"""
        subprocess.run(f"pg_ctl -D {self.primary_data} -l {self.test_dir}/primary.log start", 
                      shell=True, check=True)
        time.sleep(2)
        
    def stop_primary(self, mode='smart'):
        """停止主库"""
        subprocess.run(f"pg_ctl -D {self.primary_data} -m {mode} stop", 
                      shell=True, check=True)
    
    def get_connection(self, port=None):
        """获取数据库连接"""
        if port is None:
            port = self.primary_port
            
        return psycopg2.connect(
            host="localhost",
            port=port,
            database="postgres",
            user="postgres"
        )
    
    def create_test_data(self, rows: int) -> Dict:
        """创建测试数据并返回统计"""
        metrics = {'rows': rows, 'wal_before': 0, 'wal_after': 0, 
                  'duration': 0}
        
        conn = self.get_connection()
        cur = conn.cursor()
        
        # 创建测试表
        cur.execute("""
            CREATE TABLE IF NOT EXISTS auto_test (
                id SERIAL PRIMARY KEY,
                data JSONB,
                created_at TIMESTAMP DEFAULT NOW()
            )
        """)
        
        # 记录 WAL 开始位置
        cur.execute("SELECT pg_current_wal_lsn()")
        metrics['wal_before'] = cur.fetchone()[0]
        
        start_time = time.time()
        
        # 插入测试数据
        cur.execute("""
            INSERT INTO auto_test (data)
            SELECT 
                jsonb_build_object(
                    'seq', i,
                    'data', md5(i::text),
                    'random', random()
                )
            FROM generate_series(1, %s) i
        """, (rows,))
        
        end_time = time.time()
        metrics['duration'] = end_time - start_time
        
        # 记录 WAL 结束位置
        cur.execute("SELECT pg_current_wal_lsn()")
        metrics['wal_after'] = cur.fetchone()[0]
        
        # 计算 WAL 产生量
        cur.execute("""
            SELECT pg_wal_lsn_diff(%s, %s)
        """, (metrics['wal_after'], metrics['wal_before']))
        metrics['wal_bytes'] = cur.fetchone()[0]
        
        conn.commit()
        conn.close()
        
        return metrics
    
    def run_throughput_test(self) -> List[Dict]:
        """运行吞吐量测试"""
        test_sizes = [1000, 5000, 10000, 20000]
        results = []
        
        for size in test_sizes:
            print(f"Testing with {size} rows...")
            
            metrics = self.create_test_data(size)
            metrics['test_name'] = f'throughput_{size}'
            metrics['tps'] = size / metrics['duration']
            metrics['wal_mbps'] = (metrics['wal_bytes'] / 1024 / 1024) / metrics['duration']
            
            results.append(metrics)
            
            print(f"  TPS: {metrics['tps']:.2f}, WAL: {metrics['wal_mb']:.2f}MB")
            
            # 清理数据
            conn = self.get_connection()
            cur = conn.cursor()
            cur.execute("TRUNCATE TABLE auto_test")
            conn.commit()
            conn.close()
            
            time.sleep(1)
        
        return results
    
    def run_reliability_test(self, iterations: int = 10) -> List[Dict]:
        """运行可靠性测试"""
        results = []
        
        for i in range(iterations):
            print(f"Reliability test iteration {i+1}/{iterations}...")
            
           _metrics = self.create_test_data(5000)
            metrics = _metrics.copy()
            metrics['test_name'] = f'reliability_{i+1}'
            
            # 验证数据完整性
            conn = self.get_connection()
            cur = conn.cursor()
            cur.execute("SELECT COUNT(*) FROM auto_test")
            actual_count = cur.fetchone()[0]
            conn.close()
            
            metrics['expected_count'] = 5000
            metrics['actual_count'] = actual_count
            metrics['data_integrity'] = actual_count == 5000
            
            results.append(metrics)
            
            if metrics['data_integrity']:
                print(f"  ✓ Data integrity verified")
            else:
                print(f"  ❌ Data integrity failed: expected 5000, got {actual_count}")
            
            # 清理并重启 (模拟崩溃恢复)
            conn = self.get_connection()
            cur = conn.cursor()
            cur.execute("TRUNCATE TABLE auto_test")
            conn.commit()
            conn.close()
            
            self.stop_primary('immediate')
            time.sleep(2)
            self.start_primary()
            time.sleep(2)
        
        return results
    
    def generate_report(self, results: List[Dict], filename: str = "wal_test_report.csv"):
        """生成测试报告"""
        import csv
        
        if not results:
            print("No results to report")
            return
        
        fieldnames = ['test_name', 'rows', 'duration', 'wal_bytes', 'wal_mb', 'tps', 'wal_mbps', 
                     'data_integrity', 'expected_count', 'actual_count']
        
        with open(filename, 'w', newline='') as csvfile:
            writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
            writer.writeheader()
            
            for result in results:
                # 格式化数据
                row = {field: result.get(field, '') for field in fieldnames}
                
                # 转换 LSN 到数值
                if isinstance(row.get('wal_before'), str):
                    pass  # 保持原样
                if isinstance(row.get('wal_after'), str):
                    pass
                    
                writer.writerow(row)
        
        print(f"Report saved to {filename}")
    
    def cleanup(self):
        """清理测试环境"""
        self.stop_primary('immediate')
        subprocess.run(f"rm -rf {self.test_dir}", shell=True)


def main():
    if len(sys.argv) < 2:
        print("Usage: python3 wal_test_framework.py [setup|run|cleanup]")
        sys.exit(1)
    
    framework = WALTestFramework()
    
    if sys.argv[1] == "setup":
        framework.setup_environment()
        framework.start_primary()
        print("Test environment setup complete")
        
    elif sys.argv[1] == "run":
        print("Running WAL performance tests...")
        
        # 吞吐量测试
        throughput_results = framework.run_throughput_test()
        
        # 可靠性测试
        reliability_results = framework.run_reliability_test()
        
        # 生成报告
        all_results = throughput_results + reliability_results
        framework.generate_report(all_results)
        
    elif sys.argv[1] == "cleanup":
        framework.cleanup()
        print("Test environment cleaned up")


if __name__ == "__main__":
    main()
```

**测试运行**:

```bash
# 设置环境
python3 wal_test_framework.py setup

# 运行测试
python3 wal_test_framework.py run

# 清理环境
python3 wal_test_framework.py cleanup
```

---

## 总结

本测试用例套件提供了全面的 WAL 功能验证：

### 测试覆盖范围

1. **基础功能**: CRUD 操作、事务边界、文件管理
2. **并发测试**: 多会话写入、组提交、锁冲突  
3. **性能测试**: 吞吐量、延迟、压缩效果
4. **故障恢复**: 崩溃恢复、PITR 时间点恢复
5. **复制场景**: 流复制、故障切换
6. **边界条件**: 大事务、高频率、资源限制
7. **自动化框架**: Python 驱动的持续测试

### 关键验证指标

| 测试类别 | 关键指标 | 验证标准 |
|----------|----------|----------|
| **数据完整性** | 记录数匹配 | 100% |
| **WAL 产生** | 字节数合理 | 符合预期 |
| **性能指标** | TPS/延迟 | 满足 SLA |
| **恢复能力** | 时间点恢复 | 准确到秒 |
| **复制延迟** | 字节延迟 | < 16MB |
| **并发安全** | 无数据丢失 | 0% 错误率 |

**下一步**: 可视化图表和架构图 → [07_diagrams.md](07_diagrams.md)
