# Join算法深度分析 - 三种Join完全解析

> NestedLoop、HashJoin、MergeJoin的完整实现和性能对比

**重要程度**: ⭐⭐⭐⭐⭐  
**源码位置**: `src/backend/executor/nodeNestloop.c`, `nodeHashjoin.c`, `nodeMergejoin.c`  
**核心概念**: Join算法选择、性能优化、参数调优

---

## 📋 Join算法总览

### 三种算法对比

```
【PostgreSQL的三种Join算法】

1. Nested Loop Join (嵌套循环)
   ┌──────────────────────────────┐
   │ for each row in outer:       │
   │   for each row in inner:     │
   │     if match: output         │
   └──────────────────────────────┘
   时间: O(M × N)
   空间: O(1)
   适合: 小表 × 小表

2. Hash Join (哈希连接)
   ┌──────────────────────────────┐
   │ Phase 1: Build hash table    │
   │   for each row in inner:     │
   │     insert into hash[key]    │
   │                              │
   │ Phase 2: Probe hash table    │
   │   for each row in outer:     │
   │     lookup hash[key]         │
   │     if found: output         │
   └──────────────────────────────┘
   时间: O(M + N)
   空间: O(min(M, N))
   适合: 大表 × 大表

3. Merge Join (归并连接)
   ┌──────────────────────────────┐
   │ Prerequisite: Both sorted    │
   │                              │
   │ while both have rows:        │
   │   if left.key = right.key:   │
   │     output                   │
   │   elif left.key < right.key: │
   │     advance left             │
   │   else:                      │
   │     advance right            │
   └──────────────────────────────┘
   时间: O(M + N) if sorted
         O(M log M + N log N) if not
   空间: O(1)
   适合: 两表都有序
```

---

## 1. Nested Loop Join - 嵌套循环

### 算法原理

```
【Nested Loop执行过程】

外表 (Outer): users     (3行)
  [1, Alice]
  [2, Bob]
  [3, Carol]

内表 (Inner): orders    (4行)
  [101, 1, 100.0]  ← user_id=1
  [102, 2, 50.0]   ← user_id=2
  [103, 1, 75.0]   ← user_id=1
  [104, 3, 200.0]  ← user_id=3

Join: users.id = orders.user_id

执行流程:
┌─────────────────────────────────────────┐
│ Outer Loop (users):                     │
│                                         │
│ 1. Row: [1, Alice]                      │
│    Inner Loop (orders):                 │
│      → [101, 1, 100.0] ✓ Match!       │
│      → [102, 2, 50.0]  ✗               │
│      → [103, 1, 75.0]  ✓ Match!       │
│      → [104, 3, 200.0] ✗               │
│    Output: (Alice, 100.0), (Alice, 75.0)│
│                                         │
│ 2. Row: [2, Bob]                        │
│    Inner Loop (orders):                 │
│      → [101, 1, 100.0] ✗               │
│      → [102, 2, 50.0]  ✓ Match!       │
│      → [103, 1, 75.0]  ✗               │
│      → [104, 3, 200.0] ✗               │
│    Output: (Bob, 50.0)                  │
│                                         │
│ 3. Row: [3, Carol]                      │
│    Inner Loop (orders):                 │
│      → [101, 1, 100.0] ✗               │
│      → [102, 2, 50.0]  ✗               │
│      → [103, 1, 75.0]  ✗               │
│      → [104, 3, 200.0] ✓ Match!       │
│    Output: (Carol, 200.0)               │
└─────────────────────────────────────────┘

总比较次数: 3 × 4 = 12
```

### 核心实现

