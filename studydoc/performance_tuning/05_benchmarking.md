# PostgreSQL 性能测试和基准

> 性能测试方法和基准测试工具

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. pgbench - PostgreSQL官方基准测试工具

### 1.1 pgbench简介

pgbench是PostgreSQL自带的基准测试工具，用于测试数据库的事务处理性能（TPS）。

```bash
# 检查pgbench版本
pgbench --version
```

### 1.2 快速开始

#### 初始化测试数据

```bash
# 创建测试数据库
createdb pgbench_test

# 初始化测试数据 (scale=100表示1000万行数据)
pgbench -i -s 100 pgbench_test

# 数据量计算:
# scale=1: 100,000 rows (约15MB)
# scale=10: 1,000,000 rows (约150MB)
# scale=100: 10,000,000 rows (约1.5GB)
# scale=1000: 100,000,000 rows (约15GB)

# 初始化时的表结构:
# pgbench_accounts (主表，100000 * scale 行)
# pgbench_branches (100 * scale 行)
# pgbench_tellers (10 * scale 行)
# pgbench_history (初始为空)
```

#### 运行基准测试

```bash
# 基础测试 (默认TPC-B场景)
pgbench -c 10 -j 2 -t 1000 pgbench_test

# 参数说明:
# -c 10: 10个并发客户端
# -j 2: 2个工作线程
# -t 1000: 每个客户端执行1000个事务

# 按时间测试 (推荐)
pgbench -c 20 -j 4 -T 60 pgbench_test
# -T 60: 运行60秒

# 输出示例:
# transaction type: <builtin: TPC-B (sort of)>
# scaling factor: 100
# query mode: simple
# number of clients: 20
# number of threads: 4
# duration: 60 s
# number of transactions actually processed: 123456
# latency average = 9.712 ms
# tps = 2057.600000 (including connections establishing)
# tps = 2058.123456 (excluding connections establishing)
```

### 1.3 内置测试场景

```bash
# [1] TPC-B测试 (默认，读写混合)
pgbench -c 10 -j 2 -T 60 pgbench_test

# [2] 只读测试 (SELECT only)
pgbench -c 10 -j 2 -T 60 -S pgbench_test

# [3] 简单写入测试
pgbench -c 10 -j 2 -T 60 -N pgbench_test

# [4] 只查询主键
pgbench -c 10 -j 2 -T 60 -b select-only pgbench_test

# [5] 简单更新
pgbench -c 10 -j 2 -T 60 -b simple-update pgbench_test
```

### 1.4 自定义测试脚本

#### 示例1: 用户查询测试

```sql
-- user_query.sql
\set user_id random(1, 1000000)
SELECT * FROM users WHERE id = :user_id;
```

```bash
# 运行自定义脚本
pgbench -c 10 -j 2 -T 60 -f user_query.sql mydb
```

#### 示例2: 复杂查询测试

```sql
-- complex_query.sql
\set user_id random(1, 100000)
\set start_date '2025-01-01'
\set end_date '2025-12-31'

BEGIN;
SELECT 
    o.id,
    o.order_no,
    o.total_amount,
    u.username
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.user_id = :user_id
  AND o.created_at BETWEEN :'start_date' AND :'end_date'
ORDER BY o.created_at DESC
LIMIT 10;
COMMIT;
```

#### 示例3: 写入测试

```sql
-- insert_test.sql
\set user_id random(1, 100000)
\set amount random(10, 10000)

BEGIN;
INSERT INTO orders (user_id, order_no, total_amount, status, created_at)
VALUES (:user_id, 'ORD' || :user_id || floor(random()*1000000), :amount, 'pending', now());
COMMIT;
```

#### 示例4: 混合负载测试

```bash
# 创建多个测试脚本，指定权重
pgbench -c 20 -j 4 -T 300 \
  -f select_query.sql@70 \
  -f insert_query.sql@20 \
  -f update_query.sql@10 \
  mydb

# 权重: 70%查询, 20%插入, 10%更新
```

### 1.5 高级选项

