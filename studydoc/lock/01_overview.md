# Lock Manager 锁管理概述

## 1. 锁管理系统简介

### 1.1 什么是锁?

锁(Lock)是数据库保证数据一致性和隔离性的核心机制。PostgreSQL的锁管理器控制并发访问，防止数据竞争和不一致。

**ACID中的作用**:

```
I - Isolation (隔离性)
    ├─ 锁机制: 防止并发事务相互干扰
    ├─ MVCC: 提供非阻塞读取
    └─ Lock Manager: 管理所有锁资源

C - Consistency (一致性)  
    └─ 锁保证: 操作的原子性和完整性
```

### 1.2 PostgreSQL的4层锁机制

PostgreSQL使用4种类型的锁，从快到慢：

```
┌─────────────────────────────────────────────────────────┐
│ 1. Spinlock (自旋锁)                                     │
│    速度: 最快 (~10 CPU cycles)                           │
│    用途: 保护共享内存数据结构 (极短时间)                 │
│    特点: 忙等待，无死锁检测，自动超时(1分钟)             │
│    实现: 硬件原子操作 (test-and-set)                     │
│    文件: src/include/storage/spin.h                      │
├─────────────────────────────────────────────────────────┤
│ 2. LWLock (轻量级锁)                                     │
│    速度: 很快 (~100 CPU cycles)                          │
│    用途: 保护共享内存结构 (短时间持有)                   │
│    特点: 支持共享/排他模式，无死锁检测，自动释放         │
│    实现: Semaphore (睡眠等待)                            │
│    例子: BufferDescriptor锁，WAL插入锁                  │
│    文件: src/backend/storage/lmgr/lwlock.c               │
├─────────────────────────────────────────────────────────┤
│ 3. Regular Lock (常规锁/重量级锁) ← 本模块重点!          │
│    速度: 中等 (需要哈希查找)                             │
│    用途: 用户级锁请求 (表锁、行锁)                       │
│    特点: 8种锁模式，死锁检测，事务结束自动释放           │
│    实现: 共享内存哈希表 + 等待队列                       │
│    文件: src/backend/storage/lmgr/lock.c                 │
├─────────────────────────────────────────────────────────┤
│ 4. Predicate Lock (谓词锁)                               │
│    速度: 最慢 (复杂逻辑)                                 │
│    用途: SERIALIZABLE 隔离级别的幻读保护                 │
│    特点: 检测读写依赖冲突                                │
│    实现: SSI (Serializable Snapshot Isolation)           │
│    文件: src/backend/storage/lmgr/predicate.c            │
└─────────────────────────────────────────────────────────┘
```

**本文档聚焦于 Regular Lock (常规锁)**

---

## 2. Regular Lock 系统架构

### 2.1 整体架构图

