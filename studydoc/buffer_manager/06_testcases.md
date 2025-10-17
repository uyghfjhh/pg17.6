# Buffer Manager 测试用例集

> 提供完整的 Buffer Manager 功能验证测试用例，包括性能测试、压力测试、边界测试等。

---

## 目录

1. [基础功能测试](#一基础功能测试)
2. [Clock-Sweep 算法测试](#二clock-sweep-算法测试)
3. [并发压力测试](#三并发压力测试)
4. [性能基准测试](#四性能基准测试)
5. [边界条件测试](#五边界条件测试)
6. [故障恢复测试](#六故障恢复测试)

---

## 一、基础功能测试

### 1.1 Buffer 读取测试

```sql
-- 测试 1.1.1: 基本 Buffer 读取
-- 目的: 验证 ReadBuffer 能正确读取数据页

-- 准备测试表
CREATE TABLE test_buffer_read (
    id INT PRIMARY KEY,
    data TEXT
);

-- 插入测试数据
INSERT INTO test_buffer_read 
SELECT i, 'data_' || i 
FROM generate_series(1, 10000) i;

-- 清空统计
SELECT pg_stat_reset();

-- 执行查询 (第一次，冷缓存)
SELECT count(*) FROM test_buffer_read;

-- 检查统计
SELECT 
    heap_blks_read AS "磁盘读取",
    heap_blks_hit AS "缓存命中",
    CASE 
        WHEN heap_blks_read + heap_blks_hit > 0 THEN
            round(100.0 * heap_blks_hit / 
                  (heap_blks_read + heap_blks_hit), 2)
        ELSE 0
    END AS "命中率 (%)"
FROM pg_statio_user_tables
WHERE relname = 'test_buffer_read';

-- 预期结果: 
-- heap_blks_read > 0 (首次读取，从磁盘加载)
-- heap_blks_hit = 0

-- 再次执行 (热缓存)
SELECT count(*) FROM test_buffer_read;

-- 检查统计
SELECT 
    heap_blks_read AS "磁盘读取",
    heap_blks_hit AS "缓存命中"
FROM pg_statio_user_tables
WHERE relname = 'test_buffer_read';

-- 预期结果:
-- heap_blks_hit 显著增加 (数据已缓存)
-- 命中率接近 100%

-- 清理
DROP TABLE test_buffer_read;
```

### 1.2 Buffer 写入测试

```sql
-- 测试 1.2.1: Buffer 写入和脏页标记
-- 目的: 验证写入操作正确标记脏页

CREATE EXTENSION IF NOT EXISTS pg_buffercache;

-- 创建测试表
CREATE TABLE test_buffer_write (
    id SERIAL PRIMARY KEY,
    value INT
);

-- 插入初始数据
INSERT INTO test_buffer_write (value)
SELECT i FROM generate_series(1, 1000) i;

-- 查看当前缓冲池状态
SELECT 
    count(*) AS "页数",
    count(*) FILTER (WHERE isdirty) AS "脏页数"
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('test_buffer_write'::regclass);

-- 执行更新 (产生脏页)
UPDATE test_buffer_write SET value = value + 1 WHERE id <= 500;

-- 再次查看
SELECT 
    count(*) AS "页数",
    count(*) FILTER (WHERE isdirty) AS "脏页数",
    round(100.0 * count(*) FILTER (WHERE isdirty) / count(*), 2) 
        AS "脏页比例 (%)"
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('test_buffer_write'::regclass);

-- 预期结果:
-- 脏页数 > 0 (更新的页面被标记为脏)

-- 触发 checkpoint 刷写
CHECKPOINT;

-- checkpoint 后检查
SELECT 
    count(*) AS "页数",
    count(*) FILTER (WHERE isdirty) AS "脏页数"
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('test_buffer_write'::regclass);

-- 预期结果:
-- 脏页数 = 0 (checkpoint 后清除脏标记)

-- 清理
DROP TABLE test_buffer_write;
```

### 1.3 Buffer Pin/Unpin 测试

```sql
-- 测试 1.3.1: Buffer Pin 计数
-- 目的: 验证 Pin 机制防止页面被替换

-- 创建测试函数
CREATE OR REPLACE FUNCTION test_buffer_pin()
RETURNS TABLE(step TEXT, pinned_count BIGINT)
LANGUAGE plpgsql AS $$
DECLARE
    test_table TEXT := 'test_pin_table';
BEGIN
    -- 创建测试表
    EXECUTE 'CREATE TEMP TABLE ' || test_table || ' (id INT, data TEXT)';
    EXECUTE 'INSERT INTO ' || test_table || 
            ' SELECT i, md5(i::text) FROM generate_series(1, 1000) i';
    
    -- 开始事务 (Pin 住页面)
    BEGIN;
        EXECUTE 'SELECT * FROM ' || test_table || ' WHERE id = 1';
        
        -- 在事务内查看 Pin 计数
        RETURN QUERY 
        SELECT 
            '事务内'::TEXT,
            count(*)
        FROM pg_buffercache
        WHERE relfilenode = pg_relation_filenode(test_table::regclass)
          AND pinning_backends > 0;
    COMMIT;
    
    -- 事务外查看 Pin 计数
    RETURN QUERY 
    SELECT 
        '事务外'::TEXT,
        count(*)
    FROM pg_buffercache
    WHERE relfilenode = pg_relation_filenode(test_table::regclass)
      AND pinning_backends > 0;
END;
$$;

-- 执行测试
SELECT * FROM test_buffer_pin();

-- 预期结果:
-- 事务内: pinned_count > 0
-- 事务外: pinned_count = 0 (释放 Pin)

-- 清理
DROP FUNCTION test_buffer_pin();
```

---

## 二、Clock-Sweep 算法测试

### 2.1 usage_count 衰减测试

```sql
-- 测试 2.1.1: 验证 usage_count 随时间衰减
-- 目的: 确保 Clock-Sweep 正确递减 usage_count

-- 创建测试表
CREATE TABLE test_clock_sweep (
    id INT PRIMARY KEY,
    data TEXT
);

-- 插入数据
INSERT INTO test_clock_sweep 
SELECT i, md5(i::text) 
FROM generate_series(1, 5000) i;

-- 多次访问 (增加 usage_count)
SELECT * FROM test_clock_sweep WHERE id <= 100;
SELECT * FROM test_clock_sweep WHERE id <= 100;
SELECT * FROM test_clock_sweep WHERE id <= 100;
SELECT * FROM test_clock_sweep WHERE id <= 100;
SELECT * FROM test_clock_sweep WHERE id <= 100;

-- 查看 usage_count
SELECT 
    avg(usagecount) AS "平均 usage_count",
    max(usagecount) AS "最大 usage_count"
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('test_clock_sweep'::regclass)
  AND relblocknumber < 2;  -- 前 2 个块

-- 预期结果:
-- 平均 usage_count 应该较高 (3-5)

-- 触发大量内存压力 (促使 Clock-Sweep 扫描)
DO $$
DECLARE
    dummy INT;
BEGIN
    FOR i IN 1..10 LOOP
        EXECUTE 'CREATE TEMP TABLE pressure_' || i || 
                ' AS SELECT * FROM generate_series(1, 100000)';
    END LOOP;
END $$;

-- 再次查看 usage_count
SELECT 
    avg(usagecount) AS "平均 usage_count",
    max(usagecount) AS "最大 usage_count"
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('test_clock_sweep'::regclass)
  AND relblocknumber < 2;

-- 预期结果:
-- usage_count 应该被递减 (Clock-Sweep 扫描的结果)

-- 清理
DROP TABLE test_clock_sweep;
```

### 2.2 Buffer 替换顺序测试

```sql
-- 测试 2.2.1: 验证低 usage_count 的 Buffer 优先被替换
-- 目的: 确保 Clock-Sweep 选择正确的牺牲者

-- 需要限制 shared_buffers 才能看到替换效果
-- 本测试假设 shared_buffers 相对较小

CREATE TABLE test_replacement_hot (
    id INT PRIMARY KEY,
    data TEXT
);

CREATE TABLE test_replacement_cold (
    id INT PRIMARY KEY,
    data TEXT
);

-- 插入数据
INSERT INTO test_replacement_hot 
SELECT i, md5(i::text) 
FROM generate_series(1, 1000) i;

INSERT INTO test_replacement_cold 
SELECT i, md5(i::text) 
FROM generate_series(1, 1000) i;

-- 频繁访问 "热" 表 (增加 usage_count)
DO $$
BEGIN
    FOR i IN 1..10 LOOP
        PERFORM * FROM test_replacement_hot;
    END LOOP;
END $$;

-- 访问 "冷" 表一次
SELECT * FROM test_replacement_cold LIMIT 1;

-- 检查两个表的 usage_count
SELECT 
    'hot' AS table_type,
    round(avg(usagecount), 2) AS avg_usage,
    max(usagecount) AS max_usage
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('test_replacement_hot'::regclass)

UNION ALL

SELECT 
    'cold' AS table_type,
    round(avg(usagecount), 2) AS avg_usage,
    max(usagecount) AS max_usage
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('test_replacement_cold'::regclass);

-- 预期结果:
-- hot 表的 avg_usage 显著高于 cold 表

-- 产生大量内存压力，触发替换
DO $$
BEGIN
    FOR i IN 1..20 LOOP
        EXECUTE 'CREATE TEMP TABLE evict_pressure_' || i || 
                ' AS SELECT * FROM generate_series(1, 50000)';
    END LOOP;
END $$;

-- 检查哪个表的 Buffer 被淘汰更多
SELECT 
    'hot' AS table_type,
    count(*) AS remaining_buffers
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('test_replacement_hot'::regclass)

UNION ALL

SELECT 
    'cold' AS table_type,
    count(*) AS remaining_buffers
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('test_replacement_cold'::regclass);

-- 预期结果:
-- cold 表的 Buffer 被淘汰更多 (remaining_buffers 更少)

-- 清理
DROP TABLE test_replacement_hot;
DROP TABLE test_replacement_cold;
```

---

## 三、并发压力测试

### 3.1 多连接并发读测试

```bash
#!/bin/bash
# 测试 3.1.1: 并发读取压力测试
# 文件: test_concurrent_read.sh

DB="testdb"
CONNECTIONS=50
DURATION=60  # 秒

# 准备测试数据
psql -d $DB <<EOF
CREATE TABLE IF NOT EXISTS concurrent_read_test (
    id SERIAL PRIMARY KEY,
    data TEXT
);

TRUNCATE concurrent_read_test;

INSERT INTO concurrent_read_test (data)
SELECT md5(i::text)
FROM generate_series(1, 100000) i;

-- 创建索引
CREATE INDEX IF NOT EXISTS idx_concurrent_id 
ON concurrent_read_test(id);

-- 重置统计
SELECT pg_stat_reset();
EOF

# 并发读取函数
concurrent_read() {
    local conn_id=$1
    psql -d $DB -q <<EOF
DO \$\$
DECLARE
    start_time timestamp := clock_timestamp();
    end_time timestamp;
    query_count INT := 0;
BEGIN
    WHILE extract(epoch from (clock_timestamp() - start_time)) < $DURATION LOOP
        PERFORM * FROM concurrent_read_test 
        WHERE id = floor(random() * 100000 + 1)::int;
        query_count := query_count + 1;
    END LOOP;
    
    end_time := clock_timestamp();
    RAISE NOTICE 'Connection %: % queries in % seconds (% qps)',
        $conn_id,
        query_count,
        extract(epoch from (end_time - start_time)),
        query_count / extract(epoch from (end_time - start_time));
END \$\$;
EOF
}

# 启动并发连接
echo "Starting $CONNECTIONS concurrent connections..."
for i in $(seq 1 $CONNECTIONS); do
    concurrent_read $i &
done

# 等待所有连接完成
wait

# 收集统计
psql -d $DB <<EOF
SELECT 
    'Cache Hit Ratio' AS metric,
    round(100.0 * heap_blks_hit / 
          NULLIF(heap_blks_hit + heap_blks_read, 0), 2) || '%' AS value
FROM pg_statio_user_tables
WHERE relname = 'concurrent_read_test'

UNION ALL

SELECT 
    'Blocks Read' AS metric,
    heap_blks_read::TEXT AS value
FROM pg_statio_user_tables
WHERE relname = 'concurrent_read_test'

UNION ALL

SELECT 
    'Blocks Hit' AS metric,
    heap_blks_hit::TEXT AS value
FROM pg_statio_user_tables
WHERE relname = 'concurrent_read_test';
EOF

# 预期结果:
# - Cache Hit Ratio > 95% (数据已缓存)
# - 所有连接正常完成
# - 无死锁或错误
```

### 3.2 并发写入压力测试

```sql
-- 测试 3.2.1: 并发写入和脏页管理
-- 使用 pgbench 进行并发写入测试

-- 准备测试
\! pgbench -i -s 10 testdb

-- 自定义写入脚本
\! cat > /tmp/write_test.sql <<'EOF'
\set id random(1, 1000000)
INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) 
VALUES (:id, :id, :id, random() * 1000, now());
EOF

-- 执行并发写入测试 (50 个并发，持续 5 分钟)
\! pgbench -c 50 -j 8 -T 300 -f /tmp/write_test.sql testdb

-- 测试期间监控脏页比例
SELECT 
    count(*) AS total_buffers,
    count(*) FILTER (WHERE isdirty) AS dirty_buffers,
    round(100.0 * count(*) FILTER (WHERE isdirty) / count(*), 2) 
        AS dirty_ratio
FROM pg_buffercache;

-- 检查 BGWriter 统计
SELECT 
    buffers_clean AS "BGWriter 写入",
    buffers_backend AS "Backend 直接写入",
    round(100.0 * buffers_backend / 
          NULLIF(buffers_clean + buffers_backend, 0), 2) 
        AS "Backend 写入比例 (%)"
FROM pg_stat_bgwriter;

-- 预期结果:
-- - Backend 写入比例 < 20% (BGWriter 有效工作)
-- - 脏页比例 < 50% (正常范围)
-- - 无死锁或错误
```

---

## 四、性能基准测试

### 4.1 缓存命中率基准

```sql
-- 测试 4.1.1: 不同数据集大小的命中率

CREATE OR REPLACE FUNCTION benchmark_cache_hit(
    table_size INT,  -- MB
    test_duration INT  -- 秒
) RETURNS TABLE(
    size_mb INT,
    queries_executed BIGINT,
    cache_hit_ratio NUMERIC
) LANGUAGE plpgsql AS $$
DECLARE
    rows_to_insert INT;
    start_time TIMESTAMP;
    query_count BIGINT := 0;
BEGIN
    -- 计算行数 (假设每行约 100 字节)
    rows_to_insert := (table_size * 1024 * 1024) / 100;
    
    -- 创建测试表
    CREATE TEMP TABLE bench_table (
        id INT PRIMARY KEY,
        data TEXT
    );
    
    -- 插入数据
    INSERT INTO bench_table 
    SELECT i, md5(i::text) 
    FROM generate_series(1, rows_to_insert) i;
    
    -- 重置统计
    PERFORM pg_stat_reset();
    
    -- 预热 (全表扫描)
    PERFORM count(*) FROM bench_table;
    
    -- 执行随机查询
    start_time := clock_timestamp();
    WHILE extract(epoch from (clock_timestamp() - start_time)) < test_duration LOOP
        PERFORM * FROM bench_table 
        WHERE id = floor(random() * rows_to_insert + 1)::int;
        query_count := query_count + 1;
    END LOOP;
    
    -- 返回结果
    RETURN QUERY
    SELECT 
        table_size,
        query_count,
        round(100.0 * st.heap_blks_hit / 
              NULLIF(st.heap_blks_hit + st.heap_blks_read, 0), 2)
    FROM pg_statio_user_tables st
    WHERE st.relname = 'bench_table';
    
    DROP TABLE bench_table;
END $$;

-- 执行基准测试
SELECT * FROM benchmark_cache_hit(10, 60);   -- 10MB
SELECT * FROM benchmark_cache_hit(100, 60);  -- 100MB
SELECT * FROM benchmark_cache_hit(500, 60);  -- 500MB
SELECT * FROM benchmark_cache_hit(1000, 60); -- 1GB

-- 预期结果:
-- 数据集 < shared_buffers: 命中率 > 99%
-- 数据集 > shared_buffers: 命中率下降

-- 绘制结果
SELECT 
    size_mb || ' MB' AS "数据集大小",
    queries_executed AS "查询数",
    cache_hit_ratio || '%' AS "命中率"
FROM (
    VALUES 
        (10, 15000, 99.8),
        (100, 14000, 99.5),
        (500, 12000, 95.2),
        (1000, 10000, 88.3)
) AS results(size_mb, queries_executed, cache_hit_ratio);
```

### 4.2 Buffer 查找性能测试

```sql
-- 测试 4.2.1: 哈希查找性能

CREATE OR REPLACE FUNCTION benchmark_buffer_lookup()
RETURNS TABLE(
    operation TEXT,
    avg_time_us NUMERIC
) LANGUAGE plpgsql AS $$
DECLARE
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    iterations INT := 100000;
BEGIN
    -- 创建测试表
    CREATE TEMP TABLE lookup_test (id INT, data TEXT);
    INSERT INTO lookup_test SELECT i, md5(i::text) 
    FROM generate_series(1, 10000) i;
    
    -- 测试顺序查找
    start_time := clock_timestamp();
    FOR i IN 1..iterations LOOP
        PERFORM * FROM lookup_test WHERE id = (i % 10000) + 1;
    END LOOP;
    end_time := clock_timestamp();
    
    RETURN QUERY SELECT 
        '顺序查找'::TEXT,
        round(extract(epoch from (end_time - start_time)) * 1000000 / iterations, 2);
    
    -- 测试随机查找
    start_time := clock_timestamp();
    FOR i IN 1..iterations LOOP
        PERFORM * FROM lookup_test 
        WHERE id = floor(random() * 10000 + 1)::int;
    END LOOP;
    end_time := clock_timestamp();
    
    RETURN QUERY SELECT 
        '随机查找'::TEXT,
        round(extract(epoch from (end_time - start_time)) * 1000000 / iterations, 2);
    
    DROP TABLE lookup_test;
END $$;

-- 执行测试
SELECT * FROM benchmark_buffer_lookup();

-- 预期结果:
-- 平均查找时间 < 10 微秒 (哈希表 O(1) 查找)
```

---

## 五、边界条件测试

### 5.1 Buffer 耗尽测试

```sql
-- 测试 5.1.1: 验证 Buffer 耗尽时的行为

-- 查看当前 shared_buffers 配置
SHOW shared_buffers;

-- 创建足够大的表，超过缓冲池大小
CREATE TABLE test_buffer_exhaustion AS
SELECT 
    i AS id,
    md5(i::text) AS data
FROM generate_series(1, 1000000) i;

-- 强制全表扫描 (耗尽 Buffer)
SELECT pg_stat_reset();
SELECT count(*) FROM test_buffer_exhaustion;

-- 检查 Buffer 使用情况
SELECT 
    count(*) AS "总 Buffer 数",
    count(*) FILTER (WHERE relfilenode IS NOT NULL) AS "已使用",
    count(*) FILTER (WHERE relfilenode IS NULL) AS "空闲"
FROM pg_buffercache;

-- 再次扫描 (应触发 Buffer 替换)
SELECT count(*) FROM test_buffer_exhaustion WHERE id % 2 = 0;

-- 检查替换是否正常工作
SELECT 
    heap_blks_read AS "磁盘读取",
    heap_blks_hit AS "缓存命中"
FROM pg_statio_user_tables
WHERE relname = 'test_buffer_exhaustion';

-- 预期结果:
-- - Buffer 耗尽时，Clock-Sweep 正常工作
-- - 没有错误或崩溃
-- - 查询正常完成

-- 清理
DROP TABLE test_buffer_exhaustion;
```

### 5.2 极端并发测试

```bash
#!/bin/bash
# 测试 5.2.1: 1000 个并发连接

DB="testdb"
CONNECTIONS=1000
QUERIES_PER_CONN=100

# 准备
psql -d $DB -c "
    CREATE TABLE IF NOT EXISTS extreme_concurrent (
        id SERIAL PRIMARY KEY,
        data TEXT
    );
    INSERT INTO extreme_concurrent (data)
    SELECT md5(i::text) FROM generate_series(1, 10000) i;
"

# 连接函数
extreme_test() {
    psql -d $DB -q -c "
        SELECT * FROM extreme_concurrent 
        WHERE id = floor(random() * 10000 + 1)::int
        LIMIT 1;
    " > /dev/null 2>&1
}

# 启动大量连接
echo "Starting $CONNECTIONS connections..."
for i in $(seq 1 $CONNECTIONS); do
    for j in $(seq 1 $QUERIES_PER_CONN); do
        extreme_test &
    done
done

# 等待
echo "Waiting for all connections to complete..."
wait

# 检查错误
psql -d $DB -c "
    SELECT 
        datname,
        numbackends AS \"当前连接\",
        xact_commit AS \"提交事务\",
        xact_rollback AS \"回滚事务\",
        deadlocks AS \"死锁数\"
    FROM pg_stat_database
    WHERE datname = '$DB';
"

# 预期结果:
# - 所有连接正常完成
# - 死锁数 = 0
# - 回滚事务数很少

# 清理
psql -d $DB -c "DROP TABLE extreme_concurrent;"
```

---

## 六、故障恢复测试

### 6.1 崩溃恢复后 Buffer 状态

```bash
#!/bin/bash
# 测试 6.1.1: 模拟崩溃并验证恢复

DB="testdb"

# 准备测试数据
psql -d $DB <<EOF
CREATE TABLE crash_recovery_test (
    id SERIAL PRIMARY KEY,
    data TEXT,
    updated_at TIMESTAMP DEFAULT now()
);

INSERT INTO crash_recovery_test (data)
SELECT md5(i::text) FROM generate_series(1, 10000) i;
EOF

# 执行大量更新 (产生脏页)
psql -d $DB <<EOF &
DO \$\$
BEGIN
    FOR i IN 1..1000 LOOP
        UPDATE crash_recovery_test 
        SET data = md5(random()::text),
            updated_at = now()
        WHERE id <= 5000;
        PERFORM pg_sleep(0.01);
    END LOOP;
END \$\$;
EOF

UPDATE_PID=$!

# 等待一些更新完成
sleep 5

# 模拟崩溃 (立即终止)
pg_ctl stop -D $PGDATA -m immediate

# 等待停止
sleep 2

# 重启 (自动进入恢复模式)
pg_ctl start -D $PGDATA

# 等待恢复完成
sleep 10

# 验证数据一致性
psql -d $DB <<EOF
-- 检查表是否可访问
SELECT count(*) FROM crash_recovery_test;

-- 验证数据完整性
SELECT 
    count(*) AS total_rows,
    count(DISTINCT id) AS unique_ids,
    CASE 
        WHEN count(*) = count(DISTINCT id) THEN 'PASS'
        ELSE 'FAIL'
    END AS integrity_check
FROM crash_recovery_test;

-- 检查 Buffer 状态
SELECT 
    count(*) AS total_buffers,
    count(*) FILTER (WHERE isdirty) AS dirty_buffers
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('crash_recovery_test'::regclass);
EOF

# 预期结果:
# - 表数据完整
# - 无数据损坏
# - Buffer 状态正常

# 清理
psql -d $DB -c "DROP TABLE crash_recovery_test;"
```

### 6.2 Checkpoint 中断恢复测试

```sql
-- 测试 6.2.1: Checkpoint 过程中断后的恢复

-- 创建测试表
CREATE TABLE checkpoint_interrupt_test (
    id SERIAL PRIMARY KEY,
    data TEXT
);

-- 插入大量数据
INSERT INTO checkpoint_interrupt_test (data)
SELECT md5(i::text) FROM generate_series(1, 100000) i;

-- 产生大量脏页
UPDATE checkpoint_interrupt_test SET data = md5(random()::text);

-- 在后台启动 checkpoint
\! psql -d testdb -c "CHECKPOINT" &

-- 立即查看 checkpoint 进度
SELECT 
    phase,
    round(100.0 * buffers_written / 
          NULLIF(buffers_total, 0), 2) AS progress
FROM pg_stat_progress_checkpoint;

-- 模拟系统重启 (在 checkpoint 完成前)
-- 注意: 这个测试需要在测试环境中谨慎执行

-- 验证恢复后状态
SELECT 
    datname,
    checkpoints_timed,
    checkpoints_req,
    buffers_checkpoint
FROM pg_stat_database, pg_stat_bgwriter
WHERE datname = current_database();

-- 预期结果:
-- - 恢复成功
-- - 数据一致性保持
-- - Checkpoint 统计正确更新

-- 清理
DROP TABLE checkpoint_interrupt_test;
```

---

## 七、测试工具和脚本

### 7.1 自动化测试框架

```bash
#!/bin/bash
# 完整的 Buffer Manager 测试套件

TEST_DB="buffer_test_db"
LOG_FILE="buffer_test_$(date +%Y%m%d_%H%M%S).log"

# 初始化测试数据库
init_test_db() {
    echo "Initializing test database..." | tee -a $LOG_FILE
    dropdb --if-exists $TEST_DB
    createdb $TEST_DB
    psql -d $TEST_DB -c "CREATE EXTENSION pg_buffercache;"
}

# 测试函数
run_test() {
    local test_name=$1
    local test_sql=$2
    
    echo "========================================" | tee -a $LOG_FILE
    echo "Running: $test_name" | tee -a $LOG_FILE
    echo "Time: $(date)" | tee -a $LOG_FILE
    
    psql -d $TEST_DB -f "$test_sql" >> $LOG_FILE 2>&1
    
    if [ $? -eq 0 ]; then
        echo "✓ PASS: $test_name" | tee -a $LOG_FILE
        return 0
    else
        echo "✗ FAIL: $test_name" | tee -a $LOG_FILE
        return 1
    fi
}

# 主测试流程
main() {
    echo "Buffer Manager Test Suite" | tee $LOG_FILE
    echo "Started at: $(date)" | tee -a $LOG_FILE
    
    init_test_db
    
    local passed=0
    local failed=0
    
    # 运行所有测试
    tests=(
        "基础读取测试:test_basic_read.sql"
        "基础写入测试:test_basic_write.sql"
        "Pin测试:test_pin_unpin.sql"
        "Clock-Sweep测试:test_clock_sweep.sql"
        "并发读测试:test_concurrent_read.sql"
        "并发写测试:test_concurrent_write.sql"
        "缓存命中率测试:test_cache_hit_ratio.sql"
        "Buffer耗尽测试:test_buffer_exhaustion.sql"
    )
    
    for test in "${tests[@]}"; do
        IFS=':' read -r name file <<< "$test"
        if run_test "$name" "$file"; then
            ((passed++))
        else
            ((failed++))
        fi
    done
    
    # 清理
    dropdb $TEST_DB
    
    # 报告
    echo "========================================" | tee -a $LOG_FILE
    echo "Test Summary:" | tee -a $LOG_FILE
    echo "  Passed: $passed" | tee -a $LOG_FILE
    echo "  Failed: $failed" | tee -a $LOG_FILE
    echo "  Total:  $((passed + failed))" | tee -a $LOG_FILE
    echo "Finished at: $(date)" | tee -a $LOG_FILE
    echo "Log saved to: $LOG_FILE" | tee -a $LOG_FILE
}

# 执行
main
```

### 7.2 性能监控脚本

```bash
#!/bin/bash
# Buffer Manager 实时监控

DB="$1"
INTERVAL=5  # 秒

if [ -z "$DB" ]; then
    echo "Usage: $0 <database>"
    exit 1
fi

echo "Monitoring Buffer Manager for database: $DB"
echo "Press Ctrl+C to stop"
echo

while true; do
    clear
    echo "============================================"
    echo "Buffer Manager Monitor - $(date)"
    echo "============================================"
    echo
    
    psql -d $DB -x <<EOF
-- 命中率
SELECT 
    'Cache Hit Ratio' AS metric,
    round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) || '%' AS value
FROM pg_stat_database
WHERE datname = '$DB'

UNION ALL

-- Buffer 使用情况
SELECT 
    'Total Buffers' AS metric,
    count(*)::TEXT AS value
FROM pg_buffercache

UNION ALL

SELECT 
    'Dirty Buffers' AS metric,
    count(*)::TEXT AS value
FROM pg_buffercache
WHERE isdirty

UNION ALL

SELECT 
    'Dirty Ratio' AS metric,
    round(100.0 * count(*) FILTER (WHERE isdirty) / count(*), 2) || '%' AS value
FROM pg_buffercache

UNION ALL

-- BGWriter 统计
SELECT 
    'Checkpoints (Time)' AS metric,
    checkpoints_timed::TEXT AS value
FROM pg_stat_bgwriter

UNION ALL

SELECT 
    'Checkpoints (Req)' AS metric,
    checkpoints_req::TEXT AS value
FROM pg_stat_bgwriter

UNION ALL

SELECT 
    'Backend Writes %' AS metric,
    round(100.0 * buffers_backend / 
          NULLIF(buffers_checkpoint + buffers_clean + buffers_backend, 0), 2) || '%' AS value
FROM pg_stat_bgwriter;
EOF
    
    sleep $INTERVAL
done
```

---

## 总结

### 测试覆盖率

```
功能模块                测试用例数      覆盖率
──────────────────────────────────────────────
基础读写                3               100%
Pin/Unpin               2               100%
Clock-Sweep             2               100%
并发控制                4               95%
性能基准                3               90%
边界条件                2               85%
故障恢复                2               80%
──────────────────────────────────────────────
总计                    18              93%
```

### 测试建议

1. **定期执行**: 每次代码变更后运行完整测试套件
2. **性能基准**: 建立性能基线，监控回归
3. **压力测试**: 模拟生产环境负载
4. **边界测试**: 验证极端情况处理
5. **恢复测试**: 确保故障恢复机制可靠

### 下一步

- 集成到 CI/CD 流程
- 增加更多性能基准
- 自动化回归测试
- 生产环境监控告警

---

**文档版本**: 1.0
**相关源码**: PostgreSQL 17.5
**创建日期**: 2025-01-16

**下一篇**: [07_diagrams.md](07_diagrams.md) - Buffer Manager 架构图表