```bash
# [1] 连接模式
pgbench -M prepared -c 10 -T 60 mydb     # 预编译模式 (最快)
pgbench -M extended -c 10 -T 60 mydb     # 扩展查询协议
pgbench -M simple -c 10 -T 60 mydb       # 简单查询协议 (默认)

# [2] 报告选项
pgbench -c 10 -T 60 --progress=10 mydb   # 每10秒输出进度
pgbench -c 10 -T 60 --log mydb           # 记录每个事务的日志
pgbench -c 10 -T 60 --log-prefix=test1 mydb  # 日志文件前缀

# [3] 采样率
pgbench -c 10 -T 60 --sampling-rate=0.01 mydb  # 采样1%的事务

# [4] 限流
pgbench -c 10 -R 1000 -T 60 mydb         # 限制1000 TPS

# [5] 延迟统计
pgbench -c 10 -T 60 --latency-limit=100 mydb  # 记录超过100ms的事务

# [6] 指定主机和端口
pgbench -h localhost -p 5432 -U postgres -c 10 -T 60 mydb

# [7] 无VACUUM初始化 (加快初始化)
pgbench -i -s 100 --no-vacuum mydb
```

### 1.6 结果分析

```bash
# 详细统计输出
pgbench -c 20 -j 4 -T 60 --progress=10 --latency-limit=50 pgbench_test

# 输出分析:
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 20
number of threads: 4
duration: 60 s
number of transactions actually processed: 123456
latency average = 9.712 ms              # 平均延迟
latency stddev = 12.345 ms              # 延迟标准差
initial connection time = 45.678 ms
tps = 2057.600000 (including connections)
tps = 2058.123456 (excluding connections)  # 重点关注这个

# 百分位延迟
statement latencies in milliseconds:
         0.005  \set aid random(1, 100000 * :scale)
         0.003  \set bid random(1, 1 * :scale)
         0.002  \set tid random(1, 10 * :scale)
         0.002  \set delta random(-5000, 5000)
         0.456  BEGIN;
         2.123  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         1.234  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         1.567  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         1.890  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         1.234  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         1.567  END;

# 关键指标:
# 1. TPS (Transactions Per Second) - 每秒事务数
# 2. Latency Average - 平均响应时间
# 3. Latency Stddev - 响应时间标准差 (越小越稳定)
# 4. Statement Latencies - 各语句耗时分布
```

---

## 2. sysbench - 通用基准测试工具

### 2.1 安装sysbench

```bash
# CentOS/RHEL
yum install sysbench

# Ubuntu/Debian
apt-get install sysbench

# 验证安装
sysbench --version
```

### 2.2 PostgreSQL OLTP测试

```bash
# 准备测试数据
sysbench \
  --db-driver=pgsql \
  --pgsql-host=localhost \
  --pgsql-port=5432 \
  --pgsql-user=postgres \
  --pgsql-password=password \
  --pgsql-db=sysbench_test \
  --tables=10 \
  --table-size=1000000 \
  /usr/share/sysbench/oltp_read_write.lua prepare

# 运行测试
sysbench \
  --db-driver=pgsql \
  --pgsql-host=localhost \
  --pgsql-port=5432 \
  --pgsql-user=postgres \
  --pgsql-password=password \
  --pgsql-db=sysbench_test \
  --threads=16 \
  --time=60 \
  --report-interval=10 \
  /usr/share/sysbench/oltp_read_write.lua run

# 清理测试数据
sysbench ... oltp_read_write.lua cleanup
```

### 2.3 内置测试场景

```bash
# OLTP只读测试
sysbench --threads=16 --time=60 /usr/share/sysbench/oltp_read_only.lua run

# OLTP只写测试
sysbench --threads=16 --time=60 /usr/share/sysbench/oltp_write_only.lua run

# OLTP读写混合
sysbench --threads=16 --time=60 /usr/share/sysbench/oltp_read_write.lua run

# Point Select测试
sysbench --threads=16 --time=60 /usr/share/sysbench/select_random_points.lua run

# 范围查询测试
sysbench --threads=16 --time=60 /usr/share/sysbench/select_random_ranges.lua run
```

---

## 3. 自定义压测脚本

### 3.1 Python压测脚本

