# MVCC 完整分析文档

> PostgreSQL 17.5 Multi-Version Concurrency Control (MVCC) 深度剖析

---

## 📚 文档导航

| 序号 | 文档 | 说明 | 进度 |
|-----|------|------|------|
| 01 | [overview.md](01_overview.md) | MVCC 概述与核心概念 | ✅ |
| 02 | [data_structures.md](02_data_structures.md) | 核心数据结构详解 | ✅ |
| 03 | [implementation_flow.md](03_implementation_flow.md) | 实现流程分析 | ✅ |
| 04 | key_algorithms.md | 关键算法详解 | ⏳ |
| 05 | performance.md | 性能优化技术 | ⏳ |
| 06 | testcases.md | 测试用例集 | ⏳ |
| 07 | diagrams.md | 架构图表集 | ⏳ |

**当前完成**: 3/7 篇文档

---

## 🎯 核心概念速览

### MVCC 三大核心价值

```
1. 无锁并发
   ├─ 读不阻塞写
   ├─ 写不阻塞读
   └─ 高并发性能

2. 一致性读
   ├─ 事务级快照
   ├─ 可重复读保证
   └─ 时间点一致性

3. 简化设计
   ├─ 无需 Undo Log
   ├─ 基于多版本
   └─ 与 WAL 配合
```

### 关键数据结构

```c
// 元组头 (23 字节)
HeapTupleHeader {
    t_xmin:  插入事务 ID
    t_xmax:  删除事务 ID
    t_cid:   命令 ID
    t_ctid:  元组链指针
}

// 快照
Snapshot {
    xmin:    所有 < xmin 已提交
    xmax:    所有 >= xmax 不可见
    xip[]:   活跃事务列表
    xcnt:    活跃事务数量
}
```

---

## ⚡ 快速开始

### 查看当前事务状态

```sql
-- 查看活跃事务
SELECT pid, usename, xact_start, state, query 
FROM pg_stat_activity 
WHERE state = 'active';

-- 查看当前事务 ID
SELECT txid_current();

-- 查看事务 ID 状态
SELECT txid_status(1000);  -- committed/aborted/in progress
```

### 查看 MVCC 元数据

```sql
-- 安装 pageinspect 扩展
CREATE EXTENSION pageinspect;

-- 查看元组头信息
SELECT t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('users', 0))
LIMIT 10;

-- 查看死元组数量
SELECT relname, n_dead_tup, n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;
```

### 基础调优

```sql
-- 自动 VACUUM 配置
ALTER TABLE users SET (
    autovacuum_vacuum_scale_factor = 0.1,  -- 10% 死元组触发
    autovacuum_analyze_scale_factor = 0.05  -- 5% 变化触发
);

-- 手动 VACUUM
VACUUM VERBOSE users;

-- VACUUM FREEZE (防止事务 ID 回绕)
VACUUM FREEZE users;
```

---

## 🔬 技术深度

### 可见性判断算法

```
HeapTupleSatisfiesMVCC(tuple, snapshot):
  
  1. 检查 xmax (删除事务)
     if xmax != 0 and visible(xmax, snapshot):
       return FALSE  // 已被删除
  
  2. 检查 xmin (插入事务)
     if xmin < snapshot->xmin:
       return TRUE   // 已提交
     
     if xmin in snapshot->xip[]:
       return FALSE  // 活跃中
     
     if xmin < snapshot->xmax:
       check CLOG    // 判断提交状态
  
  3. 返回可见性结果
```

### HOT (Heap Only Tuple) 优化

```
条件:
✓ UPDATE 不修改索引键
✓ 新元组在同一页
✓ 页面有足够空间

效果:
✓ 无需更新索引 (10-100x 提升)
✓ 减少表膨胀
✓ 加速 VACUUM
```

---

## 📊 性能指标

### 关键监控指标

```sql
-- 表膨胀率
SELECT 
    schemaname || '.' || tablename AS table_name,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS bloat_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- VACUUM 统计
SELECT relname, last_vacuum, last_autovacuum, 
       vacuum_count, autovacuum_count
FROM pg_stat_user_tables
WHERE last_autovacuum IS NOT NULL
ORDER BY last_autovacuum DESC;

-- 事务 ID 使用情况
SELECT datname, age(datfrozenxid) AS xid_age,
       2^31 - age(datfrozenxid) AS xids_remaining
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

---

## 🚀 最佳实践

### 1. 避免长事务

```sql
-- 查找长事务
SELECT pid, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
  AND xact_start < now() - interval '1 hour'
