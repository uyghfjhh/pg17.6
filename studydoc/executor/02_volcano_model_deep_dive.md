# Executor火山模型深度分析

> PostgreSQL执行器的核心设计模式 - Iterator Model

**重要程度**: ⭐⭐⭐⭐⭐  
**源码位置**: `src/backend/executor/`  
**核心文件**: `execProcnode.c`, `execMain.c`

---

## 📋 什么是火山模型

### 基本概念

```
【火山模型 (Volcano Model / Iterator Model)】

核心思想:
  • 每个算子是一个Iterator（迭代器）
  • 实现统一接口: Next() → 返回一个元组
  • Pull-based: 上层算子向下层"拉取"数据
  • Pipeline执行: 一次处理一个元组

命名来源:
  Volcano: 最早在Volcano查询优化器中提出
  Iterator: 使用迭代器设计模式
```

### 为什么叫"火山"？

```
【形象比喻】
                  ┌─────────────┐
                  │   Result    │  ← 客户端
                  └──────┬──────┘
                         │ Next() 
                         ↑ 一个元组一个元组"喷发"出来
                  ┌──────┴──────┐
                  │   HashJoin  │  ← 像火山口
                  └──────┬──────┘
                    ┌────┴────┐
                    ↑         ↑
             ┌──────┴──┐  ┌──┴──────┐
             │ SeqScan │  │IndexScan│
             └─────────┘  └─────────┘
                ↑              ↑
             数据像岩浆一样层层向上流动
```

---

## 核心接口

### 统一的Iterator接口

```c
/*
 * PlanState - 所有执行节点的基类
 * 文件: src/include/nodes/execnodes.h
 */
typedef struct PlanState
{
    NodeTag     type;           // 节点类型标识
    
    Plan       *plan;           // 对应的计划节点
    EState     *state;          // 执行状态
    
    /*
     * 核心函数指针: 执行节点
     * 每个节点类型都有自己的实现
     */
    ExecProcNodeMtd ExecProcNode;  // 获取下一个元组
    
    /*
     * 其他关键函数指针
     */
    ExecProcNodeMtd ExecProcNodeReal;
    
    TupleTableSlot *ps_ResultTupleSlot;  // 结果元组槽
    ExprContext    *ps_ExprContext;      // 表达式上下文
    
    /*
     * 投影信息 (SELECT列表)
     */
    ProjectionInfo *ps_ProjInfo;
    
    /*
     * 过滤条件 (WHERE子句)
     */
    ExprState      *qual;
    
    /*
     * 左右子节点
     */
    struct PlanState *lefttree;   // 左子树
    struct PlanState *righttree;  // 右子树
    
    /*
     * 性能统计
     */
    Instrumentation *instrument;  // 运行时统计
    
} PlanState;

/*
 * 节点执行函数类型
 */
typedef TupleTableSlot *(*ExecProcNodeMtd)(struct PlanState *pstate);
```

### ExecProcNode - 核心分发函数