```python
#!/usr/bin/env python3
# pg_stress_test.py

import psycopg2
import time
import random
from multiprocessing import Pool
from datetime import datetime

DB_CONFIG = {
    'host': 'localhost',
    'port': 5432,
    'database': 'testdb',
    'user': 'postgres',
    'password': 'password'
}

def worker_task(worker_id):
    """单个worker的任务"""
    conn = psycopg2.connect(**DB_CONFIG)
    cursor = conn.cursor()
    
    success_count = 0
    error_count = 0
    total_time = 0
    
    start_time = time.time()
    duration = 60  # 运行60秒
    
    while time.time() - start_time < duration:
        query_start = time.time()
        try:
            user_id = random.randint(1, 1000000)
            cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
            result = cursor.fetchall()
            success_count += 1
        except Exception as e:
            error_count += 1
            print(f"Worker {worker_id} error: {e}")
        finally:
            query_time = time.time() - query_start
            total_time += query_time
    
    cursor.close()
    conn.close()
    
    return {
        'worker_id': worker_id,
        'success': success_count,
        'error': error_count,
        'avg_time': total_time / success_count if success_count > 0 else 0
    }

def main():
    num_workers = 20  # 并发数
    
    print(f"Starting stress test with {num_workers} workers...")
    test_start = time.time()
    
    with Pool(num_workers) as pool:
        results = pool.map(worker_task, range(num_workers))
    
    test_duration = time.time() - test_start
    
    # 汇总结果
    total_success = sum(r['success'] for r in results)
    total_error = sum(r['error'] for r in results)
    avg_time = sum(r['avg_time'] for r in results) / len(results)
    
    print(f"\n=== Test Results ===")
    print(f"Duration: {test_duration:.2f}s")
    print(f"Total Transactions: {total_success}")
    print(f"Total Errors: {total_error}")
    print(f"TPS: {total_success / test_duration:.2f}")
    print(f"Average Latency: {avg_time * 1000:.2f}ms")

if __name__ == '__main__':
    main()
```

### 3.2 Bash压测脚本

```bash
#!/bin/bash
# simple_pg_bench.sh

PGHOST="localhost"
PGPORT="5432"
PGUSER="postgres"
PGDB="testdb"
DURATION=60
CONCURRENCY=10

echo "Starting PostgreSQL stress test..."
echo "Concurrency: $CONCURRENCY"
echo "Duration: ${DURATION}s"

# 单个worker函数
worker() {
    local worker_id=$1
    local count=0
    local start=$(date +%s)
    
    while [ $(($(date +%s) - start)) -lt $DURATION ]; do
        psql -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDB -t -c \
            "SELECT * FROM users WHERE id = floor(random()*1000000);" > /dev/null 2>&1
        ((count++))
    done
    
    echo "Worker $worker_id: $count queries"
}

# 启动并发worker
for i in $(seq 1 $CONCURRENCY); do
    worker $i &
done

# 等待所有worker完成
wait

echo "Test completed!"
```

---

## 4. 性能基线建立

### 4.1 建立基线的步骤

```bash
# Step 1: 准备环境
# • 确保系统空闲
# • 重启PostgreSQL
# • 清空缓存

# Step 2: 运行基准测试
pgbench -i -s 100 pgbench_baseline
pgbench -c 20 -j 4 -T 300 --progress=30 pgbench_baseline > baseline_$(date +%Y%m%d).log

# Step 3: 记录配置
psql -c "SELECT name, setting FROM pg_settings WHERE name IN (
    'shared_buffers', 'effective_cache_size', 'work_mem',
    'maintenance_work_mem', 'max_connections', 'random_page_cost'
);" > config_baseline.txt

# Step 4: 记录系统信息
uname -a > system_info.txt
cat /proc/cpuinfo | grep "model name" | head -1 >> system_info.txt
free -h >> system_info.txt
df -h >> system_info.txt

# Step 5: 保存结果
mkdir -p baseline/$(date +%Y%m%d)
mv baseline_*.log config_baseline.txt system_info.txt baseline/$(date +%Y%m%d)/
```

### 4.2 对比测试

```bash
# 优化后再次测试
pgbench -c 20 -j 4 -T 300 --progress=30 pgbench_baseline > optimized_$(date +%Y%m%d).log

# 对比结果
echo "=== Baseline vs Optimized ==="
echo "Baseline TPS:"
grep "tps = " baseline/*/baseline_*.log | grep "excluding"
echo ""
echo "Optimized TPS:"
grep "tps = " optimized_*.log | grep "excluding"
```

---

## 5. 性能测试场景

### 5.1 OLTP场景测试

```sql
-- oltp_workload.sql
-- 模拟典型OLTP负载: 80%读 + 20%写

\set user_id random(1, 1000000)
\set product_id random(1, 10000)
\set amount random(10, 1000)

BEGIN;

-- 80%概率执行查询
\if :random_var < 8
    SELECT u.id, u.username, o.order_no, o.total_amount
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    WHERE u.id = :user_id
    ORDER BY o.created_at DESC
    LIMIT 10;
\else
    -- 20%概率执行写入
    INSERT INTO orders (user_id, product_id, total_amount, status, created_at)
    VALUES (:user_id, :product_id, :amount, 'pending', now());
\endif

COMMIT;
```

### 5.2 OLAP场景测试