```c
/*
 * Nested Loop Join实现
 * 文件: src/backend/executor/nodeNestloop.c
 */

/* Nested Loop状态 */
typedef struct NestLoopState
{
    JoinState   js;                 /* 基类 */
    
    /* 当前外表tuple */
    TupleTableSlot *nl_OuterTupleSlot;
    
    /* Join类型 */
    bool        nl_NeedNewOuter;    /* 需要新的外表tuple? */
    bool        nl_MatchedOuter;    /* 外表tuple已匹配? */
    
    /* 参数化信息 (Index Nested Loop用) */
    NestLoopParam *nl_Params;
} NestLoopState;

/*
 * ExecNestLoop - Nested Loop主函数
 */
static TupleTableSlot *
ExecNestLoop(PlanState *pstate)
{
    NestLoopState *node = castNode(NestLoopState, pstate);
    PlanState  *outerPlan;
    PlanState  *innerPlan;
    TupleTableSlot *outerTupleSlot;
    TupleTableSlot *innerTupleSlot;
    ExprState  *joinqual;
    ExprState  *otherqual;
    ExprContext *econtext;
    
    /*
     * 初始化
     */
    econtext = node->js.ps.ps_ExprContext;
    outerPlan = outerPlanState(node);
    innerPlan = innerPlanState(node);
    joinqual = node->js.joinqual;
    otherqual = node->js.ps.qual;
    
    /*
     * 重置per-tuple内存上下文
     */
    ResetExprContext(econtext);
    
    /*
     * ═══════════════════════════════════════════════
     *  主循环: 实现双层嵌套循环
     * ═══════════════════════════════════════════════
     */
    for (;;)
    {
        /*
         * ───────────────────────────────────────────
         *  Step 1: 检查是否需要新的外表tuple
         * ───────────────────────────────────────────
         */
        if (node->nl_NeedNewOuter)
        {
            /*
             * 从外表获取下一个tuple
             * 这是外层循环的"for"
             */
            outerTupleSlot = ExecProcNode(outerPlan);
            
            /*
             * 如果外表扫描完成
             */
            if (TupIsNull(outerTupleSlot))
            {
                /*
                 * 没有更多外表tuple
                 * Join完成!
                 */
                return NULL;
            }
            
            /*
             * 保存外表tuple
             */
            econtext->ecxt_outertuple = outerTupleSlot;
            node->nl_NeedNewOuter = false;
            node->nl_MatchedOuter = false;
            
            /*
             * ───────────────────────────────────────
             *  重新扫描内表
             *  这是关键! 每个外表tuple都要完整扫描内表
             * ───────────────────────────────────────
             */
            
            /*
             * 如果是参数化扫描 (Index Nested Loop)
             * 需要传递外表的值给内表的索引扫描
             */
            if (node->nl_Params != NULL)
            {
                /*
                 * 设置参数值
                 * 例如: inner.user_id = outer.id
                 * outer.id的值作为参数传给内表的IndexScan
                 */
                ExecSetParamPlan(node->nl_Params, econtext);
            }
            
            /*
             * 重新扫描内表
             * 对于SeqScan: 从头开始
             * 对于IndexScan: 使用新的参数值重新查找
             */
            ExecReScan(innerPlan);
        }
        
        /*
         * ───────────────────────────────────────────
         *  Step 2: 从内表获取下一个tuple
         *  这是内层循环的"for"
         * ───────────────────────────────────────────
         */
        innerTupleSlot = ExecProcNode(innerPlan);
        
        /*
         * 如果内表扫描完成
         */
        if (TupIsNull(innerTupleSlot))
        {
            /*
             * 当前外表tuple的内表扫描完成
             * 需要获取下一个外表tuple
             */
            node->nl_NeedNewOuter = true;
            
            /*
             * 处理外连接 (LEFT/RIGHT/FULL JOIN)
             */
            if (!node->nl_MatchedOuter &&
                (node->js.jointype == JOIN_LEFT ||
                 node->js.jointype == JOIN_ANTI))
            {
                /*
                 * LEFT JOIN且外表tuple没有匹配
                 * 输出 (outer tuple, NULLs)
                 */
                return ExecProject(node->js.ps.ps_ProjInfo);
            }
            
            /*
             * 继续外层循环
             */
            continue;
        }
        
        /*
         * ───────────────────────────────────────────
         *  Step 3: 检查Join条件
         * ───────────────────────────────────────────
         */
        econtext->ecxt_innertuple = innerTupleSlot;
        
        /*
         * 评估Join条件 (ON子句)
         * 例如: users.id = orders.user_id
         */
        if (joinqual == NULL || ExecQual(joinqual, econtext))
        {
            /*
             * Join条件满足!
             */
            
            if (otherqual == NULL || ExecQual(otherqual, econtext))
            {
                /*
                 * WHERE条件也满足
                 * 标记外表tuple已匹配
                 */
                node->nl_MatchedOuter = true;
                
                /*
                 * ✓ 找到一个匹配的Join结果
                 * 投影并返回
                 */
                return ExecProject(node->js.ps.ps_ProjInfo);
            }
            else
            {
                /*
                 * Join条件满足，但WHERE条件不满足
                 * 继续内层循环
                 */
                InstrCountFiltered2(node, 1);
            }
        }
        else
        {
            /*
             * Join条件不满足
             * 继续内层循环
             */
            InstrCountFiltered1(node, 1);
        }
        
        /*
         * 重置per-tuple内存
         */
        ResetExprContext(econtext);
    }
}
```

