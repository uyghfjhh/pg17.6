# 扫描节点深度分析 - 所有扫描方式详解

> PostgreSQL所有表扫描方式的完整对比和实现分析

**重要程度**: ⭐⭐⭐⭐⭐  
**源码位置**: `src/backend/executor/node*.c`  
**核心概念**: SeqScan, IndexScan, IndexOnlyScan, BitmapScan, TID Scan

---

## 📋 扫描节点总览

### 所有扫描类型

```
【PostgreSQL支持的扫描方式】

1. SeqScan (顺序扫描)
   ├─ 全表扫描，逐块读取
   ├─ 最简单但可能最慢
   └─ 适合: 小表，或大部分行满足条件

2. IndexScan (索引扫描)
   ├─ 通过索引查找，然后访问堆表
   ├─ 随机I/O模式
   └─ 适合: 少量行，有合适索引

3. IndexOnlyScan (仅索引扫描)
   ├─ 只扫描索引，不访问堆表
   ├─ 要求: 查询列都在索引中 + VM all-visible
   └─ 适合: 覆盖索引 + 表已VACUUM

4. BitmapScan (位图扫描)
   ├─ 分两阶段: 索引→TID位图→顺序访问堆
   ├─ 可以合并多个索引 (AND/OR)
   └─ 适合: 中等数量行，多个索引条件

5. TidScan (TID扫描)
   ├─ 直接通过TID (ctid) 访问
   ├─ 最快，但用户很少直接使用
   └─ 适合: WHERE ctid = '(0,1)'

6. SubqueryScan (子查询扫描)
   └─ 扫描子查询结果

7. FunctionScan (函数扫描)
   └─ 扫描函数返回的行

8. ValuesScan (VALUES扫描)
   └─ 扫描VALUES子句
```

---

## 1. SeqScan - 顺序扫描

### 算法原理

```
【顺序扫描流程】
┌──────────────────────────────────────┐
│ Table: users (1000 blocks)           │
│                                      │
│ Block 0: [Row1][Row2][Row3]...       │
│ Block 1: [Row50][Row51]...           │
│ Block 2: [Row100][Row101]...         │
│ ...                                  │
│ Block 999: [Row10000]                │
└──────────────────────────────────────┘
         │
         │ 按块顺序扫描
         ↓
  For block 0 to 999:
    Read block into buffer
    For each tuple in block:
      Check visibility (MVCC)
      Evaluate WHERE clause
      If matches:
        Return tuple
```

### 核心实现

