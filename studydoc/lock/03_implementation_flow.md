# Lock Manager 实现流程

本文档详细说明锁的获取、释放、等待、死锁检测等核心流程。

---

## 1. 锁获取流程 (LockAcquire)

### 1.1 完整流程图

```
LockAcquire(locktag, lockmode, dontWait)
  │
  ├─→ [1] 查找 LOCALLOCK
  │    ├─ 如果找到 && nLocks > 0
  │    │   └→ nLocks++ → 返回 ALREADY_HELD
  │    └─ 否则创建新 LOCALLOCK
  │
  ├─→ [2] 尝试快速路径 (仅限弱表锁)
  │    ├─ 如果成功 → 返回 OK
  │    └─ 否则降级到常规路径
  │
  ├─→ [3] 获取分区锁 (partition LWLock)
  │
  ├─→ [4] 查找/创建 LOCK
  │    └─ Hash查找，未找到则创建
  │
  ├─→ [5] 检查锁冲突
  │    ├─ 无冲突 → 跳到 [7]
  │    └─ 有冲突 → 继续 [6]
  │
  ├─→ [6] 等待锁
  │    ├─ 如果 dontWait=true → 返回 NOT_AVAIL
  │    ├─ 创建 PROCLOCK (waitProcLock)
  │    ├─ 更新 LOCK 计数 (requested++)
  │    ├─ 加入等待队列 (waitProcs)
  │    ├─ 释放分区锁
  │    ├─ 睡眠等待 (WaitOnLock)
  │    │   ├─ 超时检查
  │    │   ├─ 死锁检测 (1秒后启动)
  │    │   └─ 被唤醒或超时
  │    ├─ 重新获取分区锁
  │    └─ 继续 [7]
  │
  ├─→ [7] 授予锁
  │    ├─ 查找/创建 PROCLOCK
  │    ├─ 更新 LOCK 状态
  │    │   ├─ granted[mode]++
  │    │   ├─ grantMask |= (1 << mode)
  │    │   └─ nGranted++
  │    ├─ 更新 PROCLOCK 状态
  │    │   └─ holdMask |= (1 << mode)
  │    └─ 更新 LOCALLOCK 状态
  │        ├─ nLocks = 1
  │        ├─ lock = LOCK*
  │        └─ proclock = PROCLOCK*
  │
  └─→ [8] 释放分区锁 → 返回 OK
```

### 1.2 核心代码流程

```c
// src/backend/storage/lmgr/lock.c:730
LockAcquireResult
LockAcquire(const LOCKTAG *locktag,
            LOCKMODE lockmode,
            bool sessionLock,
            bool dontWait)
{
    LOCALLOCK *locallock;
    LOCK *lock;
    PROCLOCK *proclock;
    uint32 hashcode;
    LWLock *partitionLock;
    
    /* [1] 查找/创建 LOCALLOCK */
    hashcode = LockTagHashCode(locktag);
    locallock = (LOCALLOCK *) hash_search_with_hash_value(
        LockMethodLocalHash,
        (void *) locktag,
        hashcode,
        HASH_ENTER, NULL);
    
    /* 如果已持有，递增计数 */
    if (locallock->nLocks > 0)
    {
        GrantLockLocal(locallock, owner);
        return LOCKACQUIRE_ALREADY_HELD;
    }
    
    /* [2] 尝试快速路径 */
    if (EligibleForRelationFastPath(locktag, lockmode))
    {
        if (FastPathGrantRelationLock(locktag->locktag_field2, lockmode))
        {
            locallock->lock = NULL;  // 快速路径无需 LOCK
            GrantLockLocal(locallock, owner);
            return LOCKACQUIRE_OK;
        }
    }
    
    /* [3] 获取分区锁 */
    partitionLock = LockHashPartitionLock(hashcode);
    LWLockAcquire(partitionLock, LW_EXCLUSIVE);
    
    /* [4] 查找/创建 LOCK */
    lock = (LOCK *) hash_search_with_hash_value(
        LockMethodLockHash,
        (void *) locktag,
        hashcode,
        HASH_ENTER_NULL, &found);
    
    if (!found)
    {
        /* 初始化新 LOCK */
        lock->grantMask = 0;
        lock->waitMask = 0;
        MemSet(lock->requested, 0, sizeof(lock->requested));
        MemSet(lock->granted, 0, sizeof(lock->granted));
        dlist_init(&lock->procLocks);
        dclist_init(&lock->waitProcs);
    }
    
    /* [5] 检查冲突 */
    if (LockCheckConflicts(lock, lockmode, MyProc))
    {
        /* [6] 等待锁 */
        if (dontWait)
        {
            LWLockRelease(partitionLock);
            return LOCKACQUIRE_NOT_AVAIL;
        }
        
        WaitOnLock(locallock, owner, lockmode);
        // 唤醒后继续...
    }
    
    /* [7] 授予锁 */
    GrantLock(lock, proclock, lockmode);
    
    /* [8] 释放分区锁 */
    LWLockRelease(partitionLock);
    
    return LOCKACQUIRE_OK;
}
```

