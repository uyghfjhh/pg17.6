# Lock Manager 架构图表

本文档提供Lock Manager的核心架构图、流程图和数据结构关系图。

---

## 1. 整体架构图

```
┌────────────────────────────────────────────────────────────────────────┐
│                      PostgreSQL Lock Manager                            │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                    应用层 (SQL Commands)                         │  │
│  │  SELECT | INSERT | UPDATE | DELETE | LOCK TABLE | ALTER TABLE   │  │
│  └────────────────────────┬────────────────────────────────────────┘  │
│                           │                                            │
│  ┌────────────────────────▼────────────────────────────────────────┐  │
│  │              高层锁管理 (Lock Manager API)                       │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │  │
│  │  │LockAcquire() │  │LockRelease() │  │LockReleaseAll│         │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘         │  │
│  └────────────────────────┬────────────────────────────────────────┘  │
│                           │                                            │
│  ┌────────────────────────▼────────────────────────────────────────┐  │
│  │                 快速路径优化 (Fast Path)                         │  │
│  │  • 仅限弱表锁 (AccessShareLock, RowShareLock)                   │  │
│  │  • 16个槽位 (fpRelId[], fpLockBits)                             │  │
│  │  • 无哈希查找，直接数组访问                                      │  │
│  └────────────────────────┬────────────────────────────────────────┘  │
│                           │ (降级到常规路径)                           │
│  ┌────────────────────────▼────────────────────────────────────────┐  │
│  │              本地缓存 (LOCALLOCK Hash Table)                     │  │
│  │  Key: (LOCKTAG, LOCKMODE)                                       │  │
│  │  • 避免重复访问共享内存                                          │  │
│  │  • 记录持有次数 (nLocks)                                         │  │
│  └────────────────────────┬────────────────────────────────────────┘  │
│                           │                                            │
│  ┌────────────────────────▼────────────────────────────────────────┐  │
│  │          共享内存锁表 (16 个分区, LWLock保护)                    │  │
│  │  ┌──────────────────────────────────────────────────────────┐  │  │
│  │  │ LOCK Hash Table (Key: LOCKTAG)                           │  │  │
│  │  │  • 被锁对象信息                                           │  │  │
│  │  │  • 授予/等待计数                                          │  │  │
│  │  │  • 等待队列 (waitProcs)                                   │  │  │
│  │  └────────────────┬─────────────────────────────────────────┘  │  │
│  │                   │                                              │  │
│  │  ┌────────────────▼─────────────────────────────────────────┐  │  │
│  │  │ PROCLOCK Hash Table (Key: LOCK*, PGPROC*)                │  │  │
│  │  │  • 进程-锁关联关系                                        │  │  │
│  │  │  • 持有的锁模式 (holdMask)                                │  │  │
│  │  └──────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                           │                                            │
│  ┌────────────────────────▼────────────────────────────────────────┐  │
│  │                死锁检测器 (Deadlock Detector)                    │  │
│  │  • DFS 算法检测等待图环                                          │  │
│  │  • 延迟启动 (deadlock_timeout = 1s)                             │  │
│  │  • 自杀策略 (中止当前进程)                                       │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 锁获取流程图

```
LockAcquire(LOCKTAG, LOCKMODE, dontWait)
│
├─ [1] 查找 LOCALLOCK (本地哈希表)
│   ├─ 找到 && nLocks > 0
│   │   └─→ nLocks++ → 返回 ALREADY_HELD ✓
│   └─ 否则创建新条目
│
├─ [2] 尝试快速路径 (仅限弱表锁)
│   ├─ EligibleForRelationFastPath()?
│   │   ├─ Yes → FastPathGrantRelationLock()
│   │   │   ├─ 查找 fpRelId[] 槽位
│   │   │   ├─ 找到空槽 → 设置并返回 OK ✓
│   │   │   └─ 槽位已满 → 降级到 [3]
│   │   └─ No → 继续 [3]
│   └─ (快速路径成功则跳过后续步骤)
│
├─ [3] 获取分区锁 (LWLock)
│   └─ partitionLock = LockHashPartitionLock(hashcode)
│       LWLockAcquire(partitionLock, LW_EXCLUSIVE)
│
├─ [4] 查找/创建 LOCK (共享哈希表)
│   ├─ hash_search_with_hash_value(LockMethodLockHash)
│   └─ 如果新建 → 初始化 grantMask, waitMask, 计数器
│
├─ [5] 检查锁冲突
│   ├─ LockCheckConflicts(lock, lockmode)
│   │   └─ (lock->grantMask & LockConflicts[lockmode]) != 0?
│   │
│   ├─ 无冲突 → 跳到 [7]
│   └─ 有冲突 → 继续 [6]
│
├─ [6] 等待锁
│   ├─ dontWait == true?
│   │   └─→ 返回 NOT_AVAIL ✗
│   │
│   ├─ 创建 PROCLOCK (waitProcLock)
│   ├─ 更新 LOCK->requested[mode]++
│   ├─ 加入等待队列: dclist_push_tail(&lock->waitProcs, MyProc)
│   ├─ MyProc->waitLock = lock
│   ├─ MyProc->waitLockMode = lockmode
│   │
│   ├─ 释放分区锁
│   │   └─ LWLockRelease(partitionLock)
│   │
│   ├─ 等待循环:
│   │   while (true) {
│   │     PGSemaphoreLock(&MyProc->sem);  // 睡眠
│   │     
│   │     if (MyProc->waitStatus == OK)
│   │       break;  // 被授予 ✓
│   │     
│   │     if (timeout)
│   │       ERROR: lock timeout ✗
│   │     
│   │     if (now - start > deadlock_timeout) {
│   │       if (DeadLockCheck() == DEADLOCK)
│   │         ERROR: deadlock detected ✗
│   │     }
│   │   }
│   │
│   └─ 重新获取分区锁
│       └─ LWLockAcquire(partitionLock, LW_EXCLUSIVE)
│
├─ [7] 授予锁
│   ├─ 查找/创建 PROCLOCK
│   ├─ 更新 LOCK 状态:
│   │   ├─ granted[mode]++
│   │   ├─ requested[mode]++  (如果是新请求)
│   │   ├─ grantMask |= (1 << mode)
│   │   └─ nGranted++
│   │
│   ├─ 更新 PROCLOCK 状态:
│   │   └─ holdMask |= (1 << mode)
│   │
│   └─ 更新 LOCALLOCK 状态:
│       ├─ nLocks = 1
│       ├─ lock = LOCK*
│       └─ proclock = PROCLOCK*
│
└─ [8] 释放分区锁
    ├─ LWLockRelease(partitionLock)
    └─→ 返回 OK ✓