```c
/*
 * SeqScan执行 - 顺序扫描
 * 文件: src/backend/executor/nodeSeqscan.c
 */

/* SeqScan状态 */
typedef struct SeqScanState
{
    ScanState   ss;                   /* 基类 */
    Size        pscan_len;            /* 并行扫描共享状态大小 */
} SeqScanState;

/*
 * ExecSeqScan - 获取下一个元组
 */
static TupleTableSlot *
ExecSeqScan(PlanState *pstate)
{
    SeqScanState *node = castNode(SeqScanState, pstate);
    
    /*
     * 调用通用扫描函数
     * 核心工作在 SeqNext()
     */
    return ExecScan(&node->ss,
                   (ExecScanAccessMtd) SeqNext,
                   (ExecScanRecheckMtd) SeqRecheck);
}

/*
 * SeqNext - 获取下一个元组
 */
static TupleTableSlot *
SeqNext(SeqScanState *node)
{
    HeapScanDesc scandesc;
    EState     *estate;
    ScanDirection direction;
    TupleTableSlot *slot;
    
    /*
     * 获取扫描描述符
     */
    scandesc = node->ss.ss_currentScanDesc;
    estate = node->ss.ps.state;
    direction = estate->es_direction;
    slot = node->ss.ss_ScanTupleSlot;
    
    /*
     * 如果是并行扫描
     */
    if (scandesc->rs_base.rs_parallel != NULL)
    {
        /*
         * 并行SeqScan
         * 多个worker协同扫描同一个表
         */
        HeapTuple tuple = heap_parallelscan_getnext(scandesc, direction);
        
        if (tuple == NULL)
            return ExecClearTuple(slot);
        
        ExecStoreBufferHeapTuple(tuple, slot, scandesc->rs_cbuf);
        return slot;
    }
    
    /*
     * 串行扫描: 获取下一个元组
     */
    HeapTuple tuple = heap_getnext(scandesc, direction);
    
    if (tuple == NULL)
    {
        /*
         * 扫描完成，返回空slot
         */
        return ExecClearTuple(slot);
    }
    
    /*
     * 将tuple存储到slot中
     * rs_cbuf: 当前buffer
     */
    ExecStoreBufferHeapTuple(tuple, slot, scandesc->rs_cbuf);
    
    return slot;
}

/*
 * heap_getnext - 从堆表获取下一个可见元组
 * 文件: src/backend/access/heap/heapam.c
 */
HeapTuple
heap_getnext(HeapScanDesc scan, ScanDirection direction)
{
    HeapTuple tuple = &(scan->rs_ctup);
    Snapshot snapshot = scan->rs_base.rs_snapshot;
    BlockNumber page;
    
    /*
     * 如果是前向扫描
     */
    if (ScanDirectionIsForward(direction))
    {
        /*
         * 主扫描循环
         */
        for (;;)
        {
            /*
             * 如果当前block还有tuple
             */
            if (scan->rs_cindex < scan->rs_ntuples)
            {
                /*
                 * 获取下一个tuple的offset
                 */
                OffsetNumber offnum = scan->rs_vistuples[scan->rs_cindex];
                
                scan->rs_cindex++;
                
                /*
                 * 从page中获取tuple
                 */
                tuple->t_data = (HeapTupleHeader)
                    PageGetItem((Page) BufferGetPage(scan->rs_cbuf),
                               PageGetItemId((Page) BufferGetPage(scan->rs_cbuf),
                                           offnum));
                tuple->t_len = ItemIdGetLength(
                    PageGetItemId((Page) BufferGetPage(scan->rs_cbuf),
                                 offnum));
                tuple->t_tableOid = RelationGetRelid(scan->rs_base.rs_rd);
                tuple->t_self.ip_blkid.bi_hi = page >> 16;
                tuple->t_self.ip_blkid.bi_lo = page & 0xFFFF;
                tuple->t_self.ip_posid = offnum;
                
                /*
                 * 检查可见性
                 */
                if (HeapTupleSatisfiesVisibility(tuple, snapshot,
                                                scan->rs_cbuf))
                {
                    /*
                     * 这个tuple可见，返回
                     */
                    return tuple;
                }
                
                /*
                 * 不可见，继续下一个
                 */
                continue;
            }
            
            /*
             * 当前block扫描完成，移动到下一个block
             */
            
            /*
             * 释放当前buffer pin
             */
            if (BufferIsValid(scan->rs_cbuf))
                ReleaseBuffer(scan->rs_cbuf);
            
            /*
             * 获取下一个block号
             */
            page = heapgettup_advance_block(scan, direction);
            
            if (page == InvalidBlockNumber)
            {
                /*
                 * 表扫描完成
                 */
                return NULL;
            }
            
            /*
             * 读取新block
             * 这是I/O发生的地方!
             */
            scan->rs_cbuf = ReadBufferExtended(scan->rs_base.rs_rd,
                                              MAIN_FORKNUM,
                                              page,
                                              RBM_NORMAL,
                                              scan->rs_strategy);
            
            /*
             * 扫描block中的所有tuples
             */
            heapgettup_pagemode(scan, direction, page);
        }
    }
    
    return NULL;
}
```

### 性能特性

```
【SeqScan性能分析】

I/O模式:
  ┌───────────────────────────────┐
  │ 顺序读取                       │
  │ Block 0 → Block 1 → Block 2... │
  │                               │
  │ 优点:                         │
  │ ✅ 预读有效 (OS read-ahead)  │
  │ ✅ 顺序I/O快                 │
  │ ✅ 批量处理                  │
  └───────────────────────────────┘

Buffer管理:
  使用Ring Buffer (环形缓冲区)
  ┌─────────────────────────────┐
  │ [Buf0][Buf1][Buf2][Buf3]    │
  │   ↑                    ↓    │
  │   └────────────────────┘    │
  │   循环使用，不污染shared_buffers │
  └─────────────────────────────┘
  
  代码: src/backend/storage/buffer/freelist.c
  GetAccessStrategy(BAS_BULKREAD)

时间复杂度:
  O(N) - N是表的总行数
  
  即使只需要1行，最坏情况要扫描整个表

成本估算:
  cost = seq_page_cost * pages + cpu_tuple_cost * tuples
  
  默认:
  seq_page_cost = 1.0
  cpu_tuple_cost = 0.01

适用场景:
  ✅ 小表 (<1000 blocks)
  ✅ 大部分行满足条件 (>5%)
  ✅ 没有可用索引
  ✅ 并行扫描 (大表)
```

---

## 2. IndexScan - 索引扫描

### 算法原理

```
【IndexScan两步骤】

Step 1: 索引查找
  ┌──────────────────┐
  │ B-tree Index     │
  │                  │
  │ Root: [10][50]   │
  │   ├─ Leaf: [1][2][3]...    │
  │   └─ Leaf: [51][52]...     │
  └──────────────────┘
         │
         │ WHERE id = 10
         ↓
  找到TID: (0, 5)  ← Block 0, Offset 5

Step 2: 堆表回表
  ┌──────────────────┐
  │ Heap Table       │
  │                  │
  │ Block 0:         │
  │   Offset 5: → [Row: id=10, name=Alice] │
  └──────────────────┘
         ↑
         │ 根据TID访问
         
【关键点】
• 索引访问: O(log N)
• 堆表访问: 随机I/O
• 每个匹配行都要回表
```

