# VACUUM 垃圾清理模块

> PostgreSQL VACUUM机制的深度分析

**模块状态**: ✅ 完成  
**文档版本**: 1.0  
**最后更新**: 2025-10-16  
**基于版本**: PostgreSQL 17.6

---

## 📚 模块导航

| 文档 | 内容概要 | 重点 |
|------|---------|------|
| [01_overview.md](01_overview.md) | VACUUM核心原理 | ⭐⭐⭐⭐⭐ |
| [02_lazy_vacuum.md](02_lazy_vacuum.md) | Lazy VACUUM算法 | ⭐⭐⭐⭐⭐ |
| [03_autovacuum.md](03_autovacuum.md) | Autovacuum机制 | ⭐⭐⭐⭐ |
| [04_performance.md](04_performance.md) | 性能优化和调优 | ⭐⭐⭐⭐ |

---

## 🎯 VACUUM是什么？

**VACUUM**是PostgreSQL的垃圾回收机制，负责清理MVCC产生的死元组。

### 为什么需要VACUUM？

```
MVCC机制导致:
  UPDATE = 插入新版本 + 标记旧版本删除
  DELETE = 标记元组删除
  
  ↓
  
死元组累积:
  • 占用磁盘空间 (表膨胀)
  • 减慢查询速度 (需扫描死元组)
  • 浪费缓冲池 (缓存死元组)
  
  ↓
  
VACUUM清理:
  ✅ 回收死元组空间
  ✅ 更新统计信息
  ✅ 防止事务ID回卷
  ✅ 更新可见性映射 (VM)
```

---

## 🔑 核心概念

### 1. 死元组 (Dead Tuple)

```sql
-- 查看死元组数量
SELECT 
    schemaname,
    relname,
    n_live_tup,      -- 活元组
    n_dead_tup,      -- 死元组
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;
```

### 2. VACUUM类型

```
┌─────────────────┬────────────────┬────────────────┬──────────┐
│   VACUUM类型    │  空间回收      │  是否阻塞      │  推荐    │
├─────────────────┼────────────────┼────────────────┼──────────┤
│ VACUUM          │ 标记可重用     │ 不阻塞读写     │ ✅ 常用  │
│ VACUUM FULL     │ 完全回收       │ 阻塞所有操作   │ ⚠️ 慎用  │
│ Autovacuum      │ 自动VACUUM     │ 不阻塞         │ ✅ 推荐  │
└─────────────────┴────────────────┴────────────────┴──────────┘
```

### 3. 三种操作

```
1. VACUUM (Lazy Vacuum)
   • 标记死元组空间为可重用
   • 不归还空间给操作系统
   • 更新FSM (空闲空间映射)
   • 不阻塞读写
   
2. VACUUM FULL
   • 重建整个表和索引
   • 完全回收空间
   • 需要AccessExclusiveLock (阻塞所有操作)
   • 需要额外的磁盘空间
   
3. Autovacuum
   • 后台自动执行VACUUM
   • 根据死元组比例触发
   • 默认启用
```

---

## 🚀 快速开始

### 1. 手动VACUUM

```sql
-- 基础VACUUM
VACUUM table_name;

-- VACUUM并显示详情
VACUUM VERBOSE table_name;

-- VACUUM并ANALYZE (更新统计信息)
VACUUM ANALYZE table_name;

-- VACUUM所有表
VACUUM;

-- VACUUM FULL (慎用!)
VACUUM FULL table_name;
```

### 2. 检查是否需要VACUUM

```sql
-- 查看表膨胀
SELECT 
    schemaname || '.' || tablename AS table_name,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_autovacuum,
    CASE 
        WHEN last_autovacuum IS NULL THEN 'NEVER'
        ELSE (now() - last_autovacuum)::text
    END AS last_autovacuum_age
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- 建议: 
-- dead_pct > 20% → 需要VACUUM
-- dead_pct > 50% → 考虑VACUUM FULL或pg_repack
```

### 3. 配置Autovacuum

```sql
-- 全局配置
ALTER SYSTEM SET autovacuum = on;                          -- 启用
ALTER SYSTEM SET autovacuum_naptime = '1min';              -- 检查间隔
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.2;     -- 20%死元组触发
ALTER SYSTEM SET autovacuum_max_workers = 3;               -- 最多3个worker

-- 表级配置
ALTER TABLE hot_table SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- 5%触发 (更频繁)
    autovacuum_vacuum_threshold = 100,
    autovacuum_analyze_scale_factor = 0.02
);

-- 禁用某表的autovacuum (不推荐)
ALTER TABLE read_only_table SET (autovacuum_enabled = false);
```

