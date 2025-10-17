# PostgreSQL 性能调优实战指南

> PostgreSQL性能调优的系统化方法论和实战案例集

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16  
**难度**: 中级到高级

---

## 📚 文档导航

### 核心文档

1. **[性能诊断方法论](01_diagnosis_methodology.md)** 🔍
   - 性能问题分类
   - 诊断工具和方法
   - 性能指标体系
   - 问题定位流程

2. **[实战案例集](02_real_world_cases.md)** 💼
   - 慢查询优化案例
   - 高并发优化案例
   - 存储优化案例
   - 锁争用解决案例
   - 连接池优化案例

3. **[监控和工具](03_monitoring_tools.md)** 📊
   - 系统级监控
   - PostgreSQL内置监控
   - 第三方监控工具
   - 性能分析工具

4. **[调优最佳实践](04_best_practices.md)** ⭐
   - 参数配置建议
   - SQL优化原则
   - 架构设计建议
   - 容量规划

5. **[性能测试和基准](05_benchmarking.md)** 📈
   - pgbench使用
   - sysbench使用
   - 自定义测试场景
   - 性能基线建立

---

## 🎯 学习路径

### 🔰 入门级 (1周)

```
Day 1-2: 性能诊断方法论
  • 学习EXPLAIN分析
  • 掌握基础监控SQL
  • 理解性能指标

Day 3-4: 常见问题排查
  • 慢查询优化
  • 索引优化
  • 参数调优

Day 5-7: 实战练习
  • 分析实际案例
  • 动手优化测试
  • 总结经验
```

### 🚀 进阶级 (2-3周)

```
Week 1: 深度优化
  • 执行计划深度分析
  • 高级索引策略
  • 查询重写技巧

Week 2: 系统优化
  • 操作系统调优
  • 存储优化
  • 内存管理

Week 3: 架构优化
  • 分区表设计
  • 读写分离
  • 连接池优化
```

### 🏆 专家级 (持续实践)

```
• 大规模集群优化
• 复杂查询优化
• 性能容量规划
• 故障快速定位
```

---

## 🔥 快速开始

### 1. 检查当前性能状态

```sql
-- 查看最慢的10条查询
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- 查看表膨胀情况
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio
FROM pg_stat_user_tables
WHERE n_live_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 10;

-- 查看缓冲池命中率
SELECT 
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100 as cache_hit_ratio
FROM pg_statio_user_tables;
```

### 2. 基础配置检查

```sql
-- 查看关键参数
SELECT name, setting, unit, source 
FROM pg_settings 
WHERE name IN (
    'shared_buffers',
    'effective_cache_size',
    'work_mem',
    'maintenance_work_mem',
    'max_connections',
    'random_page_cost',
    'effective_io_concurrency'
);
```

### 3. 使用EXPLAIN分析

```sql
-- 分析查询执行计划
EXPLAIN (ANALYZE, BUFFERS, TIMING, VERBOSE) 
SELECT * FROM users WHERE created_at > now() - interval '1 day';
```

---

## 📊 性能优化检查清单

### ✅ 数据库级别

- [ ] shared_buffers设置为内存的25%
- [ ] effective_cache_size设置为内存的50-75%
- [ ] work_mem根据并发调整
- [ ] maintenance_work_mem设置为1-2GB
- [ ] random_page_cost (SSD: 1.1, HDD: 4.0)
- [ ] max_connections合理设置
- [ ] 开启pg_stat_statements扩展

### ✅ 查询级别

- [ ] 使用EXPLAIN ANALYZE分析
- [ ] 创建合适的索引
- [ ] 避免SELECT *
- [ ] 使用JOIN代替子查询
- [ ] 分页使用WHERE代替OFFSET
- [ ] 避免隐式类型转换

### ✅ 表级别

- [ ] 定期VACUUM
- [ ] 设置合适的fillfactor
- [ ] 合理使用分区表
- [ ] 统计信息保持最新
- [ ] 监控表膨胀

### ✅ 索引级别

- [ ] 删除未使用的索引
- [ ] 多列索引注意顺序
- [ ] 使用部分索引
- [ ] 定期REINDEX
- [ ] 监控索引膨胀

### ✅ 系统级别

- [ ] I/O调度器优化
- [ ] 文件系统选择(XFS/EXT4)
- [ ] 禁用透明大页
- [ ] 调整vm.swappiness
- [ ] 配置网络参数

---

## 🛠️ 常用工具

### 分析工具

```bash
# pg_stat_statements - 查询统计
# pgBadger - 日志分析
# pg_top - 实时监控
# pg_stat_kcache - 系统调用统计
```

### 测试工具

```bash
# pgbench - 基准测试
# sysbench - 压力测试
# ab/wrk - HTTP压测
```

### 监控工具

```bash
# Prometheus + Grafana
# pgAdmin 4
# DataDog
# New Relic
```

---

## 🎯 性能优化流程

```
[1] 性能问题识别
    ↓
[2] 收集性能数据
    ↓
[3] 分析根本原因
    ↓
[4] 制定优化方案
    ↓
[5] 实施优化措施
    ↓
[6] 验证优化效果
    ↓
[7] 持续监控
```

---

## 💡 关键原则

1. **测量优先** - 先测量再优化，避免过早优化
2. **基准对比** - 优化前后对比，量化改进效果
3. **逐步优化** - 一次改一个变量，避免混淆
4. **文档记录** - 记录所有变更和效果
5. **持续监控** - 建立监控体系，及早发现问题

---

## 📖 相关文档

- [Buffer Manager](../buffer_manager/README.md) - 缓冲池优化
- [WAL System](../wal/README.md) - WAL性能优化
- [MVCC](../mvcc/README.md) - 并发控制优化
- [Lock Manager](../lock/README.md) - 锁优化
- [VACUUM](../vacuum/README.md) - 清理优化
- [Planner/Optimizer](../planner/README.md) - 查询优化

---

**开始学习**: 从 [01_diagnosis_methodology.md](01_diagnosis_methodology.md) 开始！

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

