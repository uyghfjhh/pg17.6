# Lock Manager 关键算法

本文档详细说明Lock Manager的关键算法：死锁检测、快速路径优化、锁冲突判断等。

---

## 1. 死锁检测算法

### 1.1 等待图 (Wait-For Graph)

```
概念:
  节点 = 进程
  有向边 = 等待关系 (A → B 表示 A 等待 B 持有的锁)
  死锁 = 等待图中存在环

示例死锁:
  T1 持有 LOCK_A，等待 LOCK_B
  T2 持有 LOCK_B，等待 LOCK_C
  T3 持有 LOCK_C，等待 LOCK_A
  
  等待图: T1 → T2 → T3 → T1 (环!)
```

### 1.2 深度优先搜索 (DFS) 算法

```c
// src/backend/storage/lmgr/deadlock.c:150
DeadLockState
DeadLockCheck(PGPROC *proc)
{
    EDGE_LIST edges;
    int nProcs = 0;
    
    /* [1] 构建等待图边 */
    BuildWaitGraph(&edges, &nProcs);
    
    /* [2] DFS 检测环 */
    for (i = 0; i < nProcs; i++)
    {
        if (FindLockCycle(i, &edges, &softEdges))
        {
            /* 找到死锁环 */
            if (softEdges)
                return DS_SOFT_DEADLOCK;  // 可能自动解除
            else
                return DS_HARD_DEADLOCK;  // 必须中止事务
        }
    }
    
    return DS_NO_DEADLOCK;
}
```

### 1.3 构建等待图

```c
// 伪代码
BuildWaitGraph()
{
    foreach (PGPROC *proc in AllProcs)
    {
        if (proc->waitLock == NULL)
            continue;  // 不在等待
        
        LOCK *lock = proc->waitLock;
        
        /* 找到阻塞此进程的所有进程 */
        foreach (PROCLOCK *holder in lock->procLocks)
        {
            if (LockModeConflicts(proc->waitLockMode, holder->holdMask))
            {
                /* 添加边: proc → holder->myProc */
                AddEdge(proc, holder->myProc);
            }
        }
    }
}
```

### 1.4 DFS 环检测

```c
// src/backend/storage/lmgr/deadlock.c:450
bool
FindLockCycle(int startProc, EDGE_LIST *edges, bool *hasSoftEdge)
{
    int *visited = palloc0(nProcs * sizeof(int));
    int *path = palloc(nProcs * sizeof(int));
    int pathLen = 0;
    
    return FindLockCycleRecurse(startProc, edges, visited, path, &pathLen, hasSoftEdge);
}

static bool
FindLockCycleRecurse(int proc, EDGE_LIST *edges, int *visited, 
                     int *path, int *pathLen, bool *hasSoftEdge)
{
    /* 标记当前节点 */
    visited[proc] = 1;  // 正在访问
    path[(*pathLen)++] = proc;
    
    /* 遍历所有出边 */
    for (i = edges->edgeStart[proc]; i < edges->edgeStart[proc+1]; i++)
    {
        int blocker = edges->edgeTo[i];
        
        /* 检查是否形成环 */
        if (visited[blocker] == 1)
        {
            /* 找到环! 检查是否在当前路径中 */
            for (j = 0; j < *pathLen; j++)
            {
                if (path[j] == blocker)
                {
                    /* 环的起点就是 blocker */
                    *hasSoftEdge = CheckSoftEdge(path, j, *pathLen);
                    return true;  // 找到死锁
                }
            }
        }
        else if (visited[blocker] == 0)
        {
            /* 递归访问 */
            if (FindLockCycleRecurse(blocker, edges, visited, path, pathLen, hasSoftEdge))
                return true;
        }
    }
    
    /* 回溯 */
    (*pathLen)--;
    visited[proc] = 2;  // 访问完成
    
    return false;  // 无环
}
```

### 1.5 死锁解除策略

```c
// 选择受害者 (Victim Selection)
static PGPROC *
GetLockConflicts(LOCK *lock, LOCKMODE lockmode)
{
    /* 策略: 选择当前进程作为受害者 */
    // 原因: 
    // 1. 避免中止其他已运行很久的事务
    // 2. 让用户自行决定重试逻辑
    
    return MyProc;  // 自杀策略
}

// 中止事务
ereport(ERROR,
        (errcode(ERRCODE_T_R_DEADLOCK_DETECTED),
         errmsg("deadlock detected"),
         errdetail("Process %d waits for %s lock...",
                   MyProcPid, GetLockmodeName(...))));
```

