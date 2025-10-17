# Lock Manager (锁管理器) 模块

## 📖 模块简介

Lock Manager 是 PostgreSQL 实现并发控制的核心组件，管理所有类型的锁，防止数据竞争和死锁。它实现了完整的死锁检测、锁升级和锁等待队列管理。

### 核心特性

- ✅ **8种表级锁模式**: 从 AccessShareLock 到 AccessExclusiveLock
- ✅ **4层锁机制**: Spinlock、LWLock、Regular Lock、Predicate Lock
- ✅ **死锁检测**: 等待图（Wait-For Graph）算法
- ✅ **快速路径优化**: 本地锁缓存，减少共享内存访问
- ✅ **锁分区**: 16个分区减少锁争用
- ✅ **自动释放**: 事务结束时自动释放所有锁
- ✅ **行级锁**: 多粒度锁定

### 锁层次结构

```
应用层
  │
  ▼
表级锁 (8种模式)
  ├─ AccessShareLock        (SELECT)
  ├─ RowShareLock           (SELECT FOR UPDATE)
  ├─ RowExclusiveLock       (INSERT/UPDATE/DELETE)
  ├─ ShareUpdateExclusiveLock (VACUUM)
  ├─ ShareLock              (CREATE INDEX)
  ├─ ShareRowExclusiveLock
  ├─ ExclusiveLock          (REFRESH MATERIALIZED VIEW)
  └─ AccessExclusiveLock    (DROP TABLE, TRUNCATE)
  │
  ▼
行级锁 (4种模式)
  ├─ FOR KEY SHARE
  ├─ FOR SHARE
  ├─ FOR NO KEY UPDATE
  └─ FOR UPDATE
```

---

## 📚 文档导航

### 🎯 新手入门路径

1. **第一步**: [01_overview.md](01_overview.md) - 了解锁系统架构和8种锁模式
2. **第二步**: [02_data_structures.md](02_data_structures.md) - 理解LOCK、PROCLOCK、LOCALLOCK
3. **第三步**: [03_implementation_flow.md](03_implementation_flow.md) - 掌握锁获取和释放流程
4. **第四步**: [07_diagrams.md](07_diagrams.md) - 查看完整的架构图

### 📄 完整文档列表

| 序号 | 文档名称 | 内容概要 | 适合人群 |
|------|----------|----------|----------|
| 1 | [01_overview.md](01_overview.md) | 锁系统概述、4层锁机制、8种锁模式 | 所有人 ⭐ |
| 2 | [02_data_structures.md](02_data_structures.md) | LOCK、PROCLOCK、LOCALLOCK 核心结构 | 开发者 |
| 3 | [03_implementation_flow.md](03_implementation_flow.md) | 锁获取、释放、等待、死锁检测流程 | 开发者 ⭐ |
| 4 | [04_key_algorithms.md](04_key_algorithms.md) | 死锁检测、快速路径、锁升级算法 | 高级开发者 |
| 5 | [05_performance.md](05_performance.md) | 锁性能优化、监控、调优技巧 | DBA |
| 6 | [06_testcases.md](06_testcases.md) | 完整测试用例集合 | 测试工程师 |
| 7 | [07_diagrams.md](07_diagrams.md) | 锁冲突矩阵、等待图、死锁检测流程图 | 所有人 ⭐ |

---

## 🚀 快速开始

### 表级锁基本操作

```sql
-- 显式获取表锁
BEGIN;
LOCK TABLE accounts IN ACCESS EXCLUSIVE MODE;  -- 最强锁
-- 其他会话的 SELECT 会被阻塞
COMMIT;  -- 自动释放锁

-- 隐式锁（自动获取）
BEGIN;
SELECT * FROM accounts WHERE id = 1;  -- 自动获取 AccessShareLock
UPDATE accounts SET balance = 1000 WHERE id = 1;  -- 自动获取 RowExclusiveLock
COMMIT;
```

### 行级锁

```sql
-- SELECT FOR UPDATE (行级排他锁)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- 锁定这一行
-- 其他会话无法 UPDATE 或 DELETE 这一行
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- SELECT FOR SHARE (行级共享锁)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;  -- 允许其他会话读取
-- 但阻止其他会话 UPDATE
COMMIT;

-- NOWAIT 和 SKIP LOCKED
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;  -- 立即返回错误
SELECT * FROM accounts WHERE id IN (1,2,3) FOR UPDATE SKIP LOCKED;  -- 跳过被锁定的行
```