### 核心实现

```c
/*
 * IndexScan执行
 * 文件: src/backend/executor/nodeIndexscan.c
 */

typedef struct IndexScanState
{
    ScanState   ss;                /* 基类 */
    ExprState  *indexqualorig;     /* 原始索引条件 */
    List       *indexorderbyorig;  /* ORDER BY表达式 */
    ScanKey     iss_ScanKeys;      /* 扫描键数组 */
    int         iss_NumScanKeys;   /* 扫描键数量 */
    IndexScanDesc iss_ScanDesc;    /* 索引扫描描述符 */
    bool        iss_ReachedEnd;    /* 扫描结束标志 */
} IndexScanState;

/*
 * ExecIndexScan - 获取下一个元组
 */
static TupleTableSlot *
ExecIndexScan(PlanState *pstate)
{
    IndexScanState *node = castNode(IndexScanState, pstate);
    
    /*
     * 使用通用扫描框架
     */
    return ExecScan(&node->ss,
                   (ExecScanAccessMtd) IndexNext,
                   (ExecScanRecheckMtd) IndexRecheck);
}

/*
 * IndexNext - 通过索引获取下一个元组
 */
static TupleTableSlot *
IndexNext(IndexScanState *node)
{
    EState     *estate;
    ExprContext *econtext;
    ScanDirection direction;
    IndexScanDesc scandesc;
    TupleTableSlot *slot;
    
    estate = node->ss.ps.state;
    econtext = node->ss.ps.ps_ExprContext;
    direction = estate->es_direction;
    scandesc = node->iss_ScanDesc;
    slot = node->ss.ss_ScanTupleSlot;
    
    /*
     * 扫描循环
     */
    for (;;)
    {
        /*
         * ═══════════════════════════════════════
         *  Step 1: 从索引获取下一个TID
         * ═══════════════════════════════════════
         */
        bool found = index_getnext_tid(scandesc, direction);
        
        if (!found)
        {
            /*
             * 索引扫描完成，没有更多匹配的TID
             */
            return ExecClearTuple(slot);
        }
        
        /*
         * ═══════════════════════════════════════
         *  Step 2: 根据TID从堆表获取tuple
         * ═══════════════════════════════════════
         */
        if (!index_fetch_heap(scandesc, slot))
        {
            /*
             * tuple不可见或已被删除
             * 继续下一个
             */
            continue;
        }
        
        /*
         * ═══════════════════════════════════════
         *  Step 3: 重新检查条件 (Recheck)
         * ═══════════════════════════════════════
         * 
         * 为什么需要Recheck?
         * 1. 索引可能是有损的 (lossy)
         * 2. 某些条件无法在索引中完全检查
         */
        if (node->ss.ps.qual == NULL ||
            ExecQual(node->ss.ps.qual, econtext))
        {
            /*
             * 找到匹配的tuple!
             */
            return slot;
        }
        
        /*
         * Recheck失败，继续下一个
         */
        InstrCountFiltered2(node, 1);
    }
}

/*
 * index_getnext_tid - 从索引获取下一个TID
 * 文件: src/backend/access/index/indexam.c
 */
bool
index_getnext_tid(IndexScanDesc scan, ScanDirection direction)
{
    bool found;
    
    /*
     * 调用索引AM的gettuple方法
     * 对于B-tree: btgettuple()
     * 对于Hash: hashgettuple()
     * 对于GiST: gistgettuple()
     * ...
     */
    found = scan->indexRelation->rd_indam->amgettuple(scan, direction);
    
    if (found)
    {
        /*
         * 成功获取到一个TID
         * scan->xs_heaptid 包含TID
         */
        pgstat_count_index_tuples(scan->indexRelation, 1);
    }
    
    return found;
}

/*
 * index_fetch_heap - 根据TID从堆表获取tuple
 * 文件: src/backend/access/index/indexam.c
 */
bool
index_fetch_heap(IndexScanDesc scan, TupleTableSlot *slot)
{
    ItemPointer tid = &scan->xs_heaptid;
    Relation heapRelation = scan->heapRelation;
    Buffer buffer;
    
    /*
     * 根据TID从堆表读取tuple
     * 这通常是随机I/O!
     */
    if (!heap_fetch(heapRelation,
                   scan->xs_snapshot,
                   &tuple,
                   &buffer,
                   scan->xs_heap_continue))
    {
        /*
         * tuple不可见或不存在
         * 原因:
         * 1. MVCC: tuple对当前事务不可见
         * 2. VACUUM: tuple已被删除
         * 3. HOT更新: tuple已移动
         */
        return false;
    }
    
    /*
     * 将tuple存储到slot
     */
    ExecStoreBufferHeapTuple(&tuple, slot, buffer);
    
    return true;
}
```

### B-tree索引扫描详解