### 1.6 死锁日志示例

```
ERROR:  deadlock detected
DETAIL:  Process 12345 waits for ShareLock on relation 16384 of database 16385; blocked by process 12346.
         Process 12346 waits for ExclusiveLock on relation 16387 of database 16385; blocked by process 12345.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (0,1) in relation "test_table"
```

---

## 2. 快速路径算法

### 2.1 快速路径条件

```c
// src/backend/storage/lmgr/lock.c:690
#define EligibleForRelationFastPath(locktag, mode) \
    ((locktag)->locktag_type == LOCKTAG_RELATION && \
     (locktag)->locktag_lockmethodid == DEFAULT_LOCKMETHOD && \
     ((mode) == AccessShareLock || (mode) == RowShareLock))
```

**限制原因**:
- ✅ **仅限表锁**: 表锁最常见
- ✅ **仅限弱锁**: `AccessShareLock` 和 `RowShareLock` 冲突少
- ✅ **16个槽位**: `FP_LOCK_SLOTS_PER_BACKEND = 16`

### 2.2 快速路径获取

```c
// src/backend/storage/lmgr/lock.c:4400
static bool
FastPathGrantRelationLock(Oid relid, LOCKMODE lockmode)
{
    int i;
    int empty_slot = -1;
    
    /* [1] 查找现有槽位 */
    for (i = 0; i < FP_LOCK_SLOTS_PER_BACKEND; i++)
    {
        if (MyProc->fpRelId[i] == relid)
        {
            /* 找到相同表，检查锁模式 */
            int oldmode = FAST_PATH_GET_BITS(MyProc, i);
            
            if (oldmode == lockmode)
            {
                /* 相同模式，递增计数 */
                FastPathLocalUseCount[i]++;
                return true;
            }
            else
            {
                /* 不同模式，降级到常规路径 */
                return false;
            }
        }
        
        if (MyProc->fpRelId[i] == InvalidOid && empty_slot < 0)
            empty_slot = i;
    }
    
    /* [2] 没有现有槽位，使用空槽 */
    if (empty_slot < 0)
        return false;  // 槽位已满
    
    /* [3] 初始化新槽位 */
    MyProc->fpRelId[empty_slot] = relid;
    FAST_PATH_SET_LOCKMODE(MyProc, empty_slot, lockmode);
    FastPathLocalUseCount[empty_slot] = 1;
    
    return true;
}
```

### 2.3 fpLockBits 编码

```c
// 每个槽位占 4 位
#define FP_LOCK_BITS_PER_SLOT 4

// 设置槽位的锁模式
#define FAST_PATH_SET_LOCKMODE(proc, n, mode) \
    ((proc)->fpLockBits = \
        ((proc)->fpLockBits & ~(((uint64) 0xF) << (FP_LOCK_BITS_PER_SLOT * (n)))) | \
        (((uint64) (mode)) << (FP_LOCK_BITS_PER_SLOT * (n))))

// 获取槽位的锁模式
#define FAST_PATH_GET_BITS(proc, n) \
    (((proc)->fpLockBits >> (FP_LOCK_BITS_PER_SLOT * (n))) & 0xF)

// 示例:
// Slot 0: AccessShareLock (mode=1)
// Slot 1: RowShareLock (mode=2)
// fpLockBits = 0x...0021
//              位0-3: 0001 (slot 0)
//              位4-7: 0010 (slot 1)
```

### 2.4 快速路径性能对比

```
场景: SELECT * FROM t1 JOIN t2 JOIN t3 (3个表, 每个需要 AccessShareLock)

常规路径 (每个表):
  1. 计算 hashcode (~10 ns)
  2. 查找 LOCALLOCK 哈希表 (~50 ns)
  3. 获取分区锁 (~100 ns)
  4. 查找 LOCK 哈希表 (~50 ns)
  5. 更新 LOCK 状态 (~20 ns)
  6. 释放分区锁 (~100 ns)
  总计: ~330 ns/表

快速路径 (每个表):
  1. 线性扫描 fpRelId[] (~5 ns * 16 = 80 ns)
  2. 位操作设置 fpLockBits (~5 ns)
  总计: ~85 ns/表

加速比: 330/85 ≈ 3.9x
```

---

## 3. 锁冲突判断算法

### 3.1 冲突矩阵查找

