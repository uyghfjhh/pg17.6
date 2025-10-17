# Postmaster 主进程分析

> PostgreSQL主进程的核心源码分析

**源码文件**: `src/backend/postmaster/postmaster.c` (8000+行)  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. 核心功能

Postmaster是PostgreSQL的**主控进程**（守护进程），负责：

```
✅ 数据库系统启动和初始化
✅ 监听客户端连接 (默认端口5432)
✅ Fork backend进程处理客户端请求
✅ Fork并管理所有后台辅助进程
✅ 处理各种信号 (SIGHUP, SIGTERM, SIGQUIT等)
✅ 崩溃恢复协调和进程重启
✅ 配置文件重新加载
```

---

## 2. 启动流程

### 2.1 启动流程图

```
postgres -D /data
│
├─→ [1] main() - src/backend/main/main.c
│    └─ PostmasterMain()
│
├─→ [2] PostmasterMain() 初始化
│    ├─ 检查数据目录
│    ├─ 读取配置文件 (postgresql.conf)
│    ├─ 初始化共享内存
│    ├─ 创建监听socket
│    └─ 注册信号处理器
│
├─→ [3] 启动辅助进程
│    ├─ StartupProcess (恢复进程)
│    ├─ BgWriterProcess
│    ├─ CheckpointerProcess
│    ├─ WalWriterProcess
│    ├─ WalReceiverProcess (如果是备库)
│    └─ AutoVacuumLauncher
│
├─→ [4] 进入主循环 ServerLoop()
│    ├─ 监听连接请求
│    ├─ 处理信号
│    ├─ 检查子进程状态
│    └─ Fork backend进程
│
└─→ [5] 运行直到收到关闭信号
```

### 2.2 核心初始化代码

```c
// src/backend/postmaster/postmaster.c:400
void
PostmasterMain(int argc, char *argv[])
{
    /* [1] 处理命令行参数 */
    parse_command_line(argc, argv);
    
    /* [2] 检查数据目录 */
    checkDataDir();
    
    /* [3] 读取配置文件 */
    if (!SelectConfigFiles(userDoption, progname))
        ExitPostmaster(2);
    
    /* [4] 创建共享内存和信号量 */
    CreateSharedMemoryAndSemaphores();
    
    /* [5] 创建监听socket */
    if (ListenServerPort(PostPortNumber) != STATUS_OK)
        ereport(FATAL, ...);
    
    /* [6] 加载共享库 */
    process_shared_preload_libraries();
    
    /* [7] 注册信号处理器 */
    pqsignal(SIGHUP, SIGHUP_handler);
    pqsignal(SIGTERM, pmdie);
    pqsignal(SIGQUIT, pmdie);
    pqsignal(SIGUSR1, sigusr1_handler);
    
    /* [8] 启动恢复进程 */
    StartupPID = StartupDataBase();
    
    /* [9] 等待恢复完成，启动其他辅助进程 */
    // ...
    
    /* [10] 进入主循环 */
    ServerLoop();
}
```

---

## 3. 主循环 ServerLoop()

### 3.1 主循环流程图

```
ServerLoop()
│
while (true) {
  │
  ├─→ [1] 准备文件描述符集合
  │    └─ 监听socket + 已连接但未认证的客户端
  │
  ├─→ [2] select() 等待事件
  │    └─ 超时: 60秒
  │
  ├─→ [3] 检查是否收到信号
  │    ├─ SIGHUP → 重新加载配置
  │    ├─ SIGTERM → 优雅关闭
  │    └─ SIGQUIT → 立即关闭
  │
  ├─→ [4] 清理僵尸子进程
  │    └─ reaper() - 处理SIGCHLD
  │
  ├─→ [5] 检查新连接
  │    └─ if (有新连接)
  │        ├─ accept() 接受连接
  │        ├─ BackendStartup()
  │        └─ fork() Backend进程
  │
  └─→ [6] 维护任务
       ├─ 检查辅助进程是否需要重启
       └─ 处理其他维护任务
}
```

### 3.2 核心代码

