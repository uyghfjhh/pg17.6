# Backend 后端进程分析

> PostgreSQL Backend进程的核心源码分析

**源码文件**: `src/backend/tcop/postgres.c` (4500+行)  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. 核心功能

Backend进程是PostgreSQL的**SQL执行引擎**，每个客户端连接对应一个Backend进程：

```
✅ 处理客户端SQL请求
✅ 解析、分析、优化SQL语句
✅ 执行查询计划
✅ 管理事务状态
✅ 返回结果给客户端
✅ 维护会话状态
```

---

## 2. 完整生命周期

### 2.1 生命周期流程图

```
Postmaster fork() Backend
│
├─→ [1] PostgresMain() 初始化
│    ├─ 设置信号处理器
│    ├─ 初始化内存上下文
│    ├─ 连接到共享内存
│    └─ 身份认证 (src/backend/libpq/auth.c)
│
├─→ [2] 进入主循环
│    while (true) {
│      ├─ ReadCommand() - 读取SQL
│      ├─ 解析SQL
│      ├─ 分析SQL
│      ├─ 优化生成执行计划
│      ├─ 执行查询
│      └─ 返回结果
│    }
│
└─→ [3] 退出
     ├─ 提交/回滚未完成事务
     ├─ 释放资源
     └─ proc_exit()
```

### 2.2 PostgresMain 核心代码

```c
// src/backend/tcop/postgres.c:4000
void
PostgresMain(const char *dbname, const char *username)
{
    /* [1] 初始化 */
    BaseInit();
    InitPostgres(dbname, InvalidOid, username, InvalidOid, NULL, false);
    
    /* [2] 设置信号处理 */
    pqsignal(SIGTERM, die);
    pqsignal(SIGQUIT, quickdie);
    pqsignal(SIGUSR1, procsignal_sigusr1_handler);
    
    /* [3] 主循环 */
    for (;;)
    {
        /* 重置每条语句的状态 */
        MemoryContextSwitchTo(MessageContext);
        MemoryContextResetAndDeleteChildren(MessageContext);
        
        /* 读取客户端命令 */
        firstchar = ReadCommand(&input_message);
        
        switch (firstchar)
        {
            case 'Q':  /* 简单查询 */
                {
                    const char *query_string = pq_getmsgstring(&input_message);
                    exec_simple_query(query_string);
                }
                break;
                
            case 'P':  /* Parse (扩展协议) */
                exec_parse_message(...);
                break;
                
            case 'B':  /* Bind */
                exec_bind_message(...);
                break;
                
            case 'E':  /* Execute */
                exec_execute_message(...);
                break;
                
            case 'X':  /* Terminate */
                whereToSendOutput = DestNone;
                return;
        }
    }
}
```

---

## 3. SQL执行流程

### 3.1 完整执行流程

```
exec_simple_query("SELECT * FROM users WHERE id = 1")
│
├─→ [1] Parser (词法和语法分析)
│    源码: src/backend/parser/
│    ├─ raw_parser() - 词法分析
│    └─ parse_analyze() - 语法分析
│    输出: 解析树 (RawStmt)
│
├─→ [2] Analyzer (语义分析)
│    源码: src/backend/parser/analyze.c
│    ├─ transformStmt() - 转换语句
│    ├─ 解析表名、列名
│    ├─ 类型检查
│    └─ 权限检查
│    输出: 查询树 (Query)
│
├─→ [3] Rewriter (查询重写)
│    源码: src/backend/rewrite/
│    ├─ QueryRewrite() - 应用规则
│    ├─ 展开视图
│    └─ 应用RLS策略
│    输出: 重写后的查询树
│
├─→ [4] Planner (查询优化)
│    源码: src/backend/optimizer/
│    ├─ planner() - 生成执行计划
│    ├─ 路径选择 (顺序扫描 vs 索引扫描)
│    ├─ Join方式选择
│    └─ 代价估算
│    输出: 执行计划 (PlannedStmt)
│
├─→ [5] Executor (执行器)
│    源码: src/backend/executor/
│    ├─ standard_ExecutorStart() - 初始化
│    ├─ standard_ExecutorRun() - 执行
│    │   ├─ ExecProcNode() - 执行节点
│    │   └─ 获取元组
│    ├─ standard_ExecutorFinish() - 完成
│    └─ standard_ExecutorEnd() - 清理
│    输出: 结果集
│
└─→ [6] 返回结果
     └─ DestRemote: 发送给客户端
```