```
用户SQL
  │
  ▼
┌────────────────────────────────────────────┐
│          Lock Manager API                   │
│  LockAcquire() / LockRelease()              │
│  LockReleaseAll()                           │
└────────────────┬───────────────────────────┘
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
    ┌──────┐ ┌──────┐ ┌──────┐
    │ Fast │ │Regular│ │Dead- │
    │ Path │ │ Path  │ │lock  │
    └──────┘ └──────┘ └──────┘
        │        │        │
        ▼        ▼        ▼
┌────────────────────────────────────────────┐
│        Shared Memory (共享内存)             │
│  ┌─────────────────────────────────────┐  │
│  │ LOCK Hash Table (16 partitions)     │  │
│  │   Key: LOCKTAG                      │  │
│  │   Value: LOCK 结构                  │  │
│  └─────────────────────────────────────┘  │
│  ┌─────────────────────────────────────┐  │
│  │ PROCLOCK Hash Table                 │  │
│  │   Key: (LOCK, PGPROC)               │  │
│  │   Value: PROCLOCK 结构              │  │
│  └─────────────────────────────────────┘  │
└────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────┐
│      Local Memory (本地内存)               │
│  ┌─────────────────────────────────────┐  │
│  │ LOCALLOCK Hash Table                │  │
│  │   Key: (LOCKTAG, LOCKMODE)          │  │
│  │   Value: LOCALLOCK 结构             │  │
│  │   用途: 本地缓存，避免重复访问共享内存│  │
│  └─────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

### 2.2 三层哈希表设计

PostgreSQL使用**三层哈希表**管理锁：

| 哈希表 | 位置 | 键 | 值 | 作用 |
|--------|------|-----|-----|------|
| **LOCK Hash** | 共享内存 | LOCKTAG (对象标识) | LOCK结构 | 记录被锁对象的信息 |
| **PROCLOCK Hash** | 共享内存 | (LOCK*, PGPROC*) | PROCLOCK结构 | 记录哪个进程持有哪个锁 |
| **LOCALLOCK Hash** | 本地内存 | (LOCKTAG, LOCKMODE) | LOCALLOCK结构 | 本地缓存，避免重复查询 |

---

## 3. 8种表级锁模式

### 3.1 锁模式定义

```c
// src/include/storage/lockdefs.h:33
#define NoLock                   0  // 不加锁

#define AccessShareLock          1  // SELECT
#define RowShareLock             2  // SELECT FOR UPDATE/SHARE
#define RowExclusiveLock         3  // INSERT, UPDATE, DELETE
#define ShareUpdateExclusiveLock 4  // VACUUM, ANALYZE
#define ShareLock                5  // CREATE INDEX
#define ShareRowExclusiveLock    6  // 特殊场景
#define ExclusiveLock            7  // REFRESH MATERIALIZED VIEW
#define AccessExclusiveLock      8  // DROP, TRUNCATE, LOCK TABLE

#define MaxLockMode              8
```

### 3.2 锁模式详解

#### 锁模式1: AccessShareLock (最弱)

```sql
-- 自动获取
SELECT * FROM table_name;

-- 阻塞关系
与 AccessExclusiveLock 冲突
与其他所有锁兼容
```

**特点**:
- ✅ 允许并发读取
- ✅ 允许并发写入
- ✗ 阻止 DROP/TRUNCATE

**典型场景**: 所有SELECT查询

#### 锁模式2: RowShareLock

```sql
-- 自动获取
SELECT * FROM table_name FOR UPDATE;
SELECT * FROM table_name FOR SHARE;

-- 阻塞关系
与 ExclusiveLock, AccessExclusiveLock 冲突
```

**特点**:
- ✅ 允许并发读取
- ✅ 允许并发 INSERT/UPDATE/DELETE
- ✗ 阻止 EXCLUSIVE 及以上锁

**典型场景**: 行级锁定查询

#### 锁模式3: RowExclusiveLock

```sql
-- 自动获取
INSERT INTO table_name ...;
UPDATE table_name SET ...;
DELETE FROM table_name WHERE ...;

-- 阻塞关系
与 ShareLock, ShareRowExclusiveLock, ExclusiveLock, AccessExclusiveLock 冲突
```

**特点**:
- ✅ 允许并发读取
- ✅ 允许并发写入
- ✗ 阻止 CREATE INDEX (非CONCURRENTLY)

**典型场景**: 所有修改数据的DML语句

#### 锁模式4: ShareUpdateExclusiveLock

```sql
-- 自动获取
VACUUM table_name;
ANALYZE table_name;
CREATE INDEX CONCURRENTLY ...;
REFRESH MATERIALIZED VIEW CONCURRENTLY ...;

-- 阻塞关系
与自己冲突 (自互斥)
与 ShareUpdateExclusiveLock 及以上锁冲突
```

**特点**:
- ✅ 允许并发读写
- ✗ **防止并发VACUUM** (自互斥)
- ✗ 阻止表结构变更

**典型场景**: VACUUM、ANALYZE

#### 锁模式5: ShareLock

```sql
-- 自动获取
CREATE INDEX table_name_idx ON table_name (column);