---

## 📊 监控查询

### 1. Autovacuum活动

```sql
-- 查看正在运行的autovacuum
SELECT 
    pid,
    datname,
    usename,
    state,
    wait_event_type,
    wait_event,
    query_start,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE backend_type = 'autovacuum worker'
ORDER BY query_start;
```

### 2. VACUUM统计

```sql
-- 查看表的VACUUM历史
SELECT 
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
ORDER BY last_autovacuum DESC NULLS LAST;
```

### 3. 表膨胀估算

```sql
-- 使用pgstattuple扩展
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- 检查表膨胀
SELECT 
    schemaname || '.' || tablename AS table_name,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS size,
    round(100 * pg_relation_size(schemaname||'.'||tablename)::numeric / 
          NULLIF(pg_total_relation_size(schemaname||'.'||tablename), 0), 2) AS table_pct,
    round((pgstattuple(schemaname||'.'||tablename)).dead_tuple_percent, 2) AS dead_pct
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
  AND pg_relation_size(schemaname||'.'||tablename) > 1048576  -- > 1MB
ORDER BY pg_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
```

---

## ⚠️ 常见问题

### Q1: VACUUM和VACUUM FULL的区别？

**A**: 
- **VACUUM**: 标记空间可重用，不归还OS，不阻塞
- **VACUUM FULL**: 完全重建表，归还空间，阻塞所有操作

**推荐**: 
- 日常使用VACUUM
- 严重膨胀时使用`pg_repack`替代VACUUM FULL

### Q2: Autovacuum为什么没运行？

**A**: 检查：
```sql
-- 1. 是否启用?
SHOW autovacuum;

-- 2. 是否达到阈值?
SELECT 
    relname,
    n_dead_tup,
    n_live_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) AS dead_pct,
    pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_stat_user_tables
WHERE n_dead_tup > 0;

-- 3. 查看autovacuum日志
-- 设置: log_autovacuum_min_duration = 0
```

### Q3: VACUUM运行很慢怎么办？

**A**: 调优参数：
```sql
-- 增加维护内存
SET maintenance_work_mem = '1GB';

-- 降低延迟 (更激进)
SET vacuum_cost_delay = 0;  -- 0=不延迟

-- 增加成本限制
SET vacuum_cost_limit = 2000;  -- 默认200
```

### Q4: 事务ID回卷是什么？

**A**: 
```
PostgreSQL事务ID是32位: 0 ~ 2^32-1

如果表很久不VACUUM:
  → 老事务ID被"回卷"
  → 旧数据突然变成"未来"数据
  → 数据对所有事务"不可见" (数据丢失!)

预防:
  → autovacuum_freeze_max_age = 200,000,000
  → 强制VACUUM FREEZE

监控:
SELECT datname, age(datfrozenxid) 
FROM pg_database 
ORDER BY age(datfrozenxid) DESC;
-- 警告: age > 2,000,000,000
```

---

## 📖 学习路径

### 基础 (1-2天)
1. 阅读 [01_overview.md](01_overview.md) - 理解VACUUM原理
2. 实践手动VACUUM
3. 监控死元组和表膨胀

### 进阶 (3-5天)
4. 阅读 [02_lazy_vacuum.md](02_lazy_vacuum.md) - 深入算法
5. 阅读 [03_autovacuum.md](03_autovacuum.md) - 自动化机制
6. 调优autovacuum参数

### 专家 (1-2周)
7. 阅读 [04_performance.md](04_performance.md) - 性能优化
8. 源码阅读: `src/backend/commands/vacuum.c`
9. 处理严重表膨胀场景

---

## 🔗 相关模块

- **MVCC**: VACUUM清理MVCC产生的死元组
- **HEAP**: VACUUM操作堆表页面
- **FSM/VM**: VACUUM更新空闲空间和可见性映射
- **Autovacuum**: 后台进程自动执行VACUUM
- **Transaction**: 事务ID回卷预防

---

**下一步**: 开始阅读 [01_overview.md](01_overview.md)

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-10-16