```c
/*
 * ExecProcNode - 执行一个计划节点，返回一个元组
 * 文件: src/backend/executor/execProcnode.c
 */
static inline TupleTableSlot *
ExecProcNode(PlanState *node)
{
    TupleTableSlot *result;
    
    /*
     * 检查中断信号
     * 允许Ctrl+C取消长时间运行的查询
     */
    CHECK_FOR_INTERRUPTS();
    
    /*
     * 性能统计 - 开始计时
     */
    if (node->instrument)
        InstrStartNode(node->instrument);
    
    /*
     * 调用节点特定的执行函数
     * 这是关键: 每个节点类型有自己的实现
     */
    result = node->ExecProcNode(node);
    
    /*
     * 性能统计 - 结束计时
     */
    if (node->instrument)
        InstrStopNode(node->instrument, 
                     TupIsNull(result) ? 0.0 : 1.0);
    
    return result;
}

/*
 * 初始化时设置ExecProcNode函数指针
 * 文件: src/backend/executor/execProcnode.c
 */
PlanState *
ExecInitNode(Plan *node, EState *estate, int eflags)
{
    PlanState *result = NULL;
    
    switch (nodeTag(node))
    {
        /* =================== 扫描节点 =================== */
        case T_SeqScan:
            result = (PlanState *) ExecInitSeqScan(
                (SeqScan *) node, estate, eflags);
            break;
            
        case T_IndexScan:
            result = (PlanState *) ExecInitIndexScan(
                (IndexScan *) node, estate, eflags);
            break;
            
        case T_IndexOnlyScan:
            result = (PlanState *) ExecInitIndexOnlyScan(
                (IndexOnlyScan *) node, estate, eflags);
            break;
            
        case T_BitmapHeapScan:
            result = (PlanState *) ExecInitBitmapHeapScan(
                (BitmapHeapScan *) node, estate, eflags);
            break;
            
        /* =================== Join节点 =================== */
        case T_NestLoop:
            result = (PlanState *) ExecInitNestLoop(
                (NestLoop *) node, estate, eflags);
            break;
            
        case T_HashJoin:
            result = (PlanState *) ExecInitHashJoin(
                (HashJoin *) node, estate, eflags);
            break;
            
        case T_MergeJoin:
            result = (PlanState *) ExecInitMergeJoin(
                (MergeJoin *) node, estate, eflags);
            break;
            
        /* =================== 聚合节点 =================== */
        case T_Agg:
            result = (PlanState *) ExecInitAgg(
                (Agg *) node, estate, eflags);
            break;
            
        /* =================== 排序节点 =================== */
        case T_Sort:
            result = (PlanState *) ExecInitSort(
                (Sort *) node, estate, eflags);
            break;
            
        /* =================== 其他节点 =================== */
        case T_Limit:
            result = (PlanState *) ExecInitLimit(
                (Limit *) node, estate, eflags);
            break;
            
        // ... 更多节点类型
        
        default:
            elog(ERROR, "unrecognized node type: %d", 
                 (int) nodeTag(node));
            break;
    }
    
    /*
     * 设置ExecProcNode函数指针
     * 每个Init函数会设置对应的执行函数
     */
    ExecSetExecProcNode(result, result->ExecProcNode);
    
    return result;
}
```

---

## 执行流程详解

### 完整执行过程

```
【查询执行的完整生命周期】

1. ExecutorStart() - 初始化
   ├─ CreateExecutorState()     // 创建执行状态
   ├─ InitPlan()                // 初始化计划树
   │  └─ ExecInitNode()         // 递归初始化所有节点
   │     ├─ 分配内存
   │     ├─ 打开表/索引
   │     └─ 设置ExecProcNode函数指针
   └─ 准备完成

2. ExecutorRun() - 执行查询
   ├─ 循环调用 ExecProcNode(topNode)
   │  └─ 返回一个元组
   │     ├─ topNode调用子节点的ExecProcNode()
   │     ├─ 子节点又调用孙节点的ExecProcNode()
   │     └─ 递归向下，数据向上流动
   │  
   ├─ 处理返回的元组
   │  └─ 发送给客户端
   │  
   └─ 直到没有更多元组 (返回NULL)

3. ExecutorEnd() - 清理
   ├─ ExecEndNode()             // 递归清理所有节点
   │  ├─ 关闭表/索引
   │  ├─ 释放内存
   │  └─ 收集统计信息
   └─ FreeExecutorState()       // 释放执行状态
```

### 源码实现