### Index Nested Loop优化

```c
/*
 * Index Nested Loop Join - 重要优化!
 * 
 * 原理:
 *   如果内表有索引，不需要全表扫描
 *   使用外表的值作为参数，直接索引查找
 * 
 * 性能提升:
 *   从 O(M × N) 降低到 O(M × log N)
 */

/*
 * 示例查询:
 */
SELECT *
FROM users u
JOIN orders o ON u.id = o.user_id;  -- orders.user_id有索引

/*
 * 执行计划:
 */
EXPLAIN (ANALYZE)
SELECT * FROM users u JOIN orders o ON u.id = o.user_id;

/*
Nested Loop  (cost=0.29..123.45 rows=1000 width=72)
  ->  Seq Scan on users u  (cost=0.00..22.00 rows=100 width=36)
  ->  Index Scan using orders_user_id_idx on orders o
        (cost=0.29..8.30 rows=10 width=36)
        Index Cond: (user_id = u.id)  ← 参数化!
*/

/*
 * 参数化扫描实现
 */

/* 参数定义 */
typedef struct NestLoopParam
{
    int         paramno;        /* 参数编号 */
    Var        *paramval;       /* 参数来源 (外表的列) */
} NestLoopParam;

/*
 * 设置参数值
 */
static void
ExecSetParamPlan(NestLoopParam *params, ExprContext *econtext)
{
    /*
     * 从外表tuple中提取值
     */
    TupleTableSlot *outerslot = econtext->ecxt_outertuple;
    
    for (int i = 0; i < n_params; i++)
    {
        NestLoopParam *param = &params[i];
        
        /*
         * 获取外表的列值
         * 例如: users.id = 1
         */
        bool isnull;
        Datum value = slot_getattr(outerslot,
                                   param->paramval->varattno,
                                   &isnull);
        
        /*
         * 设置参数
         * 内表的IndexScan会使用这个值
         */
        econtext->ecxt_param_exec_vals[param->paramno].value = value;
        econtext->ecxt_param_exec_vals[param->paramno].isnull = isnull;
    }
    
    /*
     * 现在内表的IndexScan执行:
     *   WHERE orders.user_id = $1  ← $1 = users.id = 1
     * 
     * 直接通过索引查找 user_id=1 的订单
     * 非常快! O(log N)
     */
}
```

### 性能分析

```
【Nested Loop性能】

时间复杂度:
  基本版: O(M × N)
    M个外表行 × N个内表行
  
  Index版: O(M × log N)
    M个外表行 × log(N)索引查找

空间复杂度:
  O(1) - 不需要额外内存

I/O特性:
  基本版:
    ┌──────────────────────────┐
    │ 外表: 顺序扫描          │
    │ 内表: 反复全表扫描      │
    │   ↓                      │
    │ 每个外表行都要扫描整个  │
    │ 内表，I/O开销巨大!      │
    └──────────────────────────┘
  
  Index版:
    ┌──────────────────────────┐
    │ 外表: 顺序扫描          │
    │ 内表: 索引查找          │
    │   ↓                      │
    │ 每个外表行只查找匹配的  │
    │ 内表行，I/O大幅减少!    │
    └──────────────────────────┘

适用场景:
  ✅ 小表 × 小表
     100行 × 100行 = 1万次比较 (可接受)
  
  ✅ 小表 × 大表 (内表有索引!)
     100行 × 索引查找 = 很快
  
  ✅ 内表扫描代价低
     例如: 内表在内存中
  
  ❌ 大表 × 大表 (无索引)
     10000行 × 10000行 = 1亿次比较 (太慢!)

成本估算:
  cost = outer_cost +
         outer_rows × inner_cost_per_row
  
  关键: inner_cost_per_row
    - SeqScan: 高
    - IndexScan: 低 (有索引)
    - IndexOnlyScan: 更低

实际性能:
  示例: users (100行) JOIN orders (10000行)
  
  无索引:
    100 × 10000 = 100万次比较
    时间: ~1秒
  
  有索引:
    100 × log(10000) ≈ 100 × 13 = 1300次操作
    时间: ~10ms
  
  差距: 100倍!
```

