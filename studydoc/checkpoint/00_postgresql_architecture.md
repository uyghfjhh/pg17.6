# PostgreSQL 17.5 核心架构全景分析

> 本文档系统性梳理 PostgreSQL 17.5 的整体架构，为深入学习各核心模块奠定基础。

---

## 目录

1. [PostgreSQL 架构概览](#一postgresql-架构概览)
2. [进程架构](#二进程架构)
3. [内存架构](#三内存架构)
4. [存储架构](#四存储架构)
5. [核心模块分类](#五核心模块分类)
6. [模块依赖关系](#六模块依赖关系)
7. [数据流转全景](#七数据流转全景)
8. [文档导航](#八文档导航)

---

## 一、PostgreSQL 架构概览

### 1.1 三层架构模型

```
┌────────────────────────────────────────────────────────────────┐
│                     应用层 (Applications)                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │  psql    │  │  JDBC    │  │  ODBC    │  │   App    │      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘      │
└───────┼─────────────┼─────────────┼─────────────┼─────────────┘
        │             │             │             │
        └─────────────┴─────────────┴─────────────┘
                      │ (TCP/IP 或 Unix Socket)
┌─────────────────────┼─────────────────────────────────────────┐
│                     ▼                                           │
│              PostgreSQL 服务器层                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  连接管理 (Postmaster + Backend Processes)               │ │
│  └──────────────────┬───────────────────────────────────────┘ │
│                     ▼                                           │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  SQL 处理层 (Parser → Rewriter → Planner → Executor)     │ │
│  └──────────────────┬───────────────────────────────────────┘ │
│                     ▼                                           │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  事务管理层 (MVCC + Transaction Manager + Lock Manager)  │ │
│  └──────────────────┬───────────────────────────────────────┘ │
│                     ▼                                           │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  存储管理层 (Buffer Manager + WAL + Storage Manager)     │ │
│  └──────────────────┬───────────────────────────────────────┘ │
└────────────────────┼─────────────────────────────────────────┘
                     ▼
┌────────────────────────────────────────────────────────────────┐
│                     物理存储层                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Data     │  │   WAL    │  │  CLOG    │  │  Index   │      │
│  │ Files    │  │  Files   │  │  Files   │  │  Files   │      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
└────────────────────────────────────────────────────────────────┘
```

### 1.2 核心设计原则

| 原则 | 说明 | 体现 |
|------|------|------|
| **Write-Ahead Logging** | 先写日志，后写数据 | WAL 机制确保持久性 |
| **MVCC** | 多版本并发控制 | 读写不阻塞，高并发 |
| **Process-per-Connection** | 每个连接一个进程 | 隔离性好，稳定性高 |
| **Shared-Nothing Storage** | 无共享存储架构 | 简化设计，易扩展 |
| **Extensibility** | 高度可扩展 | 自定义类型、函数、索引等 |

---

## 二、进程架构

### 2.1 进程全景图

```
                    ┌─────────────────────────────────────────┐
                    │      Postmaster (主守护进程)              │
                    │  - 监听端口 (默认 5432)                   │
                    │  - Fork 各类子进程                        │
                    │  - 管理进程生命周期                       │
                    └──────────────┬──────────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
    ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
    │  Backend 进程    │  │  辅助进程        │  │  后台工作进程    │
    │  (处理客户端请求) │  │  (系统维护)      │  │  (后台任务)      │
    └─────────────────┘  └─────────────────┘  └─────────────────┘
            │                     │                     │
    ┌───────┴───────┐    ┌────────┴────────┐   ┌────────┴────────┐
    │               │    │                 │   │                 │
    ▼               ▼    ▼                 ▼   ▼                 ▼
┌────────┐    ┌────────┐ │                 │   │                 │
│Backend │    │Backend │ │                 │   │                 │
│ Proc 1 │    │ Proc 2 │ │  Checkpointer   │   │  Bgwriter       │
└────────┘    └────────┘ │  WAL Writer     │   │  Autovacuum     │
                         │  WAL Sender     │   │  Logical Worker │
                         │  WAL Receiver   │   │  Parallel Worker│
                         │  Archiver       │   │  Stats Collector│
                         │  Logger         │   └─────────────────┘
                         └─────────────────┘
```

### 2.2 进程详细说明

#### 2.2.1 Postmaster (主进程)

**源码位置**: `src/backend/postmaster/postmaster.c:1250 PostmasterMain()`

**职责**:
1. **监听连接**: 监听 TCP/IP 端口和 Unix Socket
2. **Fork Backend**: 为每个客户端连接创建独立的 Backend 进程
3. **启动辅助进程**: 启动所有后台辅助进程
4. **进程监控**: 监控子进程健康状态，崩溃时重启
5. **信号处理**: 处理 SIGHUP (重新加载配置)、SIGTERM (关闭) 等

**生命周期**:
```
pg_ctl start
    ↓
启动 Postmaster
    ↓
加载配置文件 (postgresql.conf)
    ↓
初始化共享内存
    ↓
启动所有辅助进程
    ↓
进入主循环，监听连接
    ↓
pg_ctl stop
    ↓
优雅关闭所有子进程
```

#### 2.2.2 Backend 进程 (工作进程)

**源码位置**: `src/backend/tcop/postgres.c:4280 PostgresMain()`

**职责**:
1. **接收 SQL**: 从客户端接收 SQL 命令
2. **解析 SQL**: Parser → Rewriter → Planner → Executor
3. **执行查询**: 调用存储引擎读写数据
4. **返回结果**: 将结果返回给客户端
5. **事务管理**: 管理事务的开始、提交、回滚

**关键特点**:
- **一对一**: 一个 Backend 进程服务一个客户端连接
- **独立内存**: 每个进程有独立的本地内存 (work_mem)
- **共享内存访问**: 通过共享内存访问缓冲池和锁表

**执行流程**:
```
客户端连接
    ↓
Postmaster fork() 新进程
    ↓
Backend 进程初始化
    ↓
接收 SQL 命令
    ↓
┌─────────────────────────────────┐
│ 1. Parser (gram.y)              │  将 SQL 解析为语法树
│    src/backend/parser/          │
├─────────────────────────────────┤
│ 2. Rewriter (rewriteHandler.c)  │  应用规则和视图重写
│    src/backend/rewrite/         │
├─────────────────────────────────┤
│ 3. Planner (planner.c)          │  生成最优执行计划
│    src/backend/optimizer/       │
├─────────────────────────────────┤
│ 4. Executor (execMain.c)        │  执行计划，返回结果
│    src/backend/executor/        │
└─────────────────────────────────┘
    ↓
返回结果给客户端
```

#### 2.2.3 辅助进程 (Auxiliary Processes)

| 进程名 | 源码位置 | 职责 | 数量 |
|--------|----------|------|------|
| **Checkpointer** | postmaster/checkpointer.c:173 | 定期执行 checkpoint，刷写脏页 | 1 |
| **WAL Writer** | postmaster/walwriter.c:100 | 异步刷写 WAL 缓冲区到磁盘 | 1 |
| **Bgwriter** | postmaster/bgwriter.c:150 | 后台写脏页，减轻 checkpoint 压力 | 1 |
| **Autovacuum Launcher** | postmaster/autovacuum.c:400 | 自动启动 VACUUM 任务 | 1 |
| **Stats Collector** | postmaster/pgstat.c:500 | 收集数据库统计信息 | 1 |
| **Logger** | postmaster/syslogger.c:200 | 管理日志文件 | 0-1 |
| **Archiver** | postmaster/pgarch.c:150 | 归档已完成的 WAL 段文件 | 0-1 |
| **WAL Sender** | replication/walsender.c:2000 | 发送 WAL 到备库 | 0-N |
| **WAL Receiver** | replication/walreceiver.c:200 | (备库) 接收 WAL | 0-1 |
| **Startup** | postmaster/startup.c:200 | (恢复中) 重放 WAL 日志 | 0-1 |

#### 2.2.4 后台工作进程 (Background Workers)

**特点**: 动态创建，可自定义

| 类型 | 用途 | 示例 |
|------|------|------|
| **Autovacuum Worker** | 执行 VACUUM 清理 | 自动清理死元组 |
| **Parallel Worker** | 并行查询执行 | 并行扫描、聚合、排序 |
| **Logical Replication Worker** | 逻辑复制 | 订阅端应用变更 |
| **自定义 Worker** | 扩展功能 | 通过 bgworker API 注册 |

### 2.3 进程间通信 (IPC)

```
┌───────────────────────────────────────────────────────────────┐
│                      共享内存 (Shared Memory)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │ Shared Buffer│  │  Lock Table  │  │  Proc Array  │        │
│  │    Pool      │  │              │  │              │        │
│  │  (默认 128MB) │  │  (锁管理器)   │  │ (进程信息)    │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │  WAL Buffers │  │  CLOG Buffer │  │  Checkpointer│        │
│  │   (16MB)     │  │              │  │    Shmem     │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
└───────────────────────────────────────────────────────────────┘
         ▲                    ▲                    ▲
         │                    │                    │
    ┌────┴────┐          ┌────┴────┐         ┌────┴────┐
    │Backend 1│          │Backend 2│         │Checkpntr│
    └─────────┘          └─────────┘         └─────────┘

其他 IPC 机制:
- Semaphores (信号量): 用于锁同步
- Signals (信号): 进程间通知 (如 SIGINT 触发 checkpoint)
- Pipes (管道): Logger 进程日志传输
- Message Queue: Parallel Query 协调
```

---

## 三、内存架构

### 3.1 内存全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                   PostgreSQL 内存架构                             │
└─────────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┴────────────────────┐
         │                                         │
         ▼                                         ▼
┌──────────────────────┐              ┌──────────────────────┐
│   共享内存 (Shared)   │              │  本地内存 (Local)     │
│   所有进程共享        │              │  每个进程独立         │
└──────────────────────┘              └──────────────────────┘
         │                                         │
    ┌────┴────┐                          ┌─────────┴─────────┐
    ▼         ▼                          ▼                   ▼
```

### 3.2 共享内存详解

**总大小计算**:
```c
// src/backend/storage/ipc/ipci.c:83
total_shared_memory =
    BufferPoolSize              // shared_buffers
  + WalBufferSize               // wal_buffers
  + CLOGBufferSize              // CLOG 缓冲
  + LockSpaceSize               // max_locks_per_transaction
  + ProcArraySize               // max_connections
  + CheckpointerShmemSize       // Checkpointer 状态
  + AutoVacuumShmemSize         // Autovacuum 状态
  + ... (其他各模块)
```

#### 3.2.1 Shared Buffers (共享缓冲池)

**配置**: `shared_buffers = 128MB` (默认)

**结构**:
```
shared_buffers 内存布局:
┌──────────────────────────────────────────────────┐
│  Buffer Descriptor Array (BufferDescriptors)    │  控制信息
│  每个 BufferDesc: 64 字节                        │
│  数量: shared_buffers / 8KB                      │
├──────────────────────────────────────────────────┤
│  Buffer Pool (BufferBlocks)                      │  实际数据
│  每个 Block: 8KB                                 │
│  存储数据页内容                                   │
└──────────────────────────────────────────────────┘

例如 shared_buffers = 128MB:
- 页面数: 128MB / 8KB = 16384 个页
- BufferDesc 数组: 16384 * 64B = 1MB
- 实际数据: 128MB
- 总计: ~129MB
```

**管理算法**: Clock-Sweep (时钟扫描算法)
- 源码: `src/backend/storage/buffer/freelist.c:150 StrategyGetBuffer()`

#### 3.2.2 WAL Buffers

**配置**: `wal_buffers = 16MB` (默认，自动根据 shared_buffers 调整)

**结构**:
```
┌────────────────────────────────────┐
│  WAL 缓冲区 (XLogCtlData->pages)   │
│  循环使用的环形缓冲区               │
│                                    │
│  Insert Ptr  ──>  [WAL Record]    │  当前写入位置
│                   [WAL Record]    │
│  Flush Ptr   ──>  [WAL Record]    │  已刷到磁盘位置
│                   [Free Space]    │
│                   [Free Space]    │
└────────────────────────────────────┘
```

#### 3.2.3 Lock Table (锁表)

**配置**: `max_locks_per_transaction = 64` (默认)

**总大小**: `max_locks_per_transaction * (max_connections + max_prepared_transactions)`

**结构**:
```
src/backend/storage/lmgr/lock.c:200

┌───────────────────────────────────────────┐
│  LOCK Hash Table                          │
│  - Key: (database, relation, page, tuple) │
│  - Value: LOCK 结构                        │
├───────────────────────────────────────────┤
│  PROCLOCK Hash Table                      │
│  - Key: (LOCK, PGPROC)                    │
│  - Value: PROCLOCK 结构 (持有者信息)       │
└───────────────────────────────────────────┘
```

**锁类型**: 8 种锁模式
1. AccessShareLock (SELECT)
2. RowShareLock (SELECT FOR UPDATE)
3. RowExclusiveLock (INSERT/UPDATE/DELETE)
4. ShareUpdateExclusiveLock (VACUUM)
5. ShareLock (CREATE INDEX)
6. ShareRowExclusiveLock
7. ExclusiveLock (REFRESH MATERIALIZED VIEW)
8. AccessExclusiveLock (DROP TABLE, TRUNCATE)

### 3.3 本地内存详解

**每个 Backend 进程独立拥有**:

#### 3.3.1 work_mem (工作内存)

**配置**: `work_mem = 4MB` (默认)

**用途**:
- 排序操作 (ORDER BY, CREATE INDEX)
- 哈希表 (Hash Join, Hash Aggregate)
- 位图索引扫描
- Merge Join

**注意**: 一个查询可能使用多个 work_mem 块！

**示例**:
```sql
-- 这个查询可能使用 3 * work_mem:
SELECT *
FROM t1
  JOIN t2 USING (id)       -- 1 个 work_mem (hash join)
ORDER BY t1.name, t2.value -- 2 个 work_mem (2 个排序)
```

#### 3.3.2 maintenance_work_mem (维护内存)

**配置**: `maintenance_work_mem = 64MB` (默认)

**用途**:
- VACUUM
- CREATE INDEX
- ALTER TABLE ADD FOREIGN KEY
- CLUSTER

#### 3.3.3 temp_buffers (临时表缓冲)

**配置**: `temp_buffers = 8MB` (默认)

**用途**: 每个会话的临时表缓存

### 3.4 内存上下文 (Memory Context)

PostgreSQL 使用内存上下文管理内存，避免内存泄漏。

**源码**: `src/backend/utils/mmgr/mcxt.c:100`

**层次结构**:
```
TopMemoryContext (永不释放)
    │
    ├─ ErrorContext (错误处理)
    │
    ├─ PostmasterContext (Postmaster 全局)
    │
    ├─ CacheMemoryContext (系统缓存)
    │
    ├─ MessageContext (每条消息)
    │      └─ (处理完消息后释放)
    │
    ├─ TopTransactionContext (事务顶层)
    │      ├─ CurTransactionContext (当前事务)
    │      │      ├─ ExecutorState (执行器状态)
    │      │      ├─ TupleSort (排序)
    │      │      └─ ... (算子内存)
    │      │
    │      └─ (事务结束后释放)
    │
    └─ ... (其他上下文)
```

**优点**:
1. **批量释放**: 释放上下文时，自动释放其下所有内存
2. **层次清晰**: 内存生命周期明确
3. **防止泄漏**: 异常退出时自动清理

**示例**:
```c
// 创建临时上下文
MemoryContext tempContext = AllocSetContextCreate(
    CurrentMemoryContext,
    "My Temp Context",
    ALLOCSET_DEFAULT_SIZES
);

// 在临时上下文中分配内存
MemoryContext oldContext = MemoryContextSwitchTo(tempContext);
char *temp_data = palloc(1024);  // 在 tempContext 中分配
// ... 使用 temp_data ...
MemoryContextSwitchTo(oldContext);

// 批量释放
MemoryContextDelete(tempContext);  // temp_data 自动释放
```

---

## 四、存储架构

### 4.1 物理存储布局

```
$PGDATA (数据目录)
│
├── base/                    # 数据库数据文件
│   ├── 1/                   # template1 数据库 (OID=1)
│   ├── 13000/               # template0 数据库
│   ├── 13001/               # postgres 数据库
│   └── 16384/               # 用户数据库
│       ├── 16385            # 表文件 (OID=16385)
│       ├── 16385_fsm        # Free Space Map (空闲空间映射)
│       ├── 16385_vm         # Visibility Map (可见性映射)
│       ├── 16386            # 索引文件
│       └── ...
│
├── global/                  # 全局共享对象
│   ├── pg_control           # 控制文件 (8KB)
│   ├── pg_filenode.map      # 系统表文件映射
│   └── 1260                 # pg_authid 等系统表
│
├── pg_wal/                  # WAL 日志文件
│   ├── 000000010000000000000001  # WAL 段文件 (16MB)
│   ├── 000000010000000000000002
│   ├── 000000010000000000000003
│   └── archive_status/       # 归档状态
│       ├── 000000010000000000000001.done
│       └── ...
│
├── pg_xact/                 # 事务提交状态 (CLOG)
│   ├── 0000                 # CLOG 文件 (256KB)
│   ├── 0001
│   └── ...
│
├── pg_multixact/            # 多事务状态
│   ├── members/             # MultiXact 成员
│   └── offsets/             # MultiXact 偏移
│
├── pg_subtrans/             # 子事务状态
├── pg_commit_ts/            # 提交时间戳
├── pg_twophase/             # 两阶段提交状态文件
│
├── pg_stat/                 # 统计信息
│   └── pg_stat_tmp/         # 临时统计文件
│
├── pg_tblspc/               # 表空间符号链接
│   └── 16387 -> /data/pg_tblspc1/
│
├── pg_logical/              # 逻辑复制状态
│   ├── snapshots/           # 逻辑快照
│   └── mappings/            # 文件映射
│
├── postgresql.conf          # 主配置文件
├── pg_hba.conf              # 访问控制配置
├── pg_ident.conf            # 用户映射配置
└── postmaster.pid           # PID 文件
```

### 4.2 数据文件组织

#### 4.2.1 表文件 (Heap File)

**文件大小**: 每个文件最大 1GB，超过后创建 `.1`, `.2` 等后缀

**页面结构** (8KB):
```
┌────────────────────────────────────────────────────────┐  ← 0 字节
│  PageHeaderData (24 字节)                              │
│  - pd_lsn:      最后修改的 LSN                          │
│  - pd_checksum: 页面校验和                              │
│  - pd_flags:    标志位                                  │
│  - pd_lower:    空闲空间起始位置                         │
│  - pd_upper:    空闲空间结束位置                         │
│  - pd_special:  特殊空间起始位置 (索引用)                │
├────────────────────────────────────────────────────────┤  ← 24 字节
│  Line Pointer Array (ItemIdData[])                     │
│  每个 ItemId 4 字节:                                    │
│  - lp_off:   指向元组的偏移                             │
│  - lp_flags: 标志 (unused/normal/redirect/dead)        │
│  - lp_len:   元组长度                                   │
├────────────────────────────────────────────────────────┤
│  Free Space (空闲空间)                                  │
│  ← pd_lower                                            │
│                                                        │
│  ↓ 向下增长 (Line Pointers)                            │
│  ↑ 向上增长 (Tuples)                                    │
│                                                        │
│  ← pd_upper                                            │
├────────────────────────────────────────────────────────┤
│  Tuples (元组数据，从页面末尾向前增长)                   │
│  ┌──────────────────────────────────────┐             │
│  │ HeapTupleHeaderData (23 字节)        │             │
│  │ - t_xmin: 插入事务 ID                 │             │
│  │ - t_xmax: 删除事务 ID                 │             │
│  │ - t_cid:  命令 ID                     │             │
│  │ - t_ctid: 元组位置 (TID)              │             │
│  │ - t_infomask: 标志位                  │             │
│  ├──────────────────────────────────────┤             │
│  │ Null Bitmap (变长)                   │             │
│  ├──────────────────────────────────────┤             │
│  │ User Data (实际列数据)                │             │
│  └──────────────────────────────────────┘             │
├────────────────────────────────────────────────────────┤
│  Special Space (特殊空间，索引页使用)                    │
└────────────────────────────────────────────────────────┘  ← 8192 字节
```

**源码**: `src/include/storage/bufpage.h:160`

#### 4.2.2 Free Space Map (FSM)

**作用**: 快速找到有足够空闲空间的页面

**结构**: 三层树形结构
```
FSM 页面 (8KB)
┌─────────────────────────────────────┐
│  Root FSM Page (第 3 层)            │  1 个页面
│  每个字节代表下层一个页面的空闲度    │  4096 个字节 → 指向 4096 个子页面
└──────────────────┬──────────────────┘
                   │
         ┌─────────┴─────────┐
         ▼                   ▼
  ┌──────────────┐    ┌──────────────┐
  │ L2 FSM Page  │... │ L2 FSM Page  │  ~4096 个页面
  │ (第 2 层)     │    │              │  每个指向 4096 个数据页
  └──────┬───────┘    └──────┬───────┘
         │                   │
    ┌────┴────┐         ┌────┴────┐
    ▼         ▼         ▼         ▼
┌────────┐ ┌────────┐ ┌────────┐ ...
│Data Pg │ │Data Pg │ │Data Pg │      ~16M 个数据页 (128GB)
│(第 1层)│ │        │ │        │
└────────┘ └────────┘ └────────┘
```

**空闲度**: 0-255，表示页面可用空间的近似值
- 255: 完全空
- 0: 完全满

**源码**: `src/backend/storage/freespace/freespace.c:100`

#### 4.2.3 Visibility Map (VM)

**作用**: 标记哪些页面的所有元组对所有事务可见 (all-visible)

**优化**:
1. **Index-Only Scan**: 如果页面 all-visible，索引扫描无需访问堆表
2. **VACUUM 优化**: 跳过 all-visible 页面

**结构**:
```
每个数据页对应 2 个 bit:
- Bit 0: all-visible (所有元组可见)
- Bit 1: all-frozen (所有元组已冻结)

VM 文件布局:
┌─────────────────────────────────────┐
│  Page 0 (8KB)                       │
│  Bit pairs: 表示 32768 个数据页     │  每个数据页 2 bit
│  覆盖范围: 32768 * 8KB = 256MB      │
├─────────────────────────────────────┤
│  Page 1 (8KB)                       │
│  覆盖下一个 256MB                    │
└─────────────────────────────────────┘
```

**源码**: `src/backend/access/heap/visibilitymap.c:100`

### 4.3 WAL 文件

**段文件大小**: 16MB (默认，编译时可配置)

**命名规则**: `000000010000000000000001`
- 前 8 位: Timeline ID (00000001)
- 中 8 位: 逻辑段号高位 (00000000)
- 后 8 位: 逻辑段号低位 (00000001)

**页面结构** (8KB):
```
┌────────────────────────────────────────────┐
│  XLogPageHeaderData (20-48 字节)           │
│  - xlp_magic:     魔数 (0xD10B)            │
│  - xlp_info:      标志位                    │
│  - xlp_tli:       Timeline ID              │
│  - xlp_pageaddr:  页面 LSN                 │
├────────────────────────────────────────────┤
│  WAL Records (变长)                        │
│  ┌────────────────────────────────────┐   │
│  │ XLogRecord (24 字节 + 变长数据)     │   │
│  │ - xl_tot_len: 记录总长度            │   │
│  │ - xl_xid:     事务 ID                │   │
│  │ - xl_prev:    前一条记录 LSN         │   │
│  │ - xl_info:    资源管理器 + 操作类型  │   │
│  │ - xl_rmid:    资源管理器 ID          │   │
│  │ - xl_crc:     CRC 校验和             │   │
│  ├────────────────────────────────────┤   │
│  │ Data (变长)                         │   │
│  │ - Main data block                  │   │
│  │ - FPI (Full Page Image, 如果需要)   │   │
│  └────────────────────────────────────┘   │
│  ...                                       │
└────────────────────────────────────────────┘
```

**资源管理器 (Resource Manager)**:
```c
// src/include/access/rmgrlist.h
RM_XLOG_ID      = 0   // WAL 本身的日志
RM_XACT_ID      = 1   // 事务操作
RM_SMGR_ID      = 2   // 存储管理器
RM_CLOG_ID      = 3   // CLOG 操作
RM_DBASE_ID     = 4   // 数据库操作
RM_TBLSPC_ID    = 5   // 表空间操作
RM_MULTIXACT_ID = 6   // MultiXact 操作
RM_RELMAP_ID    = 7   // 关系映射
RM_STANDBY_ID   = 8   // 备库操作
RM_HEAP2_ID     = 9   // Heap 操作 (HOT)
RM_HEAP_ID      = 10  // Heap 操作
RM_BTREE_ID     = 11  // B-Tree 索引
RM_HASH_ID      = 12  // Hash 索引
RM_GIN_ID       = 13  // GIN 索引
RM_GIST_ID      = 14  // GiST 索引
RM_SEQ_ID       = 15  // 序列
... (共 ~30 个)
```

---

## 五、核心模块分类

### 5.1 模块分类树

```
PostgreSQL 核心模块
│
├── 1. 连接与进程管理
│   ├── Postmaster (主进程管理)
│   ├── Backend Process (会话管理)
│   └── Background Workers (后台任务)
│
├── 2. SQL 处理层
│   ├── Parser (语法解析)
│   ├── Analyzer (语义分析)
│   ├── Rewriter (规则重写)
│   ├── Planner/Optimizer (查询优化)
│   └── Executor (执行器)
│
├── 3. 事务与并发控制
│   ├── Transaction Manager (事务管理)
│   ├── MVCC (多版本并发控制)
│   ├── Lock Manager (锁管理)
│   └── Snapshot Manager (快照管理)
│
├── 4. 存储管理层
│   ├── Buffer Manager (缓冲池管理) ⭐
│   ├── Storage Manager (SMGR)
│   ├── File Manager
│   └── Free Space Map (FSM)
│
├── 5. WAL 与恢复
│   ├── WAL Writer (WAL 写入) ⭐
│   ├── Checkpointer (检查点) ⭐ (已完成)
│   ├── WAL Archiver (归档)
│   ├── Recovery (崩溃恢复)
│   └── PITR (时间点恢复)
│
├── 6. 访问方法 (Access Methods)
│   ├── Heap (堆表)
│   ├── B-Tree (B 树索引)
│   ├── Hash (哈希索引)
│   ├── GiST (通用搜索树)
│   ├── GIN (通用倒排索引)
│   ├── SP-GiST (空间分区 GiST)
│   └── BRIN (块范围索引)
│
├── 7. VACUUM 与自动维护
│   ├── VACUUM (清理死元组) ⭐
│   ├── Autovacuum (自动清理)
│   ├── ANALYZE (统计信息收集)
│   └── FREEZE (事务 ID 冻结)
│
├── 8. 系统目录 (System Catalog)
│   ├── pg_class (表和索引)
│   ├── pg_attribute (列定义)
│   ├── pg_type (类型系统)
│   └── ... (100+ 系统表)
│
├── 9. 复制与高可用
│   ├── Streaming Replication (流复制)
│   ├── Logical Replication (逻辑复制)
│   ├── WAL Sender/Receiver
│   └── Replication Slots
│
└── 10. 工具与监控
    ├── Statistics Collector (统计收集)
    ├── pg_stat_* 视图
    ├── Logging (日志系统)
    └── Background Writer (后台写)
```

### 5.2 核心模块优先级

根据重要性和依赖关系，推荐学习顺序：

| 优先级 | 模块名 | 重要性 | 复杂度 | 目录名 |
|-------|--------|--------|--------|--------|
| 🔴 P0 | **Checkpoint** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ `checkpoint/` (已完成) |
| 🔴 P0 | **WAL (Write-Ahead Log)** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | `wal/` |
| 🔴 P0 | **Buffer Manager** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | `buffer_manager/` |
| 🔴 P0 | **MVCC** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | `mvcc/` |
| 🟠 P1 | **Transaction Manager** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | `transaction/` |
| 🟠 P1 | **Lock Manager** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | `lock/` |
| 🟠 P1 | **VACUUM** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | `vacuum/` |
| 🟠 P1 | **Executor** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | `executor/` |
| 🟡 P2 | **Planner/Optimizer** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | `planner/` |
| 🟡 P2 | **B-Tree Index** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | `btree/` |
| 🟡 P2 | **Heap Access Method** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | `heap/` |
| 🟡 P2 | **Storage Manager** | ⭐⭐⭐⭐ | ⭐⭐⭐ | `storage/` |
| 🟢 P3 | **Parser** | ⭐⭐⭐ | ⭐⭐⭐⭐ | `parser/` |
| 🟢 P3 | **Replication** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | `replication/` |
| 🟢 P3 | **其他索引类型** | ⭐⭐⭐ | ⭐⭐⭐ | `indexes/` |

---

## 六、模块依赖关系

### 6.1 依赖关系图

```
                    ┌─────────────────┐
                    │  SQL 查询层     │
                    │  (Parser/       │
                    │   Planner/      │
                    │   Executor)     │
                    └────────┬────────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
                ▼            ▼            ▼
        ┌────────────┐ ┌────────────┐ ┌────────────┐
        │ Transaction│ │   MVCC     │ │   Lock     │
        │  Manager   │ │            │ │  Manager   │
        └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
            ┌───────────────┐  ┌───────────────┐
            │ Buffer Manager│  │  WAL System   │
            └───────┬───────┘  └───────┬───────┘
                    │                  │
                    └──────────┬───────┘
                               │
                      ┌────────┴────────┐
                      │                 │
                      ▼                 ▼
              ┌─────────────┐   ┌─────────────┐
              │  Checkpoint │   │   Storage   │
              └─────────────┘   │   Manager   │
                                └──────┬──────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │  File System    │
                              │  (OS)           │
                              └─────────────────┘
```

### 6.2 核心依赖说明

#### 6.2.1 WAL → Buffer Manager
- **依赖**: WAL 必须先写入，才能修改缓冲池中的页面
- **原因**: Write-Ahead Logging 原则
- **体现**: `MarkBufferDirty()` 前必须调用 `XLogInsert()`

#### 6.2.2 Checkpoint → WAL + Buffer Manager
- **依赖**: Checkpoint 依赖 WAL 的 REDO 点和 Buffer Manager 的脏页刷写
- **体现**: `CreateCheckPoint()` 调用 `BufferSync()` 和 `XLogInsert()`

#### 6.2.3 MVCC → Transaction Manager
- **依赖**: MVCC 通过事务 ID (XID) 判断元组可见性
- **体现**: `HeapTupleSatisfiesMVCC()` 检查 `t_xmin` 和 `t_xmax`

#### 6.2.4 Executor → 所有下层模块
- **依赖**: 执行器调用所有存储层和事务层功能
- **体现**: `ExecScan()` → `heap_getnext()` → `ReadBuffer()` → `BufferAlloc()`

---

## 七、数据流转全景

### 7.1 写事务完整流程

```
┌───────────────────────────────────────────────────────────────┐
│                     Client Application                        │
└───────────────────────────┬───────────────────────────────────┘
                            │ INSERT INTO t VALUES (1, 'data')
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. Backend Process: Parser → Planner → Executor            │
│    src/backend/tcop/postgres.c:1053 exec_simple_query()    │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Transaction Manager: 开始事务，分配 XID                  │
│    src/backend/access/transam/xact.c:350 GetCurrentTransactionId()
│    XID = 1000                                               │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Heap Access: 找到可插入的页面                            │
│    src/backend/access/heap/heapam.c:2000 heap_insert()     │
│    - 通过 FSM 查找有空闲空间的页面                          │
│    - 或者扩展新页面                                          │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Buffer Manager: 将页面读入缓冲池                         │
│    src/backend/storage/buffer/bufmgr.c:800 ReadBuffer()    │
│    - 检查 shared_buffers 中是否存在                         │
│    - 如不存在，从磁盘读取                                    │
│    - 获取 Buffer 上的 Exclusive Lock                        │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. WAL System: 生成并写入 WAL 记录                          │
│    src/backend/access/transam/xlog.c:950 XLogInsert()      │
│    WAL Record:                                              │
│    - Type: RM_HEAP_ID, XLOG_HEAP_INSERT                    │
│    - XID: 1000                                              │
│    - Data: 插入的元组数据                                    │
│    - LSN: 0/1A000128 (新 WAL 位置)                         │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. Buffer Manager: 修改缓冲池中的页面                       │
│    - 设置页面 LSN = 0/1A000128                              │
│    - 插入新元组，设置 t_xmin = 1000                          │
│    - MarkBufferDirty() 标记为脏页                           │
│    - 释放 Buffer Lock                                       │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. Transaction Manager: 提交事务                            │
│    src/backend/access/transam/xact.c:2100 CommitTransaction()
│    - 写入 COMMIT WAL 记录                                    │
│    - 标记 XID=1000 为 COMMITTED (在 CLOG 中)                │
│    - 释放锁                                                  │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 8. Backend: 返回结果给客户端                                │
│    "INSERT 0 1"                                             │
└─────────────────────────────────────────────────────────────┘
                        │
                   (异步后台)
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 9. WAL Writer: 异步刷写 WAL 缓冲区到磁盘                    │
│    src/backend/postmaster/walwriter.c:300 WalWriterMain()  │
└───────────────────────┬─────────────────────────────────────┘
                        │
                   (周期性后台)
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 10. Checkpointer: 刷写脏页到磁盘                            │
│     src/backend/postmaster/checkpointer.c:400              │
│     - BufferSync() 扫描并写入所有脏页                       │
│     - 更新 pg_control                                       │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 读事务完整流程

```
Client: SELECT * FROM t WHERE id = 1;
    │
    ▼
┌─────────────────────────────────────────┐
│ 1. Backend: Parser → Planner           │
│    生成执行计划: SeqScan 或 IndexScan   │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ 2. Snapshot Manager: 获取快照           │
│    src/backend/utils/time/snapmgr.c    │
│    Snapshot {                           │
│      xmin: 995,  // 最小活跃事务        │
│      xmax: 1005, // 下一个 XID          │
│      xip: [998, 1001, 1003] // 活跃事务 │
│    }                                    │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ 3. Executor: 扫描表                     │
│    src/backend/executor/nodeSeqscan.c  │
│    调用 heap_getnext() 逐行读取         │
└──────────────────┬──────────────────────┘
                   │
                   ▼ (对每一行)
┌─────────────────────────────────────────┐
│ 4. Buffer Manager: ReadBuffer()        │
│    - 在 shared_buffers 中查找           │
│    - Cache Hit: 直接返回                │
│    - Cache Miss: 从磁盘读取             │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ 5. MVCC: 检查元组可见性                 │
│    HeapTupleSatisfiesMVCC()            │
│    检查 t_xmin, t_xmax 是否对快照可见   │
│    - t_xmin = 990 (< xmin) → 可见      │
│    - t_xmin = 1003 (在 xip 中) → 不可见│
│    - t_xmax 检查类似                    │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ 6. Executor: 应用过滤条件 (WHERE)       │
│    如果满足，加入结果集                  │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ 7. Backend: 返回结果给客户端            │
└─────────────────────────────────────────┘
```

### 7.3 崩溃恢复流程

```
系统崩溃
    │
    ▼
PostgreSQL 重启
    │
    ▼
┌─────────────────────────────────────────┐
│ 1. Postmaster: 读取 pg_control          │
│    - checkPoint: 0/1A000000 (LSN)      │
│    - redo: 0/19FFF000 (REDO 点)        │
│    - state: DB_IN_CRASH_RECOVERY       │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ 2. Startup Process: 进入恢复模式        │
│    src/backend/access/transam/xlog.c   │
│    StartupXLOG()                        │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ 3. 从 REDO 点开始读取 WAL               │
│    LSN = 0/19FFF000                     │
│    打开 WAL 文件: 00000001000000000019  │
└──────────────────┬──────────────────────┘
                   │
                   ▼ (对每条 WAL 记录)
┌─────────────────────────────────────────┐
│ 4. 重放 WAL 记录                        │
│    调用对应资源管理器的 redo 函数        │
│    例如:                                 │
│    - RM_HEAP_ID → heap_redo()           │
│    - RM_BTREE_ID → btree_redo()         │
│                                         │
│    操作:                                 │
│    - 读取页面到缓冲池                    │
│    - 应用 WAL 记录中的修改               │
│    - 标记为脏页                          │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ 5. 继续重放直到 WAL 结束                 │
│    LSN = 0/1A023456 (最后一条记录)      │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ 6. 刷写所有脏页到磁盘                    │
│    FlushBufferPool()                    │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ 7. 执行 End-of-Recovery Checkpoint      │
│    CreateCheckPoint(CHECKPOINT_END_OF_RECOVERY)
│    - 刷写所有脏页                        │
│    - 记录新的 checkpoint 点              │
│    - 更新 pg_control                    │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ 8. 进入正常运行状态                      │
│    state: DB_IN_PRODUCTION              │
│    开始接受客户端连接                    │
└─────────────────────────────────────────┘
```

---

## 八、文档导航

### 8.1 已完成文档

| 模块 | 文档路径 | 状态 | 内容 |
|------|----------|------|------|
| **Checkpoint** | `studydoc/checkpoint/` | ✅ 完成 | 8 篇文档，223KB，包含 18 个图表和 65+ 测试用例 |

### 8.2 计划创建的文档

基于优先级 P0-P3，我们将按以下顺序创建模块文档：

#### 🔴 第一批 (P0 - 核心基础模块)

| 模块 | 目录 | 预计页数 | 优先级 |
|------|------|----------|--------|
| **WAL (Write-Ahead Log)** | `studydoc/wal/` | ~250KB | P0 |
| **Buffer Manager** | `studydoc/buffer_manager/` | ~230KB | P0 |
| **MVCC** | `studydoc/mvcc/` | ~220KB | P0 |

#### 🟠 第二批 (P1 - 核心功能模块)

| 模块 | 目录 | 预计页数 | 优先级 |
|------|------|----------|--------|
| **Transaction Manager** | `studydoc/transaction/` | ~200KB | P1 |
| **Lock Manager** | `studydoc/lock/` | ~180KB | P1 |
| **VACUUM** | `studydoc/vacuum/` | ~210KB | P1 |
| **Executor** | `studydoc/executor/` | ~280KB | P1 |

#### 🟡 第三批 (P2 - 重要模块)

| 模块 | 目录 | 预计页数 | 优先级 |
|------|------|----------|--------|
| **Planner/Optimizer** | `studydoc/planner/` | ~300KB | P2 |
| **B-Tree Index** | `studydoc/btree/` | ~200KB | P2 |
| **Heap Access** | `studydoc/heap/` | ~180KB | P2 |
| **Storage Manager** | `studydoc/storage/` | ~150KB | P2 |

#### 🟢 第四批 (P3 - 扩展模块)

| 模块 | 目录 | 预计页数 | 优先级 |
|------|------|----------|--------|
| **Parser** | `studydoc/parser/` | ~120KB | P3 |
| **Replication** | `studydoc/replication/` | ~200KB | P3 |
| **其他索引** | `studydoc/indexes/` | ~180KB | P3 |

### 8.3 文档结构标准

每个模块文档将包含以下标准结构（参照 Checkpoint 模块）：

```
模块名称/
├── 01_overview.md              # 概述与核心功能
├── 02_data_structures.md       # 核心数据结构详解
├── 03_implementation_flow.md   # 实现流程分析
├── 04_key_algorithms.md        # 关键算法与复杂度
├── 05_performance.md           # 性能优化技术
├── 06_testcases.md             # 验证测试用例
├── 07_diagrams.md              # 可视化图表集
└── README.md                   # 模块导航与总览
```

每个文档要求：
1. ✅ **完整性**: 覆盖所有核心概念和实现细节
2. ✅ **可视化**: 包含丰富的 ASCII 图表和流程图
3. ✅ **示例**: 提供大量可执行的代码和 SQL 示例
4. ✅ **源码定位**: 标注具体文件和行号
5. ✅ **测试用例**: 包含可验证的测试脚本

---

## 附录：源码目录映射

### A.1 Backend 主要目录

| 目录 | 模块 | 说明 |
|------|------|------|
| `src/backend/access/` | 访问方法 | heap, btree, gin, gist 等 |
| `src/backend/storage/` | 存储管理 | buffer, file, smgr, lmgr |
| `src/backend/access/transam/` | 事务管理 | xlog, xact, clog, mvcc |
| `src/backend/postmaster/` | 进程管理 | postmaster, checkpointer, walwriter |
| `src/backend/executor/` | 执行器 | 各类执行节点 |
| `src/backend/optimizer/` | 优化器 | planner, path, plan |
| `src/backend/parser/` | 解析器 | gram.y, scan.l |
| `src/backend/commands/` | DDL 命令 | CREATE, ALTER, DROP |
| `src/backend/catalog/` | 系统目录 | 系统表管理 |
| `src/backend/utils/` | 工具函数 | adt, cache, mmgr, time |
| `src/backend/replication/` | 复制 | walsender, walreceiver |

### A.2 Include 主要目录

| 目录 | 说明 |
|------|------|
| `src/include/access/` | 访问方法头文件 |
| `src/include/storage/` | 存储管理头文件 |
| `src/include/catalog/` | 系统表定义 (pg_*.h) |
| `src/include/nodes/` | 节点类型定义 |
| `src/include/postmaster/` | 进程管理头文件 |

---

**文档版本**: 1.0
**基于源码**: PostgreSQL 17.5
**创建日期**: 2025-01-15

**下一步**: 开始创建 `WAL` 模块完整分析文档