```c
/*
 * ExecutorStart - 初始化查询执行
 * 文件: src/backend/executor/execMain.c
 */
void
ExecutorStart(QueryDesc *queryDesc, int eflags)
{
    EState *estate;
    MemoryContext oldcontext;
    
    /*
     * 步骤1: 创建执行状态
     * EState包含查询执行的所有上下文信息
     */
    estate = CreateExecutorState();
    queryDesc->estate = estate;
    
    /*
     * 步骤2: 设置执行参数
     */
    estate->es_param_list_info = queryDesc->params;
    estate->es_snapshot = queryDesc->snapshot;
    estate->es_crosscheck_snapshot = queryDesc->crosscheck_snapshot;
    
    /*
     * 步骤3: 初始化计划树
     * 这会递归初始化所有节点
     */
    queryDesc->planstate = ExecInitNode(queryDesc->plannedstmt->planTree,
                                       estate,
                                       eflags);
    
    /*
     * 步骤4: 初始化目标列表投影
     */
    ExecAssignExprContext(estate, &queryDesc->planstate->ps_ExprContext);
    
    /*
     * 步骤5: 打开结果relation (如果是INSERT/UPDATE/DELETE)
     */
    if (queryDesc->operation != CMD_SELECT)
    {
        InitResultRelInfo(estate, ...);
    }
}

/*
 * ExecutorRun - 执行查询，返回元组
 * 文件: src/backend/executor/execMain.c
 */
void
ExecutorRun(QueryDesc *queryDesc,
           ScanDirection direction,
           uint64 count,
           bool execute_once)
{
    EState     *estate;
    CmdType     operation;
    DestReceiver *dest;
    
    estate = queryDesc->estate;
    operation = queryDesc->operation;
    dest = queryDesc->dest;
    
    /*
     * 启动目标接收器
     */
    dest->rStartup(dest, operation, queryDesc->tupDesc);
    
    /*
     * 执行计划
     */
    if (queryDesc->plannedstmt->hasReturning ||
        operation == CMD_SELECT)
    {
        /*
         * SELECT查询 或 带RETURNING的DML
         * 使用ExecutePlan循环获取元组
         */
        ExecutePlan(estate,
                   queryDesc->planstate,
                   queryDesc->plannedstmt->parallelModeNeeded,
                   operation,
                   true,  /* sendTuples */
                   count,
                   direction,
                   dest,
                   execute_once);
    }
    else
    {
        /*
         * INSERT/UPDATE/DELETE (无RETURNING)
         */
        ExecutePlan(estate, queryDesc->planstate, ...);
    }
    
    /*
     * 关闭目标接收器
     */
    dest->rShutdown(dest);
}

/*
 * ExecutePlan - 核心执行循环
 * 文件: src/backend/executor/execMain.c
 */
static void
ExecutePlan(EState *estate,
           PlanState *planstate,
           bool use_parallel_mode,
           CmdType operation,
           bool sendTuples,
           uint64 numberTuples,
           ScanDirection direction,
           DestReceiver *dest,
           bool execute_once)
{
    TupleTableSlot *slot;
    uint64 current_tuple_count;
    
    /*
     * 初始化计数器
     */
    current_tuple_count = 0;
    
    /*
     * 主循环: 不断获取元组
     * 这是火山模型的核心循环!
     */
    for (;;)
    {
        /*
         * 重置per-tuple内存上下文
         * 避免内存泄漏
         */
        ResetPerTupleExprContext(estate);
        
        /*
         * ═══════════════════════════════════════
         *  关键调用: 获取下一个元组
         *  这会触发整个执行树的递归调用
         * ═══════════════════════════════════════
         */
        slot = ExecProcNode(planstate);
        
        /*
         * 检查是否完成
         */
        if (TupIsNull(slot))
        {
            /*
             * 没有更多元组了
             * 可能原因:
             * 1. 表扫描完成
             * 2. Join没有更多匹配
             * 3. Limit达到限制
             */
            break;
        }
        
        /*
         * 发送元组给目标接收器
         * (通常是客户端)
         */
        if (sendTuples)
        {
            /*
             * 调用dest的receiveSlot回调
             * 对于客户端查询，这会通过网络发送元组
             */
            if (!dest->receiveSlot(slot, dest))
                break;  // 客户端不想要更多数据
        }
        
        /*
         * 计数
         */
        current_tuple_count++;
        
        /*
         * 检查是否达到LIMIT
         */
        if (numberTuples && numberTuples == current_tuple_count)
            break;
    }
}

/*
 * ExecutorEnd - 清理执行器状态
 * 文件: src/backend/executor/execMain.c
 */
void
ExecutorEnd(QueryDesc *queryDesc)
{
    EState *estate;
    
    estate = queryDesc->estate;
    
    /*
     * 关闭结果relation
     */
    ExecCloseResultRelations(estate);
    
    /*
     * 递归关闭所有节点
     * 释放资源
     */
    ExecEndNode(queryDesc->planstate);
    
    /*
     * 关闭所有打开的relation
     */
    ExecCloseRangeTableRelations(estate);
    
    /*
     * 释放执行状态内存
     */
    FreeExecutorState(estate);
    
    /*
     * 清空QueryDesc中的引用
     */
    queryDesc->estate = NULL;
    queryDesc->planstate = NULL;
}
```

---

## 执行示例

### 简单查询的执行流程