```c
/*
 * B-tree特定实现
 * 文件: src/backend/access/nbtree/nbtsearch.c
 */

/*
 * btgettuple - B-tree获取下一个TID
 */
bool
btgettuple(IndexScanDesc scan, ScanDirection dir)
{
    BTScanOpaque so = (BTScanOpaque) scan->opaque;
    
    /*
     * 如果是首次调用，需要定位到起始位置
     */
    if (!BTScanPosIsValid(so->currPos))
    {
        /*
         * ═══════════════════════════════════════
         *  从Root向下查找，找到起始Leaf page
         * ═══════════════════════════════════════
         */
        
        /* Step 1: 从Root开始 */
        Buffer buf = _bt_getroot(scan->indexRelation, BT_READ);
        Page page = BufferGetPage(buf);
        BTPageOpaque opaque = (BTPageOpaque) PageGetSpecialPointer(page);
        
        /* Step 2: 向下查找直到Leaf */
        while (!P_ISLEAF(opaque))
        {
            /*
             * Internal页面
             * 根据scan key找到子节点
             */
            OffsetNumber offnum = _bt_binsrch(scan->indexRelation,
                                             buf,
                                             scan->numberOfKeys,
                                             scan->keyData,
                                             false);
            
            ItemId itemid = PageGetItemId(page, offnum);
            IndexTuple itup = (IndexTuple) PageGetItem(page, itemid);
            BlockNumber child_blkno = BTreeTupleGetDownLink(itup);
            
            /* 释放当前页 */
            _bt_relbuf(scan->indexRelation, buf);
            
            /* 读取子页 */
            buf = _bt_getbuf(scan->indexRelation, child_blkno, BT_READ);
            page = BufferGetPage(buf);
            opaque = (BTPageOpaque) PageGetSpecialPointer(page);
        }
        
        /* Step 3: 在Leaf page中二分查找 */
        OffsetNumber offnum = _bt_binsrch(scan->indexRelation,
                                         buf,
                                         scan->numberOfKeys,
                                         scan->keyData,
                                         true);
        
        /* 保存位置 */
        _bt_saveitem(so, offnum, buf, page);
    }
    
    /*
     * ═══════════════════════════════════════
     *  从当前位置获取下一个TID
     * ═══════════════════════════════════════
     */
    for (;;)
    {
        /* 获取当前tuple */
        OffsetNumber offnum = so->currPos.itemIndex;
        Page page = BufferGetPage(so->currPos.buf);
        
        if (offnum <= so->currPos.lastItem)
        {
            /*
             * 当前page还有tuple
             */
            ItemId itemid = PageGetItemId(page, offnum);
            IndexTuple itup = (IndexTuple) PageGetItem(page, itemid);
            
            /* 提取heap TID */
            scan->xs_heaptid = itup->t_tid;
            
            /* 移动到下一个 */
            so->currPos.itemIndex++;
            
            return true;
        }
        
        /*
         * 当前page扫描完成
         * 移动到下一个page
         */
        if (!_bt_next(scan, dir))
        {
            /* 没有更多page了 */
            return false;
        }
        
        /* 继续新page的扫描 */
    }
}
```

### 性能特性

```
【IndexScan性能分析】

时间复杂度:
  索引查找: O(log N)
  堆表访问: O(M) - M是匹配行数
  总计: O(log N + M)

I/O模式:
  ┌──────────────────────────────┐
  │ 随机I/O                       │
  │ Block 5 → Block 123 → Block 8 │
  │                              │
  │ 问题:                        │
  │ ❌ 无法利用OS预读            │
  │ ❌ 磁盘寻道时间长            │
  │ ❌ SSD也有随机读惩罚         │
  └──────────────────────────────┘

成本估算:
  cost = (index_pages * random_page_cost) +
         (heap_pages * random_page_cost) +
         (tuples * cpu_tuple_cost)
  
  关键参数:
  random_page_cost = 4.0  # HDD
  random_page_cost = 1.1  # SSD (应该调低!)

适用场景:
  ✅ 少量行 (<5% of table)
  ✅ 有合适的索引
  ✅ WHERE条件选择性高
  
  ❌ 大量行 (SeqScan更快!)
  ❌ 索引列选择性差

示例:
  SELECT * FROM users WHERE id = 1;
  → IndexScan! (只需1行)
  
  SELECT * FROM users WHERE age > 10;
  → SeqScan! (假设90%满足，IndexScan随机I/O太多)
```

---

## 3. IndexOnlyScan - 仅索引扫描

### 算法原理