```sql
-- olap_workload.sql
-- 分析型查询

\set start_date '2025-01-01'::date - (random() * 30)::int
\set end_date :'start_date'::date + 7

-- 复杂聚合查询
SELECT 
    date_trunc('day', created_at) as day,
    status,
    count(*) as order_count,
    sum(total_amount) as total_revenue,
    avg(total_amount) as avg_order_value,
    percentile_cont(0.5) WITHIN GROUP (ORDER BY total_amount) as median_amount
FROM orders
WHERE created_at BETWEEN :'start_date' AND :'end_date'
GROUP BY date_trunc('day', created_at), status
ORDER BY day, status;
```

### 5.3 混合负载测试

```bash
# 模拟真实业务负载
pgbench -c 50 -j 8 -T 600 \
  -f oltp_read.sql@60 \
  -f oltp_write.sql@30 \
  -f olap_report.sql@10 \
  mydb
```

---

## 6. 压测注意事项

### 6.1 测试前准备

```bash
# [1] 清空统计信息
psql -c "SELECT pg_stat_reset();"
psql -c "SELECT pg_stat_statements_reset();"

# [2] 重启PostgreSQL (冷启动测试)
systemctl restart postgresql-17

# [3] 预热缓存 (热启动测试)
psql -c "SELECT pg_prewarm('users');"
psql -c "SELECT pg_prewarm('orders');"

# [4] 检查系统负载
uptime
iostat -x 1 3
```

### 6.2 测试期间监控

```bash
# 终端1: 运行压测
pgbench -c 20 -j 4 -T 300 --progress=10 mydb

# 终端2: 监控CPU/内存
top -c -d 1

# 终端3: 监控I/O
iostat -x 1

# 终端4: 监控PostgreSQL
watch -n 1 'psql -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"'

# 终端5: 监控慢查询
psql -c "SELECT pid, now() - query_start as duration, query FROM pg_stat_activity WHERE state = 'active' AND now() - query_start > interval '1 second' ORDER BY duration DESC;"
```

### 6.3 测试结果收集

```sql
-- 测试后收集统计
-- [1] 查询统计
SELECT 
    queryid,
    calls,
    total_exec_time,
    mean_exec_time,
    stddev_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- [2] 表统计
SELECT 
    schemaname,
    tablename,
    seq_scan,
    idx_scan,
    n_tup_ins,
    n_tup_upd,
    n_tup_del
FROM pg_stat_user_tables
ORDER BY seq_scan + idx_scan DESC
LIMIT 10;

-- [3] I/O统计
SELECT 
    schemaname,
    tablename,
    heap_blks_read,
    heap_blks_hit,
    round(heap_blks_hit::numeric * 100 / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 10;
```

---

## 7. 性能测试报告模板

```markdown
# PostgreSQL性能测试报告

## 测试环境
- PostgreSQL版本: 17.6
- 操作系统: CentOS 8
- CPU: 16 cores @ 2.4GHz
- 内存: 64GB
- 磁盘: NVMe SSD 1TB
- 网络: 10Gbps

## 数据库配置
```ini
shared_buffers = 16GB
effective_cache_size = 48GB
work_mem = 32MB
maintenance_work_mem = 2GB
max_connections = 200
random_page_cost = 1.1
```

## 测试场景
- 测试工具: pgbench
- 数据规模: scale=100 (1000万行)
- 并发数: 20
- 测试时长: 300秒

## 测试结果

### 基线测试
- TPS: 2057.6
- 平均延迟: 9.7ms
- P95延迟: 18.2ms
- P99延迟: 28.5ms

### 优化后测试
- TPS: 4523.8 (+119%)
- 平均延迟: 4.4ms (-55%)
- P95延迟: 8.9ms (-51%)
- P99延迟: 15.3ms (-46%)

## 优化措施
1. 调整shared_buffers从8GB到16GB
2. 创建复合索引优化查询
3. 调整autovacuum参数
4. 启用并行查询

## 结论
通过配置优化和索引优化，TPS提升119%，延迟降低55%，达到预期目标。

## 附录
- 详细执行计划
- 系统监控数据
- 配置对比
```

---

## 总结

### 性能测试流程

```
[1] 建立基线
    ↓
[2] 识别瓶颈
    ↓
[3] 制定优化方案
    ↓
[4] 实施优化
    ↓
[5] 对比测试
    ↓
[6] 生成报告
```

### 关键原则

1. **可重复性** - 测试条件要一致
2. **真实性** - 模拟真实业务场景
3. **持续性** - 定期进行基准测试
4. **全面性** - 监控多维度指标
5. **文档化** - 记录所有测试结果

---

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

