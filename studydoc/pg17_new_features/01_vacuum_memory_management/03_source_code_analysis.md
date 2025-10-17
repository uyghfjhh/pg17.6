# VACUUM内存管理 - 源码深度分析

> TidStore核心实现源码详解

**文件位置**:
- `src/backend/access/common/tidstore.c` - TidStore实现
- `src/include/access/tidstore.h` - TidStore接口  
- `src/backend/access/heap/vacuumlazy.c` - VACUUM使用TidStore

---

## 📑 目录

1. [TidStore数据结构](#tidstore数据结构)
2. [创建TidStore](#创建tidstore)
3. [插入TID](#插入tid)
4. [查询TID](#查询tid)
5. [遍历TID](#遍历tid)
6. [内存管理](#内存管理)

---

## TidStore数据结构

### 核心结构定义

```c
/*
 * 文件: src/backend/access/common/tidstore.c:114
 * 
 * TidStore - 存储TID的核心数据结构
 */
struct TidStore
{
    /* TidStore所在的内存上下文 */
    MemoryContext context;

    /* Radix Tree使用的内存上下文 */
    MemoryContext rt_context;

    /* TID存储 - 使用本地或共享内存的Radix Tree */
    union
    {
        local_ts_radix_tree *local;     /* 本地进程的radix tree */
        shared_ts_radix_tree *shared;   /* 共享内存的radix tree */
    } tree;

    /* 如果使用共享内存，这是DSA area */
    dsa_area   *area;
};

/* 判断是否为共享TidStore */
#define TidStoreIsShared(ts) ((ts)->area != NULL)

/*
 * 设计要点:
 * 
 * 1. 内存上下文分离
 *    - context: TidStore对象本身
 *    - rt_context: Radix Tree的节点
 *    这样可以精确控制内存使用
 * 
 * 2. 本地/共享两种模式
 *    - local: 单进程VACUUM
 *    - shared: 并行VACUUM（多worker共享TidStore）
 * 
 * 3. DSA (Dynamic Shared Area)
 *    - 用于在多个进程间共享Radix Tree
 *    - 支持动态扩展
 */
```

### BlocktableEntry - 存储单个block的offsets

```c
/*
 * 文件: src/backend/access/common/tidstore.c:44
 * 
 * BlocktableEntry - Radix Tree的Value类型
 * Key是BlockNumber，Value是这个结构
 */
typedef struct BlocktableEntry
{
    struct
    {
#ifndef WORDS_BIGENDIAN
        /* 
         * flags字段
         * 需要在最低位，因为radix tree会用最低bit做标记
         */
        uint8 flags;

        /* 
         * nwords - bitmap的word数量
         * int8类型，最多127个words
         * 可以存储: 127 * 64 = 8128个offsets
         * (足够一个block的MaxOffsetNumber)
         */
        int8 nwords;
#endif

        /*
         * 小优化: 直接存储少量offsets
         * 
         * 当只有2-3个dead tuples时，
         * 直接存储在这里，避免bitmap开销
         */
        OffsetNumber full_offsets[NUM_FULL_OFFSETS];
        // NUM_FULL_OFFSETS通常是2-3个

#ifdef WORDS_BIGENDIAN
        int8 nwords;
        uint8 flags;
#endif
    } header;

    /*
     * Offset Bitmap
     * 
     * 每个bit代表一个offset是否是dead tuple
     * bit位置 = offset number
     * bit值 1 = dead tuple
     * bit值 0 = live tuple或不存在
     */
    bitmapword words[FLEXIBLE_ARRAY_MEMBER];
    // bitmapword通常是uint64 (64 bits)
} BlocktableEntry;

/*
 * 内存布局示例:
 * 
 * 场景1: 只有2个dead tuples (offset 1, 5)
 * ┌──────────────────────────────┐
 * │ flags: 0x01                  │ 1 byte
 * │ nwords: 0                    │ 1 byte (不使用bitmap)
 * │ full_offsets[0]: 1           │ 2 bytes
 * │ full_offsets[1]: 5           │ 2 bytes
 * │ (no bitmap words)            │
 * └──────────────────────────────┘
 * 总计: ~8 bytes (vs 12 bytes if using bitmap)
 * 
 * 场景2: 有10个dead tuples (offset 1,2,5,10,11,12,20,21,22,30)
 * ┌──────────────────────────────┐
 * │ flags: 0x00                  │ 1 byte
 * │ nwords: 1                    │ 1 byte
 * │ full_offsets: (unused)       │
 * │ words[0]:                    │ 8 bytes
 * │   0b00000000...00001110110111│
 * │       ↑30  ↑20-22 ↑10-12 ↑1-2,5
 * └──────────────────────────────┘
 * 总计: ~16 bytes
 */

/* 计算最大可存储的offset */
#define MAX_OFFSET_IN_BITMAP \
    Min(BITS_PER_BITMAPWORD * PG_INT8_MAX - 1, MaxOffsetNumber)
    // PG_INT8_MAX = 127
    // BITS_PER_BITMAPWORD = 64
    // 最多: 127 * 64 = 8128个offsets

/* 计算BlocktableEntry的最大大小 */
#define MaxBlocktableEntrySize \
    offsetof(BlocktableEntry, words) + \
        (sizeof(bitmapword) * WORDS_PER_PAGE(MAX_OFFSET_IN_BITMAP))
    // 约1KB
```

---

## 创建TidStore

```c
/*
 * 文件: src/backend/access/common/tidstore.c:164
 * 
 * TidStoreCreateLocal - 创建本地TidStore
 * 
 * 参数:
 *   max_bytes: 最大内存使用 (通常是maintenance_work_mem)
 *   insert_only: 是否只插入（不查询）
 * 
 * 返回: TidStore指针
 */
TidStore *
TidStoreCreateLocal(size_t max_bytes, bool insert_only)
{
    TidStore   *ts;
    size_t      initBlockSize = ALLOCSET_DEFAULT_INITSIZE;
    size_t      minContextSize = ALLOCSET_DEFAULT_MINSIZE;
    size_t      maxBlockSize = ALLOCSET_DEFAULT_MAXSIZE;

    /* 分配TidStore结构本身 */
    ts = palloc0(sizeof(TidStore));
    ts->context = CurrentMemoryContext;

    /*
     * 关键优化: 限制maxBlockSize
     * 
     * 为了避免内存浪费，将maxBlockSize设置为
     * max_bytes的1/16
     * 
     * 例如: maintenance_work_mem = 1GB
     *      maxBlockSize = 64MB
     * 
     * 这样分配的内存块不会太大，
     * 减少内部碎片
     */
    while (16 * maxBlockSize > max_bytes)
        maxBlockSize >>= 1;

    if (maxBlockSize < ALLOCSET_DEFAULT_INITSIZE)
        maxBlockSize = ALLOCSET_DEFAULT_INITSIZE;

    /*
     * 创建专用的内存上下文
     * 
     * 根据使用模式选择上下文类型:
     * - insert_only=true: 使用BumpContext
     *   只分配不释放，最快
     * - insert_only=false: 使用AllocSetContext
     *   支持分配和释放
     */
    if (insert_only)
    {
        /*
         * BumpContext - VACUUM的最佳选择
         * 
         * 特点:
         * - 只支持palloc，不支持pfree
         * - 分配极快: O(1)
         * - 内存利用率高
         * - 适合: VACUUM收集TID（只增不减）
         */
        ts->rt_context = BumpContextCreate(CurrentMemoryContext,
                                           "TID storage",
                                           minContextSize,
                                           initBlockSize,
                                           maxBlockSize);
    }
    else
    {
        /*
         * AllocSetContext - 标准内存上下文
         * 
         * 特点:
         * - 支持palloc和pfree
         * - 略慢，但更灵活
         * - 适合: 需要删除TID的场景
         */
        ts->rt_context = AllocSetContextCreate(CurrentMemoryContext,
                                               "TID storage",
                                               minContextSize,
                                               initBlockSize,
                                               maxBlockSize);
    }

    /*
     * 创建Radix Tree
     * 
     * local_ts_create宏展开为:
     * local_ts_radix_tree_create(ts->rt_context)
     * 
     * Radix Tree在rt_context中分配所有节点
     */
    ts->tree.local = local_ts_create(ts->rt_context);

    return ts;
}

/*
 * 使用示例 (vacuumlazy.c):
 */
void
lazy_vacuum_heap_rel(LVRelState *vacrel)
{
    /*
     * 计算可用内存
     * 预留一些空间给其他数据结构
     */
    size_t max_bytes = maintenance_work_mem * 1024L * 1024L;
    max_bytes -= vacrel->dead_items_info->max_bytes;

    /*
     * 创建TidStore
     * insert_only=true: VACUUM只添加TID，不删除
     */
    vacrel->dead_items = TidStoreCreateLocal(max_bytes, true);
    
    /* ... VACUUM逻辑 ... */
}
```

---

## 插入TID

```c
/*
 * 文件: src/backend/access/common/tidstore.c:368
 * 
 * TidStoreSetBlockOffsets - 为一个block设置dead tuple offsets
 * 
 * 参数:
 *   ts: TidStore
 *   blkno: Block Number
 *   offsets: Offset数组
 *   num_offsets: Offset数量
 */
void
TidStoreSetBlockOffsets(TidStore *ts, BlockNumber blkno,
                        OffsetNumber *offsets, int num_offsets)
{
    BlocktableEntry *page;
    bool        found;

    Assert(num_offsets > 0);

    /*
     * 第一步: 在Radix Tree中查找或创建BlocktableEntry
     * 
     * 如果是本地TidStore:
     *   page = local_ts_find(ts->tree.local, blkno, &found);
     * 如果是共享TidStore:
     *   page = shared_ts_find(ts->tree.shared, blkno, &found);
     */
    if (TidStoreIsShared(ts))
        page = shared_ts_find(ts->tree.shared, blkno, &found);
    else
        page = local_ts_find(ts->tree.local, blkno, &found);

    /*
     * 第二步: 根据offset数量决定存储方式
     */
    if (num_offsets <= NUM_FULL_OFFSETS)
    {
        /*
         * 小优化: 直接存储在header中
         * 
         * 当只有2-3个offsets时，
         * 直接存储更节省空间
         */
        page->header.nwords = 0;  // 标记不使用bitmap
        
        for (int i = 0; i < num_offsets; i++)
            page->header.full_offsets[i] = offsets[i];
    }
    else
    {
        /*
         * 使用Bitmap存储
         * 
         * 步骤:
         * 1. 计算需要多少个bitmapword
         * 2. 初始化bitmap为0
         * 3. 设置对应offset的bit为1
         */
        
        /* 找到最大offset */
        OffsetNumber max_offset = 0;
        for (int i = 0; i < num_offsets; i++)
        {
            if (offsets[i] > max_offset)
                max_offset = offsets[i];
        }

        /* 计算需要的words数量 */
        int nwords = WORDS_PER_PAGE(max_offset);
        
        /* 设置nwords */
        page->header.nwords = nwords;

        /* 初始化bitmap为0 */
        memset(page->words, 0, sizeof(bitmapword) * nwords);

        /* 设置每个offset的bit */
        for (int i = 0; i < num_offsets; i++)
        {
            OffsetNumber off = offsets[i];
            
            /*
             * 计算bit位置
             * 
             * wordnum = offset / 64 (哪个word)
             * bitnum = offset % 64 (word内的哪个bit)
             */
            int wordnum = WORDNUM(off);
            int bitnum = BITNUM(off);
            
            /* 设置bit */
            page->words[wordnum] |= ((bitmapword) 1 << bitnum);
        }
    }
}

/*
 * VACUUM使用示例:
 * 
 * 文件: src/backend/access/heap/vacuumlazy.c
 */
static void
lazy_scan_prune(LVRelState *vacrel, ...)
{
    OffsetNumber offsets[MaxHeapTuplesPerPage];
    int num_dead = 0;
    
    /* 扫描page，找到dead tuples */
    for (OffsetNumber offnum = FirstOffsetNumber;
         offnum <= maxoff;
         offnum = OffsetNumberNext(offnum))
    {
        ItemId itemid = PageGetItemId(page, offnum);
        
        if (ItemIdIsDead(itemid))
        {
            /* 收集dead tuple的offset */
            offsets[num_dead++] = offnum;
        }
    }
    
    /* 如果有dead tuples，插入到TidStore */
    if (num_dead > 0)
    {
        TidStoreSetBlockOffsets(vacrel->dead_items,
                                blkno,
                                offsets,
                                num_dead);
    }
}

/*
 * 性能分析:
 * 
 * 时间复杂度:
 * - Radix Tree查找: O(log n) n=block数量
 * - Bitmap设置: O(m) m=offset数量
 * 总计: O(log n + m)
 * 
 * 空间复杂度:
 * - 小优化(≤3个offsets): ~8 bytes
 * - Bitmap: header + (max_offset/64)*8 bytes
 * 平均: 4-6 bytes/offset (vs 6 bytes in PG16)
 */
```

---

## 查询TID

```c
/*
 * 文件: src/backend/access/common/tidstore.c:419
 * 
 * TidStoreIsMember - 检查TID是否在TidStore中
 * 
 * 参数:
 *   ts: TidStore
 *   tid: ItemPointer (BlockNumber + OffsetNumber)
 * 
 * 返回: true如果TID存在
 */
bool
TidStoreIsMember(TidStore *ts, ItemPointer tid)
{
    BlockNumber blkno = ItemPointerGetBlockNumber(tid);
    OffsetNumber offset = ItemPointerGetOffsetNumber(tid);
    BlocktableEntry *page;
    bool found;

    /*
     * 第一步: 在Radix Tree中查找block
     */
    if (TidStoreIsShared(ts))
        page = shared_ts_find(ts->tree.shared, blkno, &found);
    else
        page = local_ts_find(ts->tree.local, blkno, &found);

    /* Block不存在 → TID不存在 */
    if (!found)
        return false;

    /*
     * 第二步: 检查offset
     */
    if (page->header.nwords == 0)
    {
        /*
         * 使用full_offsets直接存储
         * 线性查找（因为只有2-3个）
         */
        for (int i = 0; i < NUM_FULL_OFFSETS; i++)
        {
            if (page->header.full_offsets[i] == offset)
                return true;
            if (page->header.full_offsets[i] == InvalidOffsetNumber)
                break;
        }
        return false;
    }
    else
    {
        /*
         * 使用Bitmap
         * 检查对应bit是否为1
         */
        int wordnum = WORDNUM(offset);
        int bitnum = BITNUM(offset);

        /* 检查wordnum是否越界 */
        if (wordnum >= page->header.nwords)
            return false;

        /* 检查bit */
        return (page->words[wordnum] & ((bitmapword) 1 << bitnum)) != 0;
    }
}

/*
 * 性能分析:
 * 
 * 时间复杂度:
 * - Radix Tree查找: O(log n)
 * - Bitmap查找: O(1)
 * 总计: O(log n)
 * 
 * vs PG16的二分查找: O(log m) m=总TID数量
 * PG17更快，因为n(block数) << m(TID数)
 */
```

---

## 遍历TID

```c
/*
 * 文件: src/backend/access/common/tidstore.c:441
 * 
 * TidStoreBeginIterate - 开始遍历TidStore
 * 
 * 返回: 迭代器
 */
TidStoreIter *
TidStoreBeginIterate(TidStore *ts)
{
    TidStoreIter *iter = palloc0(sizeof(TidStoreIter));
    
    iter->ts = ts;
    
    /* 创建radix tree迭代器 */
    if (TidStoreIsShared(ts))
        iter->tree_iter.shared = shared_ts_begin_iterate(ts->tree.shared);
    else
        iter->tree_iter.local = local_ts_begin_iterate(ts->tree.local);
    
    return iter;
}

/*
 * TidStoreIterateNext - 获取下一个block的TIDs
 * 
 * 返回: TidStoreIterResult结构
 *   - blkno: Block Number
 *   - offsets: Offset数组
 *   - num_offsets: Offset数量
 *   - NULL表示遍历结束
 */
TidStoreIterResult *
TidStoreIterateNext(TidStoreIter *iter)
{
    uint64 key;
    BlocktableEntry *page;
    
    /*
     * 从Radix Tree获取下一个entry
     */
    if (TidStoreIsShared(iter->ts))
        page = shared_ts_iterate_next(iter->tree_iter.shared, &key);
    else
        page = local_ts_iterate_next(iter->tree_iter.local, &key);
    
    /* 遍历结束 */
    if (page == NULL)
        return NULL;
    
    /* 提取TIDs到result */
    tidstore_iter_extract_tids(iter, (BlockNumber) key, page);
    
    return &iter->output;
}

/*
 * tidstore_iter_extract_tids - 从BlocktableEntry提取offsets
 */
static void
tidstore_iter_extract_tids(TidStoreIter *iter, BlockNumber blkno,
                           BlocktableEntry *page)
{
    iter->output.blkno = blkno;
    iter->output.num_offsets = 0;
    
    if (page->header.nwords == 0)
    {
        /*
         * 直接存储模式
         * 复制full_offsets数组
         */
        for (int i = 0; i < NUM_FULL_OFFSETS; i++)
        {
            if (page->header.full_offsets[i] == InvalidOffsetNumber)
                break;
            
            iter->output.offsets[iter->output.num_offsets++] = 
                page->header.full_offsets[i];
        }
    }
    else
    {
        /*
         * Bitmap模式
         * 遍历所有bit，找到设置的bit
         */
        for (int wordnum = 0; wordnum < page->header.nwords; wordnum++)
        {
            bitmapword w = page->words[wordnum];
            
            /* 跳过全0的word */
            if (w == 0)
                continue;
            
            /* 检查word中每个bit */
            int base_offset = wordnum * BITS_PER_BITMAPWORD;
            for (int bitnum = 0; bitnum < BITS_PER_BITMAPWORD; bitnum++)
            {
                if (w & ((bitmapword) 1 << bitnum))
                {
                    iter->output.offsets[iter->output.num_offsets++] = 
                        base_offset + bitnum;
                }
            }
        }
    }
    
    iter->output.max_offset = 
        iter->output.offsets[iter->output.num_offsets - 1];
}

/*
 * VACUUM使用迭代器的示例:
 */
static void
lazy_vacuum_all_indexes(LVRelState *vacrel)
{
    TidStoreIter *iter;
    TidStoreIterResult *result;
    
    /* 开始遍历dead tuples */
    iter = TidStoreBeginIterate(vacrel->dead_items);
    
    /*
     * 重要: 遍历按block number顺序！
     * 
     * 这对索引扫描很重要:
     * - 索引也是按block组织的
     * - 顺序访问减少随机I/O
     * - 提升缓存命中率
     */
    while ((result = TidStoreIterateNext(iter)) != NULL)
    {
        BlockNumber blkno = result->blkno;
        OffsetNumber *offsets = result->offsets;
        int num_offsets = result->num_offsets;
        
        /* 对每个索引，删除这些TIDs */
        for (int i = 0; i < vacrel->nindexes; i++)
        {
            index_vacuum_item(vacrel->indrels[i], 
                            blkno, offsets, num_offsets);
        }
    }
    
    TidStoreEndIterate(iter);
}
```

---

## 内存管理

```c
/*
 * 文件: src/backend/access/common/tidstore.c:476
 * 
 * TidStoreMemoryUsage - 查询TidStore的内存使用量
 * 
 * 返回: 字节数
 */
size_t
TidStoreMemoryUsage(TidStore *ts)
{
    /*
     * 返回rt_context的内存使用
     * 这包括:
     * - Radix Tree的所有节点
     * - 所有BlocktableEntry
     */
    return MemoryContextMemAllocated(ts->rt_context, true);
}

/*
 * VACUUM使用内存监控:
 */
static void
lazy_scan_heap(LVRelState *vacrel)
{
    BlockNumber blkno;
    
    for (blkno = 0; blkno < rel_pages; blkno++)
    {
        /* 扫描page，收集dead tuples */
        lazy_scan_prune(vacrel, blkno);
        
        /*
         * 检查内存使用
         * 
         * 如果超过限制，必须先vacuum索引
         * 然后清空TidStore，继续扫描
         */
        if (TidStoreMemoryUsage(vacrel->dead_items) > 
            vacrel->dead_items_info->max_bytes)
        {
            /*
             * 内存满了!
             * 1. Vacuum索引（删除dead tuple的索引项）
             * 2. Vacuum heap（标记page空间可重用）
             * 3. 清空TidStore
             * 4. 继续扫描
             */
            lazy_vacuum_all_indexes(vacrel);
            lazy_vacuum_heap_rel(vacrel);
            
            /* 重置TidStore */
            TidStoreDestroy(vacrel->dead_items);
            vacrel->dead_items = TidStoreCreateLocal(...);
        }
    }
}

/*
 * TidStoreDestroy - 销毁TidStore
 */
void
TidStoreDestroy(TidStore *ts)
{
    /*
     * 销毁radix tree
     * 这会释放所有BlocktableEntry
     */
    if (TidStoreIsShared(ts))
        shared_ts_free(ts->tree.shared);
    else
        local_ts_free(ts->tree.local);
    
    /*
     * 删除内存上下文
     * 这会释放所有radix tree节点
     */
    MemoryContextDelete(ts->rt_context);
    
    /* 释放TidStore结构本身 */
    pfree(ts);
}
```

---

## 总结

### 核心设计优势

1. **Radix Tree**: 
   - O(log n)查找
   - 按block顺序遍历
   - 只存储存在的block

2. **Bitmap压缩**:
   - Block Number只存一次
   - Offset用bit表示
   - 连续offset压缩效果好

3. **小优化**:
   - 少量offset直接存储
   - 避免bitmap开销
   - 提升常见场景性能

4. **内存上下文**:
   - 精确控制内存
   - BumpContext极快
   - 易于监控和限制

### 性能关键点

- **插入**: O(log n) Radix Tree
- **查询**: O(log n) 比PG16快
- **遍历**: 顺序访问，cache友好
- **内存**: 平均节省33-50%

---

**源码版本**: PostgreSQL 17.6  
**分析日期**: 2025-10-17  
**关键文件**: tidstore.c, vacuumlazy.c