```
【Index-Only Scan魔法】

前提条件:
  1. 查询的所有列都在索引中
  2. Page标记为all-visible (Visibility Map)

执行流程:
  ┌──────────────────────────────┐
  │ Step 1: 扫描索引             │
  │ ┌────────────────────┐       │
  │ │ Index: (id, name)  │       │
  │ │ [1, Alice]         │ ✓    │
  │ │ [2, Bob]           │ ✓    │
  │ │ [3, Carol]         │ ✓    │
  │ └────────────────────┘       │
  │         │                    │
  │         │ 直接返回!          │
  │         ↓                    │
  │   Result: (1, Alice)         │
  │                              │
  │ Step 2: 访问堆表?            │
  │   ❌ 不需要!                │
  │                              │
  │   但是...                    │
  │   检查VM (Visibility Map):   │
  │   ┌──────────────┐           │
  │   │ Block 0: ✓   │ all-visible │
  │   │ Block 1: ✓   │           │
  │   │ Block 2: ✗   │ 需要检查  │
  │   └──────────────┘           │
  └──────────────────────────────┘

【性能优势】
• 减少I/O: 不读堆表
• 更快: 只读索引(通常更小)
• Cache友好: 索引常驻内存
```

### 核心实现

```c
/*
 * Index-Only Scan执行
 * 文件: src/backend/executor/nodeIndexonlyscan.c
 */

typedef struct IndexOnlyScanState
{
    ScanState   ss;
    IndexScanDesc ioss_ScanDesc;
    
    /* Visibility Map相关 */
    Buffer      ioss_VMBuffer;        /* VM buffer */
    
    /* 统计 */
    long        ioss_HeapFetches;     /* 实际堆访问次数 */
} IndexOnlyScanState;

/*
 * ExecIndexOnlyScan - 获取下一个元组
 */
static TupleTableSlot *
ExecIndexOnlyScan(PlanState *pstate)
{
    IndexOnlyScanState *node = castNode(IndexOnlyScanState, pstate);
    
    return ExecScan(&node->ss,
                   (ExecScanAccessMtd) IndexOnlyNext,
                   (ExecScanRecheckMtd) IndexOnlyRecheck);
}

/*
 * IndexOnlyNext - 仅索引扫描
 */
static TupleTableSlot *
IndexOnlyNext(IndexOnlyScanState *node)
{
    IndexScanDesc scandesc;
    TupleTableSlot *slot;
    EState     *estate;
    
    scandesc = node->ioss_ScanDesc;
    estate = node->ss.ps.state;
    slot = node->ss.ss_ScanTupleSlot;
    
    for (;;)
    {
        /*
         * ═══════════════════════════════════════
         *  Step 1: 从索引获取下一个index tuple
         * ═══════════════════════════════════════
         */
        bool found = index_getnext_slot(scandesc,
                                       ForwardScanDirection,
                                       slot);
        
        if (!found)
            return ExecClearTuple(slot);
        
        /*
         * ═══════════════════════════════════════
         *  Step 2: 检查Visibility Map
         * ═══════════════════════════════════════
         *  
         *  如果page标记为all-visible:
         *    ✓ 可以信任索引数据
         *    ✓ 不需要访问堆表
         *  
         *  否则:
         *    ✗ 必须检查堆表确认可见性
         */
        
        /* 提取heap TID */
        ItemPointer tid = &scandesc->xs_heaptid;
        BlockNumber block = ItemPointerGetBlockNumber(tid);
        
        /*
         * 检查VM: 这个block是all-visible吗?
         */
        if (!VM_ALL_VISIBLE(scandesc->heapRelation,
                          block,
                          &node->ioss_VMBuffer))
        {
            /*
             * ❌ Block不是all-visible
             * 必须访问堆表检查tuple可见性
             */
            if (!index_fetch_heap(scandesc, slot))
            {
                /* tuple不可见，继续下一个 */
                continue;
            }
            
            /*
             * 统计: 堆表访问次数
             * 这个数字越少越好!
             */
            node->ioss_HeapFetches++;
        }
        
        /*
         * ✓ 找到可见的tuple!
         * 
         * 注意: slot包含的是索引数据
         * 对于Index-Only Scan，这就够了
         */
        return slot;
    }
}

/*
 * 检查EXPLAIN ANALYZE输出
 */
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name FROM users WHERE id > 100;

/*
Index Only Scan using idx_users_id_name on users
  (cost=0.29..45.30 rows=1000 width=12)
  (actual time=0.025..1.234 rows=1000 loops=1)
  Index Cond: (id > 100)
  Heap Fetches: 15      ← 只有15次堆访问! (vs 1000行)
  Buffers: shared hit=25
Planning Time: 0.123 ms
Execution Time: 1.456 ms
*/
```

### Visibility Map详解

