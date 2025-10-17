# Transaction Manager (事务管理器) 模块

## 📖 模块简介

Transaction Manager 是 PostgreSQL 实现 ACID 特性的核心组件，管理事务的整个生命周期，从事务开始到提交/回滚。它采用**三层架构设计**，实现了复杂的事务控制逻辑。

### 核心特性

- ✅ **三层事务系统**: 低层事务、中层控制、高层命令
- ✅ **事务状态机**: 双层状态管理 (TransState + TBlockState)
- ✅ **子事务支持**: 通过 SAVEPOINT 实现嵌套事务
- ✅ **两阶段提交**: 支持分布式事务 (2PC)
- ✅ **事务ID管理**: 64位 FullTransactionId，防止回卷
- ✅ **并发控制**: 与 MVCC 紧密集成
- ✅ **资源管理**: 自动清理事务资源

### 模块依赖关系

```
┌──────────────────────────────────────────┐
│         应用层 (SQL Commands)             │
│   BEGIN / COMMIT / ROLLBACK / SAVEPOINT  │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│       Transaction Manager (本模块)        │
│  ┌────────────────────────────────────┐  │
│  │ 高层: BeginTransactionBlock()      │  │
│  │       EndTransactionBlock()        │  │
│  │       DefineSavepoint()            │  │
│  ├────────────────────────────────────┤  │
│  │ 中层: StartTransactionCommand()    │  │
│  │       CommitTransactionCommand()   │  │
│  ├────────────────────────────────────┤  │
│  │ 低层: StartTransaction()           │  │
│  │       CommitTransaction()          │  │
│  │       AbortTransaction()           │  │
│  └────────────────────────────────────┘  │
└────────────────┬─────────────────────────┘
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
    ┌──────┐ ┌──────┐ ┌──────┐
    │ MVCC │ │ WAL  │ │ Lock │
    │      │ │      │ │ Mgr  │
    └──────┘ └──────┘ └──────┘
```

---

## 📚 文档导航

### 🎯 新手入门路径

1. **第一步**: [01_overview.md](01_overview.md) - 了解事务系统架构
2. **第二步**: [02_data_structures.md](02_data_structures.md) - 理解核心数据结构
3. **第三步**: [03_implementation_flow.md](03_implementation_flow.md) - 掌握事务流程
4. **第四步**: [07_diagrams.md](07_diagrams.md) - 查看完整的架构图

### 📄 完整文档列表

| 序号 | 文档名称 | 内容概要 | 适合人群 |
|------|----------|----------|----------|
| 1 | [01_overview.md](01_overview.md) | 事务系统概述、三层架构、核心概念 | 所有人 ⭐ |
| 2 | [02_data_structures.md](02_data_structures.md) | TransactionState、PGPROC、全局变量 | 开发者 |
| 3 | [03_implementation_flow.md](03_implementation_flow.md) | BEGIN/COMMIT/ROLLBACK/SAVEPOINT 详细流程 | 开发者 ⭐ |
| 4 | [04_key_algorithms.md](04_key_algorithms.md) | 事务ID分配、子事务、两阶段提交 | 高级开发者 |
| 5 | [05_performance.md](05_performance.md) | 性能优化、参数调优、最佳实践 | DBA |
| 6 | [06_testcases.md](06_testcases.md) | 完整测试用例集合 | 测试工程师 |
| 7 | [07_diagrams.md](07_diagrams.md) | 状态机图、流程图、架构图 | 所有人 ⭐ |

---

## 🚀 快速开始

### 基本事务操作

```sql
-- 1. 开始事务
BEGIN;

-- 2. 执行操作
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- 3. 提交事务
COMMIT;

-- 或回滚事务
ROLLBACK;
```

### 子事务 (SAVEPOINT)

```sql
BEGIN;
    INSERT INTO orders (id, amount) VALUES (1, 100);
    
    SAVEPOINT sp1;
    UPDATE inventory SET stock = stock - 10 WHERE product_id = 1;
    
    -- 如果库存不足，回滚到保存点
    ROLLBACK TO SAVEPOINT sp1;
    
    -- 继续其他操作
    INSERT INTO logs (message) VALUES ('Order processed');
COMMIT;
```

### 两阶段提交 (2PC)

```sql
-- 阶段1: 准备事务
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
PREPARE TRANSACTION 'txn_001';

-- 阶段2: 提交准备好的事务
COMMIT PREPARED 'txn_001';

-- 或回滚
ROLLBACK PREPARED 'txn_001';
```

---

## 🔍 核心概念速查

### 事务状态机

PostgreSQL 使用**两层状态机**管理事务：

1. **TransState** (低层状态):
   - `TRANS_DEFAULT` - 空闲状态
   - `TRANS_START` - 事务启动中
   - `TRANS_INPROGRESS` - 事务进行中
   - `TRANS_COMMIT` - 提交中
   - `TRANS_ABORT` - 回滚中
   - `TRANS_PREPARE` - 准备中 (2PC)

2. **TBlockState** (高层状态):
   - `TBLOCK_DEFAULT` - 未在事务块中
   - `TBLOCK_STARTED` - 单查询事务
   - `TBLOCK_INPROGRESS` - 事务块进行中
   - `TBLOCK_END` - 收到 COMMIT
   - `TBLOCK_ABORT` - 事务失败
   - 子事务状态: `TBLOCK_SUBBEGIN`, `TBLOCK_SUBINPROGRESS` 等

### 核心数据结构

