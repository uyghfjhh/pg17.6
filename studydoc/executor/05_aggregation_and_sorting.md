# 聚合和排序深度分析

> PostgreSQL聚合和排序算法的完整实现

**重要程度**: ⭐⭐⭐⭐⭐  
**源码位置**: `src/backend/executor/nodeAgg.c`, `nodeSort.c`, `src/backend/utils/sort/tuplesort.c`  
**核心概念**: HashAgg, GroupAgg, Sort, Tuplesort

---

## 📋 聚合算法总览

### 三种聚合方式

```
【PostgreSQL的聚合算法】

1. Plain Aggregate (简单聚合)
   ┌──────────────────────────────┐
   │ SELECT COUNT(*), AVG(amount) │
   │ FROM orders;                 │
   │                              │
   │ 无GROUP BY                   │
   │ 扫描所有行，累积结果         │
   │ 返回单行                     │
   └──────────────────────────────┘

2. HashAggregate (哈希聚合)
   ┌──────────────────────────────┐
   │ SELECT city, COUNT(*)        │
   │ FROM users                   │
   │ GROUP BY city;               │
   │                              │
   │ 使用Hash表分组               │
   │ 时间: O(N)                   │
   │ 空间: O(groups)              │
   └──────────────────────────────┘

3. GroupAggregate (分组聚合)
   ┌──────────────────────────────┐
   │ 前提: 数据已按GROUP BY排序   │
   │                              │
   │ 顺序扫描，遇到新组输出结果   │
   │ 时间: O(N)                   │
   │ 空间: O(1)                   │
   └──────────────────────────────┘
```

---

## 1. Plain Aggregate - 简单聚合

### 算法原理

```
【无GROUP BY的聚合】

查询:
  SELECT COUNT(*), SUM(amount), AVG(age)
  FROM users;

执行过程:
┌─────────────────────────────────────────┐
│ 初始化:                                 │
│   count = 0                             │
│   sum_amount = 0                        │
│   sum_age = 0                           │
│   count_age = 0                         │
│                                         │
│ 扫描所有行:                             │
│   Row 1: amount=100, age=25             │
│     count++         → 1                 │
│     sum_amount+=100 → 100               │
│     sum_age+=25     → 25                │
│     count_age++     → 1                 │
│                                         │
│   Row 2: amount=50, age=30              │
│     count++         → 2                 │
│     sum_amount+=50  → 150               │
│     sum_age+=30     → 55                │
│     count_age++     → 2                 │
│                                         │
│   ... 继续所有行                        │
│                                         │
│ 最终化:                                 │
│   count = 1000                          │
│   sum_amount = 50000                    │
│   avg_age = sum_age / count_age = 27.5 │
│                                         │
│ 输出: (1000, 50000, 27.5)               │
└─────────────────────────────────────────┘
```

### 核心实现

