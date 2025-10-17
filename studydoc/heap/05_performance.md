# HEAP 性能优化

本文档说明堆表的性能优化技术、参数调优和最佳实践。

---

## 1. Fillfactor 优化

### 1.1 原理

```
fillfactor: 控制页面填充程度，为HOT Update预留空间

默认: 100 (填满)
效果:
  - 高 fillfactor (95-100): 节省空间，但HOT Update少
  - 低 fillfactor (70-80): HOT Update多，但浪费空间
```

### 1.2 推荐设置

```sql
-- 频繁UPDATE的表
ALTER TABLE hot_table SET (fillfactor = 70);

-- 只读/追加表
ALTER TABLE readonly_table SET (fillfactor = 100);

-- 中等UPDATE频率
ALTER TABLE moderate_table SET (fillfactor = 85);

-- 重建表使设置生效
VACUUM FULL hot_table;
-- 或
CLUSTER hot_table USING hot_table_pkey;
```

### 1.3 监控HOT比例

```sql
SELECT 
    schemaname,
    relname,
    n_tup_upd AS total_upd,
    n_tup_hot_upd AS hot_upd,
    round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 2) AS hot_pct
FROM pg_stat_user_tables
WHERE n_tup_upd > 1000
ORDER BY n_tup_upd DESC;

-- 目标: hot_pct > 80%
-- 如果 < 50%: 考虑降低 fillfactor
```

---

## 2. 表膨胀处理

### 2.1 检测表膨胀

```sql
-- 使用 pgstattuple 扩展
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- 检查表膨胀
SELECT 
    schemaname || '.' || tablename AS table_name,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    round(100 * (pg_total_relation_size(schemaname||'.'||tablename) - 
           pg_relation_size(schemaname||'.'||tablename))::numeric / 
           NULLIF(pg_total_relation_size(schemaname||'.'||tablename), 0), 2) AS index_ratio,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- 死元组 > 20%: 需要VACUUM
-- 死元组 > 50%: 需要VACUUM FULL
```

### 2.2 VACUUM策略

```sql
-- 常规VACUUM (不阻塞读写)
VACUUM VERBOSE tablename;

-- VACUUM FULL (重建表，阻塞所有操作)
VACUUM FULL tablename;

-- 并发重建 (推荐，使用pg_repack)
pg_repack -t tablename dbname

-- 调整autovacuum参数
ALTER TABLE tablename SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- 5%死元组触发
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.02
);
```

---

## 3. TOAST优化

### 3.1 TOAST策略选择

```sql
-- 查看列的TOAST策略
SELECT attname, atttypid::regtype, attstorage
FROM pg_attribute
WHERE attrelid = 'your_table'::regclass AND attnum > 0;

-- attstorage 值:
-- 'p' = PLAIN (不压缩，不TOAST)
-- 'e' = EXTERNAL (不压缩，外部存储)
-- 'x' = EXTENDED (压缩+外部，默认)
-- 'm' = MAIN (压缩，优先主表)

-- 修改策略
-- 场景1: 大文本，已压缩 (如JSON) → EXTERNAL
ALTER TABLE logs ALTER COLUMN payload SET STORAGE EXTERNAL;

-- 场景2: 小文本，频繁访问 → MAIN
ALTER TABLE users ALTER COLUMN bio SET STORAGE MAIN;

-- 场景3: 大二进制，不可压缩 → EXTERNAL
ALTER TABLE files ALTER COLUMN content SET STORAGE EXTERNAL;
```

### 3.2 TOAST表监控

```sql
-- 查看TOAST表大小
SELECT 
    c.relname AS table_name,
    t.relname AS toast_table,
    pg_size_pretty(pg_relation_size(c.oid)) AS table_size,
    pg_size_pretty(pg_relation_size(t.oid)) AS toast_size,
    round(100.0 * pg_relation_size(t.oid) / 
          NULLIF(pg_relation_size(c.oid) + pg_relation_size(t.oid), 0), 2) AS toast_pct
FROM pg_class c
LEFT JOIN pg_class t ON c.reltoastrelid = t.oid
WHERE c.relkind = 'r' AND c.relnamespace = 'public'::regnamespace
  AND t.relname IS NOT NULL
ORDER BY pg_relation_size(t.oid) DESC;

-- 如果 toast_pct > 50%: 考虑优化TOAST策略
```

---

## 4. 索引优化

### 4.1 减少索引数量

```sql
-- 过多索引影响UPDATE/INSERT性能
-- 每个索引在UPDATE时都需要维护

-- 查看表的索引
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,  -- 索引使用次数
    pg_size_pretty(pg_relation_size(indexrelid)) AS idx_size
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan;

-- 删除未使用的索引 (idx_scan = 0)
DROP INDEX IF EXISTS unused_index;
```

### 4.2 部分索引