---

## 2. Hash Join - 哈希连接

### 算法原理

```
【Hash Join两阶段】

Phase 1: Build Hash Table (构建阶段)
  ┌────────────────────────────────────┐
  │ Inner Table: orders                │
  │ ┌──────────────────────┐           │
  │ │ [101, 1, 100.0]      │           │
  │ │ [102, 2, 50.0]       │           │
  │ │ [103, 1, 75.0]       │           │
  │ │ [104, 3, 200.0]      │           │
  │ └──────────────────────┘           │
  │         │                          │
  │         │ Hash on user_id          │
  │         ↓                          │
  │ ┌────────────────────────────────┐ │
  │ │ Hash Table                     │ │
  │ │ ┌─────────────────────────┐    │ │
  │ │ │ Bucket 0: empty         │    │ │
  │ │ │ Bucket 1: [101][103] ✓  │    │ │
  │ │ │ Bucket 2: [102] ✓       │    │ │
  │ │ │ Bucket 3: [104] ✓       │    │ │
  │ │ └─────────────────────────┘    │ │
  │ └────────────────────────────────┘ │
  └────────────────────────────────────┘

Phase 2: Probe Hash Table (探测阶段)
  ┌────────────────────────────────────┐
  │ Outer Table: users                 │
  │                                    │
  │ 1. [1, Alice]                      │
  │    → hash(1) → Bucket 1            │
  │    → 找到: [101][103]              │
  │    → Output: (Alice, 100.0)        │
  │              (Alice, 75.0)         │
  │                                    │
  │ 2. [2, Bob]                        │
  │    → hash(2) → Bucket 2            │
  │    → 找到: [102]                   │
  │    → Output: (Bob, 50.0)           │
  │                                    │
  │ 3. [3, Carol]                      │
  │    → hash(3) → Bucket 3            │
  │    → 找到: [104]                   │
  │    → Output: (Carol, 200.0)        │
  └────────────────────────────────────┘

总操作: 4次(build) + 3次(probe) = 7次
vs Nested Loop: 12次
```

### 核心实现