```c
/*
 * Plain Aggregate实现
 * 文件: src/backend/executor/nodeAgg.c
 */

/* Aggregate状态 */
typedef struct AggState
{
    ScanState   ss;                 /* 基类 */
    
    /* 聚合阶段 */
    AggStatePerAgg peragg;          /* 每个聚合函数的状态 */
    AggStatePerGroup pergroup;      /* 每个分组的状态 */
    
    /* 执行阶段 */
    AggPhase    current_phase;      /* 当前阶段 */
    int         numphases;          /* 总阶段数 */
    
    /* Hash表 (HashAgg用) */
    TupleHashTable hashtable;
    
    /* 排序状态 (GroupAgg用) */
    Tuplesortstate *sort_state;
    
    /* 性能统计 */
    long        hash_batches_used;  /* Hash批次数 */
    long        hash_mem_peak;      /* Hash内存峰值 */
} AggState;

/* 每个聚合函数的状态 */
typedef struct AggStatePerAggData
{
    /*
     * 聚合函数信息
     */
    Oid         aggfnoid;           /* 聚合函数OID */
    
    /* 
     * 转换函数 (Transition Function)
     * 例如: COUNT → int8inc
     *      SUM   → int8add
     *      AVG   → int8_avg_accum
     */
    FmgrInfo    transfn;            /* 转换函数 */
    FmgrInfo    finalfn;            /* 最终化函数 */
    
    /*
     * 状态值
     * 累积的中间结果
     */
    Datum       transValue;         /* 转换状态值 */
    bool        transValueIsNull;
    
    /*
     * 类型信息
     */
    Oid         transtype;          /* 转换状态类型 */
    int16       transtypeLen;
    bool        transtypeByVal;
    
} AggStatePerAggData;

/*
 * ExecAgg - 聚合执行主函数
 */
static TupleTableSlot *
ExecAgg(PlanState *pstate)
{
    AggState   *node = castNode(AggState, pstate);
    TupleTableSlot *result = NULL;
    
    /*
     * Plain Aggregate执行
     */
    if (node->aggstrategy == AGG_PLAIN)
    {
        result = agg_retrieve_direct(node);
    }
    else if (node->aggstrategy == AGG_HASHED)
    {
        /*
         * Hash Aggregate
         */
        if (!node->table_filled)
            agg_fill_hash_table(node);
        result = agg_retrieve_hash_table(node);
    }
    else
    {
        /*
         * Group Aggregate
         */
        result = ExecScan(&node->ss,
                         (ExecScanAccessMtd) agg_retrieve_grouped,
                         (ExecScanRecheckMtd) NULL);
    }
    
    return result;
}

/*
 * agg_retrieve_direct - Plain Aggregate执行
 */
static TupleTableSlot *
agg_retrieve_direct(AggState *aggstate)
{
    PlanState  *outerPlan;
    TupleTableSlot *outerslot;
    
    outerPlan = outerPlanState(aggstate);
    
    /*
     * ═══════════════════════════════════════════════
     *  Phase 1: 扫描所有输入tuple，累积状态
     * ═══════════════════════════════════════════════
     */
    for (;;)
    {
        /*
         * 获取下一个输入tuple
         */
        outerslot = ExecProcNode(outerPlan);
        
        if (TupIsNull(outerslot))
            break;  /* 输入完成 */
        
        /*
         * 对每个聚合函数调用转换函数
         * 更新累积状态
         */
        advance_aggregates(aggstate);
    }
    
    /*
     * ═══════════════════════════════════════════════
     *  Phase 2: 最终化聚合结果
     * ═══════════════════════════════════════════════
     */
    return finalize_aggregates(aggstate,
                              aggstate->peragg,
                              aggstate->pergroup);
}

/*
 * advance_aggregates - 推进聚合状态
 */
static void
advance_aggregates(AggState *aggstate)
{
    ExprContext *econtext = aggstate->tmpcontext;
    int         aggno;
    
    /*
     * 遍历所有聚合函数
     */
    for (aggno = 0; aggno < aggstate->numaggs; aggno++)
    {
        AggStatePerAgg peraggstate = &aggstate->peragg[aggno];
        AggStatePerGroup pergroupstate = &aggstate->pergroup[aggno];
        int         nargs = peraggstate->numArguments;
        
        /*
         * 评估聚合函数的参数
         * 例如: SUM(amount) → 获取amount的值
         */
        Datum      *args = (Datum *) palloc(nargs * sizeof(Datum));
        bool       *nulls = (bool *) palloc(nargs * sizeof(bool));
        
        for (int i = 0; i < nargs; i++)
        {
            args[i] = ExecEvalExpr(peraggstate->args[i],
                                  econtext,
                                  &nulls[i]);
        }
        
        /*
         * ═══════════════════════════════════════════
         *  调用转换函数 (Transition Function)
         * ═══════════════════════════════════════════
         * 
         * 转换函数格式:
         *   new_state = transfn(old_state, new_value)
         * 
         * 示例:
         *   COUNT: int8inc(state, value)
         *     → state + 1
         *   
         *   SUM: int8add(state, value)
         *     → state + value
         *   
         *   AVG: int8_avg_accum(state, value)
         *     → {sum + value, count + 1}
         */
        
        Datum newVal;
        
        if (peraggstate->transfn.fn_strict)
        {
            /*
             * 严格函数: 任何参数为NULL就跳过
             * 例如: SUM NULL值不计入
             */
            bool hasNulls = pergroupstate->transValueIsNull;
            for (int i = 0; i < nargs; i++)
            {
                if (nulls[i])
                {
                    hasNulls = true;
                    break;
                }
            }
            
            if (hasNulls)
                continue;  /* 跳过这个tuple */
        }
        
        /*
         * 调用转换函数
         */
        fcinfo->args[0].value = pergroupstate->transValue;
        fcinfo->args[0].isnull = pergroupstate->transValueIsNull;
        
        for (int i = 0; i < nargs; i++)
        {
            fcinfo->args[i + 1].value = args[i];
            fcinfo->args[i + 1].isnull = nulls[i];
        }
        
        newVal = FunctionCallInvoke(fcinfo);
        
        /*
         * 更新状态值
         */
        pergroupstate->transValue = newVal;
        pergroupstate->transValueIsNull = fcinfo->isnull;
        
        pfree(args);
        pfree(nulls);
    }
}

/*
 * finalize_aggregates - 最终化聚合结果
 */
static TupleTableSlot *
finalize_aggregates(AggState *aggstate,
                   AggStatePerAgg peraggs,
                   AggStatePerGroup pergroup)
{
    ExprContext *econtext = aggstate->ss.ps.ps_ExprContext;
    Datum      *values;
    bool       *isnull;
    int         aggno;
    
    /*
     * 分配结果数组
     */
    values = econtext->ecxt_aggvalues;
    isnull = econtext->ecxt_aggnulls;
    
    /*
     * 对每个聚合函数调用最终化函数
     */
    for (aggno = 0; aggno < aggstate->numaggs; aggno++)
    {
        AggStatePerAgg peraggstate = &peraggs[aggno];
        AggStatePerGroup pergroupstate = &pergroup[aggno];
        
        if (OidIsValid(peraggstate->finalfn_oid))
        {
            /*
             * ═══════════════════════════════════════
             *  有最终化函数
             * ═══════════════════════════════════════
             * 
             * 示例:
             *   AVG: int8_avg(state)
             *     → state.sum / state.count
             *   
             *   STDDEV: float8_stddev_samp(state)
             *     → sqrt(variance)
             */
            fcinfo->args[0].value = pergroupstate->transValue;
            fcinfo->args[0].isnull = pergroupstate->transValueIsNull;
            
            values[aggno] = FunctionCallInvoke(&peraggstate->finalfn);
            isnull[aggno] = fcinfo->isnull;
        }
        else
        {
            /*
             * 无最终化函数，直接使用状态值
             * 例如: COUNT, SUM
             */
            values[aggno] = pergroupstate->transValue;
            isnull[aggno] = pergroupstate->transValueIsNull;
        }
    }
    
    /*
     * 投影并返回结果tuple
     */
    return ExecProject(aggstate->ss.ps.ps_ProjInfo);
}
```