```c
// src/backend/storage/lmgr/lock.c:500
static inline bool
LockCheckConflicts(LOCK *lock, LOCKMODE lockmode, PGPROC *proc)
{
    LOCKMASK conflictMask = LockConflicts[lockmode];
    
    /* 检查已授予的锁是否冲突 */
    if (lock->grantMask & conflictMask)
    {
        /* 有冲突，但需要排除自己持有的锁 */
        PROCLOCK *proclock = FindProclock(lock, proc);
        
        if (proclock)
        {
            /* 从冲突掩码中移除自己持有的模式 */
            conflictMask &= ~proclock->holdMask;
        }
        
        if (lock->grantMask & conflictMask)
            return true;  // 真正的冲突
    }
    
    return false;  // 无冲突
}
```

### 3.2 位操作优化

```c
// 冲突检查只需要一次位与操作:
bool has_conflict = (lock->grantMask & LockConflicts[lockmode]) != 0;

// 示例:
// 当前已授予: AccessShareLock (bit 1), RowExclusiveLock (bit 3)
// grantMask = 0x000A (二进制: 1010)

// 请求 ShareLock (lockmode=5):
// LockConflicts[ShareLock] = 0x00F8 (冲突: 3,4,5,6,7,8)
// grantMask & LockConflicts[ShareLock]
//   = 0x000A & 0x00F8
//   = 0x0008 (bit 3, RowExclusiveLock 冲突!)
//   ≠ 0 → 有冲突
```

---

## 4. 分区锁算法

### 4.1 哈希分区

```c
// src/backend/storage/lmgr/lock.c:380
#define LOCK_HASH_PARTITIONS 16

// 计算分区号
static inline uint32
LockHashPartition(uint32 hashcode)
{
    return hashcode % LOCK_HASH_PARTITIONS;
}

// 获取分区锁
static inline LWLock *
LockHashPartitionLock(uint32 hashcode)
{
    return &MainLWLockArray[LOCK_MANAGER_LWLOCK_OFFSET + 
                            LockHashPartition(hashcode)].lock;
}
```

### 4.2 分区策略

```
为什么用 16 个分区？

权衡:
  分区太少 (如4个):
    ✗ 争用仍然较高
    ✓ 内存占用少
  
  分区太多 (如256个):
    ✓ 争用极低
    ✗ 内存占用大
    ✗ 遍历成本高 (LockReleaseAll需要遍历所有分区)
  
  16个分区:
    ✓ 平衡争用和内存
    ✓ 2^4 = 16 方便位运算
    ✓ CPU缓存友好 (16 * 64B ≈ 1KB)
```

---

## 5. 等待队列算法 (FIFO)

### 5.1 双向循环链表

```c
// 等待队列使用 dclist (doubly-linked circular list)
typedef struct dclist_head
{
    dlist_node *head;
} dclist_head;

// 加入队尾 (新等待者)
dclist_push_tail(&lock->waitProcs, &proc->links);

// 唤醒时从队头开始
dclist_foreach_modify(iter, &lock->waitProcs)
{
    PGPROC *proc = dclist_container(PGPROC, links, iter.cur);
    // 检查并唤醒...
}
```

### 5.2 FIFO 公平性保证

```
场景: 多个进程等待同一个锁

时间线:
  t0: T1 获取 ExclusiveLock
  t1: T2 等待 ShareLock (加入队列)
  t2: T3 等待 ShareLock (加入队列)
  t3: T4 等待 AccessShareLock (加入队列)
  t4: T1 释放 ExclusiveLock

唤醒顺序:
  1. 检查 T2 (ShareLock) → 无冲突 → 授予
  2. 检查 T3 (ShareLock) → 无冲突 → 授予
  3. 检查 T4 (AccessShareLock) → 无冲突 → 授予

结果: 按到达顺序授予，防止饥饿
```

---

## 6. 锁升级检测

### 6.1 检测锁升级请求

```c
// src/backend/storage/lmgr/lock.c:1100
static bool
IsLockUpgrade(LOCKMODE oldmode, LOCKMODE newmode)
{
    // PostgreSQL 定义的锁强度顺序:
    // 1=AccessShareLock (最弱)
    // 2=RowShareLock
    // 3=RowExclusiveLock
    // 4=ShareUpdateExclusiveLock
    // 5=ShareLock
    // 6=ShareRowExclusiveLock
    // 7=ExclusiveLock
    // 8=AccessExclusiveLock (最强)
    
    return newmode > oldmode;
}
```

### 6.2 拒绝锁升级