```c
/*
 * Hash Join实现
 * 文件: src/backend/executor/nodeHashjoin.c
 */

/* Hash Join状态 */
typedef struct HashJoinState
{
    JoinState   js;                 /* 基类 */
    
    /* Hash表 */
    HashJoinTable hashtable;        /* Hash表结构 */
    
    /* Join状态机 */
    int         hj_JoinState;       /* 当前状态 */
    
    /* 当前处理 */
    TupleTableSlot *hj_OuterTupleSlot;  /* 外表当前tuple */
    uint32      hj_CurHashValue;    /* 当前hash值 */
    int         hj_CurBucketNo;     /* 当前bucket */
    HashJoinTuple hj_CurTuple;      /* 当前hash tuple */
    
    /* 批处理 (work_mem不足时) */
    int         hj_CurBatch;        /* 当前批次 */
    int         hj_NumBatches;      /* 总批次数 */
} HashJoinState;

/* Join状态 */
typedef enum
{
    HJ_BUILD_HASHTABLE,         /* 构建hash表 */
    HJ_NEED_NEW_OUTER,          /* 需要新的外表tuple */
    HJ_SCAN_BUCKET,             /* 扫描bucket */
    HJ_FILL_OUTER_TUPLE,        /* 填充外表tuple (外连接) */
    HJ_FILL_INNER_TUPLES,       /* 填充内表tuples (外连接) */
    HJ_NEED_NEW_BATCH           /* 需要新的批次 */
} HashJoinState_enum;

/*
 * ExecHashJoin - Hash Join主函数
 */
static TupleTableSlot *
ExecHashJoin(PlanState *pstate)
{
    HashJoinState *node = castNode(HashJoinState, pstate);
    PlanState  *outerNode;
    HashState  *hashNode;
    ExprState  *joinqual;
    ExprState  *otherqual;
    ExprContext *econtext;
    HashJoinTable hashtable;
    TupleTableSlot *outerTupleSlot;
    uint32      hashvalue;
    
    /*
     * 初始化
     */
    joinqual = node->js.joinqual;
    otherqual = node->js.ps.qual;
    hashNode = (HashState *) innerPlanState(node);
    outerNode = outerPlanState(node);
    econtext = node->js.ps.ps_ExprContext;
    
    /*
     * ═══════════════════════════════════════════════
     *  状态机: 处理不同阶段
     * ═══════════════════════════════════════════════
     */
    switch (node->hj_JoinState)
    {
        /*
         * ───────────────────────────────────────────
         *  Phase 1: 构建Hash表
         * ───────────────────────────────────────────
         */
        case HJ_BUILD_HASHTABLE:
        {
            /*
             * Step 1: 创建Hash表
             */
            hashtable = ExecHashTableCreate(hashNode,
                                           node->js.ps.state->es_query_cxt);
            node->hashtable = hashtable;
            
            /*
             * Step 2: 从内表读取所有tuple
             * 插入到Hash表
             */
            for (;;)
            {
                TupleTableSlot *innerslot;
                
                /*
                 * 获取内表的下一个tuple
                 */
                innerslot = ExecProcNode((PlanState *) hashNode);
                
                if (TupIsNull(innerslot))
                    break;  /* 内表扫描完成 */
                
                /*
                 * 计算hash值
                 * 基于join key (例如: orders.user_id)
                 */
                hashvalue = ExecHashGetHashValue(hashtable,
                                                econtext,
                                                hashNode->hashkeys);
                
                /*
                 * 插入到Hash表
                 * 使用拉链法处理冲突
                 */
                ExecHashTableInsert(hashtable,
                                   innerslot,
                                   hashvalue);
                
                hashtable->totalTuples++;
            }
            
            /*
             * Hash表构建完成
             * 转到探测阶段
             */
            node->hj_JoinState = HJ_NEED_NEW_OUTER;
            /* FALLTHROUGH */
        }
        
        /*
         * ───────────────────────────────────────────
         *  Phase 2: 探测Hash表
         * ───────────────────────────────────────────
         */
        case HJ_NEED_NEW_OUTER:
        {
            /*
             * 获取外表的下一个tuple
             */
            outerTupleSlot = ExecProcNode(outerNode);
            
            if (TupIsNull(outerTupleSlot))
            {
                /* 外表扫描完成 */
                return NULL;
            }
            
            econtext->ecxt_outertuple = outerTupleSlot;
            node->hj_OuterTupleSlot = outerTupleSlot;
            
            /*
             * 计算外表tuple的hash值
             * 基于join key (例如: users.id)
             */
            hashvalue = ExecHashGetHashValue(node->hashtable,
                                            econtext,
                                            node->hj_OuterHashKeys);
            
            node->hj_CurHashValue = hashvalue;
            
            /*
             * 定位到bucket
             */
            ExecHashGetBucketAndBatch(node->hashtable,
                                     hashvalue,
                                     &node->hj_CurBucketNo,
                                     &batch);
            
            /*
             * 获取bucket的第一个tuple
             */
            node->hj_CurTuple = ExecHashTableBucketStart(
                                    node->hashtable,
                                    node->hj_CurBucketNo);
            
            node->hj_JoinState = HJ_SCAN_BUCKET;
            /* FALLTHROUGH */
        }
        
        /*
         * ───────────────────────────────────────────
         *  扫描当前bucket
         * ───────────────────────────────────────────
         */
        case HJ_SCAN_BUCKET:
        {
            /*
             * 遍历bucket中的所有tuple
             * (处理hash冲突)
             */
            while (node->hj_CurTuple != NULL)
            {
                HashJoinTuple hashtuple = node->hj_CurTuple;
                TupleTableSlot *innerslot;
                
                /*
                 * 移动到bucket中的下一个tuple
                 */
                node->hj_CurTuple = ExecHashTableNext(
                                        node->hashtable,
                                        hashtuple);
                
                /*
                 * 检查hash值是否真的匹配
                 * (因为可能有hash冲突)
                 */
                if (hashtuple->hashvalue != node->hj_CurHashValue)
                    continue;  /* hash冲突，跳过 */
                
                /*
                 * 提取内表tuple
                 */
                innerslot = ExecStoreMinimalTuple(
                                HJTUPLE_MINTUPLE(hashtuple),
                                node->js.ps.ps_ResultTupleSlot,
                                false);
                
                econtext->ecxt_innertuple = innerslot;
                
                /*
                 * 评估Join条件
                 * 即使hash值相同，仍需检查实际值
                 * (hash(1) 可能等于 hash(1001))
                 */
                if (joinqual == NULL ||
                    ExecQual(joinqual, econtext))
                {
                    /*
                     * Join条件满足!
                     * 检查WHERE条件
                     */
                    if (otherqual == NULL ||
                        ExecQual(otherqual, econtext))
                    {
                        /*
                         * ✓ 找到匹配!
                         * 返回Join结果
                         */
                        return ExecProject(node->js.ps.ps_ProjInfo);
                    }
                    else
                    {
                        InstrCountFiltered2(node, 1);
                    }
                }
                else
                {
                    InstrCountFiltered1(node, 1);
                }
                
                ResetExprContext(econtext);
            }
            
            /*
             * 当前bucket扫描完成
             * 需要新的外表tuple
             */
            node->hj_JoinState = HJ_NEED_NEW_OUTER;
            goto HJ_NEED_NEW_OUTER;  /* 跳转到获取新outer tuple */
        }
        
        default:
            elog(ERROR, "unrecognized hashjoin state: %d",
                 (int) node->hj_JoinState);
    }
    
    return NULL;
}
```