### 3.2 exec_simple_query 核心代码

```c
// src/backend/tcop/postgres.c:1000
static void
exec_simple_query(const char *query_string)
{
    List *parsetree_list;
    ListCell *parsetree_item;
    
    /* [1] 开始事务 (如果需要) */
    start_xact_command();
    
    /* [2] Parser - 解析SQL */
    parsetree_list = pg_parse_query(query_string);
    
    /* 遍历所有语句 (可能有多条) */
    foreach(parsetree_item, parsetree_list)
    {
        RawStmt *parsetree = lfirst_node(RawStmt, parsetree_item);
        
        /* [3] Analyzer - 语义分析 */
        querytree_list = pg_analyze_and_rewrite(parsetree, query_string, NULL, 0, NULL);
        
        /* [4] Planner - 生成执行计划 */
        plantree_list = pg_plan_queries(querytree_list, query_string, CURSOR_OPT_PARALLEL_OK, NULL);
        
        /* [5] Portal - 创建执行上下文 */
        portal = CreatePortal("", true, true);
        PortalDefineQuery(portal, NULL, query_string, CMDTAG_UNKNOWN, plantree_list, NULL);
        
        /* [6] Executor - 执行查询 */
        PortalStart(portal, NULL, 0, InvalidSnapshot);
        
        (void) PortalRun(portal,
                         FETCH_ALL,
                         true,    /* top level */
                         true,    /* execute_once */
                         receiver,
                         receiver,
                         qc);
        
        /* [7] 清理 */
        PortalDrop(portal, false);
    }
    
    /* [8] 提交事务 */
    finish_xact_command();
}
```

---

## 4. 扩展查询协议

PostgreSQL支持两种协议：

### 4.1 简单查询协议 (Simple Query)

```
客户端                Backend
  │                     │
  ├──── Q: "SELECT ..." ────→ 解析+执行
  │                     │
  ←──── RowDescription ─┤
  ←──── DataRow 1 ──────┤
  ←──── DataRow 2 ──────┤
  ←──── CommandComplete ┤
  │                     │
```

### 4.2 扩展查询协议 (Extended Query)

```
客户端                Backend
  │                     │
  ├──── Parse ─────────→ 解析SQL, 缓存
  ├──── Bind ──────────→ 绑定参数
  ├──── Describe ──────→ 描述结果
  ├──── Execute ───────→ 执行
  │                     │
  ←──── RowDescription ─┤
  ←──── DataRow ────────┤
  ←──── CommandComplete ┤
  │                     │
  ├──── Sync ──────────→ 同步
  │                     │
```

**优势**: 
- ✅ 参数化查询 (防SQL注入)
- ✅ 预编译语句 (性能提升)
- ✅ 批量执行

---

## 5. 事务管理

### 5.1 事务状态机

```
Backend事务状态:

TBLOCK_DEFAULT (空闲)
  │
  │ BEGIN
  ├──────────→ TBLOCK_STARTED (事务中)
  │              │
  │              │ SQL语句
  │              ├────→ TBLOCK_INPROGRESS
  │              │
  │              │ COMMIT
  │              ├────→ TBLOCK_END → TBLOCK_DEFAULT
  │              │
  │              │ ROLLBACK
  │              └────→ TBLOCK_ABORT → TBLOCK_DEFAULT
  │
  │ 自动提交模式 (autocommit)
  │ SQL语句自动包装在事务中
  └──────────→ BEGIN → ... → COMMIT
```

### 5.2 事务管理函数

```c
// src/backend/access/transam/xact.c

/* 开始事务 */
void StartTransactionCommand(void)
{
    switch (CurrentTransactionState->blockState)
    {
        case TBLOCK_DEFAULT:
            StartTransaction();
            s->blockState = TBLOCK_STARTED;
            break;
        // ...
    }
}

/* 提交事务 */
void CommitTransactionCommand(void)
{
    switch (s->blockState)
    {
        case TBLOCK_INPROGRESS:
            CommitTransaction();
            s->blockState = TBLOCK_DEFAULT;
            break;
        // ...
    }
}
```