### 聚合函数示例

```c
/*
 * COUNT聚合函数实现
 * 文件: src/backend/utils/adt/int8.c
 */

/*
 * 转换函数: int8inc
 * 功能: state + 1
 */
Datum
int8inc(PG_FUNCTION_ARGS)
{
    /*
     * 如果第一次调用，初始化为0
     */
    if (PG_ARGISNULL(0))
        PG_RETURN_INT64(1);
    
    /*
     * 否则，state + 1
     */
    int64 arg = PG_GETARG_INT64(0);
    
    PG_RETURN_INT64(arg + 1);
}

/*
 * COUNT(*)无最终化函数
 * 直接返回state值
 */

/*
 * SUM聚合函数
 */

/*
 * 转换函数: int8_sum
 * 功能: state + value
 */
Datum
int8_sum(PG_FUNCTION_ARGS)
{
    int64       state;
    int64       value;
    
    if (PG_ARGISNULL(0))
    {
        /* 第一次调用 */
        if (PG_ARGISNULL(1))
            PG_RETURN_NULL();  /* SUM(NULL) = NULL */
        
        PG_RETURN_INT64(PG_GETARG_INT64(1));
    }
    
    state = PG_GETARG_INT64(0);
    
    if (PG_ARGISNULL(1))
        PG_RETURN_INT64(state);  /* 跳过NULL值 */
    
    value = PG_GETARG_INT64(1);
    
    PG_RETURN_INT64(state + value);
}

/*
 * AVG聚合函数
 */

/* 内部状态: {sum, count} */
typedef struct Int8TransTypeData
{
    int64       count;
    int64       sum;
} Int8TransTypeData;

/*
 * 转换函数: int8_avg_accum
 * 功能: {state.sum + value, state.count + 1}
 */
Datum
int8_avg_accum(PG_FUNCTION_ARGS)
{
    Int8TransTypeData *transdata;
    
    if (PG_ARGISNULL(0))
    {
        /* 第一次调用，创建状态 */
        transdata = (Int8TransTypeData *) palloc0(sizeof(Int8TransTypeData));
    }
    else
    {
        transdata = (Int8TransTypeData *) PG_GETARG_POINTER(0);
    }
    
    if (!PG_ARGISNULL(1))
    {
        transdata->sum += PG_GETARG_INT64(1);
        transdata->count++;
    }
    
    PG_RETURN_POINTER(transdata);
}

/*
 * 最终化函数: int8_avg
 * 功能: state.sum / state.count
 */
Datum
int8_avg(PG_FUNCTION_ARGS)
{
    Int8TransTypeData *transdata;
    
    if (PG_ARGISNULL(0))
        PG_RETURN_NULL();
    
    transdata = (Int8TransTypeData *) PG_GETARG_POINTER(0);
    
    if (transdata->count == 0)
        PG_RETURN_NULL();
    
    /* 返回平均值 */
    PG_RETURN_NUMERIC(
        int64_div_fast_to_numeric(transdata->sum, transdata->count));
}
```