### 死锁示例

```sql
-- 会话1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 锁住 id=1

-- 会话2
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;   -- 锁住 id=2

-- 会话1
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- 等待 id=2

-- 会话2
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- 等待 id=1
-- ERROR: deadlock detected
```

---

## 🔍 核心概念速查

### 8种表级锁模式

| 锁模式 | 冲突级别 | 典型使用场景 | 阻塞 |
|--------|----------|--------------|------|
| **AccessShareLock** (1) | 最弱 | SELECT | 只阻塞 AccessExclusiveLock |
| **RowShareLock** (2) | 弱 | SELECT FOR UPDATE/SHARE | 阻塞 ExclusiveLock, AccessExclusiveLock |
| **RowExclusiveLock** (3) | 中弱 | INSERT, UPDATE, DELETE | 阻塞 ShareLock 及更强锁 |
| **ShareUpdateExclusiveLock** (4) | 中 | VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY | 阻塞自己和更强锁 |
| **ShareLock** (5) | 中强 | CREATE INDEX | 阻塞 RowExclusiveLock 及更强锁 |
| **ShareRowExclusiveLock** (6) | 强 | 特殊操作 | 阻塞 RowExclusiveLock 及更强锁 |
| **ExclusiveLock** (7) | 很强 | REFRESH MATERIALIZED VIEW | 阻塞 RowShareLock 及更强锁 |
| **AccessExclusiveLock** (8) | 最强 | DROP, TRUNCATE, LOCK TABLE, ALTER TABLE | 阻塞所有其他锁 |

### 锁冲突矩阵 (简化版)

```
          1  2  3  4  5  6  7  8
       ┌─────────────────────────┐
    1  │ ✓  ✓  ✓  ✓  ✓  ✓  ✓  ✗ │ AccessShareLock
    2  │ ✓  ✓  ✓  ✓  ✓  ✓  ✗  ✗ │ RowShareLock
    3  │ ✓  ✓  ✓  ✓  ✗  ✗  ✗  ✗ │ RowExclusiveLock
    4  │ ✓  ✓  ✓  ✗  ✗  ✗  ✗  ✗ │ ShareUpdateExclusiveLock
    5  │ ✓  ✓  ✗  ✗  ✓  ✗  ✗  ✗ │ ShareLock
    6  │ ✓  ✓  ✗  ✗  ✗  ✗  ✗  ✗ │ ShareRowExclusiveLock
    7  │ ✓  ✗  ✗  ✗  ✗  ✗  ✗  ✗ │ ExclusiveLock
    8  │ ✗  ✗  ✗  ✗  ✗  ✗  ✗  ✗ │ AccessExclusiveLock
       └─────────────────────────┘
✓ = 兼容  ✗ = 冲突
```

### 核心数据结构

```c
// LOCK: 每个被锁对象的信息 (共享内存)
typedef struct LOCK {
    LOCKTAG tag;                    // 锁标识 (对象ID)
    LOCKMASK grantMask;             // 已授予的锁类型位掩码
    LOCKMASK waitMask;              // 等待中的锁类型位掩码
    dlist_head procLocks;           // PROCLOCK链表
    dclist_head waitProcs;          // 等待进程队列
    int requested[MAX_LOCKMODES];   // 各模式请求计数
    int granted[MAX_LOCKMODES];     // 各模式授予计数
} LOCK;

// PROCLOCK: 进程持有锁的信息 (LOCK + PGPROC 的关联)
typedef struct PROCLOCK {
    PROCLOCKTAG tag;                // {LOCK*, PGPROC*}
    LOCKMASK holdMask;              // 持有的锁类型位掩码
    LOCKMASK releaseMask;           // 待释放的锁类型位掩码
    dlist_node lockLink;            // LOCK的链表节点
    dlist_node procLink;            // PGPROC的链表节点
} PROCLOCK;

// LOCALLOCK: 本地锁信息 (每个后端私有)
typedef struct LOCALLOCK {
    LOCALLOCKTAG tag;               // {LOCKTAG, LOCKMODE}
    LOCK *lock;                     // 指向共享LOCK
    PROCLOCK *proclock;             // 指向共享PROCLOCK
    int64 nLocks;                   // 持有次数
    LOCALLOCKOWNER *lockOwners;     // 资源所有者数组
} LOCALLOCK;
```