```c
/*
 * Visibility Map (VM) - 可见性映射
 * 文件: src/backend/access/heap/visibilitymap.c
 */

/*
 * VM结构:
 *   每个heap page对应2个bit
 *   bit 0: all-visible
 *   bit 1: all-frozen
 * 
 * 示例:
 *   Block 0: 11 (all-visible + all-frozen)
 *   Block 1: 10 (all-visible)
 *   Block 2: 00 (neither)
 *   Block 3: 11 (all-visible + all-frozen)
 */

/*
 * visibilitymap_test - 测试block的VM标志
 */
uint8
visibilitymap_test(Relation rel, BlockNumber heapBlk, Buffer *buf)
{
    BlockNumber mapBlock;
    uint32 mapByte;
    uint8 mapOffset;
    uint8 *map;
    uint8 result;
    
    /*
     * 计算VM中的位置
     * 1个VM page可以覆盖很多heap pages
     */
    mapBlock = HEAPBLK_TO_MAPBLOCK(heapBlk);
    mapByte = HEAPBLK_TO_MAPBYTE(heapBlk);
    mapOffset = HEAPBLK_TO_OFFSET(heapBlk);
    
    /*
     * 读取VM page
     */
    *buf = ReadBuffer(rel, mapBlock + 1);  // +1因为VM在单独fork
    LockBuffer(*buf, BUFFER_LOCK_SHARE);
    
    map = (uint8 *) PageGetContents(BufferGetPage(*buf));
    
    /*
     * 提取2个bit
     */
    result = ((map[mapByte] >> mapOffset) & 0x03);
    
    LockBuffer(*buf, BUFFER_LOCK_UNLOCK);
    
    return result;
}

/*
 * visibilitymap_set - 设置block为all-visible
 * 由VACUUM调用
 */
void
visibilitymap_set(Relation rel, BlockNumber heapBlk,
                 Buffer heapBuf, XLogRecPtr recptr,
                 Buffer vmBuf, uint8 flags)
{
    /* 标记block为all-visible */
    map[mapByte] |= (flags << mapOffset);
    
    /* WAL日志 */
    XLogBeginInsert();
    XLogRegisterData((char *) &xlrec, sizeof(xlrec));
    XLogInsert(RM_HEAP2_ID, XLOG_HEAP2_VISIBLE);
}
```

### 性能特性

```
【Index-Only Scan性能分析】

最佳情况: (Heap Fetches = 0)
  ┌───────────────────────────┐
  │ 100%命中Visibility Map    │
  │ 只读索引，不读堆表        │
  │ I/O减少50-90%             │
  └───────────────────────────┘
  
  时间: IndexScan的20-50%

一般情况: (Heap Fetches = 10-30%)
  ┌───────────────────────────┐
  │ 部分block需要检查堆表     │
  │ 还是比IndexScan快         │
  └───────────────────────────┘
  
  时间: IndexScan的60-80%

最坏情况: (Heap Fetches = 100%)
  ┌───────────────────────────┐
  │ 表刚INSERT/UPDATE过       │
  │ VM都是0                   │
  │ 每行都要检查堆表          │
  └───────────────────────────┘
  
  时间: 比IndexScan还慢! (多了VM检查)

【如何提升Index-Only Scan效率】

1. 运行VACUUM
   VACUUM users;
   → 设置VM bit

2. 创建覆盖索引
   CREATE INDEX ON users(id, name, age);
   → 包含查询所需的所有列

3. 定期ANALYZE
   ANALYZE users;
   → 更新统计信息，帮助优化器选择

4. 监控Heap Fetches
   EXPLAIN (ANALYZE) ...
   查看"Heap Fetches"数字
   理想情况 < 5%
```

---

## 4. BitmapScan - 位图扫描

### 算法原理

```
【Bitmap Scan两阶段】

Phase 1: BitmapIndexScan
  ┌──────────────────────────────┐
  │ 扫描索引，收集所有TID到位图   │
  │                              │
  │ Index: age > 18              │
  │   ├─ TID: (0, 1)             │
  │   ├─ TID: (0, 5)             │
  │   ├─ TID: (2, 3)             │
  │   └─ TID: (5, 8)             │
  │         ↓                    │
  │   转换为位图:                │
  │   Block 0: [1][  ][  ][  ][1] │
  │   Block 1: [][][]...         │
  │   Block 2: [][][1]...        │
  │   ...                        │
  │   Block 5: [...][1]          │
  └──────────────────────────────┘

Phase 2: BitmapHeapScan
  ┌──────────────────────────────┐
  │ 按block顺序访问堆表          │
  │                              │
  │ Block 0 → 读取 → 返回(1),(5) │
  │ Block 2 → 读取 → 返回(3)     │
  │ Block 5 → 读取 → 返回(8)     │
  │                              │
  │ 优势:                        │
  │ ✅ 顺序访问heap blocks      │
  │ ✅ 每个block最多读一次      │
  │ ✅ 可以合并多个索引         │
  └──────────────────────────────┘

【合并多个索引】
  WHERE age > 18 AND city = 'NYC'
  
  BitmapAnd
    ├─ BitmapIndexScan(idx_age)
    │    Bitmap1: [1][0][1][1][0]
    │
    └─ BitmapIndexScan(idx_city)
         Bitmap2: [1][1][0][1][0]
              ↓
       Result: [1][0][0][1][0]  ← AND操作
```

### 核心实现