---

## 2. Hash Aggregate - 哈希聚合

### 算法原理

```
【Hash Aggregate执行过程】

查询:
  SELECT city, COUNT(*), AVG(age)
  FROM users
  GROUP BY city;

数据:
  [Alice, NYC, 25]
  [Bob, LA, 30]
  [Carol, NYC, 28]
  [Dave, SF, 35]
  [Eve, LA, 22]

执行过程:
┌─────────────────────────────────────────┐
│ Phase 1: Build Hash Table               │
│                                         │
│ Hash Table:                             │
│ ┌───────────────────────────┐           │
│ │ Key       │ Count │ Sum   │ Count   │ │
│ ├───────────────────────────┤           │
│ │ NYC       │   0   │   0   │   0     │ │
│ │ LA        │   0   │   0   │   0     │ │
│ │ SF        │   0   │   0   │   0     │ │
│ └───────────────────────────┘           │
│                                         │
│ Row 1: [Alice, NYC, 25]                 │
│   hash('NYC') → 找到或创建entry         │
│   NYC: count=1, sum=25, count=1         │
│                                         │
│ Row 2: [Bob, LA, 30]                    │
│   hash('LA') → 找到或创建entry          │
│   LA: count=1, sum=30, count=1          │
│                                         │
│ Row 3: [Carol, NYC, 28]                 │
│   hash('NYC') → 找到已有entry           │
│   NYC: count=2, sum=53, count=2         │
│                                         │
│ Row 4: [Dave, SF, 35]                   │
│   SF: count=1, sum=35, count=1          │
│                                         │
│ Row 5: [Eve, LA, 22]                    │
│   LA: count=2, sum=52, count=2          │
│                                         │
│ 最终Hash Table:                         │
│ ┌───────────────────────────┐           │
│ │ NYC │  2  │  53  │  2    │           │
│ │ LA  │  2  │  52  │  2    │           │
│ │ SF  │  1  │  35  │  1    │           │
│ └───────────────────────────┘           │
│                                         │
│ Phase 2: Output Results                 │
│   NYC: count=2, avg=26.5                │
│   LA:  count=2, avg=26.0                │
│   SF:  count=1, avg=35.0                │
└─────────────────────────────────────────┘

时间: O(N)
空间: O(distinct_groups)
```

### 核心实现

