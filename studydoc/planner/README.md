# Planner/Optimizer 查询优化器 (精简版)

> PostgreSQL查询优化器的核心原理

**源码**: `src/backend/optimizer/`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 📚 核心概念

**Planner/Optimizer** 是PostgreSQL的**查询优化器**，负责将SQL查询转换为最优执行计划。

### 核心职责

```
✅ 路径生成 - 生成所有可能的执行路径
✅ 代价估算 - 估算每条路径的执行成本
✅ 路径选择 - 选择成本最低的路径
✅ 计划生成 - 生成最终执行计划
```

---

## 🎯 工作流程

```
SQL查询
  ↓
Parser (解析器)
  ↓
Query Tree (查询树)
  ↓
┌─────────────────────────────────┐
│   Planner/Optimizer             │
│                                  │
│  [1] 查询重写 (Rewriter)        │
│      • 展开视图                  │
│      • 应用规则                  │
│                                  │
│  [2] 路径生成 (Path Generation) │
│      • 扫描路径                  │
│      • Join路径                 │
│      • 聚合路径                  │
│                                  │
│  [3] 代价估算 (Cost Estimation) │
│      • CPU成本                  │
│      • I/O成本                  │
│      • 行数估算                  │
│                                  │
│  [4] 路径选择 (Path Selection)  │
│      • 选择最优路径              │
│      • 生成Plan Tree            │
└─────────────────────────────────┘
  ↓
Execution Plan (执行计划)
  ↓
Executor (执行器)
```

---

**详细内容**: 阅读 [01_planner_core.md](01_planner_core.md)

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