```sql
-- 替代全表索引，减少索引大小和维护代价

-- 示例: 只索引活跃订单
CREATE INDEX idx_active_orders ON orders(created_at)
WHERE status IN ('pending', 'processing');

-- 效果: 索引大小减少 80%, UPDATE速度提升 50%
```

---

## 5. 分区表优化

### 5.1 分区策略

```sql
-- 按时间分区 (常见场景)
CREATE TABLE measurements (
    time TIMESTAMP NOT NULL,
    value NUMERIC
) PARTITION BY RANGE (time);

-- 按月分区
CREATE TABLE measurements_2024_01 PARTITION OF measurements
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE measurements_2024_02 PARTITION OF measurements
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- 优势:
-- 1. 查询分区裁剪 (只扫描相关分区)
-- 2. 快速删除老数据 (DROP PARTITION)
-- 3. 分散I/O负载
```

### 5.2 分区监控

```sql
-- 查看各分区大小
SELECT 
    parent.relname AS parent_table,
    child.relname AS partition_name,
    pg_size_pretty(pg_relation_size(child.oid)) AS size
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
WHERE parent.relname = 'measurements'
ORDER BY child.relname;
```

---

## 6. 批量操作优化

### 6.1 COPY vs INSERT

```sql
-- 慢: 逐行INSERT
INSERT INTO table VALUES (1, 'a');
INSERT INTO table VALUES (2, 'b');
-- ...
-- 速度: ~1000 行/秒

-- 快: 批量INSERT
INSERT INTO table VALUES (1, 'a'), (2, 'b'), (3, 'c'), ...;
-- 速度: ~10,000 行/秒

-- 最快: COPY
COPY table FROM '/path/to/data.csv' CSV;
-- 速度: ~50,000 行/秒

-- COPY性能优化
SET maintenance_work_mem = '1GB';  -- 增加内存
ALTER TABLE table SET (autovacuum_enabled = false);  -- 临时禁用autovacuum
-- 执行COPY
-- 重建索引
CREATE INDEX CONCURRENTLY idx_col ON table(col);
ALTER TABLE table SET (autovacuum_enabled = true);
```

---

## 7. 关键参数调优

### 7.1 表级参数

```sql
ALTER TABLE hot_table SET (
    fillfactor = 70,                            -- HOT Update预留空间
    autovacuum_vacuum_scale_factor = 0.05,      -- 5%死元组触发VACUUM
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.02,     -- 2%变化触发ANALYZE
    autovacuum_vacuum_cost_delay = 10,          -- VACUUM延迟(ms)
    toast_tuple_target = 2048                   -- TOAST阈值
);
```

### 7.2 系统参数

```sql
-- shared_buffers: 影响页面缓存
shared_buffers = '4GB'  -- 通常设为RAM的25%

-- effective_cache_size: 影响查询计划
effective_cache_size = '12GB'  -- 通常设为RAM的50-75%

-- maintenance_work_mem: 影响VACUUM/CREATE INDEX速度
maintenance_work_mem = '1GB'

-- autovacuum_max_workers: 并发VACUUM进程数
autovacuum_max_workers = 4
```

---

## 8. 监控查询汇总

```sql
-- 综合健康检查
SELECT 
    schemaname || '.' || relname AS table_name,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS total_size,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    n_tup_upd,
    n_tup_hot_upd,
    round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 2) AS hot_pct,
    last_vacuum,
    last_autovacuum,
    CASE 
        WHEN last_vacuum IS NULL AND last_autovacuum IS NULL 
        THEN 'NEVER'
        WHEN last_vacuum > last_autovacuum OR last_autovacuum IS NULL
        THEN 'Manual: ' || (now() - last_vacuum)::TEXT
        ELSE 'Auto: ' || (now() - last_autovacuum)::TEXT
    END AS last_vacuum_age
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||relname) DESC
LIMIT 20;
```

---

## 9. 性能基准

### 9.1 典型指标

| 操作 | 优化前 | 优化后 | 提升 |
|------|--------|--------|------|
| UPDATE (HOT) | 1000 TPS | 5000 TPS | 5x |
| INSERT | 10K TPS | 50K TPS (COPY) | 5x |
| SELECT (索引) | 100 QPS | 1000 QPS (Index-Only) | 10x |
| VACUUM | 10 MB/s | 100 MB/s (调优) | 10x |

---

## 总结

### 优化清单

- ✅ 设置合适的 fillfactor (频繁UPDATE: 70-80)
- ✅ 监控HOT Update比例 (目标 > 80%)
- ✅ 定期VACUUM清理死元组
- ✅ 优化TOAST策略 (大字段用EXTERNAL)
- ✅ 删除未使用的索引
- ✅ 使用部分索引减少维护代价
- ✅ 批量操作用COPY而非INSERT
- ✅ 考虑分区表 (大表, >100GB)

---

**下一步**: 阅读 [06_testcases.md](06_testcases.md) 查看测试用例

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