```c
/*
 * Hash Aggregate实现
 * 文件: src/backend/executor/nodeAgg.c
 */

/* Hash表entry */
typedef struct TupleHashEntryData
{
    /* Hash表链表 */
    struct TupleHashEntryData *next;
    
    /* Hash值 */
    uint32      hash;
    
    /* Group Key值 */
    MinimalTuple firstTuple;    /* 第一个tuple (包含group key) */
    
    /* 聚合状态 */
    /* 每个聚合函数的状态值都存储在这里 */
} TupleHashEntryData;

/*
 * agg_fill_hash_table - 填充Hash表
 */
static void
agg_fill_hash_table(AggState *aggstate)
{
    PlanState  *outerPlan;
    ExprContext *econtext;
    TupleTableSlot *outerslot;
    TupleHashTable hashtable;
    
    outerPlan = outerPlanState(aggstate);
    econtext = aggstate->tmpcontext;
    hashtable = aggstate->hashtable;
    
    /*
     * ═══════════════════════════════════════════════
     *  扫描所有输入tuple
     * ═══════════════════════════════════════════════
     */
    for (;;)
    {
        /*
         * 获取下一个输入tuple
         */
        outerslot = ExecProcNode(outerPlan);
        
        if (TupIsNull(outerslot))
            break;
        
        /*
         * 准备econtext
         */
        econtext->ecxt_outertuple = outerslot;
        
        /*
         * ───────────────────────────────────────────
         *  计算GROUP BY key的hash值
         * ───────────────────────────────────────────
         */
        uint32 hash = 0;
        bool isNull;
        
        for (int i = 0; i < aggstate->numGroupCols; i++)
        {
            AttrNumber  attno = aggstate->grpColIdx[i];
            Datum       value;
            
            /* 获取GROUP BY列的值 */
            value = slot_getattr(outerslot, attno, &isNull);
            
            if (!isNull)
            {
                /* 累积hash值 */
                hash = hash_combine(hash,
                                   DatumGetUInt32(
                                       FunctionCall1(&aggstate->hashfunctions[i],
                                                    value)));
            }
        }
        
        /*
         * ───────────────────────────────────────────
         *  在Hash表中查找或创建entry
         * ───────────────────────────────────────────
         */
        bool        isnew;
        TupleHashEntry entry;
        
        entry = LookupTupleHashEntry(hashtable,
                                    outerslot,
                                    &isnew,
                                    hash);
        
        if (isnew)
        {
            /*
             * ✓ 新的group
             * 初始化聚合状态
             */
            initialize_aggregates(aggstate,
                                 aggstate->peragg,
                                 entry->additional);
        }
        
        /*
         * ───────────────────────────────────────────
         *  更新这个group的聚合状态
         * ───────────────────────────────────────────
         */
        advance_aggregates(aggstate,
                          entry->additional);
        
        /*
         * 检查内存使用
         */
        if (aggstate->hash_spill_mode == HASHAGG_NO_SPILL &&
            aggstate->mem_used > aggstate->mem_limit)
        {
            /*
             * ❌ 内存不足!
             * 需要溢出到磁盘 (Disk-based HashAgg)
             */
            hash_agg_enter_spill_mode(aggstate);
        }
    }
    
    aggstate->table_filled = true;
}

/*
 * LookupTupleHashEntry - 在Hash表中查找entry
 */
TupleHashEntry
LookupTupleHashEntry(TupleHashTable hashtable,
                    TupleTableSlot *slot,
                    bool *isnew,
                    uint32 hash)
{
    TupleHashEntry entry;
    int         bucketno;
    
    /*
     * 计算bucket号
     */
    bucketno = hash % hashtable->nbuckets;
    
    /*
     * 在bucket链表中查找
     */
    entry = hashtable->buckets[bucketno];
    
    while (entry != NULL)
    {
        if (entry->hash == hash)
        {
            /*
             * Hash值相同，检查实际key值
             */
            if (execTuplesMatch(slot,
                               entry->firstTuple,
                               hashtable->tab_hash_funcs,
                               hashtable->tab_eq_funcoids,
                               hashtable->tablecxt))
            {
                /*
                 * ✓ 找到匹配的entry
                 */
                *isnew = false;
                return entry;
            }
        }
        
        entry = entry->next;
    }
    
    /*
     * 没找到，创建新entry
     */
    entry = (TupleHashEntry) MemoryContextAlloc(
                hashtable->tablecxt,
                sizeof(TupleHashEntryData) +
                hashtable->additional_size);
    
    entry->hash = hash;
    entry->firstTuple = ExecCopySlotMinimalTuple(slot);
    
    /* 插入到bucket链表头部 */
    entry->next = hashtable->buckets[bucketno];
    hashtable->buckets[bucketno] = entry;
    
    hashtable->nentries++;
    
    *isnew = true;
    return entry;
}
```

### Disk-based HashAgg

```c
/*
 * Disk-based Hash Aggregate
 * 
 * 当内存不足时，将部分数据溢出到磁盘
 * 类似Hash Join的批处理
 */

/*
 * hash_agg_enter_spill_mode - 进入溢出模式
 */
static void
hash_agg_enter_spill_mode(AggState *aggstate)
{
    int         nbatches = 8;  /* 分成8批 */
    int         i;
    
    /*
     * 创建临时文件
     */
    aggstate->hash_spill_files = (BufFile **)
        palloc0(nbatches * sizeof(BufFile *));
    
    for (i = 0; i < nbatches; i++)
    {
        aggstate->hash_spill_files[i] = BufFileCreateTemp(false);
    }
    
    aggstate->hash_ngroups_current = aggstate->hashtable->nentries;
    aggstate->hash_nbatches = nbatches;
    aggstate->hash_spill_mode = HASHAGG_SPILL_ACTIVE;
    
    /*
     * 重新hash当前内存中的groups
     * 将部分groups写入临时文件
     */
    hash_agg_spill_tuple(aggstate);
}

/*
 * 执行计划显示:
 */
EXPLAIN (ANALYZE, BUFFERS)
SELECT city, COUNT(*), AVG(age)
FROM large_users
GROUP BY city;

/*
HashAggregate  (cost=... rows=... width=...)
  (actual time=1234.567..2345.678 rows=1000000 loops=1)
  Group Key: city
  Batches: 8  ← 使用了8批处理!
  Memory Usage: 256kB  ← 内存限制
  Disk Usage: 1024kB   ← 使用了磁盘
  ->  Seq Scan on large_users ...
Planning Time: 0.123 ms
Execution Time: 2456.789 ms
Buffers: shared hit=10000, temp read=2000 temp written=2000
*/
```

---

## 3. Group Aggregate - 分组聚合

### 算法原理