```c
// 如果检测到锁升级，返回错误
if (IsLockUpgrade(proclock->holdMask, lockmode))
{
    ereport(ERROR,
            (errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
             errmsg("lock upgrade not supported"),
             errhint("Acquire the stronger lock first.")));
}
```

---

## 7. 锁继承算法 (子事务)

### 7.1 子事务获取锁

```c
// 子事务获取的锁记录在 LOCALLOCK->lockOwners[]
typedef struct LOCALLOCKOWNER
{
    ResourceOwner owner;    // 子事务的 ResourceOwner
    int64 nLocks;           // 持有次数
} LOCALLOCKOWNER;

// 示例:
// 主事务获取 2 次，子事务获取 1 次
lockOwners[0] = {owner: TopTransactionResourceOwner, nLocks: 2}
lockOwners[1] = {owner: SubTransactionResourceOwner, nLocks: 1}
```

### 7.2 子事务回滚

```c
// src/backend/storage/lmgr/lock.c:3500
void
AtEOSubXact_Locks(bool isCommit, SubTransactionId mySubid,
                  SubTransactionId parentSubid)
{
    ResourceOwner currentOwner = CurrentResourceOwner;
    
    foreach (LOCALLOCK in LocalLockHash)
    {
        foreach (LOCALLOCKOWNER in locallock->lockOwners)
        {
            if (owner->owner == currentOwner)
            {
                if (isCommit)
                {
                    /* 提交: 锁继承给父事务 */
                    TransferToParent(owner, parentOwner);
                }
                else
                {
                    /* 回滚: 释放子事务持有的锁 */
                    ReleaseLockOwner(owner);
                }
            }
        }
    }
}
```

---

## 8. 咨询锁算法

### 8.1 咨询锁特点

```sql
-- 显式获取咨询锁
SELECT pg_advisory_lock(12345);  -- 阻塞直到获取
SELECT pg_try_advisory_lock(12345);  -- 非阻塞，返回 t/f

-- 咨询锁不与数据库对象关联，由应用定义语义
```

### 8.2 LOCKTAG 编码

```c
// 咨询锁的 LOCKTAG
SET_LOCKTAG_ADVISORY(tag,
                     MyDatabaseId,     // 数据库 OID
                     key1,             // 用户提供的 key1
                     key2,             // 用户提供的 key2
                     4);               // locktag_field4

// 示例: pg_advisory_lock(12345)
locktag.locktag_field1 = 12345;
locktag.locktag_field2 = 0;
locktag.locktag_type = LOCKTAG_ADVISORY;
```

---

## 9. 性能优化技巧

### 9.1 减少锁获取次数

```sql
-- 不好: 多次获取表锁
SELECT * FROM t WHERE id = 1;
SELECT * FROM t WHERE id = 2;
SELECT * FROM t WHERE id = 3;
-- 每次 SELECT 都获取 AccessShareLock (虽然有本地缓存)

-- 好: 一次查询
SELECT * FROM t WHERE id IN (1, 2, 3);
-- 只获取一次 AccessShareLock
```

### 9.2 避免不必要的强锁

```sql
-- 不好: SELECT FOR UPDATE 在只读场景
BEGIN;
SELECT * FROM t WHERE id = 1 FOR UPDATE;  -- RowExclusiveLock (强锁)
-- 只是读取数据...
COMMIT;

-- 好: 普通 SELECT
BEGIN;
SELECT * FROM t WHERE id = 1;  -- AccessShareLock (弱锁)
COMMIT;
```

### 9.3 缩短锁持有时间

```sql
-- 不好: 长事务
BEGIN;
SELECT * FROM big_table;  -- 持有锁
-- 执行其他业务逻辑 (耗时 10 秒)
COMMIT;

-- 好: 短事务
-- 先执行业务逻辑
-- 计算完成后再开事务
BEGIN;
SELECT * FROM big_table;
COMMIT;
```

---

## 总结

### 核心算法

1. **死锁检测**: DFS 遍历等待图，检测环，O(N²) 复杂度
2. **快速路径**: 线性扫描 16 个槽位，~85ns，3.9x 加速
3. **冲突判断**: 位掩码与操作，O(1) 时间
4. **分区锁**: 16 个分区，降低争用
5. **FIFO 队列**: 按到达顺序授予，保证公平性

### 性能优化

- ✅ 使用快速路径 (弱表锁)
- ✅ 减少锁获取次数
- ✅ 避免锁升级
- ✅ 缩短锁持有时间

---

**下一步**: 阅读 [05_performance.md](05_performance.md) 了解性能优化实践

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