```

---

## 3. 锁释放流程图

```
LockRelease(LOCKTAG, LOCKMODE)
│
├─ [1] 查找 LOCALLOCK
│   └─ 未找到 → ERROR: lock not held ✗
│
├─ [2] 递减本地计数
│   ├─ nLocks--
│   └─ if (nLocks > 0)
│       └─→ 返回 true ✓ (还有其他持有者)
│
├─ [3] 快速路径释放
│   └─ if (lock == NULL)  // 快速路径锁
│       ├─ FastPathUnGrantRelationLock()
│       │   ├─ 清除 fpRelId[slot]
│       │   └─ 清除 fpLockBits
│       ├─ 删除 LOCALLOCK
│       └─→ 返回 true ✓
│
├─ [4] 获取分区锁
│   └─ LWLockAcquire(partitionLock, LW_EXCLUSIVE)
│
├─ [5] 更新 LOCK 状态
│   ├─ granted[mode]--
│   ├─ requested[mode]--
│   ├─ nGranted--
│   ├─ nRequested--
│   │
│   └─ if (granted[mode] == 0)
│       └─ grantMask &= ~(1 << mode)
│
├─ [6] 更新 PROCLOCK 状态
│   ├─ holdMask &= ~(1 << mode)
│   │
│   └─ if (holdMask == 0)
│       ├─ 从 lock->procLocks 链表移除
│       └─ 删除 PROCLOCK (hash_search HASH_REMOVE)
│
├─ [7] 检查是否删除 LOCK
│   └─ if (nGranted == 0 && nRequested == 0)
│       └─ 删除 LOCK (hash_search HASH_REMOVE)
│
├─ [8] 唤醒等待进程
│   └─ if (lock->waitProcs 非空)
│       └─ ProcLockWakeup(lock)
│           │
│           ├─ foreach (PGPROC in waitProcs, FIFO顺序)
│           │   ├─ 检查冲突: LockCheckConflicts()
│           │   │   ├─ 无冲突:
│           │   │   │   ├─ GrantLock()
│           │   │   │   ├─ 从队列移除
│           │   │   │   ├─ proc->waitStatus = OK
│           │   │   │   └─ PGSemaphoreUnlock(&proc->sem)
│           │   │   │
│           │   │   └─ 有冲突:
│           │   │       └─ break (停止唤醒)
│           │   │
│           │   └─ 继续下一个进程...
│           │
│           └─→ 返回唤醒数量
│
├─ [9] 释放分区锁
│   └─ LWLockRelease(partitionLock)
│
└─ [10] 删除 LOCALLOCK
    ├─ hash_search(LockMethodLocalHash, HASH_REMOVE)
    └─→ 返回 true ✓
