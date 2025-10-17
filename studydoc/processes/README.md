# PostgreSQL 进程架构分析

> PostgreSQL 核心进程的深度源码分析

**模块状态**: 🚧 进行中  
**文档版本**: 1.0  
**最后更新**: 2025-10-16  
**基于版本**: PostgreSQL 17.6

---

## 📚 模块导航

| 文档 | 进程 | 核心功能 |
|------|------|---------|
| [01_overview.md](01_overview.md) | 全局架构 | 所有进程总览、进程间关系 |
| [02_postmaster.md](02_postmaster.md) | Postmaster | 主进程、启动、fork、信号处理 |
| [03_backend.md](03_backend.md) | Backend | 查询执行、会话管理 |
| [04_bgwriter.md](04_bgwriter.md) | BGWriter | 后台写入、脏页刷新 |
| [05_checkpointer.md](05_checkpointer.md) | Checkpointer | 检查点执行 |
| [06_walwriter.md](06_walwriter.md) | WAL Writer | WAL缓冲区刷盘 |
| [07_autovacuum.md](07_autovacuum.md) | Autovacuum | 自动清理死元组 |
| [08_auxiliary.md](08_auxiliary.md) | 辅助进程 | Stats、Logger等 |

---

## 🎯 PostgreSQL 进程架构总览

```
                    Operating System
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                 Postmaster (主进程)                     │ │
│  │  pid: 1234                                             │ │
│  │  • 监听端口 5432                                        │ │
│  │  • fork 所有子进程                                      │ │
│  │  • 处理信号和崩溃恢复                                   │ │
│  └─────────┬──────────────────────────────────────────────┘ │
│            │ fork()                                          │
│            ├──────────────┬──────────────┬─────────────┐    │
│            │              │              │             │    │
│  ┌─────────▼────┐  ┌──────▼──────┐ ┌────▼──────┐ ┌───▼────┐│
│  │   Backend    │  │  Backend    │ │  Backend  │ │Backend ││
│  │   Process    │  │  Process    │ │  Process  │ │Process ││
│  │  (客户端1)    │  │  (客户端2)   │ │ (客户端3) │ │  ...   ││
│  │  执行SQL      │  │  执行SQL     │ │  执行SQL  │ │        ││
│  └──────────────┘  └─────────────┘ └───────────┘ └────────┘│
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              后台辅助进程                             │  │
│  │                                                       │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐ │  │
│  │  │ BGWriter    │  │ Checkpointer │  │ WAL Writer  │ │  │
│  │  │ 后台写入器   │  │ 检查点进程    │  │ WAL写入器   │ │  │
│  │  │ 刷新脏页    │  │ 执行检查点    │  │ WAL刷盘     │ │  │
│  │  └─────────────┘  └──────────────┘  └─────────────┘ │  │
│  │                                                       │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐ │  │
│  │  │ Autovacuum  │  │Stats Collector│ │SysLogger   │ │  │
│  │  │ Launcher    │  │ 统计收集器    │  │ 日志进程    │ │  │
│  │  └──────┬──────┘  └──────────────┘  └─────────────┘ │  │
│  │         │ fork workers                               │  │
│  │  ┌──────▼──────┐  ┌──────────────┐                  │  │
│  │  │ Autovacuum  │  │ Autovacuum   │                  │  │
│  │  │ Worker 1    │  │ Worker 2     │  ...             │  │
│  │  └─────────────┘  └──────────────┘                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                共享内存段                             │  │
│  │  • Shared Buffers (缓冲池)                           │  │
│  │  • WAL Buffers (WAL缓冲区)                          │  │
│  │  • Lock Tables (锁表)                               │  │
│  │  • PGPROC Array (进程数组)                          │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔑 核心进程职责

### 1. Postmaster (主进程)

**源码**: `src/backend/postmaster/postmaster.c`

**职责**:
- ✅ 启动数据库系统
- ✅ 监听客户端连接 (端口5432)
- ✅ Fork backend进程处理连接
- ✅ Fork并管理所有辅助进程
- ✅ 处理信号 (SIGHUP, SIGTERM, SIGQUIT等)
- ✅ 崩溃恢复协调

**关键函数**: `PostmasterMain()`, `ServerLoop()`, `BackendStartup()`

---

### 2. Backend Process (后端进程)

**源码**: `src/backend/tcop/postgres.c`

**职责**:
- ✅ 为每个客户端连接服务
- ✅ 解析SQL语句
- ✅ 执行查询
- ✅ 返回结果给客户端
- ✅ 管理会话状态和事务

**关键函数**: `PostgresMain()`, `exec_simple_query()`, `PortalRun()`

---

### 3. Checkpointer (检查点进程)

**源码**: `src/backend/postmaster/checkpointer.c`

**职责**:
- ✅ 定期执行检查点
- ✅ 刷新所有脏页到磁盘
- ✅ 推进WAL重放点
- ✅ 管理WAL文件回收

**关键函数**: `CheckpointerMain()`, `CheckPointGuts()`, `BufferSync()`

---

### 4. Background Writer (后台写入器)

**源码**: `src/backend/postmaster/bgwriter.c`

**职责**:
- ✅ 持续扫描缓冲池
- ✅ 主动刷新脏页 (分散I/O)
- ✅ 减少检查点压力
- ✅ 防止缓冲池用尽

**关键函数**: `BackgroundWriterMain()`, `BgBufferSync()`

---

### 5. WAL Writer (WAL写入器)

**源码**: `src/backend/postmaster/walwriter.c`

**职责**:
- ✅ 定期刷新WAL缓冲区到磁盘
- ✅ 减少backend进程的刷盘延迟
- ✅ 提高事务提交吞吐量

**关键函数**: `WalWriterMain()`, `XLogBackgroundFlush()`

---

### 6. Autovacuum Launcher/Workers

**源码**: `src/backend/postmaster/autovacuum.c`

**职责**:
- ✅ Launcher: 定期检查表统计信息
- ✅ Launcher: Fork worker进程执行清理
- ✅ Workers: 执行VACUUM和ANALYZE
- ✅ 清理死元组，防止表膨胀

**关键函数**: `AutoVacLauncherMain()`, `AutoVacWorkerMain()`, `do_autovacuum()`

---

## 📊 进程间通信

### 1. 共享内存

```
所有进程通过共享内存通信:
- Shared Buffers: 缓冲池
- Lock Tables: 锁管理
- PGPROC Array: 进程状态
- WAL Buffers: WAL缓冲
- CLOG Buffers: 事务状态
```

### 2. 信号

```
进程间通过信号协调:
- SIGUSR1: 通用唤醒信号
- SIGTERM: 优雅退出
- SIGQUIT: 立即退出
- SIGHUP: 重新加载配置
```

### 3. Latch (闩锁)

```c
// 高效的进程间唤醒机制
WaitLatch(&MyProc->procLatch, WL_LATCH_SET | WL_TIMEOUT, timeout);
SetLatch(&proc->procLatch);  // 唤醒另一个进程
```

---

## 🔄 典型工作流程

### 客户端连接流程

```
1. 客户端连接到端口5432
   ↓