```
【Group Aggregate - 需要排序】

前提: 数据已按GROUP BY key排序

查询:
  SELECT city, COUNT(*), AVG(age)
  FROM users
  GROUP BY city
  ORDER BY city;  ← 隐式排序要求

排序后数据:
  [Alice, LA, 30]
  [Eve, LA, 22]      ← LA组
  ────────────────
  [Carol, NYC, 28]
  [Alice2, NYC, 25]  ← NYC组
  ────────────────
  [Dave, SF, 35]     ← SF组

执行过程:
┌─────────────────────────────────────────┐
│ 当前组: 无                              │
│ 状态: count=0, sum=0                    │
│                                         │
│ Row 1: [Alice, LA, 30]                  │
│   新组! LA                              │
│   初始化: count=1, sum=30               │
│                                         │
│ Row 2: [Eve, LA, 22]                    │
│   同组 LA                               │
│   更新: count=2, sum=52                 │
│                                         │
│ Row 3: [Carol, NYC, 28]                 │
│   新组! NYC                             │
│   ✓ 输出LA: count=2, avg=26.0          │
│   初始化: count=1, sum=28               │
│                                         │
│ Row 4: [Alice2, NYC, 25]                │
│   同组 NYC                              │
│   更新: count=2, sum=53                 │
│                                         │
│ Row 5: [Dave, SF, 35]                   │
│   新组! SF                              │
│   ✓ 输出NYC: count=2, avg=26.5         │
│   初始化: count=1, sum=35               │
│                                         │
│ 结束:                                    │
│   ✓ 输出SF: count=1, avg=35.0          │
└─────────────────────────────────────────┘

优势:
  • 时间: O(N) - 单次扫描
  • 空间: O(1) - 只保存当前组状态
  • 无需Hash表
```

### 核心实现

```c
/*
 * Group Aggregate实现
 * 文件: src/backend/executor/nodeAgg.c
 */

/*
 * agg_retrieve_grouped - Group Aggregate执行
 */
static TupleTableSlot *
agg_retrieve_grouped(AggState *aggstate)
{
    TupleTableSlot *outerslot;
    TupleTableSlot *firstSlot;
    
    /*
     * 如果是第一次调用
     */
    if (aggstate->grp_firstTuple == NULL)
    {
        /*
         * 获取第一个tuple
         */
        outerslot = ExecProcNode(outerPlanState(aggstate));
        
        if (TupIsNull(outerslot))
            return NULL;  /* 空输入 */
        
        /*
         * 初始化第一个group
         */
        aggstate->grp_firstTuple = ExecCopySlotHeapTuple(outerslot);
        
        /*
         * 初始化聚合状态
         */
        initialize_aggregates(aggstate,
                            aggstate->peragg,
                            aggstate->pergroup);
        
        /*
         * 处理第一个tuple
         */
        advance_aggregates(aggstate, aggstate->pergroup);
    }
    
    /*
     * ═══════════════════════════════════════════════
     *  主循环: 处理同一组的tuples
     * ═══════════════════════════════════════════════
     */
    for (;;)
    {
        /*
         * 获取下一个tuple
         */
        outerslot = ExecProcNode(outerPlanState(aggstate));
        
        if (TupIsNull(outerslot))
        {
            /*
             * 输入结束
             * 输出最后一个group
             */
            return finalize_aggregates(aggstate,
                                      aggstate->peragg,
                                      aggstate->pergroup);
        }
        
        /*
         * ───────────────────────────────────────────
         *  比较GROUP BY key
         *  检查是否还是同一组
         * ───────────────────────────────────────────
         */
        econtext->ecxt_outertuple = outerslot;
        
        if (!execTuplesMatch(aggstate->grp_firstTuple,
                           outerslot,
                           aggstate->numGroupCols,
                           aggstate->grpColIdx,
                           aggstate->eqfunctions,
                           econtext->ecxt_per_tuple_memory))
        {
            /*
             * ✓ 新组!
             * 输出当前组的结果
             */
            TupleTableSlot *result;
            
            result = finalize_aggregates(aggstate,
                                        aggstate->peragg,
                                        aggstate->pergroup);
            
            /*
             * 保存新组的第一个tuple
             */
            aggstate->grp_firstTuple = ExecCopySlotHeapTuple(outerslot);
            
            /*
             * 重新初始化聚合状态
             */
            initialize_aggregates(aggstate,
                                aggstate->peragg,
                                aggstate->pergroup);
            
            /*
             * 处理新组的第一个tuple
             */
            advance_aggregates(aggstate, aggstate->pergroup);
            
            /*
             * 返回上一组的结果
             */
            return result;
        }
        
        /*
         * 同组，继续累积
         */
        advance_aggregates(aggstate, aggstate->pergroup);
    }
}
```

---

## 4. Sort - 排序实现

### Tuplesort算法

