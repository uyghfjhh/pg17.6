# B-Tree 索引核心分析 (精简版)

> PostgreSQL B-Tree索引的核心原理和算法

**源码**: `src/backend/access/nbtree/`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. B-Tree概述

### 1.1 为什么是B+树？

```
B+树特点:
✅ 平衡树 - 所有叶子节点在同一层
✅ 高扇出 - 每页存储数百个键 (树高低)
✅ 有序存储 - 支持范围查询
✅ 右链 - 叶子节点链接，支持顺序扫描
✅ 写优化 - 页分裂和合并机制

PostgreSQL特色:
✅ 支持多列索引
✅ 支持唯一性约束
✅ 支持部分索引
✅ 并发安全 (latch保护)
```

### 1.2 树结构

```
                    Root Page (层级3)
                         │
        ┌────────────────┼────────────────┐
        │                │                │
    Internal Page    Internal Page    Internal Page (层级2)
        │                │                │
    ┌───┼───┐        ┌───┼───┐        ┌───┼───┐
    │   │   │        │   │   │        │   │   │
   Leaf Leaf Leaf   Leaf Leaf Leaf   Leaf Leaf Leaf (层级1)
    │   │   │        │   │   │        │   │   │
    ↓   ↓   ↓        ↓   ↓   ↓        ↓   ↓   ↓
   TIDs TIDs TIDs   TIDs TIDs TIDs   TIDs TIDs TIDs

右链 (Rightlink):
Leaf → Leaf → Leaf → Leaf → ... (顺序扫描)
```

---

## 2. 页面结构

```
B-Tree Page (8KB):
┌─────────────────────────────────────────────────┐
│ PageHeader (24B)                                │
├─────────────────────────────────────────────────┤
│ BTPageOpaque (16B) - B-Tree特有                │
│  • btpo_prev, btpo_next (左右兄弟页)           │
│  • btpo_level (层级: 0=叶子, 1+=内部)          │
│  • btpo_flags (是否root/leaf等)                │
├─────────────────────────────────────────────────┤
│ ItemId[] (行指针数组)                           │
├─────────────────────────────────────────────────┤
│ Free Space                                      │
├─────────────────────────────────────────────────┤
│ Index Tuples (从底部向上)                       │
│  每个元组: [key | TID(s)]                       │
└─────────────────────────────────────────────────┘
```

---

## 3. 核心操作

### 3.1 搜索算法

```c
// _bt_search() - 从根到叶子
BTScanPos
_bt_search(Relation rel, BTScanInsert key)
{
    Buffer buf = _bt_getroot(rel);  // 获取根页
    Page page;
    
    for (;;)
    {
        page = BufferGetPage(buf);
        opaque = (BTPageOpaque) PageGetSpecialPointer(page);
        
        if (P_ISLEAF(opaque))
            break;  // 到达叶子节点
        
        /* 在内部节点中二分查找 */
        offnum = _bt_binsrch(rel, key, buf);
        
        /* 获取子页面指针 */
        ItemId itemid = PageGetItemId(page, offnum);
        IndexTuple itup = (IndexTuple) PageGetItem(page, itemid);
        BlockNumber childblk = BTreeInnerTupleGetDownLink(itup);
        
        /* 下降到子节点 */
        buf = _bt_getbuf(rel, childblk, BT_READ);
    }
    
    /* 在叶子节点中定位 */
    offnum = _bt_binsrch(rel, key, buf);
    return offnum;
}

复杂度: O(log N)
```

### 3.2 插入算法

```
_bt_insert(Relation rel, IndexTuple itup)
│
├─→ [1] 搜索插入位置
│    └─ _bt_search() → 定位到叶子页
│
├─→ [2] 检查是否有空间
│    └─ PageGetFreeSpace(page) >= tupsize?
│        ├─ Yes → 直接插入 ✓
│        └─ No → 需要分裂
│
├─→ [3] 页面分裂
│    └─ _bt_split()
│        ├─ 分配新页面 (右兄弟)
│        ├─ 将一半元组移到新页
│        ├─ 更新右链: old → new → next
│        ├─ 在父节点插入分隔键
│        └─ 可能触发父节点分裂 (递归)
│
└─→ [4] 写WAL日志
```