2. Postmaster 接受连接
   ↓
3. Postmaster fork() 新的 Backend 进程
   ↓
4. Backend 进程继承连接socket
   ↓
5. Backend 进程处理客户端请求
   ↓
6. 客户端断开 → Backend 进程退出
```

### 查询执行流程

```
Backend进程中:
1. 接收SQL语句
   ↓
2. Parser: 词法分析、语法分析 → 解析树
   ↓
3. Analyzer: 语义分析 → 查询树
   ↓
4. Rewriter: 重写规则 → 重写后的查询树
   ↓
5. Planner: 查询优化 → 执行计划
   ↓
6. Executor: 执行查询 → 结果集
   ↓
7. 返回结果给客户端
```

---

## 🚀 快速监控

### 查看所有进程

```sql
-- 查看所有PostgreSQL进程
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    backend_start,
    state,
    query
FROM pg_stat_activity
ORDER BY backend_start;

-- 查看后台进程
SELECT pid, backend_type 
FROM pg_stat_activity 
WHERE backend_type LIKE '%ground%' OR backend_type LIKE '%checkpoint%';
```

### 系统层面

```bash
# 查看进程树
ps auxf | grep postgres

# 示例输出:
# postgres  1234  Ss   postmaster -D /data
# postgres  1235  S     \_ postgres: checkpointer
# postgres  1236  S     \_ postgres: background writer
# postgres  1237  S     \_ postgres: walwriter
# postgres  1238  S     \_ postgres: autovacuum launcher
# postgres  1239  Ss    \_ postgres: user db [local] idle
```

---

## 📖 学习路径

### 基础 (1-2天)
1. 理解进程架构总览
2. 了解Postmaster的职责
3. 理解Backend进程执行流程

### 进阶 (3-5天)
4. 深入Checkpointer和BGWriter
5. 理解WAL Writer工作机制
6. 学习Autovacuum调优

### 专家 (1-2周)
7. 源码阅读: `src/backend/postmaster/`
8. 分析进程间同步机制
9. 调优各进程参数

---

**下一步**: 阅读 [01_overview.md](01_overview.md) 深入了解进程架构

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