---

## 6. 内存管理

### 6.1 内存上下文层次

```
TopMemoryContext (永不释放)
  │
  ├─ MessageContext (每条SQL后重置)
  │
  ├─ TransactionContext (事务结束后释放)
  │   │
  │   └─ PortalContext (查询结束后释放)
  │       │
  │       └─ ExecutorContext (执行期间使用)
  │
  └─ CacheMemoryContext (缓存元数据)
```

### 6.2 内存上下文切换

```c
// 典型用法
MemoryContext oldcontext;

/* 切换到事务上下文 */
oldcontext = MemoryContextSwitchTo(TransactionContext);

/* 分配内存 (在TransactionContext中) */
ptr = palloc(size);

/* 恢复原上下文 */
MemoryContextSwitchTo(oldcontext);

/* 事务结束时自动释放TransactionContext */
```

---

## 7. 并发控制

### 7.1 Backend与共享资源

```
Backend进程访问共享资源:

┌─────────────┐
│  Backend    │
│  Process    │
└──────┬──────┘
       │
       ├──→ Shared Buffers (缓冲池)
       │     └─ 需要获取Buffer锁
       │
       ├──→ Lock Tables (锁表)
       │     └─ 需要获取Partition锁
       │
       ├──→ WAL Buffers (WAL缓冲)
       │     └─ 需要获取WAL插入锁
       │
       └──→ PGPROC (进程槽)
             └─ 记录当前事务状态
```

### 7.2 锁获取示例

```c
// 查询执行时自动获取表锁
Relation rel = table_open(relid, AccessShareLock);

// 使用表...
// 扫描、读取数据

// 释放锁 (通常在事务结束时)
table_close(rel, AccessShareLock);
```

---

## 8. 性能监控

### 8.1 查看Backend活动

```sql
-- 查看所有Backend
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    backend_start,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE backend_type = 'client backend'
ORDER BY backend_start;
```

### 8.2 查看慢查询

```sql
-- 运行超过5秒的查询
SELECT 
    pid,
    now() - query_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
  AND (now() - query_start) > interval '5 seconds'
ORDER BY duration DESC;
```

---

## 9. 架构图

```
Backend 进程架构:

┌────────────────────────────────────────────────────┐
│            Backend Process (PostgresMain)          │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │  SQL接收和协议处理                            │ │
│  │  • ReadCommand()                              │ │
│  │  • 简单查询 vs 扩展协议                       │ │
│  └────────────┬─────────────────────────────────┘ │
│               │                                    │
│  ┌────────────▼─────────────────────────────────┐ │
│  │  SQL处理流程                                  │ │
│  │  ┌──────┐  ┌────────┐  ┌────────┐  ┌──────┐ │ │
│  │  │Parser│→│Analyzer│→│Rewriter│→│Planner│ │ │
│  │  └──────┘  └────────┘  └────────┘  └──────┘ │ │
│  └────────────┬─────────────────────────────────┘ │
│               │                                    │
│  ┌────────────▼─────────────────────────────────┐ │
│  │  Executor (执行器)                            │ │
│  │  • ExecProcNode() - 节点执行                 │ │
│  │  • 访问Buffer Manager                        │ │
│  │  • 获取锁                                     │ │
│  └────────────┬─────────────────────────────────┘ │
│               │                                    │
│  ┌────────────▼─────────────────────────────────┐ │
│  │  结果返回                                     │ │
│  │  • DestRemote: 发送给客户端                  │ │
│  └──────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────┘
```

---

## 总结

### Backend核心职责

1. **SQL处理**: 解析→分析→优化→执行
2. **事务管理**: BEGIN/COMMIT/ROLLBACK
3. **并发控制**: 获取锁、访问共享资源
4. **内存管理**: 上下文切换、自动清理
5. **结果返回**: 协议处理、数据传输

### 关键特性

- ✅ **进程隔离**: 每个连接独立Backend
- ✅ **自动清理**: 事务结束释放资源
- ✅ **并发安全**: MVCC + 锁机制

---

**下一步**: 阅读 [03_checkpointer.md](03_checkpointer.md) 了解检查点进程

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