```c
// src/backend/postmaster/postmaster.c:1700
static int
ServerLoop(void)
{
    fd_set readmask;
    struct timeval timeout;
    
    for (;;)
    {
        /* [1] 准备select的文件描述符集 */
        FD_ZERO(&readmask);
        FD_SET(ListenSocket[i], &readmask);
        
        /* [2] 设置超时 */
        timeout.tv_sec = 60;
        timeout.tv_usec = 0;
        
        /* [3] 等待事件 */
        selres = select(nSockets, &readmask, NULL, NULL, &timeout);
        
        /* [4] 检查信号标志 */
        if (pending_pm_reload_request)
        {
            ProcessConfigFile(PGC_SIGHUP);
            pending_pm_reload_request = false;
        }
        
        if (pending_pm_shutdown_request)
        {
            pmdie(pending_pm_shutdown_request);
        }
        
        /* [5] 处理新连接 */
        if (selres > 0)
        {
            for (i = 0; i < MAXLISTEN; i++)
            {
                if (FD_ISSET(ListenSocket[i], &readmask))
                {
                    Port *port = ConnCreate(ListenSocket[i]);
                    BackendStartup(port);
                }
            }
        }
        
        /* [6] 清理死亡子进程 */
        if (pending_pm_child_exit)
        {
            reaper();
            pending_pm_child_exit = false;
        }
    }
}
```

---

## 4. Fork Backend进程

### 4.1 Backend启动流程

```
BackendStartup(Port *port)
│
├─→ [1] 检查连接限制
│    ├─ max_connections
│    └─ reserved_connections
│
├─→ [2] 分配Backend slot
│    └─ 在PGPROC数组中分配位置
│
├─→ [3] fork() 创建子进程
│    └─ pid = fork()
│
├─→ [4] 父进程 (Postmaster)
│    ├─ 记录子进程PID
│    ├─ 关闭客户端socket (子进程继承)
│    └─ 返回主循环
│
└─→ [5] 子进程 (Backend)
     ├─ 关闭监听socket
     ├─ 初始化信号处理器
     ├─ 初始化Backend环境
     └─ PostgresMain() - 进入Backend主循环
```

### 4.2 核心代码

```c
// src/backend/postmaster/postmaster.c:4200
static int
BackendStartup(Port *port)
{
    Backend *bn;
    pid_t pid;
    
    /* [1] 分配Backend结构 */
    bn = (Backend *) malloc(sizeof(Backend));
    bn->child_slot = MyPMChildSlot = AssignPostmasterChildSlot();
    
    /* [2] 记录启动时间 */
    bn->start_time = time(NULL);
    
    /* [3] fork子进程 */
    pid = fork_process();
    
    if (pid == 0)  /* 子进程 */
    {
        /* 关闭不需要的文件描述符 */
        ClosePostmasterPorts(false);
        
        /* 初始化Backend */
        InitPostgres(port->database_name, InvalidOid, port->user_name,
                     InvalidOid, NULL, false);
        
        /* 进入Backend主循环 */
        PostgresMain(port);
        
        /* 永远不会到达这里 */
        ExitPostgres();
    }
    
    /* 父进程 */
    bn->pid = pid;
    bn->is_autovacuum = false;
    DLInitElem(&bn->elem, bn);
    DLAddHead(&BackendList, &bn->elem);
    
    return STATUS_OK;
}
```

---

## 5. 信号处理

### 5.1 信号类型

```c
// 信号处理器注册
pqsignal(SIGHUP, SIGHUP_handler);     // 重新加载配置
pqsignal(SIGTERM, pmdie);             // 优雅关闭
pqsignal(SIGINT, pmdie);              // 快速关闭
pqsignal(SIGQUIT, pmdie);             // 立即关闭
pqsignal(SIGUSR1, sigusr1_handler);   // 通用通知
pqsignal(SIGUSR2, dummy_handler);     // 未使用
pqsignal(SIGCHLD, reaper);            // 子进程退出
```

### 5.2 关闭流程对比

