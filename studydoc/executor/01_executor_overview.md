# Executor 执行器核心分析

> PostgreSQL执行器架构、扫描节点、Join节点、聚合和排序的完整分析

**源码**: `src/backend/executor/`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. 执行器架构

### 1.1 总体架构

```
                  Backend进程
                      │
     ┌────────────────┼────────────────┐
     │                │                │
  Parser         Analyzer          Planner
     │                │                │
     └────────────────┼────────────────┘
                      │
                      ▼
               Executor (执行器)
                      │
     ┌────────────────┼────────────────┐
     │                │                │
  扫描节点        Join节点          聚合节点
  │                │                │
  ├─ SeqScan       ├─ NestedLoop    ├─ Agg
  ├─ IndexScan     ├─ HashJoin      ├─ HashAgg
  ├─ BitmapScan    └─ MergeJoin     └─ GroupAgg
  └─ ...
```

### 1.2 核心函数

```c
// src/backend/executor/execMain.c

/* [1] 执行器初始化 */
void ExecutorStart(QueryDesc *queryDesc, int eflags)
{
    EState *estate = CreateExecutorState();
    InitPlan(queryDesc, eflags);
}

/* [2] 执行查询 */
void ExecutorRun(QueryDesc *queryDesc, ScanDirection direction, uint64 count)
{
    TupleTableSlot *slot;
    
    while (true)
    {
        /* 获取下一个元组 */
        slot = ExecProcNode(queryDesc->planstate);
        
        if (TupIsNull(slot))
            break;  // 没有更多元组
        
        /* 发送给客户端 */
        (*queryDesc->dest->receiveSlot)(slot, queryDesc->dest);
    }
}

/* [3] 清理 */
void ExecutorEnd(QueryDesc *queryDesc)
{
    ExecutePlan(queryDesc->estate, queryDesc->planstate, ...);
}
```

---

## 2. 火山模型 (Iterator Model)

### 2.1 核心思想

```
火山模型特点:
  • 每个节点实现 ExecProcNode() 接口
  • 每次调用返回一个元组 (TupleTableSlot)
  • 按需获取 (pull-based)
  • 流水线执行

示例执行:
  HashJoin->ExecProcNode()
    ├─→ left->ExecProcNode()  // 获取左表元组
    │    └─→ SeqScan->ExecProcNode()  // 扫描表
    │         └─→ 返回一个元组
    │
    ├─→ right->ExecProcNode() // 获取右表元组
    │    └─→ IndexScan->ExecProcNode()
    │         └─→ 返回一个元组
    │
    └─→ 执行Hash Join逻辑
         └─→ 返回连接后的元组
```

### 2.2 ExecProcNode实现

```c
// src/backend/executor/execProcnode.c
TupleTableSlot *
ExecProcNode(PlanState *node)
{
    TupleTableSlot *result;
    
    /* 检查中断 */
    CHECK_FOR_INTERRUPTS();
    
    /* 根据节点类型调用相应函数 */
    switch (nodeTag(node))
    {
        case T_SeqScanState:
            result = ExecSeqScan((SeqScanState *) node);
            break;
            
        case T_IndexScanState:
            result = ExecIndexScan((IndexScanState *) node);
            break;
            
        case T_HashJoinState:
            result = ExecHashJoin((HashJoinState *) node);
            break;
            
        case T_AggState:
            result = ExecAgg((AggState *) node);
            break;
            
        // ... 更多节点类型
    }
    
    return result;
}
```

---

## 3. 扫描节点

### 3.1 顺序扫描 (SeqScan)

```c
// src/backend/executor/nodeSeqscan.c
TupleTableSlot *
ExecSeqScan(SeqScanState *node)
{
    HeapScanDesc scandesc = node->ss_currentScanDesc;
    TupleTableSlot *slot = node->ss.ps.ps_ResultTupleSlot;
    
    while (true)
    {
        /* 获取下一个元组 */
        HeapTuple tuple = heap_getnext(scandesc, ForwardScanDirection);
        
        if (tuple == NULL)
            return ExecClearTuple(slot);  // 扫描结束
        
        /* 存储到slot */
        ExecStoreBufferHeapTuple(tuple, slot, scandesc->rs_cbuf);
        
        /* 应用过滤条件 */
        if (node->ss.ps.qual == NULL || 
            ExecQual(node->ss.ps.qual, econtext))
        {
            return slot;  // 满足条件，返回
        }
        
        /* 不满足条件，继续下一个 */
    }
}

// 执行计划示例:
EXPLAIN SELECT * FROM users WHERE age > 18;

/*
Seq Scan on users  (cost=0.00..35.50 rows=1000 width=40)
  Filter: (age > 18)
*/
```