---

## 2. 快速路径流程

### 2.1 快速路径条件

```c
// 只有弱表锁才能使用快速路径:
static inline bool
EligibleForRelationFastPath(const LOCKTAG *locktag, LOCKMODE lockmode)
{
    return (locktag->locktag_type == LOCKTAG_RELATION &&
            locktag->locktag_lockmethodid == DEFAULT_LOCKMETHOD &&
            (lockmode == AccessShareLock || lockmode == RowShareLock));
}
```

### 2.2 快速路径获取

```c
// src/backend/storage/lmgr/lock.c:4400
static bool
FastPathGrantRelationLock(Oid relid, LOCKMODE lockmode)
{
    uint32 f;
    uint32 unused_slot = FP_LOCK_SLOTS_PER_BACKEND;
    
    /* 查找空槽或已存在的槽 */
    for (f = 0; f < FP_LOCK_SLOTS_PER_BACKEND; f++)
    {
        if (MyProc->fpRelId[f] == relid)
        {
            /* 找到相同表，递增计数 */
            Assert(FastPathLocalUseCount[f] > 0);
            FastPathLocalUseCount[f]++;
            return true;
        }
        if (MyProc->fpRelId[f] == InvalidOid)
            unused_slot = f;
    }
    
    /* 没有空槽，降级到常规路径 */
    if (unused_slot >= FP_LOCK_SLOTS_PER_BACKEND)
        return false;
    
    /* 使用空槽 */
    MyProc->fpRelId[unused_slot] = relid;
    MyProc->fpLockBits |= FAST_PATH_BITS(unused_slot, lockmode);
    FastPathLocalUseCount[unused_slot] = 1;
    
    return true;
}
```

---

## 3. 锁等待流程 (WaitOnLock)

### 3.1 等待流程图

```
WaitOnLock(locallock, owner, lockmode)
  │
  ├─→ [1] 准备等待
  │    ├─ 创建 PROCLOCK (waitProcLock)
  │    ├─ 更新 LOCK->requested[mode]++
  │    ├─ 加入等待队列 (dclist_push_tail)
  │    ├─ MyProc->waitLock = lock
  │    └─ MyProc->waitLockMode = lockmode
  │
  ├─→ [2] 释放分区锁
  │    └─ LWLockRelease(partitionLock)
  │
  ├─→ [3] 等待循环
  │    while (true)
  │    {
  │      ├─ PGSemaphoreLock(&MyProc->sem)  // 睡眠
  │      │
  │      ├─ 检查是否被授予
  │      │   └─ if (MyProc->waitStatus == PROC_WAIT_STATUS_OK)
  │      │       └→ break  // 成功获取锁
  │      │
  │      ├─ 检查超时
  │      │   └─ if (timeout && now > deadline)
  │      │       └→ 返回 ERROR (超时)
  │      │
  │      ├─ 死锁检测 (1秒后)
  │      │   └─ if (now - start > DeadlockTimeout)
  │      │       ├─ 启动死锁检测器
  │      │       └─ if (检测到死锁)
  │      │           └→ 返回 ERROR (死锁)
  │      │
  │      └─ 继续等待
  │    }
  │
  └─→ [4] 重新获取分区锁
       └─ LWLockAcquire(partitionLock, LW_EXCLUSIVE)
```

### 3.2 死锁超时参数

```sql
-- 死锁检测延迟 (默认 1秒)
deadlock_timeout = 1s

-- 锁等待超时 (默认禁用)
lock_timeout = 0  -- 0表示无限等待

-- 语句超时
statement_timeout = 0
```

---

## 4. 锁释放流程 (LockRelease)

### 4.1 完整流程图

```
LockRelease(locktag, lockmode, sessionLock)
  │
  ├─→ [1] 查找 LOCALLOCK
  │    └─ 如果未找到 → 错误
  │
  ├─→ [2] 递减本地计数
  │    ├─ nLocks--
  │    └─ if (nLocks > 0) → 返回 (还有其他持有者)
  │
  ├─→ [3] 检查快速路径
  │    └─ if (lock == NULL) → FastPathUnGrantRelationLock()
  │
  ├─→ [4] 获取分区锁
  │    └─ LWLockAcquire(partitionLock, LW_EXCLUSIVE)
  │
  ├─→ [5] 更新 LOCK 状态
  │    ├─ granted[mode]--
  │    ├─ requested[mode]--
  │    ├─ nGranted--
  │    └─ if (granted[mode] == 0)
  │        └─ grantMask &= ~(1 << mode)
  │
  ├─→ [6] 更新 PROCLOCK 状态
  │    ├─ holdMask &= ~(1 << mode)
  │    └─ if (holdMask == 0)
  │        └─ 删除 PROCLOCK
  │
  ├─→ [7] 检查是否删除 LOCK
  │    └─ if (nGranted == 0 && nRequested == 0)
  │        └─ 删除 LOCK
  │
  ├─→ [8] 唤醒等待进程
  │    └─ if (有等待进程)
  │        └─ ProcLockWakeup(lock)
  │
  ├─→ [9] 释放分区锁
  │    └─ LWLockRelease(partitionLock)
  │
  └─→ [10] 删除 LOCALLOCK
       └─ hash_search(HASH_REMOVE)
```