```
优雅关闭 (SIGTERM / pg_ctl stop -m smart):
  1. 不再接受新连接
  2. 等待所有客户端断开
  3. 所有Backend进程完成后关闭
  4. 执行最后一次检查点
  5. 关闭辅助进程
  6. Postmaster退出

快速关闭 (SIGINT / pg_ctl stop -m fast):
  1. 向所有Backend发送SIGTERM
  2. 等待Backend退出 (回滚事务)
  3. 执行检查点
  4. 关闭辅助进程
  5. Postmaster退出

立即关闭 (SIGQUIT / pg_ctl stop -m immediate):
  1. 向所有进程发送SIGQUIT
  2. 不等待，强制终止所有进程
  3. 不执行检查点
  4. 下次启动需要崩溃恢复
```

### 5.3 信号处理代码

```c
// src/backend/postmaster/postmaster.c:2800
static void
pmdie(SIGNAL_ARGS)
{
    int save_errno = errno;
    
    switch (postgres_signal_arg)
    {
        case SIGTERM:  /* 优雅关闭 */
            pending_pm_shutdown_request = SmartShutdown;
            break;
            
        case SIGINT:   /* 快速关闭 */
            pending_pm_shutdown_request = FastShutdown;
            /* 向所有Backend发送SIGTERM */
            SignalChildren(SIGTERM);
            break;
            
        case SIGQUIT:  /* 立即关闭 */
            pending_pm_shutdown_request = ImmediateShutdown;
            /* 向所有进程发送SIGQUIT */
            SignalChildren(SIGQUIT);
            break;
    }
    
    errno = save_errno;
}
```

---

## 6. 崩溃恢复

### 6.1 崩溃检测

```c
// 子进程异常退出时触发
static void
reaper(SIGNAL_ARGS)
{
    pid_t pid;
    int exitstatus;
    
    while ((pid = waitpid(-1, &exitstatus, WNOHANG)) > 0)
    {
        /* 检查是否是异常退出 */
        if (WIFSIGNALED(exitstatus))
        {
            /* 进程被信号杀死 - 可能崩溃 */
            HandleChildCrash(pid, exitstatus);
        }
    }
}
```

### 6.2 崩溃恢复流程

```
检测到Backend崩溃:
  ↓
[1] 向所有进程发送SIGQUIT (立即退出)
  ↓
[2] 等待所有进程退出
  ↓
[3] 重置共享内存 (重要!)
  ↓
[4] 重启StartupProcess (执行崩溃恢复)
  ↓
[5] StartupProcess重放WAL日志
  ↓
[6] 恢复完成，重启所有辅助进程
  ↓
[7] 恢复正常服务
```

---

## 7. 架构图

```
Postmaster 进程职责图:

┌──────────────────────────────────────────────────────────┐
│                    Postmaster进程                         │
│                                                           │
│  [监听]           [管理]           [信号]        [恢复]   │
│   │               │                 │             │      │
│   ├─ Socket监听   ├─ Fork子进程    ├─ SIGHUP    ├─检测崩溃│
│   ├─ accept()     ├─ 进程追踪      ├─ SIGTERM   └─协调恢复│
│   └─ 连接管理     ├─ 资源分配      └─ SIGQUIT            │
│                   └─ 进程重启                             │
└──────────────────────────────────────────────────────────┘
         │                    │                    │
         │ fork()             │ fork()             │ fork()
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Backend进程    │  │  Checkpointer   │  │   BGWriter      │
│  (每客户端1个)   │  │  (1个)          │  │   (1个)         │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

---

## 8. 关键参数

```sql
-- 最大连接数
max_connections = 100

-- 保留连接 (超级用户)
superuser_reserved_connections = 3

-- 监听地址
listen_addresses = '*'

-- 监听端口
port = 5432

-- Unix socket目录
unix_socket_directories = '/tmp'
```

---

## 总结

### Postmaster核心职责

1. **启动管理**: 初始化系统，启动所有进程
2. **连接管理**: 监听并fork Backend进程
3. **进程管理**: 监控所有子进程状态
4. **信号处理**: 响应关闭、重载等信号
5. **崩溃恢复**: 检测并协调崩溃恢复

### 关键特性

- ✅ **单点管理**: 所有进程由Postmaster创建
- ✅ **故障隔离**: Backend崩溃不影响其他客户端
- ✅ **快速恢复**: 自动检测并重启子进程

---

**下一步**: 阅读 [02_backend.md](02_backend.md) 了解Backend进程

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