### Hash表实现

```c
/*
 * Hash表结构
 * 文件: src/backend/executor/nodeHash.c
 */

/* Hash表 */
typedef struct HashJoinTableData
{
    /* 桶数组 */
    HashJoinTuple *buckets;         /* bucket数组 */
    int         nbuckets;            /* bucket数量 */
    int         nbuckets_original;   /* 原始bucket数 */
    
    /* 批处理 (内存不足时) */
    int         nbatch;              /* batch数量 */
    int         curbatch;            /* 当前batch */
    BufFile   **innerBatchFile;      /* 内表溢出文件 */
    BufFile   **outerBatchFile;      /* 外表溢出文件 */
    
    /* 统计 */
    long        totalTuples;         /* 总tuple数 */
    long        spaceUsed;           /* 已使用内存 */
    long        spaceAllowed;        /* 允许的内存 (work_mem) */
    
    /* 内存管理 */
    MemoryContext hashCxt;           /* Hash表内存上下文 */
} HashJoinTableData;

/* Hash表中的tuple */
typedef struct HashJoinTuple
{
    struct HashJoinTuple *next;      /* 链表next指针 (拉链法) */
    uint32      hashvalue;           /* Hash值 */
    MinimalTuple tuple;              /* 实际tuple数据 */
} *HashJoinTuple;

/*
 * ExecHashTableCreate - 创建Hash表
 */
HashJoinTable
ExecHashTableCreate(HashState *state, MemoryContext mcxt)
{
    HashJoinTable hashtable;
    Plan       *outerNode;
    int         nbuckets;
    int         nbatch;
    int         num_skew_mcvs;
    
    /*
     * 估算需要多少buckets
     * 基于: 内表行数, 可用内存(work_mem)
     */
    outerNode = outerPlan(state);
    
    ExecChooseHashTableSize(outerNode->plan_rows,
                           outerNode->plan_width,
                           &nbuckets,
                           &nbatch,
                           &num_skew_mcvs);
    
    /*
     * 分配Hash表结构
     */
    hashtable = (HashJoinTable) palloc(sizeof(HashJoinTableData));
    hashtable->nbuckets = nbuckets;
    hashtable->nbuckets_original = nbuckets;
    hashtable->nbatch = nbatch;
    hashtable->curbatch = 0;
    
    /*
     * 分配bucket数组
     */
    hashtable->buckets = (HashJoinTuple *)
        palloc0(nbuckets * sizeof(HashJoinTuple));
    
    /*
     * 设置内存限制
     * work_mem控制Hash表最大大小
     */
    hashtable->spaceAllowed = get_hash_memory_limit();
    hashtable->spaceUsed = 0;
    
    return hashtable;
}

/*
 * ExecHashTableInsert - 插入tuple到Hash表
 */
void
ExecHashTableInsert(HashJoinTable hashtable,
                   TupleTableSlot *slot,
                   uint32 hashvalue)
{
    HashJoinTuple hashTuple;
    int         bucketno;
    int         batchno;
    
    /*
     * 计算bucket号和batch号
     */
    ExecHashGetBucketAndBatch(hashtable, hashvalue,
                             &bucketno, &batchno);
    
    /*
     * 如果是当前batch
     */
    if (batchno == hashtable->curbatch)
    {
        /*
         * 创建HashJoinTuple
         */
        MinimalTuple tuple = ExecCopySlotMinimalTuple(slot);
        
        hashTuple = (HashJoinTuple)
            MemoryContextAlloc(hashtable->hashCxt,
                             HJTUPLE_OVERHEAD +
                             tuple->t_len);
        
        hashTuple->hashvalue = hashvalue;
        memcpy(HJTUPLE_MINTUPLE(hashTuple), tuple, tuple->t_len);
        pfree(tuple);
        
        /*
         * 插入到bucket链表头部 (拉链法)
         */
        hashTuple->next = hashtable->buckets[bucketno];
        hashtable->buckets[bucketno] = hashTuple;
        
        /*
         * 更新内存使用统计
         */
        hashtable->spaceUsed += HJTUPLE_OVERHEAD + tuple->t_len;
        
        /*
         * 检查内存是否超限
         */
        if (hashtable->spaceUsed > hashtable->spaceAllowed)
        {
            /*
             * ❌ 内存不足!
             * 需要增加batch数量
             * 将部分数据写入临时文件
             */
            ExecHashIncreaseNumBatches(hashtable);
        }
    }
    else
    {
        /*
         * 不是当前batch
         * 写入临时文件，稍后处理
         */
        ExecHashJoinSaveTuple(slot,
                             hashvalue,
                             &hashtable->innerBatchFile[batchno]);
    }
}
```