### 4.2 核心代码

```c
// src/backend/storage/lmgr/lock.c:2000
bool
LockRelease(const LOCKTAG *locktag,
            LOCKMODE lockmode,
            bool sessionLock)
{
    LOCALLOCK *locallock;
    LOCK *lock;
    PROCLOCK *proclock;
    
    /* [1] 查找 LOCALLOCK */
    locallock = GetLocalLock(locktag, lockmode, HASH_FIND);
    if (!locallock)
        elog(ERROR, "lock not held");
    
    /* [2] 递减本地计数 */
    if (!RemoveLocalLock(locallock, owner))
        return true;  // 还有其他持有者
    
    /* [3] 快速路径 */
    if (locallock->lock == NULL)
    {
        FastPathUnGrantRelationLock(locktag->locktag_field2, lockmode);
        RemoveLocalLock(locallock);
        return true;
    }
    
    /* [4] 获取分区锁 */
    LWLockAcquire(partitionLock, LW_EXCLUSIVE);
    
    lock = locallock->lock;
    proclock = locallock->proclock;
    
    /* [5-6] 更新 LOCK 和 PROCLOCK */
    UnGrantLock(lock, lockmode, proclock);
    
    /* [7] 删除空 LOCK */
    if (lock->nGranted == 0)
        hash_search(LockMethodLockHash, &lock->tag, HASH_REMOVE, NULL);
    
    /* [8] 唤醒等待进程 */
    ProcLockWakeup(lock);
    
    /* [9] 释放分区锁 */
    LWLockRelease(partitionLock);
    
    /* [10] 删除 LOCALLOCK */
    RemoveLocalLock(locallock);
    
    return true;
}
```

---

## 5. 唤醒等待进程 (ProcLockWakeup)

### 5.1 唤醒策略

```c
// src/backend/storage/lmgr/proc.c:1450
int
ProcLockWakeup(LOCK *lock)
{
    dclist_head *waitQueue = &lock->waitProcs;
    int nwoken = 0;
    
    /* 遍历等待队列 (FIFO顺序) */
    dclist_foreach_modify(iter, waitQueue)
    {
        PGPROC *proc = dclist_container(PGPROC, links, iter.cur);
        LOCKMODE lockmode = proc->waitLockMode;
        
        /* 检查是否可以授予 */
        if (LockCheckConflicts(lock, lockmode, proc))
            break;  // 有冲突，停止
        
        /* 授予锁 */
        GrantLock(lock, proc->waitProcLock, lockmode);
        
        /* 从等待队列移除 */
        dclist_delete_from(waitQueue, iter.cur);
        
        /* 唤醒进程 */
        proc->waitStatus = PROC_WAIT_STATUS_OK;
        PGSemaphoreUnlock(&proc->sem);
        nwoken++;
    }
    
    return nwoken;
}
```

### 5.2 FIFO 公平性

```
等待队列示例:

LOCK->waitProcs: [P1:ShareLock] → [P2:ExclusiveLock] → [P3:ShareLock]

当前持有: [P0:RowExclusiveLock] 释放

唤醒过程:
  1. 检查 P1 (ShareLock) → 无冲突 → 授予并唤醒
  2. 检查 P2 (ExclusiveLock) → 与P1冲突 → 停止
  3. P3 继续等待 (即使无冲突，也要等P2先获取)

结果: FIFO 防止饥饿
```

---

## 6. 批量释放锁 (LockReleaseAll)

### 6.1 使用场景

```c
// 事务结束时批量释放
void
AtEOXact_Locks(bool isCommit)
{
    LockReleaseAll(DEFAULT_LOCKMETHOD, false);
}

// 子事务回滚时部分释放
void
AtEOSubXact_Locks(bool isCommit, SubTransactionId mySubid)
{
    // 只释放子事务持有的锁
}
```

### 6.2 批量释放流程