```
【Tuplesort - 多路归并排序】

算法: External Merge Sort (外部归并排序)

Phase 1: 构建初始runs
┌───────────────────────────────────┐
│ 输入数据流:                       │
│ [5][2][8][1][9][3][7][4][6]...    │
│         ↓                         │
│ 在内存中排序 (work_mem大小):      │
│ Run 1: [1][2][5][8]               │
│ Run 2: [3][4][7][9]               │
│ Run 3: [6][10][12][15]            │
│ ...                               │
│         ↓                         │
│ 写入临时文件                       │
└───────────────────────────────────┘

Phase 2: 归并runs
┌───────────────────────────────────┐
│ 多路归并:                         │
│                                   │
│ Run 1: [1] [2] [5] [8]            │
│         ↓                         │
│ Run 2: [3] [4] [7] [9]            │
│         ↓                         │
│ Run 3: [6][10][12][15]            │
│         ↓                         │
│    ┌────────────┐                │
│    │ Min-Heap   │                │
│    │   (1)      │                │
│    │  /   \     │                │
│    │ (3)  (6)   │                │
│    └────────────┘                │
│         ↓                         │
│ 输出: [1][2][3][4][5][6]...       │
└───────────────────────────────────┘

复杂度:
  时间: O(N log N)
  空间: O(work_mem)
  I/O: 如果不fit in memory，需要多次pass
```

### 核心实现