### 3.2 索引扫描 (IndexScan)

```c
// src/backend/executor/nodeIndexscan.c
TupleTableSlot *
ExecIndexScan(IndexScanState *node)
{
    IndexScanDesc scandesc = node->iss_ScanDesc;
    TupleTableSlot *slot = node->ss.ps.ps_ResultTupleSlot;
    
    while (true)
    {
        /* 从索引获取下一个TID */
        bool found = index_getnext_tid(scandesc, ForwardScanDirection);
        
        if (!found)
            return ExecClearTuple(slot);
        
        /* 根据TID获取堆元组 */
        HeapTuple tuple = index_fetch_heap(scandesc);
        
        if (tuple == NULL)
            continue;  // 元组可能已被VACUUM
        
        ExecStoreBufferHeapTuple(tuple, slot, scandesc->xs_cbuf);
        
        /* 应用过滤条件 (IndexCond已在索引扫描时应用) */
        if (node->ss.ps.qual == NULL ||
            ExecQual(node->ss.ps.qual, econtext))
        {
            return slot;
        }
    }
}

// 执行计划示例:
EXPLAIN SELECT * FROM users WHERE id = 1;

/*
Index Scan using users_pkey on users  (cost=0.29..8.30 rows=1 width=40)
  Index Cond: (id = 1)
*/
```

### 3.3 仅索引扫描 (IndexOnlyScan)

```c
// 条件: 查询的列都在索引中 + 页面all-visible (VM)
TupleTableSlot *
ExecIndexOnlyScan(IndexOnlyScanState *node)
{
    IndexScanDesc scandesc = node->ioss_ScanDesc;
    
    while (true)
    {
        /* 从索引获取下一个元组 */
        bool found = index_getnext_slot(scandesc, ForwardScanDirection, slot);
        
        if (!found)
            return ExecClearTuple(slot);
        
        /* 检查可见性映射 */
        if (!VM_ALL_VISIBLE(rel, heapBlk, &vmbuffer))
        {
            /* 页面不是all-visible，需要检查堆元组 */
            HeapTuple tuple = index_fetch_heap(scandesc);
            // 检查可见性...
        }
        
        /* 索引数据已足够，不需要访问堆表 ✓ */
        return slot;
    }
}

// 执行计划示例:
EXPLAIN SELECT id FROM users WHERE id > 100;

/*
Index Only Scan using users_pkey on users  (cost=0.29..45.30 rows=1000 width=4)
  Index Cond: (id > 100)
*/
```

### 3.4 位图扫描 (BitmapScan)

```
特点: 先用索引收集所有TID，再批量访问堆表

优势:
  ✅ 顺序访问堆表 (减少随机I/O)
  ✅ 可以合并多个索引 (Bitmap OR/AND)

两阶段执行:
  Phase 1: BitmapIndexScan
    └─ 扫描索引，收集TID到位图
  
  Phase 2: BitmapHeapScan
    └─ 按TID顺序访问堆表
```

```sql
-- 执行计划示例:
EXPLAIN SELECT * FROM users WHERE age > 18 AND city = 'NYC';

/*
Bitmap Heap Scan on users  (cost=12.50..95.30 rows=500 width=40)
  Recheck Cond: ((age > 18) AND (city = 'NYC'::text))
  ->  BitmapAnd  (cost=12.50..12.50 rows=500 width=0)
        ->  Bitmap Index Scan on idx_age  (cost=0.00..4.25 rows=1000 width=0)
              Index Cond: (age > 18)
        ->  Bitmap Index Scan on idx_city  (cost=0.00..8.00 rows=800 width=0)
              Index Cond: (city = 'NYC'::text)
*/
```

---

## 4. Join节点

### 4.1 嵌套循环 (NestedLoop)

```
算法:
  for each row R in left table:
    for each row S in right table:
      if R join S matches:
        output joined tuple

复杂度: O(M × N)

适用场景:
  ✅ 小表 join 小表
  ✅ 内表有索引 (变成 Index Nested Loop)
  ✗ 大表 join 大表 (太慢!)
```

