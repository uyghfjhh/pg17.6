# VACUUM 核心原理与算法

> VACUUM的核心原理、Lazy VACUUM算法、Autovacuum机制、性能优化

**源码**: `src/backend/commands/vacuum.c`, `src/backend/postmaster/autovacuum.c`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. VACUUM核心原理

### 1.1 为什么需要VACUUM？

```
MVCC导致的问题:

UPDATE users SET name = 'Bob' WHERE id = 1;

实际操作:
  1. 插入新元组 (t_xmin = 当前XID)
  2. 标记旧元组删除 (t_xmax = 当前XID)
  
结果:
  • 旧元组仍占用空间 (死元组)
  • 表膨胀
  • 查询变慢 (需跳过死元组)

DELETE users WHERE id = 1;

实际操作:
  1. 标记元组删除 (t_xmax = 当前XID)
  2. 不立即回收空间
  
结果:
  • 死元组累积
  • 需要VACUUM清理
```

### 1.2 VACUUM的作用

```
✅ 回收死元组空间 (标记为可重用)
✅ 更新FSM (Free Space Map)
✅ 更新VM (Visibility Map)
✅ 防止事务ID回卷
✅ 更新统计信息 (如果带ANALYZE)
✅ 触发索引清理
```

---

## 2. Lazy VACUUM算法

### 2.1 完整流程

```
VACUUM table_name
│
├─→ [1] 获取表锁 (ShareUpdateExclusiveLock)
│    └─ 允许SELECT/INSERT/UPDATE/DELETE
│        但阻塞ALTER TABLE、CREATE INDEX等
│
├─→ [2] 扫描堆表
│    for (each page) {
│      │
│      ├─ 获取页面独占锁
│      │
│      ├─ 检查每个元组可见性
│      │   └─ HeapTupleSatisfiesVacuum()
│      │       ├─ 可见 → 跳过
│      │       ├─ 死元组 → 记录TID
│      │       └─ 最近死亡 → 可能跳过
│      │
│      ├─ 标记死元组为LP_DEAD
│      ├─ 压实页面空间 (PageRepairFragmentation)
│      ├─ 更新FSM (记录空闲空间)
│      └─ 释放页面锁
│    }
│
├─→ [3] 清理索引
│    └─ 对每个索引调用ambulkdelete()
│        ├─ 扫描索引
│        ├─ 删除指向死元组的索引项
│        └─ 压缩索引页面
│
├─→ [4] 第二次扫描堆表
│    └─ 回收在步骤[2]和[3]之间新产生的死元组
│
├─→ [5] 更新VM (Visibility Map)
│    └─ 标记all-visible页面
│
├─→ [6] 清理FSM
│    └─ vacuum_rel() → FreeSpaceMapVacuum()
│
├─→ [7] 更新pg_class
│    └─ reltuples, relpages等统计信息
│
└─→ [8] 释放表锁
```

### 2.2 核心代码流程

```c
// src/backend/commands/vacuum.c:1800
static void
lazy_scan_heap(LVRelState *vacrel)
{
    BlockNumber nblocks = vacrel->rel_pages;
    BlockNumber blkno;
    Buffer buf;
    Page page;
    OffsetNumber offnum, maxoff;
    
    /* [1] 扫描所有页面 */
    for (blkno = 0; blkno < nblocks; blkno++)
    {
        /* 跳过all-visible页面 (VM优化) */
        if (VM_ALL_VISIBLE(vacrel->rel, blkno, &vmbuffer))
        {
            vacrel->skipped_allvis++;
            continue;
        }
        
        /* 读取页面 */
        buf = ReadBufferExtended(vacrel->rel, MAIN_FORKNUM, blkno, RBM_NORMAL, vac_strategy);
        LockBuffer(buf, BUFFER_LOCK_EXCLUSIVE);
        page = BufferGetPage(buf);
        
        maxoff = PageGetMaxOffsetNumber(page);
        
        /* [2] 扫描页面中的所有元组 */
        for (offnum = FirstOffsetNumber; offnum <= maxoff; offnum = OffsetNumberNext(offnum))
        {
            ItemId itemid = PageGetItemId(page, offnum);
            HeapTupleData tuple;
            
            if (!ItemIdIsNormal(itemid))
                continue;
            
            ItemPointerSet(&(tuple.t_self), blkno, offnum);
            tuple.t_data = (HeapTupleHeader) PageGetItem(page, itemid);
            tuple.t_len = ItemIdGetLength(itemid);
            
            /* [3] 检查元组状态 */
            switch (HeapTupleSatisfiesVacuum(&tuple, OldestXmin, buf))
            {
                case HEAPTUPLE_DEAD:
                    /* 死元组 - 记录TID */
                    lazy_record_dead_tuple(vacrel, &(tuple.t_self));
                    tups_vacuumed++;
                    break;
                    
                case HEAPTUPLE_LIVE:
                    /* 活元组 - 跳过 */
                    num_tuples++;
                    break;
                    
                case HEAPTUPLE_RECENTLY_DEAD:
                    /* 最近死亡 - 可能还有活动快照引用 */
                    num_tuples++;
                    break;
                    
                // ...
            }
        }
        
        /* [4] 如果有死元组，执行页面剪枝 */
        if (tups_vacuumed > 0)
        {
            lazy_vacuum_heap_page(vacrel, blkno, buf, 0, &vmbuffer);
        }
        
        /* [5] 更新FSM */
        RecordPageWithFreeSpace(vacrel->rel, blkno, freespace);
        
        UnlockReleaseBuffer(buf);
    }
    
    /* [6] 清理索引 */
    lazy_vacuum_all_indexes(vacrel);
    
    /* [7] 第二次扫描 */
    lazy_vacuum_heap_rel(vacrel);
}
```