```c
/*
 * Tuplesort实现
 * 文件: src/backend/utils/sort/tuplesort.c
 */

/* Tuplesort状态 */
typedef struct Tuplesortstate
{
    /* 排序方法 */
    TupSortStatus status;       /* 当前状态 */
    
    /* 内存tuples */
    SortTuple  *memtuples;      /* 内存中的tuples */
    int         memtupcount;    /* tuple数量 */
    int         memtupsize;     /* 数组大小 */
    
    /* Work memory限制 */
    int64       allowedMem;     /* 允许的内存 (work_mem) */
    int64       availMem;       /* 剩余内存 */
    
    /* 临时文件 (外部排序) */
    int         maxTapes;       /* 最大tape数 */
    int         tapeRange;      /* Tape范围 */
    LogicalTape **memtupsl;     /* Tape数组 */
    
    /* 比较函数 */
    SortSupportData sortKey;
    
    /* 统计 */
    int64       spaceUsed;      /* 已用空间 */
} Tuplesortstate;

/* 排序状态 */
typedef enum
{
    TSS_INITIAL,                /* 初始状态 */
    TSS_BUILDRUNS,              /* 构建runs */
    TSS_SORTEDINMEM,            /* 内存排序完成 */
    TSS_SORTEDONTAPE,           /* 磁盘排序完成 */
    TSS_FINALMERGE              /* 最终归并 */
} TupSortStatus;

/*
 * tuplesort_begin_heap - 开始排序
 */
Tuplesortstate *
tuplesort_begin_heap(TupleDesc tupDesc,
                    int nkeys,
                    AttrNumber *attNums,
                    Oid *sortOperators,
                    Oid *sortCollations,
                    bool *nullsFirstFlags,
                    int workMem,
                    SortCoordinate coordinate,
                    bool randomAccess)
{
    Tuplesortstate *state;
    
    /*
     * 分配state
     */
    state = (Tuplesortstate *) palloc0(sizeof(Tuplesortstate));
    
    state->status = TSS_INITIAL;
    state->randomAccess = randomAccess;
    
    /*
     * 设置内存限制
     */
    state->allowedMem = workMem * 1024L;
    state->availMem = state->allowedMem;
    
    /*
     * 分配初始内存数组
     */
    state->memtupsize = 1024;
    state->memtuples = (SortTuple *) palloc(
        state->memtupsize * sizeof(SortTuple));
    
    /*
     * 设置比较函数
     */
    PrepareSortSupportFromOrderingOp(attNums[0],
                                    sortOperators[0],
                                    &state->sortKey);
    
    return state;
}

/*
 * tuplesort_puttupleslot - 插入tuple
 */
void
tuplesort_puttupleslot(Tuplesortstate *state,
                      TupleTableSlot *slot)
{
    MemoryContext oldcontext = MemoryContextSwitchTo(state->sortcontext);
    SortTuple   stup;
    
    /*
     * 复制tuple
     */
    stup.tuple = ExecCopySlotMinimalTuple(slot);
    
    /*
     * 计算排序key
     */
    stup.datum1 = slot_getattr(slot,
                               state->sortKey.ssup_attno,
                               &stup.isnull1);
    
    /*
     * 插入到内存数组
     */
    if (state->memtupcount >= state->memtupsize)
    {
        /*
         * 数组满了，扩容
         */
        grow_memtuples(state);
    }
    
    state->memtuples[state->memtupcount++] = stup;
    
    /*
     * 检查内存使用
     */
    state->availMem -= GetMemoryChunkSpace(stup.tuple);
    
    if (state->availMem < 0)
    {
        /*
         * ❌ 内存不足!
         * 需要溢出到磁盘
         */
        dumptuples(state, false);
    }
    
    MemoryContextSwitchTo(oldcontext);
}

/*
 * tuplesort_performsort - 执行排序
 */
void
tuplesort_performsort(Tuplesortstate *state)
{
    switch (state->status)
    {
        case TSS_INITIAL:
            /*
             * ═══════════════════════════════════════
             *  内存排序 (数据fit in memory)
             * ═══════════════════════════════════════
             */
            
            /*
             * 使用qsort排序内存数组
             */
            tuplesort_sort_memtuples(state);
            
            state->status = TSS_SORTEDINMEM;
            state->current = 0;
            break;
        
        case TSS_BUILDRUNS:
            /*
             * ═══════════════════════════════════════
             *  外部排序 (数据不fit in memory)
             * ═══════════════════════════════════════
             */
            
            /*
             * 完成最后一个run
             */
            dumptuples(state, true);
            
            /*
             * 开始归并所有runs
             */
            mergeruns(state);
            
            state->status = TSS_SORTEDONTAPE;
            break;
        
        default:
            elog(ERROR, "invalid tuplesort state");
    }
}

/*
 * tuplesort_sort_memtuples - 内存排序
 */
static void
tuplesort_sort_memtuples(Tuplesortstate *state)
{
    /*
     * 使用qsort with 3-way partitioning
     * 针对有很多重复值优化
     */
    qsort_tuple_comparator comparator;
    
    comparator = state->comparator;
    
    /*
     * 调用qsort
     */
    qsort_arg(state->memtuples,
             state->memtupcount,
             sizeof(SortTuple),
             comparator,
             state);
}

/*
 * dumptuples - 将内存tuples写入磁盘
 */
static void
dumptuples(Tuplesortstate *state, bool alltuples)
{
    int         i;
    
    /*
     * 排序内存中的tuples
     */
    tuplesort_sort_memtuples(state);
    
    /*
     * 分配新tape
     */
    LogicalTape *tape = LogicalTapeCreate(state->tapeset);
    
    /*
     * 写入tape
     */
    for (i = 0; i < state->memtupcount; i++)
    {
        writetup(state, tape, &state->memtuples[i]);
    }
    
    /*
     * 记录run
     */
    state->currentRun++;
    
    /*
     * 重置内存
     */
    state->memtupcount = 0;
    state->availMem = state->allowedMem;
}

/*
 * mergeruns - 归并所有runs
 */
static void
mergeruns(Tuplesortstate *state)
{
    int         numInputTapes = state->currentRun;
    int         numTapes;
    
    /*
     * 多路归并
     * 使用堆选择最小值
     */
    
    while (numInputTapes > 1)
    {
        /*
         * 一次归并pass
         * 每次将多个tapes归并成更少的tapes
         */
        int tapenum;
        
        for (tapenum = 0; tapenum < numInputTapes; tapenum += state->maxTapes)
        {
            /*
             * 归并最多maxTapes个tapes
             */
            mergeonerun(state, 
                       tapenum,
                       Min(tapenum + state->maxTapes, numInputTapes));
        }
        
        numInputTapes = (numInputTapes + state->maxTapes - 1) / state->maxTapes;
    }
}
```

---

## 性能对比总结

```
┌──────────────────────────────────────────────────────────┐
│ 算法          │ 时间复杂度│ 空间复杂度│ 适用场景        │
├──────────────────────────────────────────────────────────┤
│ Plain Agg     │ O(N)      │ O(1)      │ 无GROUP BY      │
│ Hash Agg      │ O(N)      │ O(groups) │ GROUP BY        │
│ Group Agg     │ O(N)      │ O(1)      │ 已排序          │
│ Sort          │ O(N logN) │ O(work_mem)│ ORDER BY        │
└──────────────────────────────────────────────────────────┘

【选择建议】
Group BY选择:
  ├─ groups数量 < work_mem
  │  └─→ Hash Aggregate (最快)
  │
  └─ groups数量 > work_mem
     ├─ 数据已排序? → Group Aggregate
     └─ 数据未排序? → Hash Aggregate (spill to disk)

ORDER BY处理:
  └─ work_mem足够? 
     ├─ 是 → In-Memory Sort
     └─ 否 → External Merge Sort
```

---

**文档版本**: PostgreSQL 17.6  
**创建日期**: 2025-10-17  
**重要程度**: ⭐⭐⭐⭐⭐