```sql
-- 示例查询
SELECT name, age 
FROM users 
WHERE age > 18 
ORDER BY age;
```

```
【执行计划树】
                Sort
                 │
              SeqScan (users)
                 │
           (Filter: age > 18)

【执行流程 - 火山模型】

步骤1: ExecutorStart()
  ├─ 初始化Sort节点
  │  └─ 分配tuplesort内存
  │
  └─ 初始化SeqScan节点
     ├─ 打开users表
     └─ 准备heap扫描

步骤2: ExecutorRun() - 第1次调用ExecProcNode(Sort)
  │
  ├─ Sort节点: "我需要所有数据才能排序"
  │  └─ 循环调用 ExecProcNode(SeqScan)
  │     │
  │     ├─ SeqScan返回: (Alice, 25) ✓ age>18
  │     ├─ SeqScan返回: (Bob, 15)   ✗ age<=18, 过滤掉
  │     ├─ SeqScan返回: (Carol, 30) ✓ age>18
  │     ├─ SeqScan返回: (Dave, 20)  ✓ age>18
  │     └─ SeqScan返回: NULL (扫描完成)
  │
  ├─ Sort节点收集到3个元组:
  │  (Alice, 25), (Carol, 30), (Dave, 20)
  │
  └─ Sort节点执行排序:
     (Dave, 20), (Alice, 25), (Carol, 30)

步骤3: ExecutorRun() - 后续调用ExecProcNode(Sort)
  │
  ├─ 第2次: Sort返回 (Dave, 20)   → 发送给客户端
  ├─ 第3次: Sort返回 (Alice, 25)  → 发送给客户端
  ├─ 第4次: Sort返回 (Carol, 30)  → 发送给客户端
  └─ 第5次: Sort返回 NULL         → 完成

步骤4: ExecutorEnd()
  ├─ 清理Sort节点 (释放tuplesort内存)
  └─ 清理SeqScan节点 (关闭users表)
```

### 复杂查询的执行流程

```sql
-- 复杂查询: Join + 聚合
SELECT u.city, COUNT(*), AVG(o.amount)
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.age > 18
GROUP BY u.city;
```

```
【执行计划树】
            HashAggregate
            (GROUP BY city)
                 │
              HashJoin
              (u.id = o.user_id)
            ┌─────┴─────┐
         SeqScan      IndexScan
         (users)      (orders)
      (Filter: age>18)

【执行流程 - 递归调用链】

ExecProcNode(HashAggregate) ← 顶层调用
  │
  └─→ 需要所有输入，循环调用子节点:
      │
      ExecProcNode(HashJoin) ← HashAggregate调用
        │
        ├─→ Phase 1: Build哈希表
        │   │
        │   └─→ 循环调用: ExecProcNode(SeqScan on users)
        │       ├─ 返回: (1, Alice, NYC, 25) ✓
        │       ├─ 返回: (2, Bob, LA, 15)    ✗ 过滤
        │       ├─ 返回: (3, Carol, NYC, 30) ✓
        │       └─ 返回: NULL
        │       
        │       哈希表构建完成:
        │       Hash[1] = (1, Alice, NYC, 25)
        │       Hash[3] = (3, Carol, NYC, 30)
        │
        └─→ Phase 2: Probe哈希表
            │
            └─→ 循环调用: ExecProcNode(IndexScan on orders)
                ├─ 返回: (101, 1, 50.0)  → Join成功 → (Alice, NYC, 50.0)
                ├─ 返回: (102, 3, 75.0)  → Join成功 → (Carol, NYC, 75.0)
                ├─ 返回: (103, 1, 30.0)  → Join成功 → (Alice, NYC, 30.0)
                └─ 返回: NULL
  │
  └─→ HashAggregate收集到Join结果:
      (Alice, NYC, 50.0)
      (Carol, NYC, 75.0)
      (Alice, NYC, 30.0)
      
      按city分组聚合:
      NYC: COUNT=3, AVG=51.67
      
      返回: (NYC, 3, 51.67)

【调用栈可视化】
ExecutePlan()
  └─ ExecProcNode(HashAggregate)
      └─ ExecProcNode(HashJoin)
          ├─ ExecProcNode(SeqScan users)    ← 最底层
          │   └─ heap_getnext()             ← 访问存储
          │
          └─ ExecProcNode(IndexScan orders) ← 最底层
              └─ index_getnext_tid()        ← 访问索引
```