```c
// 事务状态 (每个事务一个)
typedef struct TransactionStateData {
    FullTransactionId fullTransactionId;  // 完整事务ID
    SubTransactionId subTransactionId;    // 子事务ID
    char *name;                           // SAVEPOINT 名称
    TransState state;                     // 低层状态
    TBlockState blockState;               // 高层状态
    int nestingLevel;                     // 嵌套层次
    TransactionId *childXids;             // 子事务ID数组
    struct TransactionStateData *parent;  // 父事务指针
} TransactionStateData;

// 进程结构 (每个后端一个)
struct PGPROC {
    TransactionId xid;       // 当前事务ID
    TransactionId xmin;      // 最小活跃事务ID
    int pid;                 // 进程ID
    // ... 更多字段
};
```

### 三层事务系统

| 层次 | 函数 | 调用时机 |
|------|------|----------|
| **高层** (用户命令) | `BeginTransactionBlock()` | 执行 BEGIN |
| | `EndTransactionBlock()` | 执行 COMMIT |
| | `UserAbortTransactionBlock()` | 执行 ROLLBACK |
| | `DefineSavepoint()` | 执行 SAVEPOINT |
| **中层** (命令处理) | `StartTransactionCommand()` | 每个查询前 |
| | `CommitTransactionCommand()` | 每个查询后 |
| | `AbortCurrentTransaction()` | 错误处理 |
| **低层** (实际执行) | `StartTransaction()` | 真正启动事务 |
| | `CommitTransaction()` | 真正提交事务 |
| | `AbortTransaction()` | 真正回滚事务 |

---

## 📊 源码位置

### 核心文件

| 文件 | 路径 | 主要内容 |
|------|------|----------|
| xact.c | `src/backend/access/transam/xact.c` | 事务管理主逻辑 (6300+ 行) |
| varsup.c | `src/backend/access/transam/varsup.c` | 事务ID分配 |
| twophase.c | `src/backend/access/transam/twophase.c` | 两阶段提交 |
| proc.h | `src/include/storage/proc.h` | PGPROC 结构定义 |
| xact.h | `src/include/access/xact.h` | 事务管理接口 |

### 关键函数定位

```bash
# 事务开始
src/backend/access/transam/xact.c:2011    StartTransaction()
src/backend/access/transam/xact.c:2841    BeginTransactionBlock()

# 事务提交
src/backend/access/transam/xact.c:2167    CommitTransaction()
src/backend/access/transam/xact.c:2904    EndTransactionBlock()

# 事务回滚
src/backend/access/transam/xact.c:2630    AbortTransaction()
src/backend/access/transam/xact.c:3069    UserAbortTransactionBlock()

# 子事务
src/backend/access/transam/xact.c:5048    StartSubTransaction()
src/backend/access/transam/xact.c:5168    CommitSubTransaction()
src/backend/access/transam/xact.c:4143    DefineSavepoint()

# 事务ID分配
src/backend/access/transam/varsup.c:76    GetNewTransactionId()
```

---

## 🎓 学习建议

### 学习顺序

```
第1天: 理解三层架构和状态机
    ↓
第2天: 掌握 BEGIN/COMMIT/ROLLBACK 流程
    ↓
第3天: 深入子事务和 SAVEPOINT
    ↓
第4天: 学习两阶段提交
    ↓
第5天: 性能优化和调优
```

### 实践练习

1. **基础练习**: 编写包含事务的 SQL 脚本
2. **进阶练习**: 使用 SAVEPOINT 处理复杂业务逻辑
3. **高级练习**: 实现跨数据库的两阶段提交
4. **性能练习**: 优化长事务和大事务

### 常见问题

1. **什么时候分配事务ID?**
   - 只有当事务执行第一个修改操作时才分配XID
   - 纯查询事务不会分配XID

2. **子事务的开销?**
   - 每个子事务需要一个子事务ID
   - 过多子事务会影响性能

3. **两阶段提交的限制?**
   - 需要 `max_prepared_transactions > 0`
   - 准备好的事务会持有锁，需要尽快完成

---

## 📈 监控与调试

### 查看当前事务

```sql
-- 查看所有活跃事务
SELECT pid, xact_start, state, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;

-- 查看长事务
SELECT pid, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle'
  AND xact_start < now() - interval '5 minutes'
ORDER BY xact_start;

-- 查看准备好的事务
SELECT * FROM pg_prepared_xacts;
```

### 事务统计

```sql
-- 事务提交/回滚统计
SELECT datname, 
       xact_commit, 
       xact_rollback,
       round(xact_rollback::numeric / NULLIF(xact_commit + xact_rollback, 0) * 100, 2) AS rollback_pct
FROM pg_stat_database;
```

---

## 🔗 相关模块

- **MVCC**: 提供多版本并发控制机制
- **WAL**: 记录事务的所有修改
- **Lock Manager**: 管理事务锁
- **Buffer Manager**: 管理脏页刷写
- **VACUUM**: 清理死元组

---

## 💡 最佳实践

1. ✅ **避免长事务**: 长事务阻碍 VACUUM，导致表膨胀
2. ✅ **合理使用 SAVEPOINT**: 处理复杂业务逻辑的部分失败
3. ✅ **谨慎使用 2PC**: 只在必要时使用，及时完成准备好的事务
4. ✅ **监控事务统计**: 定期检查回滚率和长事务
5. ✅ **优化事务粒度**: 批量操作时适当拆分事务

---

**文档版本**: 1.0  
**PostgreSQL 版本**: 17.6  
**最后更新**: 2025-01-16  
**维护者**: PostgreSQL 学习小组

**下一步**: 阅读 [01_overview.md](01_overview.md) 深入了解事务系统架构