```c
// src/backend/executor/nodeNestloop.c
TupleTableSlot *
ExecNestLoop(NestLoopState *node)
{
    PlanState *outerPlan = outerPlanState(node);
    PlanState *innerPlan = innerPlanState(node);
    
    while (true)
    {
        /* 如果没有外表元组，获取下一个 */
        if (TupIsNull(node->nl_OuterTupleSlot))
        {
            outerTupleSlot = ExecProcNode(outerPlan);
            if (TupIsNull(outerTupleSlot))
                return NULL;  // 外表扫描完成
            
            node->nl_OuterTupleSlot = outerTupleSlot;
            
            /* 重新扫描内表 */
            ExecReScan(innerPlan);
        }
        
        /* 获取内表元组 */
        innerTupleSlot = ExecProcNode(innerPlan);
        
        if (TupIsNull(innerTupleSlot))
        {
            /* 内表扫描完成，清除外表元组，进入下一轮 */
            node->nl_OuterTupleSlot = NULL;
            continue;
        }
        
        /* 检查join条件 */
        if (joinqual == NULL || ExecQual(joinqual, econtext))
        {
            /* Join成功，返回结果 */
            return ExecProject(node->js.ps.ps_ProjInfo);
        }
    }
}
```

### 4.2 哈希连接 (HashJoin)

```
算法:
  Phase 1 (Build): 构建哈希表 (小表)
    for each row R in right table:
      hash(R.join_key) → insert into hash table
  
  Phase 2 (Probe): 探测哈希表 (大表)
    for each row S in left table:
      lookup hash table with hash(S.join_key)
      if found:
        output joined tuple

复杂度: O(M + N)

适用场景:
  ✅ 大表 join 大表
  ✅ 等值连接 (=)
  ✗ 非等值连接 (>, <, !=)
```

```c
// src/backend/executor/nodeHashjoin.c
TupleTableSlot *
ExecHashJoin(HashJoinState *node)
{
    switch (node->hj_JoinState)
    {
        case HJ_BUILD_HASHTABLE:
            /* Phase 1: 构建哈希表 */
            ExecHashTableCreate(hashNode, ...);
            
            while (true)
            {
                outerTupleSlot = ExecProcNode(outerPlan);
                if (TupIsNull(outerTupleSlot))
                    break;
                
                /* 插入哈希表 */
                ExecHashTableInsert(hashtable, outerTupleSlot, hashvalue);
            }
            
            node->hj_JoinState = HJ_NEED_NEW_OUTER;
            // fall through
            
        case HJ_NEED_NEW_OUTER:
            /* Phase 2: 探测哈希表 */
            innerTupleSlot = ExecProcNode(innerPlan);
            
            if (TupIsNull(innerTupleSlot))
                return NULL;  // 完成
            
            /* 计算哈希值并查找 */
            hashvalue = ExecHashGetHashValue(hashtable, econtext, ...);
            ExecHashGetBucket(hashtable, hashvalue, &bucketno);
            
            /* 遍历bucket中的匹配元组 */
            while ((tuple = ExecHashGetNext(hashtable)) != NULL)
            {
                if (ExecQual(joinqual, econtext))
                    return ExecProject(...);  // Join成功
            }
            
            // 继续下一个内表元组
            break;
    }
}
```

### 4.3 归并连接 (MergeJoin)

```
算法:
  前提: 两表都已按join key排序
  
  pointer_left = first row of left table
  pointer_right = first row of right table
  
  while both tables have rows:
    if left.key < right.key:
      advance left pointer
    else if left.key > right.key:
      advance right pointer
    else:  // keys match
      output joined tuple
      advance both pointers

复杂度: O(M + N) (如果已排序)
        O(M log M + N log N) (如果需要排序)

适用场景:
  ✅ 两表都有序 (索引或已排序)
  ✅ 大表 join 大表
  ✅ 非等值连接也支持
```

```sql
-- 执行计划示例:
EXPLAIN SELECT * FROM users u JOIN orders o ON u.id = o.user_id;

/*
Merge Join  (cost=135.75..245.80 rows=5000 width=80)
  Merge Cond: (u.id = o.user_id)
  ->  Index Scan using users_pkey on users u  (cost=0.29..45.30 rows=1000 width=40)
  ->  Sort  (cost=135.45..148.95 rows=5000 width=40)
        Sort Key: o.user_id
        ->  Seq Scan on orders o  (cost=0.00..85.00 rows=5000 width=40)
*/
```