-- 显式获取
LOCK TABLE table_name IN SHARE MODE;

-- 阻塞关系
与 RowExclusiveLock 及以上锁冲突
```

**特点**:
- ✅ 允许并发读取
- ✗ 阻止并发写入 (INSERT/UPDATE/DELETE)
- ✗ 阻止 CREATE INDEX

**典型场景**: CREATE INDEX (非并发)

#### 锁模式6: ShareRowExclusiveLock

```sql
-- 显式获取
LOCK TABLE table_name IN SHARE ROW EXCLUSIVE MODE;

-- 很少使用！
```

**特点**:
- ✅ 允许并发读取
- ✗ 阻止并发写入
- ✗ 阻止其他 ShareRowExclusiveLock

#### 锁模式7: ExclusiveLock

```sql
-- 自动获取
REFRESH MATERIALIZED VIEW mat_view;

-- 显式获取
LOCK TABLE table_name IN EXCLUSIVE MODE;
```

**特点**:
- ✅ 允许并发读取
- ✗ 阻止并发写入
- ✗ 阻止 SELECT FOR UPDATE

**典型场景**: REFRESH MATERIALIZED VIEW

#### 锁模式8: AccessExclusiveLock (最强)

```sql
-- 自动获取
DROP TABLE table_name;
TRUNCATE TABLE table_name;
REINDEX TABLE table_name;
VACUUM FULL table_name;
LOCK TABLE table_name;  -- 默认模式
ALTER TABLE table_name ...;  -- 大多数情况

-- 阻塞关系
与所有其他锁模式冲突 (完全排他)
```

**特点**:
- ✗ 阻止任何并发操作 (包括SELECT)
- ✗ 完全排他访问

**典型场景**: DDL操作、表结构变更

### 3.3 锁冲突矩阵 (完整版)

```
Lock Conflict Matrix
====================

请求锁 →     1   2   3   4   5   6   7   8
当前持有锁 ↓ AS  RS  RE  SUE S   SRE E   AE

1  AccessShare       ✓   ✓   ✓   ✓   ✓   ✓   ✓   ✗
2  RowShare          ✓   ✓   ✓   ✓   ✓   ✓   ✗   ✗
3  RowExclusive      ✓   ✓   ✓   ✓   ✗   ✗   ✗   ✗
4  ShareUpdateExcl   ✓   ✓   ✓   ✗   ✗   ✗   ✗   ✗
5  Share              ✓   ✓   ✗   ✗   ✓   ✗   ✗   ✗
6  ShareRowExcl      ✓   ✓   ✗   ✗   ✗   ✗   ✗   ✗
7  Exclusive         ✓   ✗   ✗   ✗   ✗   ✗   ✗   ✗
8  AccessExclusive   ✗   ✗   ✗   ✗   ✗   ✗   ✗   ✗

✓ = 兼容 (Compatible)
✗ = 冲突 (Conflict)

规则: 
- 对角线: 是否自冲突
  - AccessShareLock: ✓ (多个会话可以同时持有)
  - ShareUpdateExclusiveLock: ✗ (自冲突，防止并发VACUUM)
  - AccessExclusiveLock: ✗ (完全排他)
  