---

## 3. 索引清理

### 3.1 索引VACUUM流程

```
lazy_vacuum_all_indexes(vacrel)
│
for (each index) {
  │
  ├─→ [1] 收集死元组TID列表
  │
  ├─→ [2] 调用索引的ambulkdelete()
  │    └─ B-Tree: btbulkdelete()
  │        │
  │        ├─ 扫描所有叶子页面
  │        ├─ 对每个索引项:
  │        │   └─ 检查TID是否在死元组列表
  │        │       ├─ 是 → 标记删除
  │        │       └─ 否 → 保留
  │        │
  │        └─ 页面清理和压缩
  │
  └─→ [3] 调用amvacuumcleanup()
       └─ B-Tree: btvacuumcleanup()
           ├─ 统计索引大小
           └─ 更新元数据
}
```

---

## 4. Autovacuum机制

### 4.1 Launcher主循环

```
AutoVacLauncherMain()
│
while (true) {
  │
  ├─→ [1] 睡眠 autovacuum_naptime (默认1分钟)
  │
  ├─→ [2] 连接到每个数据库
  │    for (each database) {
  │      │
  │      ├─ 查询pg_stat_user_tables
  │      │
  │      ├─ 计算是否需要VACUUM:
  │      │   threshold = autovacuum_vacuum_threshold + 
  │      │              (n_live_tup * autovacuum_vacuum_scale_factor)
  │      │   
  │      │   if (n_dead_tup > threshold)
  │      │     → 需要VACUUM
  │      │
  │      └─ 按优先级排序:
  │          1. 接近XID回卷的表 (最高优先级!)
  │          2. 死元组最多的表
  │          3. 统计信息过期的表
  │    }
  │
  ├─→ [3] 选择最需要VACUUM的表
  │
  ├─→ [4] 检查worker数量限制
  │    └─ 如果 active_workers < autovacuum_max_workers
  │        └─ fork() AutoVacWorker
  │
  └─→ [5] 回到 [1]
}
```

### 4.2 Worker执行

```
AutoVacWorkerMain()
│
├─→ [1] 接收要处理的表信息
│
├─→ [2] 连接到目标数据库
│
├─→ [3] 执行VACUUM
│    └─ vacuum()
│        └─ Lazy VACUUM流程 (同手动VACUUM)
│
├─→ [4] 如果需要，执行ANALYZE
│    └─ do_analyze_rel()
│        ├─ 采样表数据
│        ├─ 计算统计信息
│        └─ 更新pg_statistic
│
└─→ [5] 退出
```

---

## 5. 事务ID回卷预防

### 5.1 FREEZE机制

```
事务ID空间: 32位 = 2^32 = 4,294,967,296

PostgreSQL使用循环事务ID:
  0 → 1 → 2 → ... → 2^31-1 → 0 (回卷!)

如果表很久不VACUUM:
  旧事务ID突然变成"未来"事务
  → 旧数据对所有事务不可见
  → 数据"丢失"!

FREEZE解决方案:
  对足够老的元组执行FREEZE:
    t_xmin = FrozenTransactionId (2)
  
  FrozenTransactionId对所有事务可见
  → 永远不会被回卷影响
```

