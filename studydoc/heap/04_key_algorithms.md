# HEAP 关键算法

本文档详细说明堆表的关键算法：HOT Update、页面剪枝、FSM/VM管理等。

---

## 1. HOT Update 算法

### 1.1 核心思想

**HOT (Heap-Only Tuple)**: 同页更新优化，避免索引更新

```
传统UPDATE代价 = 元组更新 + N个索引更新
HOT UPDATE代价 = 元组更新 (索引更新=0)

性能提升: 当N=5时，HOT可提升5-6倍性能
```

### 1.2 条件判断算法

```c
// src/backend/access/heap/heapam.c

bool
HeapSatisfiesHOTandKeyUpdate(Relation rel, Bitmapset *hot_attrs,
                              HeapTuple oldtup, HeapTuple newtup)
{
    // 条件1: 更新未改变任何索引列
    if (HeapDetermineModifiedColumns(rel, hot_attrs, oldtup, newtup))
        return false;  // 索引列改变 → 不能HOT
    
    // 条件2: 新元组在同一页有空间
    // (由调用者检查 PageGetFreeSpace())
    
    return true;  // 可以HOT Update
}
```

### 1.3 HOT链遍历算法

```c
// 索引扫描时跟随HOT链找到最新版本
ItemPointer
heap_get_latest_tid(Relation rel, Snapshot snapshot, ItemPointer tid)
{
    ItemPointer result = *tid;
    
    for (;;) {
        // 读取当前TID指向的元组
        HeapTuple tuple = heap_fetch(rel, snapshot, result);
        
        // 检查是否是HOT链的中间节点
        if (!(tuple->t_infomask2 & HEAP_HOT_UPDATED))
            break;  // 到达链尾
        
        // 跟随 t_ctid 到下一个版本
        result = &(tuple->t_data->t_ctid);
        
        // 防止无限循环
        if (ItemPointerEquals(result, &tuple->t_self))
            break;
    }
    
    return result;
}

// 时间复杂度: O(链长度) 通常 ≤ 5
```

---

## 2. 页面剪枝算法

### 2.1 剪枝触发判断

```c
// src/backend/access/heap/pruneheap.c

void
heap_page_prune_opt(Relation relation, Buffer buffer)
{
    Page page = BufferGetPage(buffer);
    TransactionId prune_xid = PageGetPruneXid(page);
    TransactionId OldestXmin = GetOldestXmin(relation);
    
    // 快速检查: 如果没有可剪枝的元组，跳过
    if (!TransactionIdIsValid(prune_xid) ||
        !TransactionIdPrecedes(prune_xid, OldestXmin))
        return;  // 无需剪枝
    
    // 执行剪枝
    heap_page_prune(relation, buffer, OldestXmin);
}
```

### 2.2 HOT链剪枝算法

```c
// 剪枝一条HOT链
static void
heap_prune_chain(Page page, OffsetNumber rootoffnum,
                 TransactionId OldestXmin)
{
    OffsetNumber offnum = rootoffnum;
    HeapTupleHeader htup;
    
    // 遍历HOT链
    while (OffsetNumberIsValid(offnum)) {
        ItemId itemid = PageGetItemId(page, offnum);
        
        if (ItemIdIsRedirected(itemid)) {
            // 已经是重定向，继续
            offnum = ItemIdGetRedirect(itemid);
            continue;
        }
        
        htup = (HeapTupleHeader) PageGetItem(page, itemid);
        
        // 检查此版本是否可剪枝
        if (HeapTupleHeaderIsHeapOnly(htup)) {
            // 这是HOT链的中间节点
            if (TransactionIdPrecedes(htup->t_xmax, OldestXmin)) {
                // 标记为DEAD
                ItemIdSetDead(itemid);
                // 记录为可回收空间
                prunable_tuples++;
            }
        }
        
        // 跟随链到下一个版本
        if (htup->t_infomask & HEAP_HOT_UPDATED)
            offnum = ItemPointerGetOffsetNumber(&htup->t_ctid);
        else
            break;  // 链尾
    }
    
    // 如果链头可剪枝，设置为REDIRECT
    if (prunable_tuples > 0) {
        ItemId rootitem = PageGetItemId(page, rootoffnum);
        ItemIdSetRedirect(rootitem, latest_offnum);
    }
}
```

### 2.3 空间压实算法