- 强度递增: 1 < 2 < 3 < 4 < 5 < 6 < 7 < 8
```

---

## 4. 行级锁

PostgreSQL还支持**行级锁**，粒度更细，冲突更少。

### 4.1 4种行级锁模式

```c
// src/include/nodes/lockoptions.h:49
typedef enum LockTupleMode
{
    LockTupleKeyShare,        // FOR KEY SHARE
    LockTupleShare,           // FOR SHARE
    LockTupleNoKeyExclusive,  // FOR NO KEY UPDATE
    LockTupleExclusive,       // FOR UPDATE
} LockTupleMode;
```

### 4.2 行级锁详解

| 行锁模式 | SQL语法 | 用途 | 阻塞 |
|----------|---------|------|------|
| **FOR KEY SHARE** | `SELECT ... FOR KEY SHARE` | 防止删除 | 阻止 DELETE 和 FOR UPDATE |
| **FOR SHARE** | `SELECT ... FOR SHARE` | 防止修改 | 阻止 UPDATE 和 DELETE |
| **FOR NO KEY UPDATE** | `SELECT ... FOR NO KEY UPDATE` | 更新非键列 | 阻止所有修改 |
| **FOR UPDATE** | `SELECT ... FOR UPDATE` | 排他锁定 | 阻止所有修改和其他行锁 |

### 4.3 行锁冲突矩阵

```
                    正在持有的锁 →
请求的锁 ↓        KEY   FOR    NO KEY  FOR
                  SHARE SHARE  UPDATE  UPDATE
FOR KEY SHARE      ✓     ✓      ✓       ✗
FOR SHARE          ✓     ✓      ✗       ✗
FOR NO KEY UPDATE  ✓     ✗      ✗       ✗
FOR UPDATE         ✗     ✗      ✗       ✗
```

### 4.4 行锁示例

```sql
-- FOR UPDATE: 最强行锁
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- 锁定这一行，其他会话无法修改或删除
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- FOR SHARE: 共享行锁
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;
-- 允许其他会话读取，但阻止修改
-- 其他会话可以: SELECT, SELECT FOR SHARE
-- 其他会话不可以: UPDATE, DELETE, SELECT FOR UPDATE
COMMIT;

-- FOR KEY SHARE: 最弱行锁
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR KEY SHARE;
-- 只阻止删除和键列更新
-- 允许更新非键列
COMMIT;

-- NOWAIT: 立即失败而不等待
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
-- 如果无法立即获取锁，返回错误

-- SKIP LOCKED: 跳过已锁定的行
SELECT * FROM queue WHERE status = 'pending' 
FOR UPDATE SKIP LOCKED LIMIT 10;
-- 获取未锁定的10行，跳过已锁定的行
-- 适用于队列处理场景
```

---

## 5. 锁的粒度层次

PostgreSQL支持**多粒度锁定**，从粗到细：

```
数据库级别
    │
    ├─ 表级锁 (8种模式)
    │   └─ 对整个表加锁
    │
    ├─ 页级锁 (内部使用)
    │   └─ 扩展表时的页面锁
    │
    ├─ 行级锁 (4种模式)
    │   └─ 对单个元组加锁
    │
    └─ 其他锁
        ├─ 事务ID锁 (等待事务完成)
        ├─ 虚拟事务ID锁
        ├─ 扩展锁
        └─ 关系扩展锁