```

---

## 4. 死锁检测流程图

```
DeadLockCheck()
│
├─ [1] 延迟启动
│   └─ 等待时间 < deadlock_timeout (1s)?
│       └─→ 返回 NO_DEADLOCK (还没到时间)
│
├─ [2] 构建等待图
│   │
│   BuildWaitGraph():
│   ├─ foreach (PGPROC *proc in AllProcs)
│   │   ├─ if (proc->waitLock == NULL)
│   │   │   └─→ 跳过 (不在等待)
│   │   │
│   │   ├─ lock = proc->waitLock
│   │   │
│   │   └─ foreach (PROCLOCK in lock->procLocks)
│   │       └─ if (LockModeConflicts(proc->waitLockMode, 
│   │                                 proclock->holdMask))
│   │           └─ AddEdge(proc → proclock->myProc)
│   │
│   └─→ 生成邻接表: edges[]
│
├─ [3] DFS 检测环
│   │
│   FindLockCycle():
│   ├─ 初始化 visited[] = 0
│   ├─ 起点 = MyProc
│   │
│   └─ FindLockCycleRecurse(MyProc):
│       │
│       ├─ visited[MyProc] = 1  // 正在访问
│       ├─ path[pathLen++] = MyProc
│       │
│       ├─ foreach (edge: MyProc → blocker)
│       │   │
│       │   ├─ if (visited[blocker] == 1)  // 正在访问
│       │   │   └─ if (blocker in path)
│       │   │       └─→ 找到环! ✓
│       │   │           ├─ 记录死锁路径
│       │   │           └─→ 返回 DEADLOCK
│       │   │
│       │   └─ if (visited[blocker] == 0)  // 未访问
│       │       └─ 递归: FindLockCycleRecurse(blocker)
│       │           └─ if (返回 DEADLOCK)
│       │               └─→ 返回 DEADLOCK ✓
│       │
│       ├─ 回溯:
│       │   ├─ pathLen--
│       │   └─ visited[MyProc] = 2  // 访问完成
│       │
│       └─→ 返回 NO_DEADLOCK
│
├─ [4] 选择受害者
│   └─ 策略: 选择当前进程 (MyProc)
│       └─ 原因: 避免中止其他已运行久的事务
│
└─ [5] 中止事务
    └─ ereport(ERROR,
               (errcode(ERRCODE_T_R_DEADLOCK_DETECTED),
                errmsg("deadlock detected"),
                errdetail("...")));

示例死锁图:
  T1: 持有 LOCK_A, 等待 LOCK_B
  T2: 持有 LOCK_B, 等待 LOCK_A
  
  等待图:
    T1 → T2 → T1  (环!)
    
  DFS 路径:
    起点: T1
    T1 → T2 → T1 (检测到 T1 正在访问) → 死锁!