```c
// src/backend/storage/page/bufpage.c
void
PageRepairFragmentation(Page page)
{
    Offset pd_lower = ((PageHeader) page)->pd_lower;
    Offset pd_upper = ((PageHeader) page)->pd_upper;
    ItemId itemidbase = (ItemId) ((char *) page + sizeof(PageHeaderData));
    int nitems = (pd_lower - sizeof(PageHeaderData)) / sizeof(ItemIdData);
    
    // 计算元组的目标位置
    Offset new_upper = BLCKSZ - sizeof(PageHeaderData);
    
    // 从右向左压实元组
    for (i = 0; i < nitems; i++) {
        ItemId itemid = &itemidbase[i];
        
        if (ItemIdIsNormal(itemid)) {
            Size size = ItemIdGetLength(itemid);
            new_upper -= MAXALIGN(size);
            
            // 移动元组
            memmove((char *) page + new_upper,
                    (char *) page + ItemIdGetOffset(itemid),
                    size);
            
            // 更新ItemId
            ItemIdSetNormal(itemid, new_upper, size);
        }
    }
    
    // 更新页头
    ((PageHeader) page)->pd_upper = new_upper;
}
```

---

## 3. FSM 管理算法

### 3.1 FSM 查找算法

```c
// src/backend/storage/freespace/freespace.c

BlockNumber
GetPageWithFreeSpace(Relation rel, Size spaceNeeded)
{
    // 将所需空间转换为FSM的slot值 (0-255)
    uint8 min_cat = fsm_space_needed_to_cat(spaceNeeded);
    
    // 在FSM中搜索
    BlockNumber blkno = fsm_search(rel, min_cat);
    
    if (blkno == InvalidBlockNumber) {
        // FSM中没有足够空间
        // 尝试扩展表
        return RelationGetNumberOfBlocks(rel);
    }
    
    return blkno;
}

// FSM三层搜索
static BlockNumber
fsm_search(Relation rel, uint8 min_cat)
{
    // Level 2 (Root): 找到最大空间的分支
    // Level 1 (Branch): 找到最大空间的叶子
    // Level 0 (Leaf): 找到实际的页号
    
    // 使用贪心算法: 总是选择最大空间的分支
    for (level = FSM_ROOT_LEVEL; level >= 0; level--) {
        // 读取FSM页面
        Buffer buf = ReadBuffer(rel->rd_smgr, FSM_FORKNUM, page);
        FSMPage fsmpage = BufferGetPage(buf);
        
        // 查找满足条件的slot
        int slot = fsm_search_avail(fsmpage, min_cat);
        
        if (slot == -1)
            return InvalidBlockNumber;  // 无可用空间
        
        // 下降到下一层
        page = fsm_get_child(page, slot);
    }
    
    return page;  // Level 0的page就是实际的块号
}
```

### 3.2 FSM 更新算法

```c
void
RecordPageWithFreeSpace(Relation rel, BlockNumber blkno, Size freespace)
{
    uint8 cat = fsm_space_avail_to_cat(freespace);
    
    // 更新叶子节点
    fsm_set_avail(rel, blkno, cat);
    
    // 向上传播最大值
    while (blkno != FSM_ROOT_ADDRESS) {
        blkno = fsm_get_parent(blkno);
        uint8 max_child = fsm_get_max_child_avail(rel, blkno);
        fsm_set_avail(rel, blkno, max_child);
    }
}
```

---

## 4. Visibility Map 算法

### 4.1 VM 标记算法

```c
// src/backend/access/heap/visibilitymap.c

void
visibilitymap_set(Relation rel, BlockNumber heapBlk,
                  uint8 flags)  // VISIBILITYMAP_ALL_VISIBLE | VISIBILITYMAP_ALL_FROZEN
{
    // 计算VM页号和位偏移
    BlockNumber mapBlock = HEAPBLK_TO_MAPBLOCK(heapBlk);
    uint32 mapByte = HEAPBLK_TO_MAPBYTE(heapBlk);
    uint8 mapBit = HEAPBLK_TO_MAPBIT(heapBlk);
    
    // 读取VM页
    Buffer mapBuffer = vm_readbuf(rel, mapBlock, true);
    Page mapPage = BufferGetPage(mapBuffer);
    
    // 设置位
    uint8 *map = (uint8 *) PageGetContents(mapPage);
    map[mapByte] |= (flags << mapBit);
    
    // 标记脏页
    MarkBufferDirty(mapBuffer);
    
    // 记录WAL
    XLogRecPtr lsn = log_heap_visible(rel->rd_node, heapBlk, flags);
    PageSetLSN(mapPage, lsn);
}
```

### 4.2 VM 检查算法

