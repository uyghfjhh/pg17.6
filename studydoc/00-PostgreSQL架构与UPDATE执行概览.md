# PostgreSQL架构与UPDATE执行流程概览

> **系列文档导读** - PostgreSQL UPDATE语句完整执行机制深度分析
>
> **版本**: PostgreSQL 17.6
> **最后更新**: 2025年
> **作者**: 基于PostgreSQL源码深度分析

---

## 📚 文档系列说明

本文档是**PostgreSQL UPDATE语句执行流程系列文档**的总览和架构说明,作为整个系列的**第0号文档**。

### 完整文档列表

1. ✅ **00-PostgreSQL架构与UPDATE执行概览.md** (本文档)
   - PostgreSQL层次架构总览
   - 各层职责详解
   - UPDATE的层次穿透路径
   - 源码目录导航

2. 📄 **UPDATE语句执行流程完整分析.md** (主文档,约10000行)
   - 从词法分析到事务提交的完整流程
   - 每个阶段的详细代码分析
   - 函数调用链和数据结构转换

3. 📊 **UPDATE流程图集.md** (约3500行)
   - 14个大型ASCII流程图
   - 可视化各阶段的执行过程
   - 数据流转图

4. 🔍 **UPDATE示例跟踪.md** (约4500行)
   - 3个完整的真实案例
   - 从SQL到磁盘的完整跟踪
   - 性能数据和优化对比

5. 📐 **UPDATE数据结构详解.md** (约3500行)
   - 所有关键数据结构的C定义
   - 内存布局图
   - 字段详解和使用场景

6. ⚡ **UPDATE性能优化指南.md** (约2500行)
   - HOT更新优化技巧
   - 索引策略和监控方法
   - 实际优化案例

7. 💾 **UPDATE磁盘存储布局详解.md** (约3500行)
   - 页面物理结构
   - 元组在磁盘上的布局
   - TOAST、FSM、VM机制

8. 🗂️ **UPDATE文件定位与扫描机制详解.md** (约4500行)
   - 从SQL到文件路径的完整映射
   - 顺序扫描和索引扫描的实现
   - Buffer与磁盘的交互细节

**总计**: 约34,000行深度技术文档

---

## 📖 目录