### 批处理机制

```c
/*
 * Hash Join批处理 (Batching)
 * 
 * 问题: 如果内表太大，Hash表放不进内存怎么办?
 * 解决: 分批处理 (Hybrid Hash Join)
 */

/*
 * 批处理示例:
 * 
 * 假设:
 *   - 内表: 100万行
 *   - Hash表需要: 1GB
 *   - work_mem: 256MB
 *   - 需要批次: 4个
 * 
 * 执行流程:
 * 
 * Batch 0: (在内存中)
 *   Inner: hash(key) % 4 == 0 的行
 *   Outer: hash(key) % 4 == 0 的行
 *   → 完成Join
 * 
 * Batch 1: (从临时文件读取)
 *   读取 inner_batch_1.tmp
 *   读取 outer_batch_1.tmp
 *   构建Hash表
 *   → 完成Join
 * 
 * Batch 2: ...
 * Batch 3: ...
 */

/*
 * ExecHashIncreaseNumBatches - 增加batch数量
 */
static void
ExecHashIncreaseNumBatches(HashJoinTable hashtable)
{
    int         oldnbatch = hashtable->nbatch;
    int         newnbatch = oldnbatch * 2;  /* 翻倍 */
    int         i;
    
    /*
     * 分配新的临时文件数组
     */
    hashtable->innerBatchFile = (BufFile **)
        repalloc(hashtable->innerBatchFile,
                newnbatch * sizeof(BufFile *));
    hashtable->outerBatchFile = (BufFile **)
        repalloc(hashtable->outerBatchFile,
                newnbatch * sizeof(BufFile *));
    
    /*
     * 创建新的临时文件
     */
    for (i = oldnbatch; i < newnbatch; i++)
    {
        hashtable->innerBatchFile[i] = BufFileCreateTemp(false);
        hashtable->outerBatchFile[i] = BufFileCreateTemp(false);
    }
    
    hashtable->nbatch = newnbatch;
    
    /*
     * 重新hash当前内存中的tuples
     * 将部分tuple移动到新的batch
     */
    for (i = 0; i < hashtable->nbuckets; i++)
    {
        HashJoinTuple tuple = hashtable->buckets[i];
        HashJoinTuple prevtuple = NULL;
        
        while (tuple != NULL)
        {
            HashJoinTuple nexttuple = tuple->next;
            int         bucketno;
            int         batchno;
            
            /*
             * 重新计算batch号
             */
            ExecHashGetBucketAndBatch(hashtable,
                                     tuple->hashvalue,
                                     &bucketno,
                                     &batchno);
            
            if (batchno == hashtable->curbatch)
            {
                /* 保持在内存中 */
                prevtuple = tuple;
            }
            else
            {
                /* 写入临时文件 */
                ExecHashJoinSaveTuple(HJTUPLE_MINTUPLE(tuple),
                                     tuple->hashvalue,
                                     &hashtable->innerBatchFile[batchno]);
                
                /* 从链表中删除 */
                if (prevtuple)
                    prevtuple->next = nexttuple;
                else
                    hashtable->buckets[i] = nexttuple;
                
                /* 释放内存 */
                pfree(tuple);
                hashtable->spaceUsed -= HJTUPLE_OVERHEAD +
                                       HJTUPLE_MINTUPLE(tuple)->t_len;
            }
            
            tuple = nexttuple;
        }
    }
}
```