```

---

## 5. 数据结构关系图

```
┌────────────────────────────────────────────────────────────────────┐
│                        LOCK 结构                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ LOCKTAG tag = {relation OID, database OID, ...}              │ │
│  │ LOCKMASK grantMask = 0x000A  (bits: 1,3 已授予)             │ │
│  │ LOCKMASK waitMask  = 0x0080  (bit: 7 等待中)                │ │
│  │                                                               │ │
│  │ int requested[9] = {0, 3, 0, 1, 0, 0, 0, 1, 0}              │ │
│  │    索引:           0  1  2  3  4  5  6  7  8                │ │
│  │    模式:           -  AS RS RE SU S  SR EX AE                │ │
│  │                                                               │ │
│  │ int granted[9]   = {0, 3, 0, 1, 0, 0, 0, 0, 0}              │ │
│  │                                                               │ │
│  │ dlist_head procLocks  → [PROCLOCK₁] ⇄ [PROCLOCK₂] ⇄ ...    │ │
│  │ dclist_head waitProcs → [PGPROC₃] ⇄ [PGPROC₄] ⇄ ...        │ │
│  └──────────────────────────────────────────────────────────────┘ │
└─────────────────┬──────────────────────────────────────────────────┘
                  │ procLocks 链表
                  ▼
┌────────────────────────────────────────────────────────────────────┐
│                     PROCLOCK 结构                                   │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ PROCLOCKTAG tag = {myLock: LOCK*, myProc: PGPROC*}          │ │
│  │ LOCKMASK holdMask = 0x0002  (持有 AccessShareLock)          │ │
│  │                                                               │ │
│  │ dlist_node lockLink  (在 LOCK->procLocks 中)                │ │
│  │ dlist_node procLink  (在 PGPROC->myProcLocks 中)            │ │
│  └──────────────────────────────────────────────────────────────┘ │
└─────────────────┬──────────────────────────────────────────────────┘
                  │ myProc 指针
                  ▼
┌────────────────────────────────────────────────────────────────────┐
│                      PGPROC 结构                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ int pid = 12345                                              │ │
│  │ LOCK *waitLock = NULL  (不在等待)                            │ │
│  │ LOCKMODE waitLockMode                                        │ │
│  │                                                               │ │
│  │ dlist_head myProcLocks → [PROCLOCK₁] ⇄ [PROCLOCK₂] ⇄ ...   │ │
│  │                                                               │ │
│  │ /* 快速路径 */                                                │ │
│  │ Oid fpRelId[16] = {16384, 16385, 0, ...}                    │ │
│  │ uint64 fpLockBits = 0x...0021                                │ │
│  │   slot 0: mode=1 (AccessShareLock)                           │ │
│  │   slot 1: mode=2 (RowShareLock)                              │ │
│  │   ...                                                         │ │
│  └──────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│                   LOCALLOCK 结构 (本地内存)                         │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ LOCALLOCKTAG tag = {LOCKTAG, LOCKMODE=AccessShareLock}      │ │
│  │                                                               │ │
│  │ LOCK *lock         → 指向共享 LOCK                           │ │
│  │ PROCLOCK *proclock → 指向共享 PROCLOCK                       │ │
│  │                                                               │ │
│  │ int64 nLocks = 3  (递归获取 3 次)                            │ │
│  │                                                               │ │
│  │ LOCALLOCKOWNER lockOwners[] =                                │ │
│  │   [{owner: TopTransactionResourceOwner, nLocks: 2},         │ │
│  │    {owner: SubTransactionResourceOwner, nLocks: 1}]         │ │
│  └──────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

---

## 6. 锁冲突矩阵

```
请求锁 ↓  已持有锁 →  │ AS  RS  RE  SU  S   SR  EX  AE
────────────────────────┼────────────────────────────────
1. AccessShareLock      │  ✓   ✓   ✓   ✓   ✓   ✓   ✓   ✗
2. RowShareLock         │  ✓   ✓   ✓   ✓   ✓   ✓   ✗   ✗
3. RowExclusiveLock     │  ✓   ✓   ✓   ✓   ✗   ✗   ✗   ✗
4. ShareUpdateExcLock   │  ✓   ✓   ✓   ✗   ✗   ✗   ✗   ✗
5. ShareLock            │  ✓   ✓   ✗   ✗   ✓   ✗   ✗   ✗
6. ShareRowExclusiveLock│  ✓   ✓   ✗   ✗   ✗   ✗   ✗   ✗
7. ExclusiveLock        │  ✓   ✗   ✗   ✗   ✗   ✗   ✗   ✗
8. AccessExclusiveLock  │  ✗   ✗   ✗   ✗   ✗   ✗   ✗   ✗

图例:
  ✓ = 可以共存
  ✗ = 冲突 (阻塞)

常见操作的锁模式:
  SELECT             → AccessShareLock (AS)
  SELECT FOR SHARE   → RowShareLock (RS)
  INSERT/UPDATE/DELETE → RowExclusiveLock (RE)
  CREATE INDEX       → ShareLock (S)
  VACUUM             → ShareUpdateExclusiveLock (SU)
  ALTER TABLE        → AccessExclusiveLock (AE)
```