```c
/*
 * BitmapHeapScan执行
 * 文件: src/backend/executor/nodeBitmapHeapscan.c
 */

typedef struct BitmapHeapScanState
{
    ScanState   ss;
    
    /* 位图迭代器 */
    TBMIterator *tbmiterator;      /* 精确TID */
    TBMSharedIterator *shared_tbmiterator;  /* 并行扫描 */
    
    /* 当前正在处理的block */
    BlockNumber tbmres_blockno;
    OffsetNumber *tbmres_offsets;
    int         tbmres_ntuples;
    int         tbmres_index;
    
    /* 预取 */
    bool        prefetch_pages;
    
    /* 统计 */
    long        exact_pages;       /* 精确页数 */
    long        lossy_pages;       /* 有损页数 */
} BitmapHeapScanState;

/*
 * ExecBitmapHeapScan - 位图堆扫描
 */
static TupleTableSlot *
ExecBitmapHeapScan(PlanState *pstate)
{
    BitmapHeapScanState *node = castNode(BitmapHeapScanState, pstate);
    
    return ExecScan(&node->ss,
                   (ExecScanAccessMtd) BitmapHeapNext,
                   (ExecScanRecheckMtd) BitmapHeapRecheck);
}

/*
 * BitmapHeapNext - 获取下一个元组
 */
static TupleTableSlot *
BitmapHeapNext(BitmapHeapScanState *node)
{
    TBMIterateResult *tbmres;
    OffsetNumber targoffset;
    TupleTableSlot *slot;
    
    slot = node->ss.ss_ScanTupleSlot;
    
    for (;;)
    {
        /*
         * ═══════════════════════════════════════
         *  如果当前block还有tuple
         * ═══════════════════════════════════════
         */
        if (node->tbmres_index < node->tbmres_ntuples)
        {
            /*
             * 获取下一个offset
             */
            targoffset = node->tbmres_offsets[node->tbmres_index];
            node->tbmres_index++;
            
            /*
             * 从page获取tuple
             */
            HeapTuple tuple = heap_fetch_tuple(node->ss.ss_currentRelation,
                                              node->ss.ss_currentScanDesc->rs_cbuf,
                                              targoffset);
            
            if (tuple == NULL)
                continue;  /* 跳过已删除的tuple */
            
            /*
             * 检查可见性和条件
             */
            if (HeapTupleSatisfiesVisibility(tuple, ...))
            {
                ExecStoreBufferHeapTuple(tuple, slot, ...);
                
                /* Recheck条件 (lossy bitmap需要) */
                if (node->ss.ps.qual == NULL ||
                    ExecQual(node->ss.ps.qual, ...))
                {
                    return slot;
                }
            }
            
            continue;
        }
        
        /*
         * ═══════════════════════════════════════
         *  当前block完成，获取下一个block
         * ═══════════════════════════════════════
         */
        tbmres = tbm_iterate(node->tbmiterator);
        
        if (tbmres == NULL)
        {
            /* 位图扫描完成 */
            return ExecClearTuple(slot);
        }
        
        node->tbmres_blockno = tbmres->blockno;
        
        /*
         * 检查是否是lossy page
         */
        if (tbmres->ntuples >= 0)
        {
            /*
             * ✓ Exact (精确): 有具体的offset列表
             */
            node->tbmres_offsets = tbmres->offsets;
            node->tbmres_ntuples = tbmres->ntuples;
            node->exact_pages++;
        }
        else
        {
            /*
             * ✗ Lossy (有损): 整个page都可能有匹配
             * 需要扫描page中的所有tuple
             */
            node->tbmres_offsets = NULL;
            node->tbmres_ntuples = MaxHeapTuplesPerPage;
            node->lossy_pages++;
        }
        
        node->tbmres_index = 0;
        
        /*
         * 读取heap block
         */
        Buffer buf = ReadBufferExtended(node->ss.ss_currentRelation,
                                       MAIN_FORKNUM,
                                       node->tbmres_blockno,
                                       RBM_NORMAL,
                                       node->ss.ss_currentScanDesc->rs_strategy);
        
        node->ss.ss_currentScanDesc->rs_cbuf = buf;
        
        /*
         * 预取下一批blocks
         */
        if (node->prefetch_pages)
        {
            BitmapPrefetch(node, ...);
        }
    }
}
```

### Bitmap数据结构