```

### 5.1 LOCKTAG - 锁标识

```c
// src/include/storage/lock.h:85
typedef struct LOCKTAG
{
    uint32 locktag_field1;  // 对象ID (如: relation OID)
    uint32 locktag_field2;  // 数据库OID 或 page number
    uint32 locktag_field3;  // 其他标识
    uint16 locktag_field4;  // 元组偏移等
    uint8  locktag_type;    // 锁类型 (见下表)
    uint8  locktag_lockmethodid;  // 锁方法ID
} LOCKTAG;
```

**锁类型 (locktag_type)**:

| 类型 | 值 | 说明 |
|------|-----|------|
| LOCKTAG_RELATION | 0 | 表锁 |
| LOCKTAG_RELATION_EXTEND | 1 | 表扩展锁 |
| LOCKTAG_PAGE | 2 | 页锁 |
| LOCKTAG_TUPLE | 3 | 行锁 |
| LOCKTAG_TRANSACTION | 4 | 事务ID锁 |
| LOCKTAG_VIRTUALTRANSACTION | 5 | 虚拟事务ID锁 |
| LOCKTAG_SPECULATIVE_TOKEN | 6 | 插入冲突检测 |
| LOCKTAG_OBJECT | 7 | 数据库对象锁 |
| LOCKTAG_USERLOCK | 8 | 用户自定义锁 |
| LOCKTAG_ADVISORY | 9 | 咨询锁 |

---

## 6. 快速路径优化 (Fast Path)

为了提高性能，PostgreSQL为**关系锁**实现了**快速路径**。

### 6.1 快速路径条件

满足以下条件时使用快速路径：

```
1. 锁类型 = LOCKTAG_RELATION (表锁)
2. 锁模式为弱锁 (AccessShareLock 或 RowShareLock)
3. 该进程持有的此类锁 < FP_LOCK_SLOTS_PER_BACKEND (16个)
```

### 6.2 快速路径 vs 常规路径

| 特性 | 快速路径 | 常规路径 |
|------|----------|----------|
| **存储位置** | PGPROC->fpRelId[] | LOCK/PROCLOCK 哈希表 |
| **访问开销** | 低 (本地数组) | 高 (共享内存哈希查找) |
| **锁争用** | 无 (本地) | 有 (LWLock保护) |
| **适用场景** | 弱表锁 (常见) | 强锁或溢出情况 |
| **最大数量** | 16个/进程 | 无限制 |

### 6.3 快速路径示例

```sql
-- 快速路径: SELECT (AccessShareLock)
SELECT * FROM t1, t2, t3, t4, t5;
-- 5个表的锁都走快速路径 (如果<16个)

-- 溢出到常规路径
SELECT * FROM t1, t2, ..., t20;  -- 20个表
-- 前16个表走快速路径
-- 后4个表走常规路径

-- 强锁必须走常规路径
LOCK TABLE accounts IN EXCLUSIVE MODE;
-- ExclusiveLock 必须走常规路径
```

---

## 7. 锁分区 (Lock Partitioning)

为了减少锁争用，PostgreSQL将LOCK哈希表分为**16个分区**。

### 7.1 分区机制

```c
// src/backend/storage/lmgr/lock.c
#define LOG2_NUM_LOCK_PARTITIONS  4
#define NUM_LOCK_PARTITIONS  (1 << LOG2_NUM_LOCK_PARTITIONS)  // 16

// 分区算法
partition = LOCKTAG的哈希值 % 16

// 每个分区有独立的LWLock
LWLock *partitionLock = &MainLWLockArray[
    LOCK_MANAGER_LWLOCK_OFFSET + partition
].lock;
```

**好处**:
- ✅ 减少锁争用 (16个分区并行)
- ✅ 提高并发性能
- ✅ 降低LWLock竞争

### 7.2 分区示例

```
LOCK Hash Table (16 Partitions)
================================

Partition 0 [LWLock 0]
  ├─ LOCK(table1, AccessShareLock)
  └─ LOCK(table5, RowExclusiveLock)

Partition 1 [LWLock 1]
  ├─ LOCK(table2, ShareLock)
  └─ ...

...

Partition 15 [LWLock 15]
  └─ LOCK(table100, AccessExclusiveLock)

不同分区可以并行访问，无争用！
```

---

## 8. 死锁检测

PostgreSQL使用**等待图算法**检测死锁。

### 8.1 死锁场景

```sql
-- 会话1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- 持有 id=1 的行锁

-- 会话2
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
-- 持有 id=2 的行锁

-- 会话1
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- 等待 id=2 的锁 (被会话2持有)