---

## 7. 快速路径架构

```
┌──────────────────────────────────────────────────────────────────┐
│                    PGPROC 快速路径                                │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Oid fpRelId[FP_LOCK_SLOTS_PER_BACKEND=16]                  │ │
│  │   [0] = 16384  (table OID)                                 │ │
│  │   [1] = 16385                                              │ │
│  │   [2] = InvalidOid (空)                                    │ │
│  │   ...                                                       │ │
│  │   [15] = InvalidOid                                        │ │
│  │                                                             │ │
│  │ uint64 fpLockBits (64位, 每个槽4位)                        │ │
│  │   位 0-3:   0001 (slot 0: AccessShareLock)                │ │
│  │   位 4-7:   0010 (slot 1: RowShareLock)                   │ │
│  │   位 8-11:  0000 (slot 2: empty)                          │ │
│  │   ...                                                       │ │
│  │   位 60-63: 0000 (slot 15: empty)                         │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘

快速路径查找流程:
  LockAcquire(table=16384, mode=AccessShareLock)
    ├─→ 线性扫描 fpRelId[0..15]
    ├─→ 找到 fpRelId[0] == 16384
    ├─→ 检查 fpLockBits 的 slot 0 (位 0-3)
    ├─→ mode == AccessShareLock (1) 匹配
    └─→ 返回成功! (总耗时 ~85ns)

对比常规路径 (~330ns):
  ├─→ 计算 hashcode
  ├─→ 查找 LOCALLOCK 哈希表
  ├─→ 获取分区锁 (LWLock)
  ├─→ 查找 LOCK 哈希表
  ├─→ 更新 LOCK/PROCLOCK
  └─→ 释放分区锁

加速比: 330ns / 85ns ≈ 3.9x
```

---

## 8. 锁等待队列 FIFO

```
场景: 多个进程等待同一个锁

时间线:
  t0: P0 持有 ExclusiveLock
  t1: P1 请求 ShareLock (加入队列)
  t2: P2 请求 ShareLock (加入队列)
  t3: P3 请求 AccessShareLock (加入队列)
  t4: P0 释放 ExclusiveLock

LOCK->waitProcs (双向循环链表):
  ┌─────────────────────────────────────────────────┐
  │  [P1:ShareLock] ⇄ [P2:ShareLock] ⇄ [P3:AS] ⇄  │
  └─────────────────────────────────────────────────┘
   ↑                                                 │
   └─────────────────────────────────────────────────┘

P0 释放锁后，ProcLockWakeup() 流程:
  1. 检查 P1 (ShareLock) → 无冲突 → 授予并唤醒 ✓
  2. 检查 P2 (ShareLock) → 无冲突 → 授予并唤醒 ✓
  3. 检查 P3 (AS) → 无冲突 → 授予并唤醒 ✓

结果: 按到达顺序 (FIFO) 授予，保证公平性

反例 (如果不是FIFO):
  如果P3先获取 → P1, P2 可能一直等待 (饥饿问题)
```

---

## 总结

### 核心架构

1. **四层锁系统**: 快速路径 → LOCALLOCK → LOCK → PROCLOCK
2. **16个分区**: 降低锁争用
3. **死锁检测**: DFS 算法检测等待图环
4. **FIFO 队列**: 保证公平性，防止饥饿

### 性能优化

- ✅ **快速路径**: 3.9x 加速 (弱表锁)
- ✅ **本地缓存**: 避免重复访问共享内存
- ✅ **位操作**: 快速冲突检查 (O(1))
- ✅ **分区锁**: 降低全局争用

---

**完成**: Lock Manager 模块所有文档已完成！

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