---

## 火山模型的优缺点

### 优点

```
✅ 1. 简单优雅
   - 统一的Iterator接口
   - 每个算子独立实现
   - 易于理解和维护

✅ 2. 组合灵活
   - 算子可以任意组合
   - 新增算子容易
   - 计划树构造简单

✅ 3. Pipeline执行
   - 流水线处理
   - 内存占用小 (一次一个元组)
   - 支持LIMIT优化 (提前终止)

✅ 4. 控制流清晰
   - Pull-based模型
   - 控制流自然 (递归调用)
   - 易于调试

示例: LIMIT优化
  SELECT * FROM users LIMIT 10;
  
  SeqScan只需要扫描10个元组就停止
  不需要扫描整个表!
```

### 缺点

```
❌ 1. 函数调用开销
   - 每个元组都需要函数调用
   - 虚函数调用代价高
   - CPU分支预测困难

   示例:
   扫描100万行:
   - ExecProcNode() 调用: 100万次
   - 每次调用: ~5-10ns开销
   - 总开销: 5-10ms (不可忽视!)

❌ 2. Cache不友好
   - 每个元组单独处理
   - 无法利用SIMD
   - 数据局部性差

   对比:
   火山模型: 处理100行 → 100次函数调用
   向量化:   处理100行 → 1次调用 + SIMD

❌ 3. 难以向量化
   - 一次一个元组的设计
   - 无法批量处理
   - CPU利用率低

❌ 4. 中间结果物化
   - Sort/HashAgg需要全部数据
   - 无法真正pipeline
   - 内存压力大
```

---

## PostgreSQL的优化

### 1. JIT编译优化

```c
/*
 * PostgreSQL 11+ 支持JIT (Just-In-Time)编译
 * 减少ExecProcNode的开销
 */

-- 启用JIT
SET jit = on;
SET jit_above_cost = 100000;

-- 查看JIT效果
EXPLAIN (ANALYZE, BUFFERS) 
SELECT SUM(amount) FROM large_table WHERE amount > 100;

/*
Planning Time: 0.123 ms
JIT:
  Functions: 3
  Options: Inlining true, Optimization true, ...
  Timing: Generation 1.234 ms, Inlining 0.456 ms, ...
Execution Time: 1234.567 ms
*/

【JIT优化原理】
传统执行:
  for each tuple:
    ExecProcNode()  ← 函数调用
      ExecQual()    ← 函数调用
        evaluate expression ← 函数调用

JIT编译:
  编译期: 生成机器码
  for each tuple:
    [直接执行机器码] ← 无函数调用!
```

### 2. 表达式预编译

```c
/*
 * ExprState - 预编译的表达式
 * 文件: src/include/nodes/execnodes.h
 */

/*
 * 传统方式: 每次evaluate都要解释表达式树
 */
Datum eval_expression(Expr *expr) {
    switch (expr->type) {
        case T_Var:
            return fetch_variable(...);
        case T_Const:
            return expr->constvalue;
        case T_OpExpr:
            left = eval_expression(expr->left);
            right = eval_expression(expr->right);
            return apply_operator(expr->op, left, right);
        // 每次都要switch和递归!
    }
}

/*
 * 优化方式: 预编译成步骤数组
 */
typedef struct ExprState {
    /* 预编译的步骤 */
    struct ExprEvalStep *steps;
    int steps_len;
    
    /* 执行函数 */
    ExprStateEvalFunc evalfunc;
} ExprState;

/*
 * 执行预编译表达式: 顺序执行步骤
 */
Datum ExecEvalExpr(ExprState *state) {
    for (int i = 0; i < state->steps_len; i++) {
        ExprEvalStep *step = &state->steps[i];
        
        switch (step->opcode) {
            case EEOP_SCAN_FETCHSOME:
                slot_getsomeattrs(step->resultslot, ...);
                break;
            case EEOP_INNER_VAR:
                *op->resvalue = slot->values[attnum];
                *op->resnull = slot->isnull[attnum];
                break;
            case EEOP_FUNCEXPR:
                *op->resvalue = FunctionCall(...);
                break;
            // 简单的switch，无递归
        }
    }
    return result;
}
```