### 3.3 删除算法

```
_bt_doinsert(Relation rel, IndexTuple itup, bool is_delete)
│
├─→ [1] 标记删除 (不立即回收)
│    └─ ItemIdMarkDead(itemid)
│
├─→ [2] VACUUM时真正删除
│    └─ btvacuumpage()
│        ├─ 回收死元组空间
│        ├─ 如果页面为空 → 标记可删除
│        └─ 下次VACUUM回收页面
│
└─→ [3] 页面合并 (可选)
     └─ 如果页面太空 → 与兄弟页合并
```

---

## 4. 页面分裂详解

### 4.1 分裂策略

```
原页面: [1, 3, 5, 7, 9, 11, 13, 15] (满)
插入: 10

分裂点选择:
  • 一般: 中间位置 (50/50)
  • 顺序插入: 右边界 (95/5) - 优化连续插入

结果:
  左页: [1, 3, 5, 7, 9]
  右页: [10, 11, 13, 15]
  
父节点更新:
  ... → [分隔键: 10] → 右页
```

### 4.2 并发控制

```
分裂过程使用Latch保护:
  1. 获取父节点写锁
  2. 获取当前页写锁
  3. 执行分裂
  4. 更新右链
  5. 释放所有锁

读操作:
  • 使用rightlink追赶移动的元组
  • 无需等待分裂完成
```

---

## 5. 唯一性检查

```c
// _bt_check_unique() - 检查唯一性约束
_bt_check_unique(Relation rel, IndexTuple itup, ...)
{
    /* 搜索相同键 */
    while ((tuple = _bt_next(scan)) != NULL)
    {
        if (_bt_compare(itup, tuple) == 0)
        {
            /* 找到重复键 */
            
            /* 检查元组是否可见 */
            if (HeapTupleSatisfiesVisibility(htup, snapshot))
            {
                /* 冲突! */
                ereport(ERROR,
                    (errcode(ERRCODE_UNIQUE_VIOLATION),
                     errmsg("duplicate key violates unique constraint")));
            }
            
            /* 死元组，可以重用 */
        }
    }
}
```

---

## 6. 性能优化

### 6.1 填充因子

```sql
-- 默认 fillfactor = 90 (预留10%空间给更新)
CREATE INDEX idx_name ON table(col) WITH (fillfactor = 90);

-- 只读表: fillfactor = 100 (节省空间)
CREATE INDEX idx_readonly ON readonly_table(col) WITH (fillfactor = 100);

-- 频繁更新: fillfactor = 70 (更多空间，减少分裂)
CREATE INDEX idx_hotupdate ON hot_table(col) WITH (fillfactor = 70);
```

### 6.2 索引膨胀

```sql
-- 检查索引膨胀
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- REINDEX重建索引
REINDEX INDEX CONCURRENTLY idx_name;  -- 不阻塞
```

---

## 7. 监控查询

```sql
-- 查看索引使用情况
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,  -- 使用次数
    idx_tup_read,  -- 读取行数
    idx_tup_fetch  -- 返回行数
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan;

-- 未使用的索引
SELECT 
    schemaname,
    tablename,
    indexname
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey';

-- 删除未使用的索引
DROP INDEX CONCURRENTLY idx_unused;
```

---

## 总结

### B-Tree核心要点

1. **搜索**: O(log N) - 二分查找
2. **插入**: 可能触发页面分裂
3. **删除**: 延迟回收，VACUUM真正删除
4. **唯一性**: 在插入时检查
5. **并发**: Latch + Rightlink机制

### 最佳实践

- ✅ 高选择性列创建索引
- ✅ 多列索引注意列顺序
- ✅ 使用部分索引减少大小
- ✅ 定期REINDEX防止膨胀
- ✅ 删除未使用的索引

---

**完成**: B-Tree索引核心文档完成！

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