### 5.2 FREEZE触发

```c
// 参数
vacuum_freeze_min_age = 50,000,000           // 5000万事务后freeze
vacuum_freeze_table_age = 150,000,000        // 1.5亿事务后全表freeze
autovacuum_freeze_max_age = 200,000,000      // 2亿事务强制vacuum

// 触发逻辑
age = current_xid - table_frozenxid

if (age > autovacuum_freeze_max_age)
{
    // 强制VACUUM FREEZE (最高优先级!)
    // 即使autovacuum_enabled = false也会执行
}
else if (age > vacuum_freeze_table_age)
{
    // VACUUM并扫描整表 (不跳过all-visible页面)
}
else
{
    // 常规VACUUM (可以跳过all-visible页面)
}
```

### 5.3 监控XID年龄

```sql
-- 检查数据库XID年龄
SELECT 
    datname,
    age(datfrozenxid) AS xid_age,
    datfrozenxid
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- 警告阈值:
-- age > 200,000,000: 强制autovacuum
-- age > 1,000,000,000: 严重警告
-- age > 2,000,000,000: 危险! 接近回卷

-- 检查表XID年龄
SELECT 
    schemaname,
    relname,
    age(relfrozenxid) AS xid_age,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size
FROM pg_stat_user_tables
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

---

## 6. 性能优化

### 6.1 关键参数

```sql
-- 维护内存 (VACUUM使用)
maintenance_work_mem = '1GB'    -- 默认64MB，增大可提速

-- VACUUM成本限制 (防止影响业务)
vacuum_cost_delay = 0           -- 默认0 (无延迟)
vacuum_cost_limit = 200         -- 默认200

-- Autovacuum worker数量
autovacuum_max_workers = 3      -- 默认3

-- 触发阈值
autovacuum_vacuum_scale_factor = 0.2  -- 20%死元组
autovacuum_vacuum_threshold = 50      -- 基础阈值

-- 表级调优 (频繁更新的表)
ALTER TABLE hot_table SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- 5%触发
    autovacuum_vacuum_cost_delay = 0,       -- 不延迟
    autovacuum_vacuum_cost_limit = 2000     -- 更激进
);
```

### 6.2 VACUUM FULL替代方案

```
VACUUM FULL的问题:
  ✗ 需要AccessExclusiveLock (阻塞所有操作)
  ✗ 需要额外磁盘空间 (原表大小)
  ✗ 耗时长 (可能几小时)

推荐替代方案: pg_repack
  ✅ 在线重建表
  ✅ 不阻塞读写 (只在最后短暂阻塞)
  ✅ 自动重建索引

使用pg_repack:
  $ pg_repack -t table_name -d dbname
```

---

## 7. 监控和诊断

### 7.1 监控autovacuum

```sql
-- 查看autovacuum活动
SELECT 
    pid,
    datname,
    usename,
    state,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE backend_type = 'autovacuum worker';

-- 查看VACUUM历史
SELECT 
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    n_dead_tup,
    n_live_tup
FROM pg_stat_user_tables
ORDER BY last_autovacuum DESC NULLS LAST;
```

### 7.2 表膨胀检测

```sql
-- 使用pgstattuple
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- 精确检查 (慢)
SELECT * FROM pgstattuple('table_name');

-- 快速估算
SELECT * FROM pgstattuple_approx('table_name');

-- 批量检查
SELECT 
    schemaname || '.' || tablename AS table_name,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS size,
    round((pgstattuple_approx(schemaname||'.'||tablename)).dead_tuple_percent, 2) AS dead_pct
FROM pg_tables
WHERE schemaname = 'public'
  AND pg_relation_size(schemaname||'.'||tablename) > 1048576
ORDER BY pg_relation_size(schemaname||'.'||tablename) DESC;
```

---

## 总结

### VACUUM核心要点

1. **Lazy VACUUM**: 标记空间可重用，不归还OS
2. **索引清理**: 删除指向死元组的索引项
3. **Autovacuum**: 自动化VACUUM，防止膨胀
4. **FREEZE**: 防止事务ID回卷
5. **性能优化**: 调整参数，使用pg_repack

### 最佳实践

- ✅ 启用autovacuum (默认启用)
- ✅ 监控死元组比例 (目标 < 20%)
- ✅ 监控XID年龄 (< 2亿)
- ✅ 频繁更新的表降低触发阈值
- ✅ 使用pg_repack代替VACUUM FULL

---

**完成**: VACUUM模块核心文档已完成！

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