-- 会话2
UPDATE accounts SET balance = balance + 50 WHERE id = 1;
-- 等待 id=1 的锁 (被会话1持有)
-- 形成环! PostgreSQL检测到死锁
-- ERROR: deadlock detected
-- DETAIL: Process 12345 waits for ShareLock on transaction 1000; 
--         blocked by process 12346.
--         Process 12346 waits for ShareLock on transaction 999; 
--         blocked by process 12345.
```

### 8.2 死锁检测算法

```
1. 进程等待锁时，设置定时器 (deadlock_timeout, 默认1秒)
2. 超时后，调用 DeadLockCheck()
3. 构建等待图 (Wait-For Graph)
   - 节点: 进程
   - 边: A等待B持有的锁 → A→B
4. DFS检测环
5. 如果检测到环:
   - 选择一个受害者进程 (通常是检测者自己)
   - 中止该进程的事务
   - 释放所有锁，唤醒等待者
6. 如果无环:
   - 继续等待
```

---

## 9. 锁管理器的关键特性

### 9.1 自动锁管理

```sql
-- PostgreSQL 自动管理锁!
BEGIN;
    SELECT * FROM t1;       -- 自动: AccessShareLock
    UPDATE t1 SET ...;      -- 自动: RowExclusiveLock
    SELECT * FROM t2 FOR UPDATE;  -- 自动: RowShareLock + 行锁
COMMIT;  -- 自动释放所有锁
```

### 9.2 锁升级

PostgreSQL **不支持自动锁升级**（行锁→表锁）。

原因: 防止死锁和复杂性

### 9.3 锁降级

不支持。一旦获取强锁，必须持有到事务结束。

### 9.4 锁超时

```sql
-- 设置锁等待超时
SET lock_timeout = '5s';

BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- 如果5秒内无法获取锁，抛出错误
COMMIT;
```

---

## 10. 源码导航

### 10.1 核心文件

| 文件 | 行数 | 主要内容 |
|------|------|----------|
| `lock.c` | 5000+ | 锁管理主逻辑 |
| `deadlock.c` | 1500+ | 死锁检测算法 |
| `proc.c` | 1800+ | 进程锁等待 |
| `lock.h` | 600+ | 锁相关定义 |
| `lockdefs.h` | 60+ | 锁模式定义 |

### 10.2 关键函数

```
锁获取:
  LockAcquire()              lock.c:763
  LockAcquireExtended()      lock.c:1264
  FastPathGrantRelationLock() lock.c:555

锁释放:
  LockRelease()              lock.c:2131
  LockReleaseAll()           lock.c:2364
  FastPathUnGrantRelationLock() lock.c:612

死锁检测:
  DeadLockCheck()            deadlock.c:108
  FindLockCycle()            deadlock.c:377
  GetLockConflicts()         deadlock.c:555

等待队列:
  ProcSleep()                proc.c:1241
  ProcWakeup()               proc.c:1451
```

---

## 11. 总结

### 11.1 核心要点

1. **4层锁机制**: Spinlock、LWLock、Regular Lock、Predicate Lock
2. **8种表锁模式**: 从 AccessShareLock 到 AccessExclusiveLock
3. **4种行锁模式**: FOR KEY SHARE、FOR SHARE、FOR NO KEY UPDATE、FOR UPDATE
4. **三层哈希表**: LOCK、PROCLOCK、LOCALLOCK
5. **快速路径优化**: 本地缓存弱表锁
6. **16个锁分区**: 减少争用
7. **自动死锁检测**: 等待图算法
8. **自动锁管理**: 事务结束时释放

### 11.2 学习路径

```
第1步: 理解8种锁模式和冲突矩阵 (本文档)
   ↓
第2步: 学习核心数据结构 (02_data_structures.md)
   ↓
第3步: 掌握锁获取和释放流程 (03_implementation_flow.md)
   ↓
第4步: 深入死锁检测和快速路径 (04_key_algorithms.md)
   ↓
第5步: 性能优化和调优 (05_performance.md)
```

---

**下一步**: 阅读 [02_data_structures.md](02_data_structures.md) 深入了解锁的数据结构

**源码版本**: PostgreSQL 17.6  
**文档版本**: 1.0  
**最后更新**: 2025-01-16