1. [PostgreSQL层次架构总览](#1-postgresql层次架构总览)
2. [各层职责详解](#2-各层职责详解)
3. [UPDATE语句的层次穿透](#3-update语句的层次穿透)
4. [模块间交互关系](#4-模块间交互关系)
5. [关键子系统详解](#5-关键子系统详解)
6. [源码目录结构](#6-源码目录结构)
7. [阅读建议](#7-阅读建议)

---

## 1. PostgreSQL层次架构总览

### 1.1 经典五层架构

PostgreSQL采用经典的**分层架构**设计,从客户端到磁盘共分为5个主要层次:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Client Layer                                │
│                        (客户端层)                                   │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ psql / JDBC Driver / ODBC / libpq / pg_dump / pgAdmin         ││
│  │ Application Programs                                           ││
│  └────────────────────────────────────────────────────────────────┘│
└──────────────────────────┬──────────────────────────────────────────┘
                           │ PostgreSQL Wire Protocol
                           │ TCP/IP or Unix Socket
                           ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   Process Management Layer                          │
│                     (进程管理层)                                    │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ Postmaster (Master Daemon)                                     ││
│  │   ├─ Backend Process 1  ← 客户端连接1                         ││
│  │   ├─ Backend Process 2  ← 客户端连接2                         ││
│  │   ├─ Backend Process N  ← 客户端连接N                         ││
│  │   │                                                             ││
│  │   ├─ Background Writer  (脏页异步刷写)                        ││
│  │   ├─ WAL Writer         (WAL日志刷写)                         ││
│  │   ├─ Checkpointer       (检查点管理)                          ││
│  │   ├─ Autovacuum Launcher(自动清理)                            ││
│  │   ├─ Stats Collector    (统计信息收集)                        ││
│  │   └─ Archiver           (归档进程)                            ││
│  └────────────────────────────────────────────────────────────────┘│
└──────────────────────────┬──────────────────────────────────────────┘
                           │ Shared Memory (共享内存区)
                           │ - Shared Buffers
                           │ - WAL Buffers
                           │ - Lock Table
                           ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    Query Processing Layer                           │
│                      (查询处理层)                                   │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ 子层1: Parser (解析器)                                         ││
│  │   ┌──────────────────────────────────────────────────────────┐││
│  │   │ 1.1 Lexer: scan.l                                        │││
│  │   │     SQL文本 → Token流                                    │││
│  │   │     "UPDATE users SET ..." → [UPDATE][users][SET]...     │││
│  │   ├──────────────────────────────────────────────────────────┤││
│  │   │ 1.2 Parser: gram.y                                       │││
│  │   │     Token流 → Parse Tree (AST)                          │││
│  │   │     使用Bison生成的LALR(1)解析器                        │││
│  │   ├──────────────────────────────────────────────────────────┤││
│  │   │ 1.3 Analyzer: analyze.c                                  │││
│  │   │     Parse Tree → Query Tree                             │││
│  │   │     - 表名解析 (users → OID 16385)                      │││
│  │   │     - 类型检查                                           │││
│  │   │     - 权限检查                                           │││
│  │   └──────────────────────────────────────────────────────────┘││
│  │   输出: Query {commandType=CMD_UPDATE, ...}                   ││
│  ├────────────────────────────────────────────────────────────────┤│
│  │ 子层2: Rewriter (重写器)                                      ││
│  │   ┌──────────────────────────────────────────────────────────┐││
│  │   │ rewriteHandler.c                                         │││
│  │   │   ├─ 应用规则 (RULE)                                    │││
│  │   │   ├─ 展开视图 (VIEW)                                    │││
│  │   │   ├─ 应用RLS策略 (Row-Level Security)                   │││
│  │   │   └─ 处理INSTEAD OF触发器                               │││
│  │   └──────────────────────────────────────────────────────────┘││
│  │   输出: Rewritten Query Tree                                  ││
│  ├────────────────────────────────────────────────────────────────┤│
│  │ 子层3: Planner (优化器)                                       ││
│  │   ┌──────────────────────────────────────────────────────────┐││
│  │   │ planner.c                                                │││
│  │   │   ├─ 生成访问路径 (Path)                                │││
│  │   │   │   • SeqScan (顺序扫描)                              │││
│  │   │   │   • IndexScan (索引扫描)                            │││
│  │   │   │   • IndexOnlyScan (纯索引扫描)                      │││
│  │   │   │   • BitmapScan (位图扫描)                           │││
│  │   │   ├─ 成本估算 (Cost Estimation)                         │││
│  │   │   │   • CPU cost + I/O cost                             │││
│  │   │   │   • 基于pg_statistic统计信息                       │││
│  │   │   ├─ 选择最优路径                                       │││
│  │   │   └─ 生成执行计划 (Path → Plan)                         │││
│  │   └──────────────────────────────────────────────────────────┘││
│  │   输出: PlannedStmt {planTree: ModifyTable → IndexScan}       ││
│  ├────────────────────────────────────────────────────────────────┤│
│  │ 子层4: Executor (执行器)                                      ││
│  │   ┌──────────────────────────────────────────────────────────┐││
│  │   │ execMain.c, nodeModifyTable.c, node*.c                   │││
│  │   │                                                           │││
│  │   │ ExecutorStart()                                          │││
│  │   │   ├─ 创建EState (执行状态)                              │││
│  │   │   ├─ InitPlan() - 初始化计划节点                        │││
│  │   │   ├─ 打开表和索引                                       │││
│  │   │   └─ 初始化触发器                                       │││
│  │   │                                                           │││
│  │   │ ExecutorRun()                                            │││
│  │   │   └─ ExecProcNode() - 递归执行计划树                    │││
│  │   │       └─ ExecModifyTable()                               │││
│  │   │           ├─ 扫描要更新的行                             │││
│  │   │           ├─ ExecUpdate()                                │││
│  │   │           │   ├─ BEFORE触发器                           │││
│  │   │           │   ├─ heap_update()  ← 核心!                 │││
│  │   │           │   ├─ 更新索引                               │││
│  │   │           │   └─ AFTER触发器                            │││
│  │   │           └─ 返回结果 (RETURNING)                       │││
│  │   │                                                           │││
│  │   │ ExecutorFinish()                                         │││
│  │   │   └─ AFTER STATEMENT触发器                              │││
│  │   │                                                           │││
│  │   │ ExecutorEnd()                                            │││
│  │   │   └─ 关闭资源,释放内存                                  │││
│  │   └──────────────────────────────────────────────────────────┘││
│  └────────────────────────────────────────────────────────────────┘│
└──────────────────────────┬──────────────────────────────────────────┘
                           │
┌──────────────────────────┴──────────────────────────────────────────┐
│                   Storage Management Layer                          │
│                     (存储管理层)                                    │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ Access Methods (访问方法)                                      ││
│  │   ┌──────────────────────────────────────────────────────────┐││
│  │   │ Heap AM (堆表访问方法) - heapam.c                       │││
│  │   │   ├─ heap_insert()   - 插入元组                         │││
│  │   │   ├─ heap_update()   - 更新元组 ★                       │││
│  │   │   ├─ heap_delete()   - 删除元组                         │││
│  │   │   └─ heap_fetch()    - 获取元组                         │││
│  │   ├──────────────────────────────────────────────────────────┤││
│  │   │ Index AM (索引访问方法)                                  │││
│  │   │   ├─ B-tree: nbtree/  (btinsert, btgettuple, ...)      │││
│  │   │   ├─ Hash: hash/                                         │││
│  │   │   ├─ GiST: gist/                                         │││
│  │   │   ├─ GIN: gin/                                           │││
│  │   │   ├─ BRIN: brin/                                         │││
│  │   │   └─ SP-GiST: spgist/                                    │││
│  │   └──────────────────────────────────────────────────────────┘││
│  ├────────────────────────────────────────────────────────────────┤│
│  │ Buffer Manager (缓冲区管理) - bufmgr.c                         ││
│  │   ┌──────────────────────────────────────────────────────────┐││
│  │   │ Shared Buffers Pool                                      │││
│  │   │   • 默认128MB, 可配置到数百GB                           │││
│  │   │   • 页面缓存 (8KB per page)                             │││
│  │   │                                                           │││
│  │   │ 核心函数:                                                │││
│  │   │   ├─ ReadBuffer()      - 读取页面                       │││
│  │   │   │   └─ BufTable查找 → Hit/Miss                        │││
│  │   │   ├─ MarkBufferDirty() - 标记脏页                       │││
│  │   │   ├─ FlushBuffer()     - 刷写脏页                       │││
│  │   │   └─ ReleaseBuffer()   - 释放引用                       │││
│  │   │                                                           │││
│  │   │ 替换策略:                                                │││
│  │   │   └─ Clock-sweep算法 (类LRU)                            │││
│  │   └──────────────────────────────────────────────────────────┘││
│  ├────────────────────────────────────────────────────────────────┤│
│  │ Transaction Manager (事务管理)                                 ││
│  │   ┌──────────────────────────────────────────────────────────┐││
│  │   │ Transaction Control - xact.c                             │││
│  │   │   ├─ StartTransaction()                                  │││
│  │   │   │   └─ 分配XID                                         │││
│  │   │   ├─ CommitTransaction()  ★                              │││
│  │   │   │   ├─ 写COMMIT记录到WAL                              │││
│  │   │   │   ├─ XLogFlush() - 刷WAL                            │││
│  │   │   │   ├─ 更新CLOG                                       │││
│  │   │   │   └─ 释放锁                                         │││
│  │   │   └─ AbortTransaction()                                  │││
│  │   │       └─ 清理资源                                       │││
│  │   ├──────────────────────────────────────────────────────────┤││
│  │   │ WAL Manager - xlog.c                                     │││
│  │   │   ├─ XLogInsert()    - 写WAL记录                        │││
│  │   │   │   ├─ 分配WAL空间                                    │││
│  │   │   │   ├─ 写入WAL buffers                                │││
│  │   │   │   ├─ 计算CRC                                        │││
│  │   │   │   └─ 返回LSN                                        │││
│  │   │   └─ XLogFlush()     - 刷WAL到磁盘                      │││
│  │   │       └─ write() + fsync()                              │││
│  │   ├──────────────────────────────────────────────────────────┤││
│  │   │ CLOG (Commit Log) - clog/                                │││
│  │   │   └─ TransactionIdSetStatusBit()                         │││
│  │   │       • IN_PROGRESS (00)                                 │││
│  │   │       • COMMITTED (01)                                   │││
│  │   │       • ABORTED (10)                                     │││
│  │   └──────────────────────────────────────────────────────────┘││
│  └────────────────────────────────────────────────────────────────┘│
└──────────────────────────┬──────────────────────────────────────────┘
                           │
┌──────────────────────────┴──────────────────────────────────────────┐
│                    Disk Management Layer                            │
│                      (磁盘管理层)                                   │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ Storage Manager (存储管理器)                                   ││
│  │   ┌──────────────────────────────────────────────────────────┐││
│  │   │ Interface - smgr.c                                       │││
│  │   │   ├─ smgropen()  - 打开relation                         │││
│  │   │   ├─ smgrread()  - 读取block                            │││
│  │   │   ├─ smgrwrite() - 写入block                            │││
│  │   │   └─ smgrsync()  - fsync文件                            │││
│  │   ├──────────────────────────────────────────────────────────┤││
│  │   │ Implementation - md.c (Magnetic Disk)                    │││
│  │   │   ├─ mdopen()    - 打开文件                             │││
│  │   │   ├─ mdread()    - pread() 系统调用                     │││
│  │   │   ├─ mdwrite()   - pwrite() 系统调用                    │││
│  │   │   └─ mdsync()    - fsync() 系统调用                     │││
│  │   ├──────────────────────────────────────────────────────────┤││
│  │   │ VFD (Virtual File Descriptor) - fd.c                     │││
│  │   │   • 文件描述符缓存和管理                                │││
│  │   │   • 避免超过OS的fd限制                                  │││
│  │   │   • 自动关闭不活跃的fd                                  │││
│  │   └──────────────────────────────────────────────────────────┘││
│  ├────────────────────────────────────────────────────────────────┤│
│  │ File System Layout (文件系统布局)                              ││
│  │   ┌──────────────────────────────────────────────────────────┐││
│  │   │ $PGDATA/                                                 │││
│  │   │   ├─ base/{dboid}/{relfilenode}      - 表文件           │││
│  │   │   ├─ base/{dboid}/{relfilenode}_fsm  - FSM              │││
│  │   │   ├─ base/{dboid}/{relfilenode}_vm   - Visibility Map   │││
│  │   │   ├─ pg_wal/                          - WAL日志目录     │││
│  │   │   │   └─ 000000010000000000000001     - WAL文件(16MB)  │││
│  │   │   ├─ pg_xact/                         - CLOG目录        │││
│  │   │   │   └─ 0000                         - CLOG文件       │││
│  │   │   ├─ pg_tblspc/                       - 表空间          │││
│  │   │   ├─ pg_control                       - 控制文件        │││
│  │   │   └─ ...                                                │││
│  │   └──────────────────────────────────────────────────────────┘││
│  └────────────────────────────────────────────────────────────────┘│
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      Operating System                               │
│                       (操作系统层)                                  │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ System Calls                                                    ││
│  │   ├─ open() / close()        - 文件打开/关闭                  ││
│  │   ├─ pread() / pwrite()      - 定位读写                       ││
│  │   ├─ fsync() / fdatasync()   - 同步到磁盘                     ││
│  │   ├─ mmap() / munmap()       - 内存映射                       ││
│  │   ├─ shmget() / shmat()      - 共享内存                       ││
│  │   └─ semop() / semctl()      - 信号量                         ││
│  ├────────────────────────────────────────────────────────────────┤│
│  │ File System (ext4, xfs, zfs, btrfs, ...)                       ││
│  ├────────────────────────────────────────────────────────────────┤│
│  │ Page Cache / Buffer Cache                                       ││
│  ├────────────────────────────────────────────────────────────────┤│
│  │ Block Device Layer                                              ││
│  │   └─ I/O Scheduler (CFQ, Deadline, NOOP)                       ││
│  └────────────────────────────────────────────────────────────────┘│
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ↓
                    Physical Disk
                     (物理磁盘)
               HDD / SSD / NVMe / RAID
```

### 1.2 水平子系统 (横切关注点)

除了垂直分层,PostgreSQL还包含多个横切整个系统的子系统:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Horizontal Subsystems                            │
│                     (水平子系统)                                    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ Memory Management (内存管理)                                   ││
│  │   ├─ MemoryContext (分层内存管理)                             ││
│  │   │   ├─ TopMemoryContext                                      ││
│  │   │   ├─ TransactionContext (事务期间使用)                    ││
│  │   │   ├─ CacheMemoryContext (缓存)                            ││
│  │   │   └─ ...                                                   ││
│  │   ├─ palloc() / pfree() (内存分配)                            ││
│  │   └─ Shared Memory (进程间共享)                               ││
│  │       • Shared Buffers                                         ││
│  │       • Lock Table                                             ││
│  │       • PGPROC/PGXACT Arrays                                   ││
│  └────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ Lock Management (锁管理)                                       ││
│  │   ├─ Spinlock (自旋锁) - 最底层                              ││
│  │   │   • 保护小的临界区                                        ││
│  │   │   • 持有时间极短 (微秒级)                                ││
│  │   ├─ LWLock (轻量级锁)                                        ││
│  │   │   • 保护共享内存结构                                      ││
│  │   │   • Shared/Exclusive模式                                  ││
│  │   ├─ Regular Lock (常规锁)                                    ││
│  │   │   • 表锁 (8种模式)                                        ││
│  │   │   • 行锁                                                   ││
│  │   │   • 支持死锁检测                                          ││
│  │   └─ Advisory Lock (咨询锁)                                   ││
│  └────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ System Catalog (系统目录)                                      ││
│  │   ├─ System Tables                                             ││
│  │   │   ├─ pg_class      (表/索引/序列)                         ││
│  │   │   ├─ pg_attribute  (列定义)                               ││
│  │   │   ├─ pg_type       (数据类型)                             ││
│  │   │   ├─ pg_proc       (函数/过程)                            ││
│  │   │   ├─ pg_database   (数据库)                               ││
│  │   │   └─ ...                                                   ││
│  │   ├─ Syscache (系统目录缓存)                                  ││
│  │   │   └─ SearchSysCache() - 内存中查找                        ││
│  │   └─ RelCache (Relation缓存)                                  ││
│  │       └─ RelationIdGetRelation() - 获取Relation描述符         ││
│  └────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ Utility Systems (工具子系统)                                   ││
│  │   ├─ Error Reporting (ereport, elog)                           ││
│  │   ├─ Logging (日志管理)                                        ││
│  │   ├─ Statistics (pg_stat_*)                                    ││
│  │   ├─ GUC (Grand Unified Configuration)                         ││
│  │   │   └─ postgresql.conf参数管理                              ││
│  │   └─ Query Jumbling (查询指纹)                                ││
│  └────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 各层职责详解

### 2.1 客户端层 (Client Layer)

**职责**:
- 建立与PostgreSQL服务器的连接
- 发送SQL语句
- 接收查询结果
- 错误处理和事务控制

**主要组件**:

#### libpq (C客户端库)
```c
// 连接管理
PGconn *PQconnectdb(const char *conninfo);

// 查询执行
PGresult *PQexec(PGconn *conn, const char *command);
PGresult *PQprepare(PGconn *conn, const char *stmtName, ...);
PGresult *PQexecPrepared(PGconn *conn, const char *stmtName, ...);

// 结果处理
char *PQgetvalue(PGresult *res, int row_number, int column_number);
int PQntuples(PGresult *res);
```

#### psql (命令行工具)
- 交互式SQL shell
- 支持元命令 (\d, \dt, \l, ...)
- 批处理模式

#### JDBC/ODBC Driver
- Java/其他语言的接口
- 实现标准的JDBC/ODBC API

**通信协议**: PostgreSQL Wire Protocol
- 基于TCP/IP (默认端口5432)
- 二进制协议
- 支持prepared statements
- 支持COPY协议(批量数据传输)

---

### 2.2 进程管理层 (Process Management)

PostgreSQL使用**多进程架构**(而非多线程):

#### Postmaster (守护进程)

源文件: `src/backend/postmaster/postmaster.c`

```c
// 主循环伪代码
PostmasterMain()
{
    // 1. 初始化
    LoadConfigurationFiles();
    CreateSharedMemory();

    // 2. 监听端口
    listen_sock = socket(...);
    bind(listen_sock, port=5432);
    listen(listen_sock);

    // 3. 启动后台进程
    StartBackgroundWorkers();

    // 4. 主循环
    for (;;) {
        // 接受连接
        client_sock = accept(listen_sock);

        // fork新的backend进程
        pid = fork();
        if (pid == 0) {
            // 子进程
            BackendRun(client_sock);
            exit(0);
        }

        // 父进程继续监听
    }
}
```

**职责**:
- 监听TCP端口接受连接
- fork后端进程处理客户端请求
- 管理后台工作进程
- 处理信号(SIGHUP重新加载配置,SIGTERM关闭)
- 崩溃恢复

#### Backend Process (后端进程)

每个客户端连接对应一个Backend进程:

```c
PostgresMain(int argc, char *argv[], ...)
{
    // 1. 初始化
    InitPostgres(...);

    // 2. 主循环
    for (;;) {
        // 接收命令
        ReadCommand(&input_message);

        // 处理命令
        switch (input_message.type) {
        case 'Q':  // Simple Query
            exec_simple_query(input_message.data);
            break;
        case 'P':  // Parse (Prepared Statement)
            exec_parse_message(...);
            break;
        case 'B':  // Bind
            exec_bind_message(...);
            break;
        case 'E':  // Execute
            exec_execute_message(...);
            break;
        }

        // 发送结果
        SendResult(...);
    }
}
```

**特点**:
- 独立的内存空间
- 独立的事务上下文
- 崩溃不影响其他连接
- 通过共享内存通信

#### 后台进程

##### Background Writer (bgwriter)
```c
// 异步刷写脏页
BgWriterMain()
{
    for (;;) {
        pg_usleep(bgwriter_delay * 1000L);  // 默认200ms

        BgBufferSync();  // 刷写脏页
    }
}
```

##### WAL Writer
```c
// 定期刷WAL buffers
WalWriterMain()
{
    for (;;) {
        pg_usleep(wal_writer_delay * 1000L);

        XLogBackgroundFlush();
    }
}
```

##### Checkpointer
```c
// 创建检查点
CheckpointerMain()
{
    for (;;) {
        // 等待checkpoint_timeout或WAL填满
        WaitForCheckpoint();

        CreateCheckPoint(CHECKPOINT_CAUSE_TIME);
    }
}
```

##### Autovacuum Launcher
```c
// 自动清理
AutoVacLauncherMain()
{
    for (;;) {
        // 检查需要VACUUM的表
        table_list = GetTablesNeedingVacuum();

        // 启动worker进程
        for (table in table_list) {
            StartAutovacuumWorker(table);
        }

        pg_usleep(autovacuum_naptime);
    }
}
```

---

### 2.3 查询处理层 (Query Processing)

这是UPDATE语句的核心处理层,分为4个子层:

#### 子层1: Parser (解析器)

**源文件**: `src/backend/parser/`

##### 1.1 词法分析 (Lexer)

文件: `scan.l` (Flex生成)

```
SQL文本:
"UPDATE users SET name = 'John' WHERE id = 1;"

Token流:
[UPDATE (keyword)]
[users (identifier)]
[SET (keyword)]
[name (identifier)]
[= (operator)]
['John' (string constant)]
[WHERE (keyword)]
[id (identifier)]
[= (operator)]
[1 (numeric constant)]
[; (punctuation)]
```

##### 1.2 语法分析 (Parser)

文件: `gram.y` (Bison生成)

语法规则(UpdateStmt):
```c
UpdateStmt: opt_with_clause UPDATE relation_expr_opt_alias
            SET set_clause_list
            from_clause
            where_or_current_clause
            returning_clause
            {
                UpdateStmt *n = makeNode(UpdateStmt);
                n->relation = $3;
                n->targetList = $5;
                n->fromClause = $6;
                n->whereClause = $7;
                n->returningList = $8;
                n->withClause = $1;
                $$ = (Node *) n;
            }
```

输出: UpdateStmt (Parse Tree)

##### 1.3 语义分析 (Analyzer)

文件: `analyze.c`

关键函数:
```c
Query *transformUpdateStmt(ParseState *pstate, UpdateStmt *stmt)
{
    Query *qry = makeNode(Query);

    qry->commandType = CMD_UPDATE;

    // 1. 解析表名 → OID
    qry->resultRelation = setTargetTable(pstate, stmt->relation, ...);
    // "users" → 查询pg_class → OID 16385

    // 2. 处理FROM子句
    transformFromClause(pstate, stmt->fromClause);

    // 3. 转换WHERE子句
    qry->whereClause = transformWhereClause(pstate, stmt->whereClause, ...);

    // 4. 转换SET子句
    qry->targetList = transformUpdateTargetList(pstate, stmt->targetList);

    // 5. 转换RETURNING子句
    qry->returningList = transformReturningList(pstate, stmt->returningList, ...);

    return qry;
}
```

输出: Query Tree

#### 子层2: Rewriter (重写器)

**源文件**: `src/backend/rewrite/rewriteHandler.c`

```c
List *QueryRewrite(Query *parsetree)
{
    List *querylist;

    // 1. 应用规则系统
    querylist = fireRIRrules(parsetree, NIL);

    // 2. 展开视图
    //    如果users是视图,展开为底层表的查询

    // 3. 应用RLS (Row-Level Security)
    //    添加安全策略的WHERE条件

    // 4. 处理INSTEAD OF触发器

    return querylist;
}
```

**功能**:
- **规则(RULE)**: 自动查询重写
- **视图(VIEW)**: 展开为底层SQL
- **RLS**: 行级安全策略
- **触发器**: INSTEAD OF类型

#### 子层3: Planner (优化器)

**源文件**: `src/backend/optimizer/plan/planner.c`

```c
PlannedStmt *standard_planner(Query *parse, ...)
{
    PlannerInfo *root;
    RelOptInfo *final_rel;
    Path *best_path;
    Plan *top_plan;

    // 1. 创建PlannerInfo
    root = subquery_planner(glob, parse, ...);

    // 2. 生成访问路径
    //    对于UPDATE的WHERE条件:
    //      - SeqScan: 全表扫描
    //      - IndexScan: 使用users_pkey索引
    //      - BitmapScan: 位图索引扫描

    // 3. 成本估算
    cost_seqscan(...);      // CPU cost + I/O cost
    cost_index(...);        // 基于pg_statistic统计

    // 4. 选择最优路径
    best_path = choose_best_path(root, final_rel);

    // 5. Path → Plan
    top_plan = create_plan(root, best_path);

    // 6. 对于UPDATE,包装为ModifyTable节点
    ModifyTable *mt = makeNode(ModifyTable);
    mt->operation = CMD_UPDATE;
    mt->plans = list_make1(top_plan);

    return result;
}
```

**输出**: 执行计划
```
ModifyTable (operation=UPDATE)
  └─ IndexScan on users using users_pkey
      Index Cond: (id = 1)
```

#### 子层4: Executor (执行器)

**源文件**: `src/backend/executor/`

```c
// 1. 初始化
void ExecutorStart(QueryDesc *queryDesc, ...)
{
    EState *estate = CreateExecutorState();

    InitPlan(queryDesc, eflags);
    // ├─ ExecInitModifyTable()
    // │   ├─ 打开表: table_open(users, RowExclusiveLock)
    // │   ├─ 打开索引: index_open(users_pkey, RowExclusiveLock)
    // │   └─ 初始化触发器
    // └─ 递归初始化子计划
}

// 2. 执行
void ExecutorRun(QueryDesc *queryDesc, ...)
{
    ExecutePlan(estate, planstate, ...);
    // └─ ExecProcNode(planstate)
    //     └─ ExecModifyTable()
    //         ├─ 获取要更新的元组
    //         │   slot = ExecProcNode(subplanstate);
    //         │   // IndexScan返回 (id=1, ctid=(0,5), ...)
    //         ├─ ExecUpdate()
    //         │   ├─ ExecBRUpdateTriggers() [BEFORE触发器]
    //         │   ├─ heap_update(relation, ctid, newtup, ...)
    //         │   ├─ ExecInsertIndexTuples() [更新索引]
    //         │   └─ ExecARUpdateTriggers() [AFTER触发器]
    //         └─ 返回结果 (RETURNING)
}

// 3. 完成
void ExecutorFinish(QueryDesc *queryDesc)
{
    // AFTER STATEMENT触发器
}

// 4. 清理
void ExecutorEnd(QueryDesc *queryDesc)
{
    // 关闭表和索引
    // 释放内存
}
```

---

### 2.4 存储管理层 (Storage Management)

#### Access Methods (访问方法)

##### Heap AM (堆表访问)

源文件: `src/backend/access/heap/heapam.c`

```c
TM_Result heap_update(Relation relation,
                      ItemPointer otid,    // 旧元组TID
                      HeapTuple newtup,    // 新元组
                      CommandId cid,
                      Snapshot crosscheck,
                      bool wait,
                      TM_FailureData *tmfd,
                      LockTupleMode *lockmode,
                      TU_UpdateIndexes *update_indexes)
{
    Buffer buffer;
    Page page;
    HeapTupleData oldtup;

    // 1. 读取旧元组所在页面
    buffer = ReadBuffer(relation, ItemPointerGetBlockNumber(otid));
    LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);

    page = BufferGetPage(buffer);

    // 2. 定位旧元组
    ItemId lp = PageGetItemId(page, ItemPointerGetOffsetNumber(otid));
    oldtup.t_data = (HeapTupleHeader) PageGetItem(page, lp);

    // 3. MVCC检查
    result = HeapTupleSatisfiesUpdate(&oldtup, cid, buffer);
    if (result != TM_Ok)
        return result;  // 并发冲突

    // 4. 判断HOT更新
    bool use_hot_update = false;
    if (/* 未修改索引列 && 同一页面 && 有空间 */)
        use_hot_update = true;

    // 5. 写入新元组
    RelationPutHeapTuple(relation, buffer, newtup);

    // 6. 更新旧元组
    oldtup.t_data->t_xmax = GetCurrentTransactionId();
    oldtup.t_data->t_ctid = newtup->t_self;  // 指向新版本

    if (use_hot_update)
        HeapTupleSetHotUpdated(&oldtup);

    // 7. 标记buffer为脏
    MarkBufferDirty(buffer);

    // 8. 记录WAL
    XLogRecPtr recptr = log_heap_update(relation, buffer, ...);
    PageSetLSN(page, recptr);

    // 9. 释放buffer
    UnlockReleaseBuffer(buffer);

    // 10. 返回是否需要更新索引
    *update_indexes = use_hot_update ? TU_None : TU_All;

    return TM_Ok;
}
```

##### Index AM (索引访问)

B-tree示例: `src/backend/access/nbtree/`

```c
// 搜索索引
bool btgettuple(IndexScanDesc scan, ScanDirection dir)
{
    BTScanOpaque so = (BTScanOpaque) scan->opaque;

    // 首次调用,定位到起始位置
    if (!BTScanPosIsValid(so->currPos)) {
        _bt_first(scan, dir);  // B-tree搜索
    }

    // 获取下一个匹配项
    while (...) {
        offnum = _bt_readpage(scan, dir, offnum);

        // 返回heap TID
        scan->xs_heaptid = so->currPos->items[offnum].heapTid;
        return true;
    }

    return false;
}

// B-tree搜索
static bool _bt_first(IndexScanDesc scan, ScanDirection dir)
{
    Relation rel = scan->indexRelation;
    BTStack stack;
    Buffer buf;

    // 从root向下搜索
    stack = _bt_search(rel, scankey, &buf, BT_READ, scan->xs_snapshot);

    // 到达leaf page
    // 二分查找匹配项
    offnum = _bt_binsrch(rel, scankey, buf);

    return true;
}
```

#### Buffer Manager

源文件: `src/backend/storage/buffer/bufmgr.c`

```c
Buffer ReadBuffer_common(SMgrRelation smgr, ...)
{
    BufferDesc *bufHdr;
    BufferTag newTag;
    uint32 newHash;
    bool found;

    // 1. 构建buffer tag
    INIT_BUFFERTAG(newTag, smgr->smgr_rnode, forkNum, blockNum);
    newHash = BufTableHashCode(&newTag);

    // 2. 在hash表中查找
    bufHdr = BufTableLookup(&newTag, newHash);

    if (bufHdr != NULL) {
        // ===== Buffer Hit =====
        PinBuffer(bufHdr, strategy);
        return BufferDescriptorGetBuffer(bufHdr);
    }

    // ===== Buffer Miss =====

    // 3. 选择victim buffer (Clock-sweep)
    bufHdr = StrategyGetBuffer(strategy, &buf_state);

    // 4. 如果victim是脏页,先刷盘
    if (buf_state & BM_DIRTY)
        FlushBuffer(bufHdr, NULL);

    // 5. 从hash表删除旧映射,插入新映射
    BufTableDelete(&oldTag, oldHash);
    BufTableInsert(&newTag, newHash, bufHdr->buf_id);

    // 6. 从磁盘读取
    smgrread(smgr, forkNum, blockNum, (char *) bufBlock);
    //   └─> mdread() → pg_pread() → 系统调用pread()

    // 7. 设置buffer状态
    bufHdr->tag = newTag;
    bufHdr->flags |= BM_VALID;

    PinBuffer(bufHdr, strategy);
    return BufferDescriptorGetBuffer(bufHdr);
}
```

**Clock-sweep替换算法**:
```c
BufferDesc *StrategyGetBuffer(...)
{
    for (;;) {
        buf = GetBufferDescriptor(ClockSweepTick());

        if (usage_count > 0) {
            usage_count--;  // 减1,继续
            continue;
        }

        if (refcount == 0) {
            return buf;  // 找到victim!
        }
    }
}
```

#### Transaction Manager

##### 事务控制

源文件: `src/backend/access/transam/xact.c`

```c
void CommitTransaction(void)
{
    TransactionId xid = GetCurrentTransactionId();

    // 1. 执行AFTER触发器
    AfterTriggerEndQuery(estate);

    // 2. 写COMMIT记录到WAL
    XLogRecPtr recptr = XLogInsert(RM_XACT_ID, XLOG_XACT_COMMIT);

    // 3. 刷WAL到磁盘 (关键!)
    XLogFlush(XactLastRecEnd);
    //   └─> XLogWrite() → write() + fsync()

    // 4. 更新CLOG
    TransactionIdCommitTree(xid, nchildren, children);
    //   └─> 设置XID状态为COMMITTED

    // 5. 释放锁
    LockReleaseAll(DEFAULT_LOCKMETHOD, true);

    // 6. 清理资源
    AtEOXact_RelationCache(true);
    MemoryContextDelete(TransactionContext);

    // 7. 重置状态
    CurrentTransactionState->state = TRANS_DEFAULT;
}
```

##### WAL Manager

源文件: `src/backend/access/transam/xlog.c`

```c
XLogRecPtr XLogInsert(RmgrId rmid, uint8 info)
{
    XLogRecPtr EndPos;

    // 1. 准备WAL记录
    XLogRecData *rdata = XLogRecordAssemble(...);

    // 2. 分配WAL空间
    ReserveXLogInsertLocation(rechdr->xl_tot_len, &StartPos, &EndPos, ...);

    // 3. 计算CRC
    COMP_CRC32C(rdata_crc, rechdr, ...);

    // 4. 复制到WAL buffers
    CopyXLogRecordToWAL(rechdr, rdata, StartPos, EndPos);

    // 5. 返回LSN
    return EndPos;
}

void XLogFlush(XLogRecPtr record)
{
    // 确保record及之前的所有WAL都已写入磁盘
    XLogWrite(WriteRqst, ...);
    //   └─> write(wal_fd, buffer, size)
    //       fsync(wal_fd)  ← 强制刷盘!
}
```

---

### 2.5 磁盘管理层 (Disk Management)

#### Storage Manager

**接口**: `src/backend/storage/smgr/smgr.c`

```c
void smgrread(SMgrRelation reln, ForkNumber forknum,
              BlockNumber blocknum, char *buffer)
{
    mdread(reln, forknum, blocknum, buffer);
}

void smgrwrite(SMgrRelation reln, ForkNumber forknum,
               BlockNumber blocknum, char *buffer, bool skipFsync)
{
    mdwrite(reln, forknum, blocknum, buffer, skipFsync);
}
```

**实现**: `src/backend/storage/smgr/md.c`

```c
void mdread(SMgrRelation reln, ForkNumber forknum,
            BlockNumber blocknum, char *buffer)
{
    MdfdVec *v;
    off_t seekpos;
    int nbytes;

    // 1. 获取文件段
    v = _mdfd_getseg(reln, forknum, blocknum, ...);

    // 2. 计算偏移
    seekpos = (off_t) BLCKSZ * (blocknum % RELSEG_SIZE);

    // 3. 读取
    nbytes = FileRead(v->mdfd_vfd, buffer, BLCKSZ, seekpos, ...);
    //        └─> pg_pread(fd, buffer, 8192, offset)
    //            └─> 系统调用: pread(fd, buf, count, offset)
}

void mdwrite(SMgrRelation reln, ForkNumber forknum,
             BlockNumber blocknum, char *buffer, bool skipFsync)
{
    MdfdVec *v;
    off_t seekpos;

    v = _mdfd_getseg(reln, forknum, blocknum, ...);
    seekpos = (off_t) BLCKSZ * (blocknum % RELSEG_SIZE);

    nbytes = FileWrite(v->mdfd_vfd, buffer, BLCKSZ, seekpos, ...);
    //        └─> pg_pwrite(fd, buffer, 8192, offset)
    //            └─> 系统调用: pwrite(fd, buf, count, offset)

    if (!skipFsync)
        register_dirty_segment(reln, forknum, v);
}
```

---

## 3. UPDATE语句的层次穿透

一条UPDATE语句如何穿透整个PostgreSQL架构?下面详细展示从SQL到磁盘的完整路径:

```
╔═════════════════════════════════════════════════════════════════════╗
║ SQL: UPDATE users SET name='Bob' WHERE id=1;                        ║
╚═════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────┐
│ 层1: 客户端层 (Client Layer)                                       │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
    psql> UPDATE users SET name='Bob' WHERE id=1;
           │
           ├─> libpq: PQexec()
           │     └─> 发送Query消息 (type='Q')
           │         通过TCP socket发送到postmaster
           ↓

┌─────────────────────────────────────────────────────────────────────┐
│ 层2: 进程管理层 (Process Management)                               │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
    Postmaster进程
      └─> accept()连接 → fork() Backend进程
            │
    Backend进程:
      ├─> PostgresMain() 主循环
      │     └─> ReadCommand() 接收SQL
      │           └─> exec_simple_query()  ← 进入查询处理
      ↓

┌─────────────────────────────────────────────────────────────────────┐
│ 层3: 查询处理层 (Query Processing)                                 │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
【子层3.1: Parser - 解析器】
      │
      ├─> raw_parser()  [scan.l + gram.y]
      │     Input:  "UPDATE users SET name='Bob' WHERE id=1"
      │     Output: UpdateStmt {
      │               relation = RangeVar("users"),
      │               targetList = [name='Bob'],
      │               whereClause = OpExpr(id=1)
      │             }
      │
      ├─> parse_analyze()  [analyze.c]
      │     └─> transformUpdateStmt()
      │           ├─ 解析表名 "users"
      │           │   └─> SearchSysCache(RELNAMENSP, "users")
      │           │       └─> 在pg_class中查找 → OID 16385
      │           ├─ 检查权限: pg_class_aclcheck()
      │           ├─ 转换SET子句 → TargetEntry列表
      │           └─ 转换WHERE子句 → Expr树
      │     Output: Query {
      │               commandType = CMD_UPDATE,
      │               resultRelation = 1 (RTEIndex),
      │               targetList = [...],
      │               whereClause = ...
      │             }
      ↓

【子层3.2: Rewriter - 重写器】
      │
      ├─> QueryRewrite()  [rewriteHandler.c]
      │     └─> fireRIRrules()
      │           ├─ 检查是否有VIEW (如果users是视图)
      │           │   └─> 展开视图定义
      │           ├─ 检查是否有RULE
      │           │   └─> 应用规则重写
      │           └─ 检查RLS策略
      │               └─> 添加安全过滤条件
      │     Output: List of Query (通常只有一个)
      ↓

【子层3.3: Planner - 优化器】
      │
      ├─> standard_planner()  [planner.c]
      │     │
      │     ├─> 生成扫描路径:
      │     │     1. SeqScan Path
      │     │        Cost: startup=0, total=15.0 (假设100行)
      │     │     2. IndexScan Path on users_pkey (id列索引)
      │     │        Cost: startup=0.29, total=8.30
      │     │        ★ 选中此路径!
      │     │
      │     └─> create_plan()
      │           └─> ModifyTable Plan
      │                 └─> IndexScan Plan
      │                       index: users_pkey
      │                       indexqual: (id = 1)
      │     Output: PlannedStmt {
      │               planTree = ModifyTable {
      │                 operation = UPDATE,
      │                 plans = [IndexScan on users_pkey]
      │               }
      │             }
      ↓

【子层3.4: Executor - 执行器】
      │
      ├─> CreateQueryDesc()
      │     └─> 包装PlannedStmt为QueryDesc
      │
      ├─> ExecutorStart()  [execMain.c]
      │     └─> InitPlan()
      │           └─> ExecInitModifyTable()
      │                 ├─ 打开表: table_open()
      │                 │   └─> relation_open(16385, RowExclusiveLock)
      │                 │         • 在RelCache中查找
      │                 │         • 加RowExclusiveLock锁
      │                 ├─ 打开索引: ExecOpenIndices()
      │                 │   └─> index_open(users_pkey_oid, RowExclusiveLock)
      │                 └─> ExecInitIndexScan()
      │                       └─> index_beginscan()
      │
      ├─> ExecutorRun()  [execMain.c]
      │     └─> ExecutePlan()
      │           └─> ExecProcNode() 递归执行
      │                 └─> ExecModifyTable()  [nodeModifyTable.c]
      │                       │
      │                       ├─ 第1步: 扫描要更新的行
      │                       │   └─> ExecProcNode(subplan)
      │                       │         └─> ExecIndexScan()
      │                       │               └─> index_getnext_tid()
      │                       │                     ↓ 调用索引AM
      │                       │                   btgettuple()  ← B-tree AM
      │                       │                     └─> 返回 TID = (0, 5)
      │                       │                           表示block 0, offset 5
      │                       │
      │                       ├─ 第2步: 执行更新
      │                       │   └─> ExecUpdate()  [nodeModifyTable.c]
      │                       │         │
      │                       │         ├─ BEFORE ROW触发器
      │                       │         │   ExecBRUpdateTriggers()
      │                       │         │
      │                       │         ├─ ★核心更新★
      │                       │         │   table_tuple_update()
      │                       │         │     └─> heapam_tuple_update()
      │                       │         │           └─> heap_update()  ← 见下层
      │                       │         │
      │                       │         ├─ 更新索引
      │                       │         │   ExecInsertIndexTuples()
      │                       │         │     └─> index_insert()
      │                       │         │           └─> btinsert() [B-tree]
      │                       │         │
      │                       │         └─ AFTER ROW触发器
      │                       │             ExecARUpdateTriggers()
      │                       │
      │                       └─ 返回: affected rows = 1
      ↓

┌─────────────────────────────────────────────────────────────────────┐
│ 层4: 存储管理层 (Storage Management)                               │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
【Access Method - heap_update()】
      │
      └─> heap_update(relation, old_tid=(0,5), newtup, ...)
            [heapam.c:3189行起]
            │
            ├─ 步骤1: 读取旧元组所在页面
            │   └─> ReadBuffer(relation, blocknum=0)  ← 调用Buffer Manager
            │         └─> ReadBuffer_common()
            │               ├─ BufTableLookup() 查找hash表
            │               │   └─ HIT! 在buffer pool中找到
            │               └─> PinBuffer() 增加引用计数
            │                     返回 Buffer ID = 123
            │
            ├─ 步骤2: 加锁页面
            │   └─> LockBuffer(buffer=123, BUFFER_LOCK_EXCLUSIVE)
            │
            ├─ 步骤3: 定位旧元组
            │   page = BufferGetPage(buffer);
            │   ItemId lp = PageGetItemId(page, offset=5);
            │   HeapTupleHeader oldtup = PageGetItem(page, lp);
            │
            ├─ 步骤4: MVCC可见性检查
            │   └─> HeapTupleSatisfiesUpdate(&oldtup, curcid, buffer)
            │         检查 t_xmin, t_xmax, 当前事务ID
            │         → 返回 TM_Ok (可以更新)
            │
            ├─ 步骤5: 判断是否HOT更新
            │   if (未修改索引列 && 同一页面有空间)
            │       use_hot_update = true;  ← 优化!
            │   else
            │       use_hot_update = false;
            │
            ├─ 步骤6: 在页面中写入新元组
            │   └─> RelationPutHeapTuple(relation, buffer, newtup)
            │         └─> 找到空闲空间 (offset=6)
            │             memcpy新元组数据到页面
            │             更新page header
            │
            ├─- 步骤7: 更新旧元组的xmax和ctid
            │   oldtup->t_xmax = GetCurrentTransactionId();  // 如 XID 1001
            │   oldtup->t_ctid = (0, 6);  // 指向新版本
            │   if (use_hot_update)
            │       HeapTupleSetHotUpdated(oldtup);
            │
            ├─ 步骤8: 标记buffer为脏页
            │   └─> MarkBufferDirty(buffer=123)
            │         设置 BM_DIRTY 标志
            │
            ├─ 步骤9: ★记录WAL日志★
            │   └─> log_heap_update()
            │         └─> XLogInsert(RM_HEAP_ID, XLOG_HEAP_UPDATE)
            │               ├─ 准备xl_heap_update结构
            │               │   • old_tid = (0, 5)
            │               │   • new_tid = (0, 6)
            │               │   • 新元组数据
            │               ├─ XLogBeginInsert()
            │               ├─ XLogRegisterData(...) 注册数据
            │               ├─ XLogRegisterBuffer(buffer) 注册buffer
            │               └─> XLogInsert()  ← 见WAL Manager
            │                     返回 LSN = 0/1A2B3C4D
            │
            ├─ 步骤10: 更新页面LSN
            │   PageSetLSN(page, LSN=0/1A2B3C4D);
            │
            ├─ 步骤11: 解锁并释放buffer
            │   UnlockReleaseBuffer(buffer);
            │
            └─ 返回: TM_Ok, update_indexes = (HOT ? TU_None : TU_All)

【WAL Manager - XLogInsert()】
      │
      └─> XLogInsert(RM_HEAP_ID, XLOG_HEAP_UPDATE)
            [xlog.c]
            │
            ├─ XLogRecordAssemble() 组装记录
            │     ├─ xl_heap_update 主记录
            │     ├─ 新元组数据块
            │     └─ buffer引用
            │
            ├─ ReserveXLogInsertLocation() 分配WAL空间
            │     在WAL buffer中分配 ~200 bytes
            │
            ├─- COMP_CRC32C() 计算CRC校验
            │
            ├─ CopyXLogRecordToWAL() 复制到WAL buffers
            │
            └─> 返回 LSN = 0/1A2B3C4D

【Buffer Manager】
      │
      当事务提交时:
      │
      └─> Background Writer / Checkpointer
            └─> 异步刷写脏页到磁盘
                  └─> FlushBuffer(buffer=123)
                        ↓ 调用磁盘层

┌─────────────────────────────────────────────────────────────────────┐
│ 层5: 磁盘管理层 (Disk Management)                                  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
【Storage Manager】
      │
      └─> FlushBuffer(bufHdr)
            └─> smgrwrite()  [smgr.c]
                  └─> mdwrite()  [md.c]
                        │
                        ├─ 确定文件路径:
                        │   relpath(rnode, MAIN_FORKNUM)
                        │   → "$PGDATA/base/16384/16385"
                        │        数据库OID ─┘      └─ 表的relfilenode
                        │
                        ├─ 打开/获取文件描述符:
                        │   _mdfd_getseg(reln, MAIN_FORKNUM, blocknum=0)
                        │     └─> PathNameOpenFile()
                        │           └─> VFD系统管理文件描述符
                        │                 fd = 15
                        │
                        ├─ 计算文件偏移:
                        │   seekpos = blocknum * BLCKSZ
                        │            = 0 * 8192 = 0
                        │
                        ├─ 写入8KB页面:
                        │   FileWrite(vfd=15, buffer, size=8192, offset=0)
                        │     └─> pg_pwrite()
                        │           └─> 系统调用: pwrite(fd=15, buf, 8192, 0)
                        │
                        └─ 注册dirty segment:
                            register_dirty_segment()
                              └─> 稍后checkpoint时fsync

【WAL写入 - 事务提交时】
      │
      └─> CommitTransaction()
            └─> XLogFlush(LSN=0/1A2B3C4D)
                  [xlog.c]
                  │
                  ├─ XLogWrite() 写WAL buffer到文件
                  │   │
                  │   ├─ 确定WAL文件:
                  │   │   XLogFileInit()
                  │   │   → "$PGDATA/pg_wal/000000010000000000000001"
                  │   │
                  │   └─> write(wal_fd, buffer, size)
                  │         系统调用写WAL
                  │
                  └─> fsync(wal_fd)  ★强制刷盘!★
                        确保WAL已持久化到磁盘

┌─────────────────────────────────────────────────────────────────────┐
│ Operating System (操作系统)                                        │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
    系统调用:
      ├─ pwrite(fd, buffer, 8192, offset)
      │   → 写数据页到: base/16384/16385
      │
      ├─ write(wal_fd, buffer, size)
      │   → 写WAL到: pg_wal/000000010000000000000001
      │
      └─ fsync(wal_fd)
            ↓
    文件系统 (ext4/xfs):
      └─> Page Cache → I/O Scheduler → 块设备
                                          ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Physical Disk (物理磁盘)                                            │
│   SSD / HDD / NVMe                                                  │
│                                                                     │
│   磁盘布局:                                                         │
│   /var/lib/postgresql/17/main/                                     │
│     ├─ base/16384/16385          ← 表数据文件                      │
│     │    (Page 0 包含更新后的元组)                                 │
│     ├─ base/16384/16385_fsm       ← Free Space Map                 │
│     ├─ base/16384/16385_vm        ← Visibility Map                 │
│     └─ pg_wal/000000010000000000000001  ← WAL日志                  │
│          (包含本次UPDATE的日志记录)                                │
└─────────────────────────────────────────────────────────────────────┘

【最终结果】

客户端收到:
    UPDATE 1   ← 更新了1行

磁盘上:
    ├─ 表文件: base/16384/16385
    │    Page 0:
    │      ├─ Line Pointer 5: 旧元组 (t_xmax=1001, t_ctid=(0,6))
    │      └─ Line Pointer 6: 新元组 (t_xmin=1001, name='Bob')
    │
    ├─ 索引文件: base/16384/16386 (users_pkey)
    │    B-tree结构指向新元组 TID = (0, 6)
    │
    └─ WAL日志: pg_wal/000000010000000000000001
         记录: XLOG_HEAP_UPDATE {old=(0,5), new=(0,6), data=...}

事务提交后,此UPDATE永久生效!
```

---

## 4. 模块间交互关系

PostgreSQL的各个模块通过明确定义的接口进行交互:

### 4.1 核心模块依赖图

```
                        ┌──────────────┐
                        │   Executor   │
                        │   执行器     │
                        └───────┬──────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
                ↓               ↓               ↓
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │  Access  │    │  Buffer  │    │   WAL    │
        │  Method  │───→│ Manager  │←───│ Manager  │
        │ 访问方法 │    │ 缓冲管理 │    │ WAL管理  │
        └────┬─────┘    └────┬─────┘    └────┬─────┘
             │               │               │
             └───────────────┼───────────────┘
                            ↓
                    ┌──────────────┐
                    │   Storage    │
                    │   Manager    │
                    │  存储管理器  │
                    └───────┬──────┘
                            │
                            ↓
                    ┌──────────────┐
                    │   Operating  │
                    │    System    │
                    │   操作系统   │
                    └──────────────┘
```

### 4.2 关键交互接口

#### Executor → Access Method
```c
// 执行器通过Table AM接口访问表
const TableAmRoutine *tableam = relation->rd_tableam;

// 更新元组
tableam->tuple_update(relation, tupleid, slot, cid, ...);
  └─> 对于heap表,调用 heapam_tuple_update()
        └─> heap_update()
```

#### Access Method → Buffer Manager
```c
// heap_update() 调用缓冲区管理器
Buffer buffer = ReadBuffer(relation, blocknum);
LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);
Page page = BufferGetPage(buffer);
... // 修改页面
MarkBufferDirty(buffer);
UnlockReleaseBuffer(buffer);
```

#### Access Method → WAL Manager
```c
// 记录WAL
XLogBeginInsert();
XLogRegisterData(&xlrec, sizeof(xlrec));
XLogRegisterBuffer(0, buffer, REGBUF_STANDARD);
XLogRecPtr recptr = XLogInsert(RM_HEAP_ID, XLOG_HEAP_UPDATE);
PageSetLSN(page, recptr);
```

#### Buffer Manager → Storage Manager
```c
// 缓冲区管理器读取页面
void ReadBuffer_common(...)
{
    ... // buffer miss
    smgrread(smgr, forknum, blocknum, buffer);
    //  └─> mdread() → pread()
}

// 刷写脏页
void FlushBuffer(...)
{
    smgrwrite(smgr, forknum, blocknum, buffer, false);
    //  └─> mdwrite() → pwrite()
}
```

#### Storage Manager → VFD → OS
```c
// md.c 使用VFD管理文件
int FileWrite(File file, char *buffer, int amount, off_t offset, ...)
{
    ... // VFD查找实际fd
    return pg_pwrite(vfdP->fd, buffer, amount, offset);
    //       └─> 系统调用 pwrite()
}
```

### 4.3 UPDATE语句的模块调用链

```
ExecutorRun()
  └─> ExecModifyTable()
        └─> ExecUpdate()
              ├─> table_tuple_update()  [Table AM接口]
              │     └─> heap_update()  [Heap AM实现]
              │           ├─> ReadBuffer()  [Buffer Manager]
              │           │     └─> smgrread()  [Storage Manager]
              │           │           └─> mdread()  [MD层]
              │           │                 └─> pread()  [OS]
              │           │
              │           ├─> XLogInsert()  [WAL Manager]
              │           │     └─> CopyXLogRecordToWAL()
              │           │
              │           └─> MarkBufferDirty()  [Buffer Manager]
              │
              └─> ExecInsertIndexTuples()
                    └─> index_insert()  [Index AM接口]
                          └─> btinsert()  [B-tree AM实现]
```

---

## 5. 关键子系统详解

### 5.1 RelFileNode - 文件定位的核心

**如何从表名找到磁盘文件?**

```c
// 1. 表名 → OID
SearchSysCache(RELNAMENSP, "users", namespace_oid)
  → 查询pg_class系统表
  → 返回 pg_class.oid = 16385

// 2. OID → RelFileNode
typedef struct RelFileNode {
    Oid spcNode;   // 表空间OID (如 pg_default = 1663)
    Oid dbNode;    // 数据库OID (如 mydb = 16384)
    Oid relNode;   // 表的relfilenode (如 16385)
} RelFileNode;

// 3. RelFileNode → 文件路径
char *relpath(RelFileNode rnode, ForkNumber forknum)
{
    if (rnode.spcNode == DEFAULTTABLESPACE_OID)
        return psprintf("base/%u/%u", rnode.dbNode, rnode.relNode);
        //      → "base/16384/16385"
    else
        return psprintf("pg_tblspc/%u/%s/%u/%u",
                        rnode.spcNode, TABLESPACE_VERSION_DIRECTORY,
                        rnode.dbNode, rnode.relNode);
}

// 4. 最终磁盘文件
$PGDATA/base/16384/16385        ← 主数据fork (MAIN_FORKNUM)
$PGDATA/base/16384/16385_fsm    ← Free Space Map
$PGDATA/base/16384/16385_vm     ← Visibility Map
$PGDATA/base/16384/16385.1      ← 第2个segment (如果>1GB)
```

### 5.2 页面结构 (Page Layout)

PostgreSQL的基本存储单位是8KB的页面:

```
┌─────────────────────────────────────────────────────────┐ ← 页面起始
│                   PageHeaderData (24 bytes)             │
│  ┌─────────────────────────────────────────────────┐   │
│  │ pd_lsn: LSN (8 bytes)        - 最后修改的WAL位置│   │
│  │ pd_checksum: uint16 (2)      - 页面校验和       │   │
│  │ pd_flags: uint16 (2)         - 标志位           │   │
│  │ pd_lower: uint16 (2)         - 空闲空间起始     │   │
│  │ pd_upper: uint16 (2)         - 空闲空间结束     │   │
│  │ pd_special: uint16 (2)       - 特殊空间偏移     │   │
│  │ pd_pagesize_version: uint16  - 页面大小和版本   │   │
│  │ pd_prune_xid: TransactionId  - 可清理的最小XID  │   │
│  └─────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────┤
│              Line Pointer Array (ItemIdData[])          │
│  每个4 bytes:                                           │
│    ┌──────────────────────────────────────┐            │
│    │ lp_off: 15 bits  - 元组偏移          │            │
│    │ lp_flags: 2 bits - 状态标志          │            │
│    │ lp_len: 15 bits  - 元组长度          │            │
│    └──────────────────────────────────────┘            │
│  [0] → offset=8064, len=120  (第1个元组)               │
│  [1] → offset=7944, len=120  (第2个元组)               │
│  [2] → offset=7824, len=120  (第3个元组)               │
│  ... (可以有约290个line pointers)                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│                   Free Space                            │
│                  (可用空间)                             │
│         ← pd_lower指向这里                              │
│         ← pd_upper指向这里                              │
│                                                         │
├─────────────────────────────────────────────────────────┤ ← pd_upper
│                   Tuple Data                            │
│                  (元组数据)                             │
│  ┌──────────────────────────────────────────────┐      │
│  │ HeapTupleHeaderData (23 bytes)               │      │
│  │  ├─ t_xmin: TransactionId    - 插入事务      │      │
│  │  ├─ t_xmax: TransactionId    - 删除/更新事务 │      │
│  │  ├─ t_cid: CommandId         - 命令ID        │      │
│  │  ├─ t_ctid: ItemPointerData  - 新版本位置    │      │
│  │  ├─ t_infomask: uint16       - 标志位        │      │
│  │  ├─ t_hoff: uint8            - header大小    │      │
│  │  └─ t_bits[]: null bitmap    - NULL标志      │      │
│  ├──────────────────────────────────────────────┤      │
│  │ User Data (实际列数据)                       │      │
│  │  • column1 value                             │      │
│  │  • column2 value                             │      │
│  │  • ...                                       │      │
│  └──────────────────────────────────────────────┘      │
│  (从高地址向低地址增长)                                 │
├─────────────────────────────────────────────────────────┤
│                Special Space (特殊空间)                 │
│  索引页使用,堆表页面为空                                │
└─────────────────────────────────────────────────────────┘ ← 页面结束 (8KB)
```

### 5.3 MVCC实现机制

PostgreSQL通过在每个元组头部记录事务信息实现MVCC:

```c
typedef struct HeapTupleHeaderData {
    TransactionId t_xmin;  // 插入此版本的事务ID
    TransactionId t_xmax;  // 删除/更新此版本的事务ID (0表示活跃)
    CommandId t_cid;       // 插入/删除的命令ID
    ItemPointerData t_ctid; // 指向新版本的TID (UPDATE时使用)
    uint16 t_infomask;     // 标志位
    uint8 t_hoff;          // header大小
    bits8 t_bits[FLEXIBLE_ARRAY_MEMBER];  // NULL bitmap
} HeapTupleHeaderData;
```

**可见性判断** (HeapTupleSatisfiesMVCC):

```c
bool HeapTupleSatisfiesMVCC(HeapTuple htup, Snapshot snapshot, Buffer buffer)
{
    TransactionId xmin = HeapTupleHeaderGetXmin(tuple);
    TransactionId xmax = HeapTupleHeaderGetXmax(tuple);

    // 1. 检查t_xmin (插入事务)
    if (xmin >= snapshot->xmax)
        return false;  // 在快照之后插入,不可见

    if (!TransactionIdDidCommit(xmin))
        return false;  // 插入事务未提交,不可见

    // 2. 检查t_xmax (删除/更新事务)
    if (!TransactionIdIsValid(xmax))
        return true;  // 未被删除/更新,可见

    if (xmax >= snapshot->xmax)
        return true;  // 删除事务在快照之后,仍可见

    if (!TransactionIdDidCommit(xmax))
        return true;  // 删除事务未提交,仍可见

    return false;  // 已被删除/更新,不可见
}
```

**UPDATE示例**:

```
Before UPDATE (XID 1000):
  Page 0, Offset 5: (t_xmin=999, t_xmax=0, t_ctid=(0,5), name='Alice')
                      ↑插入此行   ↑活跃       ↑指向自己

During UPDATE (XID 1001):
  Page 0, Offset 5: (t_xmin=999, t_xmax=1001, t_ctid=(0,6), name='Alice')
                                     ↑本事务      ↑指向新版本
  Page 0, Offset 6: (t_xmin=1001, t_xmax=0, t_ctid=(0,6), name='Bob')
                      ↑本事务插入  ↑活跃       ↑指向自己

After COMMIT:
  XID 1001 的状态 → COMMITTED (在CLOG中)

  对于快照xmax=1000的事务:
    Offset 5可见 (xmax=1001 > snapshot.xmax)

  对于快照xmax=1002的事务:
    Offset 5不可见 (xmax=1001已提交且 < snapshot.xmax)
    Offset 6可见 (xmin=1001已提交且 < snapshot.xmax)
```

---

## 6. 源码目录结构

PostgreSQL源码的关键目录布局:

```
postgresql-17.6/
├─ src/
│  ├─ backend/          ← 服务器端核心代码
│  │  ├─ access/        ← 访问方法
│  │  │  ├─ common/     ← 通用访问方法代码
│  │  │  ├─ heap/       ← 堆表访问方法 ★
│  │  │  │  ├─ heapam.c       ← heap_update()等
│  │  │  │  ├─ heapam_handler.c
│  │  │  │  └─ hio.c          ← Heap I/O
│  │  │  ├─ index/      ← 索引通用代码
│  │  │  ├─ nbtree/     ← B-tree索引 ★
│  │  │  │  ├─ nbtsearch.c    ← B-tree搜索
│  │  │  │  ├─ nbtinsert.c    ← B-tree插入
│  │  │  │  └─ nbtpage.c      ← B-tree页面管理
│  │  │  ├─ hash/       ← Hash索引
│  │  │  ├─ gin/        ← GIN索引
│  │  │  ├─ gist/       ← GiST索引
│  │  │  ├─ spgist/     ← SP-GiST索引
│  │  │  ├─ brin/       ← BRIN索引
│  │  │  └─ transam/    ← 事务管理 ★
│  │  │     ├─ xact.c         ← CommitTransaction()等
│  │  │     ├─ xlog.c         ← WAL管理 XLogInsert()
│  │  │     ├─ clog.c         ← Commit Log
│  │  │     └─ multixact.c    ← MultiXact
│  │  │
│  │  ├─ parser/        ← 解析器 ★
│  │  │  ├─ scan.l            ← 词法分析(Flex)
│  │  │  ├─ gram.y            ← 语法分析(Bison)
│  │  │  ├─ analyze.c         ← 语义分析
│  │  │  └─ parse_utilcmd.c
│  │  │
│  │  ├─ rewrite/       ← 重写器 ★
│  │  │  └─ rewriteHandler.c  ← QueryRewrite()
│  │  │
│  │  ├─ optimizer/     ← 优化器 ★
│  │  │  ├─ path/             ← 路径生成
│  │  │  ├─ plan/             ← 计划生成
│  │  │  │  └─ planner.c      ← standard_planner()
│  │  │  └─ util/             ← 优化器工具
│  │  │
│  │  ├─ executor/      ← 执行器 ★
│  │  │  ├─ execMain.c        ← ExecutorRun()等
│  │  │  ├─ nodeModifyTable.c ← ExecModifyTable()
│  │  │  ├─ execIndexing.c    ← 索引更新
│  │  │  ├─ nodeSeqscan.c     ← 顺序扫描
│  │  │  ├─ nodeIndexscan.c   ← 索引扫描
│  │  │  └─ node*.c           ← 各种执行节点
│  │  │
│  │  ├─ storage/       ← 存储管理 ★
│  │  │  ├─ buffer/           ← 缓冲区管理
│  │  │  │  ├─ bufmgr.c       ← ReadBuffer()等
│  │  │  │  ├─ freelist.c     ← Clock-sweep算法
│  │  │  │  └─ buf_table.c    ← Buffer hash表
│  │  │  ├─ smgr/             ← 存储管理器接口
│  │  │  │  ├─ smgr.c         ← smgrread/write()
│  │  │  │  └─ md.c           ← 磁盘管理 ★
│  │  │  ├─ file/             ← 文件管理
│  │  │  │  └─ fd.c           ← VFD系统
│  │  │  ├─ freespace/        ← FSM (Free Space Map)
│  │  │  ├─ lmgr/             ← 锁管理
│  │  │  └─ page/             ← 页面操作
│  │  │
│  │  ├─ catalog/       ← 系统目录
│  │  │  ├─ pg_class.c
│  │  │  ├─ pg_type.c
│  │  │  └─ namespace.c       ← 命名空间查找
│  │  │
│  │  ├─ utils/         ← 工具函数
│  │  │  ├─ cache/            ← 缓存系统
│  │  │  │  ├─ syscache.c     ← 系统目录缓存
│  │  │  │  └─ relcache.c     ← Relation缓存
│  │  │  ├─ mmgr/             ← 内存管理
│  │  │  │  ├─ mcxt.c         ← MemoryContext
│  │  │  │  └─ aset.c         ← AllocSet
│  │  │  └─ time/             ← 时间/快照
│  │  │     └─ snapmgr.c      ← Snapshot管理
│  │  │
│  │  ├─ postmaster/    ← 进程管理
│  │  │  ├─ postmaster.c      ← 主守护进程
│  │  │  ├─ bgwriter.c        ← Background Writer
│  │  │  ├─ walwriter.c       ← WAL Writer
│  │  │  ├─ checkpointer.c    ← Checkpointer
│  │  │  └─ autovacuum.c      ← Autovacuum
│  │  │
│  │  └─ tcop/          ← 流量控制/命令处理
│  │     └─ postgres.c        ← PostgresMain()
│  │
│  ├─ include/          ← 头文件
│  │  ├─ access/
│  │  │  ├─ heapam.h
│  │  │  ├─ htup_details.h    ← HeapTupleHeaderData定义
│  │  │  └─ xlog.h
│  │  ├─ storage/
│  │  │  ├─ bufmgr.h
│  │  │  ├─ smgr.h
│  │  │  └─ buf_internals.h   ← BufferDesc定义
│  │  ├─ nodes/
│  │  │  └─ parsenodes.h      ← UpdateStmt定义
│  │  └─ executor/
│  │     └─ executor.h
│  │
│  └─ bin/              ← 客户端工具
│     └─ psql/          ← psql命令行工具
│
└─ contrib/             ← 扩展模块
   └─ pg_stat_statements/
```

**UPDATE相关的关键文件**:

| 文件路径 | 关键函数/结构 | 说明 |
|---------|-------------|------|
| `src/backend/parser/gram.y` | UpdateStmt语法规则 | UPDATE语法定义 |
| `src/backend/parser/analyze.c` | transformUpdateStmt() | UPDATE语义分析 |
| `src/backend/optimizer/plan/planner.c` | standard_planner() | 生成执行计划 |
| `src/backend/executor/nodeModifyTable.c` | ExecUpdate() | UPDATE执行 |
| `src/backend/access/heap/heapam.c` | heap_update() | 堆表UPDATE实现 |
| `src/backend/storage/buffer/bufmgr.c` | ReadBuffer() | 缓冲区管理 |
| `src/backend/access/transam/xlog.c` | XLogInsert() | WAL日志记录 |
| `src/backend/storage/smgr/md.c` | mdwrite() | 磁盘写入 |
| `src/include/access/htup_details.h` | HeapTupleHeaderData | 元组头部结构 |

---

## 7. 阅读建议

### 7.1 文档阅读顺序

**对于初学者**:
1. 先读本文档 (00号) - 理解整体架构
2. 再读 **UPDATE流程图集.md** (03号) - 通过图形化理解流程
3. 然后读 **UPDATE示例跟踪.md** (04号) - 通过实例巩固理解
4. 最后读 **UPDATE语句执行流程完整分析.md** (02号) - 深入细节

**对于有经验的开发者**:
1. 直接阅读 **UPDATE语句执行流程完整分析.md** (02号)
2. 需要时参考 **UPDATE数据结构详解.md** (05号)
3. 性能优化参考 **UPDATE性能优化指南.md** (06号)
4. 存储细节参考 **UPDATE磁盘存储布局详解.md** (07号) 和 **UPDATE文件定位与扫描机制详解.md** (08号)

### 7.2 源码阅读路线

**路线1: 自顶向下**
```
PostgresMain() (tcop/postgres.c)
  → exec_simple_query()
    → pg_parse_query() (parser/)
      → QueryRewrite() (rewrite/)
        → pg_plan_queries() (optimizer/)
          → PortalRun() + ExecutorRun() (executor/)
            → ExecModifyTable()
              → heap_update() (access/heap/)
```

**路线2: 自底向上**
```
系统调用 pwrite()
  → FileWrite() (storage/file/fd.c)
    → mdwrite() (storage/smgr/md.c)
      → smgrwrite() (storage/smgr/smgr.c)
        → FlushBuffer() (storage/buffer/bufmgr.c)
          → MarkBufferDirty()
            → heap_update() (access/heap/heapam.c)
```

**路线3: 按层次**
1. **解析层**: scan.l → gram.y → analyze.c
2. **重写层**: rewriteHandler.c
3. **优化层**: planner.c → 各种cost_*函数
4. **执行层**: execMain.c → nodeModifyTable.c
5. **存储层**: heapam.c → bufmgr.c → md.c

### 7.3 调试建议

**启用详细日志**:
```sql
-- postgresql.conf
log_statement = 'all'
log_min_messages = DEBUG5
log_min_duration_statement = 0
```

**使用GDB调试**:
```bash
# 编译debug版本
./configure --enable-debug CFLAGS="-O0 -g3"
make && make install

# 启动并attach到backend进程
ps aux | grep postgres
gdb -p <backend_pid>

# 设置断点
(gdb) break heap_update
(gdb) break XLogInsert
(gdb) continue
```

**查看执行计划**:
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
UPDATE users SET name='Bob' WHERE id=1;
```

### 7.4 扩展阅读

**官方文档**:
- PostgreSQL Documentation: https://www.postgresql.org/docs/17/
- Chapter 73: Database Physical Storage
- Chapter 30: Write-Ahead Logging

**推荐资料**:
- *PostgreSQL Internals* by Hironobu Suzuki
- *The Internals of PostgreSQL* (online book)
- PostgreSQL源码中的 `README` 文件

**重要博客和论文**:
- Postgres Professional Blog
- MVCC in PostgreSQL系列
- PostgreSQL Wiki: https://wiki.postgresql.org/

---

## 总结

本文档作为PostgreSQL UPDATE语句系列文档的架构总览,介绍了:

1. **五层架构**: 客户端层 → 进程管理层 → 查询处理层 → 存储管理层 → 磁盘管理层
2. **查询处理四子层**: Parser → Rewriter → Planner → Executor
3. **UPDATE完整路径**: 从SQL文本到磁盘文件的所有步骤
4. **模块交互**: 各层之间的清晰接口和调用关系
5. **关键概念**: RelFileNode、Page Layout、MVCC、WAL等
6. **源码导航**: 34,000行代码的关键文件定位

接下来的7个文档将深入探讨UPDATE执行的各个方面,总计约34,000行详细技术文档。

**下一步**: 阅读 **UPDATE语句执行流程完整分析.md** 了解每个函数的详细实现!