```c
bool
visibilitymap_test(Relation rel, BlockNumber heapBlk, uint8 *flags)
{
    BlockNumber mapBlock = HEAPBLK_TO_MAPBLOCK(heapBlk);
    uint32 mapByte = HEAPBLK_TO_MAPBYTE(heapBlk);
    uint8 mapBit = HEAPBLK_TO_MAPBIT(heapBlk);
    
    Buffer mapBuffer = vm_readbuf(rel, mapBlock, false);
    if (!BufferIsValid(mapBuffer))
        return false;  // VM页不存在
    
    Page mapPage = BufferGetPage(mapBuffer);
    uint8 *map = (uint8 *) PageGetContents(mapPage);
    
    *flags = (map[mapByte] >> mapBit) & VISIBILITYMAP_VALID_BITS;
    
    ReleaseBuffer(mapBuffer);
    
    return (*flags != 0);
}
```

---

## 5. TOAST 压缩算法

### 5.1 LZ 压缩

```c
// src/common/pg_lzcompress.c

int32
pglz_compress(const char *source, int32 slen,
              char *dest, const PGLZ_Strategy *strategy)
{
    // 使用LZ77变种算法
    // 查找重复的3字节序列
    
    const char *dp = dest;
    const char *sp = source;
    int32 match_len, match_off;
    
    while (sp < source + slen) {
        // 在历史窗口中查找最长匹配
        match_len = pglz_find_match(sp, source, sp - source, &match_off);
        
        if (match_len >= PGLZ_MIN_MATCH) {
            // 输出 (offset, length) 对
            dp = pglz_out_ctrl(dp, match_off, match_len);
            sp += match_len;
        } else {
            // 输出字面字符
            *dp++ = *sp++;
        }
    }
    
    return dp - dest;  // 返回压缩后大小
}

// 典型压缩比: 2-4x (文本数据)
```

---

## 6. 表扩展策略

```c
// src/backend/access/heap/hio.c

#define HEAP_DEFAULT_FILLFACTOR 100
#define HEAP_MIN_FILLFACTOR     10

// 扩展大小计算
static BlockNumber
RelationGetTargetPageFreeSpace(Relation relation, int fillfactor)
{
    Size targetfree;
    
    // 计算目标空闲空间
    targetfree = BLCKSZ - BLCKSZ * fillfactor / 100;
    
    // 至少留1个ItemId + 1个最小元组的空间
    if (targetfree < sizeof(ItemIdData) + MAXALIGN(SizeofHeapTupleHeader))
        targetfree = sizeof(ItemIdData) + MAXALIGN(SizeofHeapTupleHeader);
    
    return targetfree;
}

// 批量扩展策略
static void
RelationAddExtraBlocks(Relation relation, Buffer buffer)
{
    BlockNumber nblocks = RelationGetNumberOfBlocks(relation);
    BlockNumber extend_by;
    
    // 动态扩展大小:
    // 小表: 逐页扩展
    // 大表: 批量扩展 (最多512页 = 4MB)
    if (nblocks < 8)
        extend_by = 1;
    else if (nblocks < 64)
        extend_by = nblocks / 8;
    else
        extend_by = Min(512, nblocks / 512);
    
    // 执行扩展
    for (i = 0; i < extend_by; i++)
        mdextend(relation->rd_smgr, MAIN_FORKNUM, nblocks + i, zeroblk);
}
```

---

## 7. 性能优化算法

### 7.1 Fillfactor 优化

```sql
-- 计算最优 fillfactor
-- 目标: 最大化 HOT Update 比例

-- 公式:
-- avg_tuple_size = total_tuple_size / n_live_tup
-- avg_updates_per_tuple = n_tup_upd / n_live_tup
-- target_hot_ratio = 0.8  (80%)

-- 最优 fillfactor ≈ 100 - (avg_tuple_size * avg_updates_per_tuple / BLCKSZ) * 100

示例:
  平均元组: 200B
  平均更新次数: 3次
  页大小: 8192B
  
  最优 fillfactor ≈ 100 - (200 * 3 / 8192) * 100
                  ≈ 100 - 7.3
                  ≈ 93 → 设置为 90
```

---

## 总结

### 核心算法复杂度

| 算法 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| HOT链遍历 | O(链长) ≈ O(1) | O(1) |
| 页面剪枝 | O(页内元组数) | O(1) |
| FSM查找 | O(log N) | O(1) |
| VM标记/查询 | O(1) | O(N/32KB) |
| LZ压缩 | O(N) | O(窗口大小) |
| 空间压实 | O(N) | O(1) |

### 性能影响

- ✅ **HOT Update**: 最大性能提升 (5-10x for UPDATE-heavy)
- ✅ **页面剪枝**: 减少VACUUM频率 (2-3x less I/O)
- ✅ **FSM**: 快速空间分配 (O(log N) vs O(N))
- ✅ **VM**: 跳过已知可见页 (VACUUM 50%+ faster)

---

**下一步**: 阅读 [05_performance.md](05_performance.md) 了解性能优化

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