ORDER BY duration DESC;

-- 终止长事务 (谨慎!)
SELECT pg_terminate_backend(pid);
```

### 2. 合理配置 VACUUM

```ini
# postgresql.conf

# 自动 VACUUM 基础配置
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 1min

# 触发阈值
autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 50
autovacuum_vacuum_scale_factor = 0.2
autovacuum_analyze_scale_factor = 0.1

# 防止事务 ID 回绕
autovacuum_freeze_max_age = 200000000
vacuum_freeze_table_age = 150000000
```

### 3. 监控表膨胀

```bash
# 定期检查
psql -c "
SELECT schemaname || '.' || tablename, 
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)),
       round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) AS bloat_pct
FROM pg_stat_user_tables 
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
"
```

---

## 🔗 与其他模块的关系

```
┌──────────────────────────────────────┐
│            MVCC System               │
└────────┬─────────────────────────────┘
         │
    ┌────┴─────┬─────────┬──────────┐
    │          │         │          │
    ▼          ▼         ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ Buffer │ │  WAL   │ │ CLOG   │ │ VACUUM │
│ Manager│ │ System │ │ (事务)  │ │ (清理) │
└────────┘ └────────┘ └────────┘ └────────┘
```

**依赖关系**:
- MVCC → Buffer Manager (读取元组)
- MVCC → WAL (持久化修改)
- MVCC → CLOG (查询事务状态)
- VACUUM → MVCC (清理死元组)

---

## 📝 常见问题

### Q1: 为什么表越来越大？

**A**: 表膨胀原因：
1. 频繁 UPDATE/DELETE
2. VACUUM 不及时
3. 长事务阻止清理
4. autovacuum 配置不当

**解决方案**:
- 检查死元组比例
- 手动 VACUUM
- 终止长事务
- 调整 autovacuum 参数
- 考虑 VACUUM FULL (需要锁表)

### Q2: 什么是事务 ID 回绕？

**A**: 
- TransactionId 是 32 位,最多 42 亿
- 用完后回绕到 3
- 旧数据可能变成"未来"
- VACUUM FREEZE 解决 (设置 t_xmin=2)

### Q3: HOT Update 什么时候失效？

**A**: HOT Update 失效场景：
1. 更新了索引键列
2. 新元组不在同一页
3. 页面空间不足
4. 表有 TOAST 数据

---

## 🎓 学习路径

### 初学者

```
1. 01_overview.md
   → 理解 MVCC 基本概念

2. 02_data_structures.md
   → 学习元组头和快照结构

3. 查看实际数据
   → 使用 pageinspect 查看元组
```

### 进阶

```
1. 03_implementation_flow.md
   → 掌握 INSERT/UPDATE/DELETE 流程

2. 性能调优
   → VACUUM 配置和监控

3. 源码阅读
   → heapam.c, snapmgr.c
```

### 专家

```
1. 深入算法
   → 可见性判断优化

2. 极端场景
   → 事务 ID 回绕处理

3. 源码贡献
   → 优化 MVCC 性能
```

---

## 📚 相关资源

### 官方文档
- [MVCC Introduction](https://www.postgresql.org/docs/17/mvcc-intro.html)
- [Routine Vacuuming](https://www.postgresql.org/docs/17/routine-vacuuming.html)

### 推荐阅读
- PostgreSQL 内核分析 - MVCC 章节
- "PostgreSQL: Up and Running"
- Tom Lane 的 MVCC 演讲

### 相关工具
- pageinspect: 查看页面内部结构
- pg_visibility: 可见性映射分析
- pgstattuple: 表膨胀分析

---

## 更新日志

### v1.0 (2025-01-16)
- ✅ 完成核心数据结构分析
- ✅ 完成实现流程分析
- ⏳ 关键算法文档进行中

---

**文档版本**: 1.0
**基于源码**: PostgreSQL 17.5
**创建日期**: 2025-01-16

**相关模块**: 
- [Buffer Manager](../buffer_manager/) - 缓冲池管理
- [WAL System](../wal/) - 预写式日志
- [Checkpoint](../checkpooint/) - 检查点系统