### 3. 批量处理优化

```c
/*
 * 虽然是火山模型，但PostgreSQL在某些地方使用批量处理
 */

/*
 * 示例: Bitmap Scan批量访问
 */
TupleTableSlot *
ExecBitmapHeapScan(BitmapHeapScanState *node)
{
    // 不是一次一个TID，而是批量预取
    prefetch_pages(tbmres->blockno, prefetch_pages);
    
    // 然后顺序处理这个页面的所有元组
    for (offnum = FirstOffsetNumber; ...; offnum++)
    {
        // 一次处理多个元组
    }
}

/*
 * 示例: Hash Join批量构建
 */
ExecHashJoin()
{
    // Phase 1: 批量构建哈希表
    while ((outerTuple = ExecProcNode(outerPlan)) != NULL)
    {
        ExecHashTableInsert(hashtable, outerTuple, hashvalue);
        // 不是逐个返回，而是全部收集
    }
    
    // Phase 2: 批量探测
    // ...
}
```

---

## 与其他模型对比

### 火山模型 vs 向量化模型

```
【火山模型 (PostgreSQL, MySQL)】
  一次处理一个元组
  
  for each tuple in table:
    if eval_qual(tuple):
      project(tuple)
      send_to_client(tuple)
  
  优点: 简单，内存占用小
  缺点: 函数调用多，Cache miss

【向量化模型 (ClickHouse, DuckDB)】
  一次处理一批元组 (通常1024个)
  
  while has_more_data:
    batch = fetch_batch(1024 tuples)
    mask = eval_qual_vectorized(batch)  // SIMD
    result = project_vectorized(batch, mask)
    send_to_client(result)
  
  优点: SIMD, Cache友好, 高吞吐
  缺点: 内存占用大，实现复杂

【性能对比】
任务: 扫描1亿行，简单过滤

火山模型:
  - 函数调用: 1亿次
  - 时间: 10秒

向量化:
  - 函数调用: 10万次 (1亿/1024)
  - SIMD加速: 4-8倍
  - 时间: 1-2秒

差距: 5-10倍!
```

### 火山模型 vs 编译执行

```
【火山模型】
  解释执行 + JIT优化
  
  运行时:
    解释器循环
    或JIT编译生成代码

【编译执行 (HyPer, Peloton)】
  将整个查询编译成机器码
  
  编译期:
    Query → LLVM IR → 机器码
  
  运行时:
    直接执行编译好的机器码
    无解释开销
  
  for (int i = 0; i < table_size; i++) {
      if (table[i].age > 18) {  // 直接内联
          sum += table[i].amount;
      }
  }

【性能对比】
复杂聚合查询:

火山模型 + JIT: 5秒
编译执行: 1秒

差距: 5倍
```

---

## 总结

### 火山模型核心要点

1. **Iterator接口**: 所有节点实现统一的Next()接口
2. **Pull-based**: 上层主动向下层拉取数据
3. **Pipeline执行**: 流水线处理，一次一个元组
4. **递归调用**: 自然的控制流

### 为什么PostgreSQL坚持火山模型？

```
✅ 1. 成熟稳定
   - 30年的优化和测试
   - Bug少，可靠性高

✅ 2. 实现简单
   - 代码易读易维护
   - 新功能容易添加

✅ 3. 内存友好
   - 适合OLTP工作负载
   - 支持大量并发连接

✅ 4. JIT弥补差距
   - PostgreSQL 11+ JIT编译
   - 性能逼近向量化引擎

⚠️  但在OLAP场景下:
   - DuckDB (向量化) 可能快10倍+
   - ClickHouse (向量化) 可能快100倍+
```

### 最佳实践

```sql
-- 1. 对于OLAP查询，考虑启用JIT
SET jit = on;
SET jit_above_cost = 100000;  -- 只对大查询启用

-- 2. 调整work_mem让Hash/Sort在内存中完成
SET work_mem = '256MB';

-- 3. 使用并行查询加速大表扫描
SET max_parallel_workers_per_gather = 4;

-- 4. 创建合适的索引减少扫描量
CREATE INDEX ON orders(user_id) WHERE amount > 100;
```

---

**文档版本**: PostgreSQL 17.6  
**创建日期**: 2025-10-17  
**重要程度**: ⭐⭐⭐⭐⭐