```c
/*
 * TIDBitmap - TID位图
 * 文件: src/backend/nodes/tidbitmap.c
 */

/*
 * 位图有两种表示:
 * 1. Exact (精确): 记录具体的TID
 * 2. Lossy (有损): 只记录block号
 * 
 * 内存不足时，自动从Exact转为Lossy
 */

typedef struct TIDBitmap
{
    NodeTag     type;
    
    /* 哈希表: block号 → offset列表 */
    HTAB       *pagetable;         /* 精确模式 */
    
    /* 简单位图: block号 → 1 bit */
    BlockNumber *lossy_pages;      /* 有损模式 */
    int         nlossy_pages;
    
    /* 内存限制 */
    long        maxentries;        /* 最大entry数 */
    long        npages;            /* 当前page数 */
    
    MemoryContext mcxt;
} TIDBitmap;

/*
 * tbm_add_tuples - 添加TID到位图
 */
void
tbm_add_tuples(TIDBitmap *tbm, const ItemPointer tids,
              int ntids, bool recheck)
{
    for (int i = 0; i < ntids; i++)
    {
        BlockNumber blk = ItemPointerGetBlockNumber(&tids[i]);
        OffsetNumber off = ItemPointerGetOffsetNumber(&tids[i]);
        
        /*
         * 查找或创建page entry
         */
        PagetableEntry *page = tbm_get_pageentry(tbm, blk);
        
        if (page->status == TBM_EMPTY)
        {
            /*
             * 新page，添加到精确模式
             */
            page->status = TBM_EXACT;
            page->blockno = blk;
            tbm->npages++;
        }
        
        if (page->status == TBM_EXACT)
        {
            /*
             * 精确模式: 记录具体offset
             */
            page->offsets[page->ntuples++] = off;
            
            /*
             * 检查是否超过内存限制
             */
            if (tbm->npages > tbm->maxentries)
            {
                /*
                 * 内存不足!
                 * 转换为有损模式
                 */
                tbm_lossify(tbm);
            }
        }
    }
}

/*
 * tbm_lossify - 转换为有损模式
 */
static void
tbm_lossify(TIDBitmap *tbm)
{
    /*
     * 将精确的offset列表丢弃
     * 只保留block号
     * 
     * 影响:
     * - 减少内存占用 (~100倍)
     * - 需要Recheck (扫描整个page)
     * - 性能略有下降
     */
    HASH_SEQ_STATUS status;
    PagetableEntry *page;
    
    hash_seq_init(&status, tbm->pagetable);
    
    while ((page = hash_seq_search(&status)) != NULL)
    {
        if (page->status == TBM_EXACT)
        {
            /* 丢弃offset信息 */
            page->status = TBM_LOSSY;
            pfree(page->offsets);
            page->offsets = NULL;
            page->ntuples = -1;  /* -1表示lossy */
        }
    }
}
```

### 性能特性

```
【Bitmap Scan性能分析】

I/O模式:
  ✅ 顺序访问heap blocks
  ✅ 每个block最多读一次
  ✅ 可以预取 (prefetch)
  
  vs IndexScan:
    IndexScan: Block 5 → 2 → 8 → 1 → 5 (重复!)
    BitmapScan: Block 1 → 2 → 5 → 8 (顺序，不重复)

内存使用:
  精确模式: ~24 bytes per TID
  有损模式: ~1 bit per block
  
  work_mem控制最大内存:
    SET work_mem = '4MB';

适用场景:
  ✅ 中等数量行 (1-20% of table)
  ✅ 多个索引条件 (AND/OR)
  ✅ 内表有索引但选择性不高
  
  示例:
    WHERE age > 18 AND city = 'NYC'
    → BitmapAnd两个索引

成本估算:
  cost = (index_pages + heap_pages) * seq_page_cost +
         cpu_tuple_cost * tuples
  
  注意: 使用seq_page_cost (不是random!)
        因为heap访问是顺序的

EXPLAIN输出:
  Bitmap Heap Scan on users
    Recheck Cond: ((age > 18) AND (city = 'NYC'))
    →  BitmapAnd
         →  Bitmap Index Scan on idx_age
               Index Cond: (age > 18)
         →  Bitmap Index Scan on idx_city
               Index Cond: (city = 'NYC')
```

---

## 扫描方式对比总结

###对比表

```
┌──────────────────────────────────────────────────────────────┐
│ 扫描类型      │ 时间复杂度 │ I/O模式  │ 适用场景          │
├──────────────────────────────────────────────────────────────┤
│ SeqScan       │ O(N)      │ 顺序     │ 小表/大部分行     │
│ IndexScan     │ O(logN+M) │ 随机     │ 少量行(<5%)       │
│ IndexOnlyScan │ O(logN+M) │ 仅索引   │ 覆盖索引+VM       │
│ BitmapScan    │ O(logN+M) │ 半顺序   │ 中等行/多索引     │
│ TidScan       │ O(1)      │ 直接     │ ctid已知          │
└──────────────────────────────────────────────────────────────┘

【选择建议】
┌────────────────────────────────────────────┐
│ 返回行数 < 100:                            │
│   → IndexScan / IndexOnlyScan             │
│                                           │
│ 返回行数 100-10000:                        │
│   → BitmapScan (多条件)                   │
│   → IndexScan (单条件)                    │
│                                           │
│ 返回行数 > 10000:                          │
│   → SeqScan                               │
│   → 并行SeqScan (表很大)                  │
│                                           │
│ 覆盖索引 + 表clean:                        │
│   → IndexOnlyScan (最快!)                 │
└────────────────────────────────────────────┘
```

---

**文档版本**: PostgreSQL 17.6  
**创建日期**: 2025-10-17  
**重要程度**: ⭐⭐⭐⭐⭐