```
LockReleaseAll(lockmethodid, allLocks)
  │
  ├─→ [1] 遍历所有 LOCALLOCK
  │    └─ HASH_SEQ_SEARCH(LockMethodLocalHash)
  │
  ├─→ [2] 对每个 LOCALLOCK:
  │    ├─ 检查是否需要释放
  │    │   └─ 根据 ResourceOwner 判断
  │    │
  │    ├─ 快速路径释放
  │    │   └─ if (lock == NULL)
  │    │       └→ FastPathUnGrantRelationLock()
  │    │
  │    ├─ 常规路径释放
  │    │   ├─ 获取分区锁
  │    │   ├─ 更新 LOCK/PROCLOCK
  │    │   ├─ 唤醒等待进程
  │    │   └─ 释放分区锁
  │    │
  │    └─ 删除 LOCALLOCK
  │
  └─→ [3] 清理快速路径
       └─ 批量清零 fpRelId[] 和 fpLockBits
```

---

## 7. 死锁检测流程

### 7.1 检测触发

```c
// 等待锁超过 1 秒后触发
if (get_timeout_active(DEADLOCK_TIMEOUT))
{
    if (CheckDeadLockAlert())
    {
        /* 启动死锁检测 */
        DeadLockCheck();
    }
}
```

### 7.2 死锁检测算法 (DFS)

```
CheckForDeadlock()
  │
  ├─→ [1] 初始化 visited[] 数组
  │
  ├─→ [2] 从当前进程开始 DFS
  │    └─ DeadLockCheckRecurse(MyProc)
  │
  └─→ [3] DFS 递归:
       │
       DeadLockCheckRecurse(proc)
       {
         ├─ 标记 visited[proc] = true
         │
         ├─ 获取 proc 等待的 LOCK
         │   └─ waitLock = proc->waitLock
         │
         ├─ 遍历持有 waitLock 的所有进程
         │   foreach (PROCLOCK in waitLock->procLocks)
         │   {
         │     blocker = PROCLOCK->myProc;
         │     
         │     ├─ if (blocker == MyProc)
         │     │   └→ 找到环! 返回 DEADLOCK
         │     │
         │     ├─ if (visited[blocker])
         │     │   └→ 跳过 (已访问)
         │     │
         │     └─ 递归检查
         │         └─ DeadLockCheckRecurse(blocker)
         │   }
         │
         └─ 返回 NO_DEADLOCK
       }
```

### 7.3 死锁示例

```
场景:
  T1: 持有 LOCK_A，等待 LOCK_B
  T2: 持有 LOCK_B，等待 LOCK_A

检测过程:
  1. T1 触发死锁检测
  2. DFS: T1 → (等待B) → T2 → (等待A) → T1
  3. 找到环! 检测到死锁
  4. 选择 T1 作为受害者 (当前进程)
  5. 中止 T1: ERROR: deadlock detected
```

---

## 8. 锁升级 (Lock Promotion)

PostgreSQL **不支持自动锁升级**，只支持**锁降级**（从强锁释放到弱锁）。

### 8.1 为什么不支持锁升级？

```
假设支持锁升级:

T1: 持有 AccessShareLock，尝试升级到 ExclusiveLock
T2: 持有 AccessShareLock，尝试升级到 ExclusiveLock

结果: 死锁!
  T1 等待 T2 释放 AccessShareLock
  T2 等待 T1 释放 AccessShareLock

PostgreSQL 选择: 禁止锁升级，避免此类死锁
```

### 8.2 手动模拟锁升级

```sql
-- 错误示例 (会死锁):
BEGIN;
SELECT * FROM t WHERE id = 1;  -- 获取 AccessShareLock
UPDATE t SET val = 2 WHERE id = 1;  -- 需要 RowExclusiveLock (冲突!)

-- 正确方法: 一开始就获取足够强的锁
BEGIN;
SELECT * FROM t WHERE id = 1 FOR UPDATE;  -- RowExclusiveLock
UPDATE t SET val = 2 WHERE id = 1;  -- 已持有，无需升级
COMMIT;
```

---

## 总结

### 关键流程

1. **LockAcquire**: 查找 LOCALLOCK → 快速路径 → 冲突检查 → 等待/授予
2. **WaitOnLock**: 加入等待队列 → 睡眠 → 死锁检测 → 唤醒
3. **LockRelease**: 递减计数 → 更新状态 → 唤醒等待者
4. **ProcLockWakeup**: FIFO 遍历 → 冲突检查 → 批量唤醒

### 性能优化

- ✅ **LOCALLOCK 缓存**: 避免重复访问共享内存
- ✅ **快速路径**: 弱表锁零哈希查找
- ✅ **分区锁**: 16个分区降低争用
- ✅ **批量释放**: 事务结束时一次性清理

---

**下一步**: 阅读 [04_key_algorithms.md](04_key_algorithms.md) 了解死锁检测算法

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