---

## 📊 源码位置

### 核心文件

| 文件 | 路径 | 主要内容 |
|------|------|----------|
| lock.c | `src/backend/storage/lmgr/lock.c` | 锁管理主逻辑 (5000+ 行) |
| deadlock.c | `src/backend/storage/lmgr/deadlock.c` | 死锁检测算法 |
| proc.c | `src/backend/storage/lmgr/proc.c` | 进程锁管理 |
| lock.h | `src/include/storage/lock.h` | 锁相关定义 |
| lockdefs.h | `src/include/storage/lockdefs.h` | 锁模式定义 |

### 关键函数定位

```bash
# 锁获取
src/backend/storage/lmgr/lock.c:763    LockAcquire()
src/backend/storage/lmgr/lock.c:1264   LockAcquireExtended()

# 锁释放
src/backend/storage/lmgr/lock.c:2131   LockRelease()
src/backend/storage/lmgr/lock.c:2364   LockReleaseAll()

# 死锁检测
src/backend/storage/lmgr/deadlock.c:108    DeadLockCheck()
src/backend/storage/lmgr/deadlock.c:377    FindLockCycle()

# 快速路径
src/backend/storage/lmgr/lock.c:555    FastPathGrantRelationLock()
src/backend/storage/lmgr/lock.c:612    FastPathUnGrantRelationLock()
```

---

## 🎓 学习建议

### 学习顺序

```
第1天: 理解8种锁模式和冲突矩阵
    ↓
第2天: 掌握锁获取和释放流程
    ↓
第3天: 深入死锁检测算法
    ↓
第4天: 学习快速路径优化
    ↓
第5天: 性能优化和调优
```

### 实践练习

1. **基础练习**: 尝试不同锁模式的组合
2. **进阶练习**: 制造并观察死锁
3. **高级练习**: 分析 `pg_locks` 视图
4. **性能练习**: 优化锁争用场景

### 常见问题

1. **什么时候需要显式加锁?**
   - 大多数情况下，PostgreSQL 自动管理锁
   - 需要确保操作原子性时，使用 `LOCK TABLE`
   - 避免死锁时，统一加锁顺序

2. **如何避免锁等待?**
   - 使用 `NOWAIT` 或 `SKIP LOCKED`
   - 缩短事务时间
   - 合理设计表结构

3. **死锁如何处理?**
   - PostgreSQL 自动检测并中止一个事务
   - 应用层重试被中止的事务
   - 统一锁获取顺序避免死锁

---

## 📈 监控与调试

### 查看当前锁

```sql
-- 查看所有锁
SELECT 
    locktype,
    database,
    relation::regclass,
    page,
    tuple,
    virtualxid,
    transactionid,
    mode,
    granted,
    pid
FROM pg_locks
ORDER BY pid;

-- 查看被阻塞的查询
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### 锁等待超时

```sql
-- 设置锁等待超时
SET lock_timeout = '5s';  -- 5秒后超时

-- 设置死锁超时
SET deadlock_timeout = '1s';  -- 1秒后检测死锁
```

---

## 🔗 相关模块

- **Transaction Manager**: 事务结束时释放锁
- **MVCC**: 减少锁需求，提高并发
- **Buffer Manager**: Buffer访问需要锁保护
- **WAL**: 某些WAL操作需要锁

---

## 💡 最佳实践

1. ✅ **避免长事务**: 长时间持有锁会阻塞其他会话
2. ✅ **统一加锁顺序**: 防止死锁
3. ✅ **使用合适的锁模式**: 不要过度加锁
4. ✅ **监控锁等待**: 定期检查 `pg_locks`
5. ✅ **使用行级锁**: 比表级锁冲突少
6. ✅ **考虑 SKIP LOCKED**: 队列处理场景

---

**文档版本**: 1.0  
**PostgreSQL 版本**: 17.6  
**最后更新**: 2025-01-16  
**维护者**: PostgreSQL 学习小组

**下一步**: 阅读 [01_overview.md](01_overview.md) 深入了解锁系统架构