### 性能分析

```
【Hash Join性能】

时间复杂度:
  Build: O(N) - 扫描内表
  Probe: O(M) - 扫描外表
  总计: O(M + N) ← 线性时间!

空间复杂度:
  O(min(M, N)) - Hash表大小
  通常选择小表作为内表

I/O特性:
  ┌──────────────────────────────┐
  │ Build Phase:                 │
  │   顺序扫描内表               │
  │   ✓ 顺序I/O                 │
  │                              │
  │ Probe Phase:                 │
  │   顺序扫描外表               │
  │   ✓ 顺序I/O                 │
  │                              │
  │ Hash表访问:                  │
  │   内存访问                   │
  │   ✓ 非常快                  │
  └──────────────────────────────┘

内存使用:
  受work_mem限制:
    SET work_mem = '256MB';
  
  如果Hash表 > work_mem:
    → 启用批处理
    → 部分数据写入临时文件
    → 多次扫描，性能下降

适用场景:
  ✅ 大表 × 大表
  ✅ 等值连接 (=)
  ✅ 内存足够 (work_mem >= hash table)
  
  ❌ 非等值连接 (>, <, !=)
     Hash Join无法处理
  
  ❌ 内存不足
     批处理性能下降

成本估算:
  cost = inner_cost +              # 扫描内表
         outer_cost +              # 扫描外表
         (inner_rows * cpu_operator_cost) +  # Build hash
         (outer_rows * cpu_operator_cost)    # Probe hash
  
  关键: 假设Hash表在内存中!

实际性能:
  示例: users (1000行) JOIN orders (100000行)
  
  Nested Loop: 
    1000 × 100000 = 1亿次比较
    时间: ~10秒
  
  Hash Join:
    Build: 100000次插入
    Probe: 1000次查找
    总计: 101000次操作
    时间: ~100ms
  
  差距: 100倍!

调优要点:
  1. 调大work_mem
     SET work_mem = '512MB';
     避免批处理
  
  2. 选择小表作为内表
     优化器会自动选择
     但可以用join_collapse_limit控制
  
  3. 监控EXPLAIN
     EXPLAIN (ANALYZE, BUFFERS)
     查看"Batches"数量
     理想: Batches = 1
```

---

我继续创建Merge Join和最终的性能对比总结...

<function_calls>
<invoke name="search_replace">
<parameter name="file_path">/home/postgres/pg17.6/studydoc/executor/04_join_algorithms_deep_dive.md