---

## 5. 聚合节点

### 5.1 普通聚合 (Agg)

```c
// src/backend/executor/nodeAgg.c
TupleTableSlot *
ExecAgg(AggState *aggstate)
{
    switch (aggstate->phase)
    {
        case AGG_FETCH:
            /* 获取所有输入元组并累积 */
            while (true)
            {
                slot = ExecProcNode(outerPlan);
                if (TupIsNull(slot))
                    break;
                
                /* 更新聚合状态 */
                advance_aggregates(aggstate, pergroup);
            }
            
            aggstate->phase = AGG_OUTPUT;
            // fall through
            
        case AGG_OUTPUT:
            /* 输出聚合结果 */
            return project_aggregates(aggstate);
    }
}

// 示例:
SELECT COUNT(*), AVG(age) FROM users;

/*
Aggregate  (cost=22.50..22.51 rows=1 width=16)
  ->  Seq Scan on users  (cost=0.00..22.00 rows=1000 width=4)
*/
```

### 5.2 哈希聚合 (HashAgg)

```
特点: 使用哈希表分组

算法:
  hash_table = {}
  
  for each input row:
    group_key = row.group_by_cols
    hash_table[group_key].accumulate(row)
  
  for each group in hash_table:
    output aggregate result

复杂度: O(N)

适用: GROUP BY查询
```

```sql
-- 示例:
SELECT city, COUNT(*), AVG(age) FROM users GROUP BY city;

/*
HashAggregate  (cost=27.00..29.00 rows=100 width=48)
  Group Key: city
  ->  Seq Scan on users  (cost=0.00..22.00 rows=1000 width=20)
*/
```

---

## 6. 排序节点

```c
// src/backend/executor/nodeSort.c
TupleTableSlot *
ExecSort(SortState *node)
{
    switch (node->sort_Done)
    {
        case false:
            /* 第一次调用: 收集并排序所有元组 */
            while (true)
            {
                slot = ExecProcNode(outerPlan);
                if (TupIsNull(slot))
                    break;
                
                /* 插入到tuplesort */
                tuplesort_puttupleslot(tuplesortstate, slot);
            }
            
            /* 执行排序 */
            tuplesort_performsort(tuplesortstate);
            node->sort_Done = true;
            // fall through
            
        case true:
            /* 后续调用: 返回排序后的元组 */
            if (tuplesort_gettupleslot(tuplesortstate, true, slot, NULL))
                return slot;
            else
                return ExecClearTuple(slot);  // 完成
    }
}

// 示例:
SELECT * FROM users ORDER BY age;

/*
Sort  (cost=65.83..68.33 rows=1000 width=40)
  Sort Key: age
  ->  Seq Scan on users  (cost=0.00..22.00 rows=1000 width=40)
*/
```

---

## 7. 性能优化

### 7.1 选择合适的Join算法

```
NestedLoop:
  • 小表 × 小表
  • 内表有索引

HashJoin:
  • 大表 × 大表
  • 等值连接
  • 内存足够 (work_mem)

MergeJoin:
  • 两表都有序
  • 非等值连接
```

### 7.2 关键参数

```sql
-- 工作内存 (排序、哈希)
work_mem = '256MB'  -- 默认4MB，增大可提速

-- 随机页面代价
random_page_cost = 1.1  -- SSD设为1.1-1.5

-- 并行查询
max_parallel_workers_per_gather = 4
```

---

## 总结

### 执行器核心

1. **火山模型**: Iterator接口，按需获取元组
2. **扫描节点**: SeqScan、IndexScan、IndexOnlyScan、BitmapScan
3. **Join节点**: NestedLoop、HashJoin、MergeJoin
4. **聚合节点**: Agg、HashAgg
5. **排序节点**: Sort

### 性能要点

- ✅ 使用索引 (IndexScan、IndexOnlyScan)
- ✅ 选择合适的Join算法
- ✅ 调整work_mem (哈希和排序)
- ✅ 并行查询 (大表扫描)

---

**完成**: Executor模块核心文档已完成！  
**完成**: 所有P1模块已完成！🎉

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

