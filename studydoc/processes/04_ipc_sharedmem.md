# 进程间通信和共享内存

> PostgreSQL进程间通信(IPC)和共享内存机制的核心分析

**源码目录**: `src/backend/storage/ipc/`, `src/include/storage/`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. 共享内存架构

### 1.1 共享内存布局

```
共享内存段 (Shared Memory Segment):
┌─────────────────────────────────────────────────────────┐
│ Size: shared_buffers + 其他结构 (几GB到几十GB)          │
│                                                          │
│ ┌────────────────────────────────────────────────────┐ │
│ │ Shared Buffers (缓冲池)                            │ │
│ │ 默认: 128MB, 推荐: RAM的25%                         │ │
│ │ BufferDesc数组 + 实际页面数据                       │ │
│ └────────────────────────────────────────────────────┘ │
│                                                          │
│ ┌────────────────────────────────────────────────────┐ │
│ │ WAL Buffers (WAL缓冲区)                            │ │
│ │ 默认: 16MB (shared_buffers的1/32)                  │ │
│ └────────────────────────────────────────────────────┘ │
│                                                          │
│ ┌────────────────────────────────────────────────────┐ │
│ │ Lock Tables (锁表)                                  │ │
│ │ • LOCK hash table                                   │ │
│ │ • PROCLOCK hash table                               │ │
│ └────────────────────────────────────────────────────┘ │
│                                                          │
│ ┌────────────────────────────────────────────────────┐ │
│ │ PGPROC Array (进程数组)                             │ │
│ │ max_connections + autovacuum_max_workers + ...      │ │
│ │ 每个进程的状态、锁持有情况等                         │ │
│ └────────────────────────────────────────────────────┘ │
│                                                          │
│ ┌────────────────────────────────────────────────────┐ │
│ │ CLOG Buffers (事务状态缓存)                         │ │
│ │ 记录事务提交/中止状态                               │ │
│ └────────────────────────────────────────────────────┘ │
│                                                          │
│ ┌────────────────────────────────────────────────────┐ │
│ │ Lightweight Locks (LWLock数组)                      │ │
│ │ Buffer锁、WAL锁、锁管理器锁等                        │ │
│ └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 1.2 共享内存创建

```c
// src/backend/storage/ipc/ipci.c:80
void
CreateSharedMemoryAndSemaphores(void)
{
    PGShmemHeader *shim = NULL;
    Size size;
    
    /* [1] 计算总大小 */
    size = CalculateShmemSize();
    
    /* [2] 创建共享内存段 */
    shim = PGSharedMemoryCreate(size);
    
    /* [3] 初始化各个子系统 */
    InitShmemAllocation();
    
    /*
     * 分配各个组件的共享内存:
     */
    
    /* Buffer Manager */
    InitBufferPool();  // shared_buffers
    
    /* Lock Manager */
    InitLocks();       // 锁表
    
    /* Process Array */
    InitProcGlobal();  // PGPROC数组
    
    /* WAL */
    XLogShmemInit();   // WAL buffers
    
    /* CLOG */
    CLOGShmemInit();   // 事务状态
    
    /* 其他子系统... */
}
```

---

## 2. 进程间通信机制

### 2.1 通信方式对比

```
┌───────────────┬────────────┬──────────────┬─────────────┐
│   通信方式    │  速度      │  用途        │  复杂度     │
├───────────────┼────────────┼──────────────┼─────────────┤
│ 共享内存      │ 最快       │ 数据共享     │ 需要锁保护   │
│ 信号 (Signal) │ 快         │ 进程唤醒     │ 简单        │
│ Latch        │ 快         │ 高效唤醒     │ 中等        │
│ 管道 (Pipe)   │ 中等       │ 少量数据传输 │ 简单        │
│ 共享队列      │ 中等       │ 消息传递     │ 复杂        │
└───────────────┴────────────┴──────────────┴─────────────┘
```

### 2.2 共享内存访问示例

```c
// Backend访问共享缓冲区
Buffer buf = ReadBuffer(rel, blockNum);

/* 内部流程: */
// 1. 查找buffer映射表 (共享哈希表)
// 2. 如果找到 → 增加refcount
// 3. 如果未找到 → 分配新buffer，从磁盘读取
// 4. 返回buffer引用

// 访问buffer内容 (需要先加锁)
LockBuffer(buf, BUFFER_LOCK_SHARE);
Page page = BufferGetPage(buf);
// 访问页面数据...
LockBuffer(buf, BUFFER_LOCK_UNLOCK);

ReleaseBuffer(buf);
```

---

## 3. 信号机制

### 3.1 常用信号

```c
// Backend进程的信号处理器
pqsignal(SIGTERM, die);                    // 终止进程
pqsignal(SIGQUIT, quickdie);               // 立即退出
pqsignal(SIGUSR1, procsignal_sigusr1_handler); // 通用通知

// SIGUSR1的多种用途 (通过共享内存标志区分):
typedef enum
{
    PROCSIG_CATCHUP_INTERRUPT,    // 备库追赶
    PROCSIG_NOTIFY_INTERRUPT,     // LISTEN/NOTIFY
    PROCSIG_PARALLEL_MESSAGE,     // 并行查询消息
    PROCSIG_WALSND_INIT_STOPPING, // WAL发送停止
    PROCSIG_RECOVERY_CONFLICT_BUFFERPIN, // 恢复冲突
    // ...
} ProcSignalReason;
```

### 3.2 信号发送示例

```c
// 通知另一个进程
void
SendProcSignal(pid_t pid, ProcSignalReason reason)
{
    /* [1] 在共享内存中设置标志 */
    ProcSignalSetBit(proc, reason);
    
    /* [2] 发送信号唤醒进程 */
    kill(pid, SIGUSR1);
}

// 接收端处理
static void
procsignal_sigusr1_handler(SIGNAL_ARGS)
{
    /* 检查各种标志位 */
    if (ProcSignalCheckBit(PROCSIG_NOTIFY_INTERRUPT))
        HandleNotifyInterrupt();
    
    if (ProcSignalCheckBit(PROCSIG_CATCHUP_INTERRUPT))
        HandleCatchupInterrupt();
    
    // ...
}
```

---

## 4. Latch (闩锁)机制

### 4.1 Latch概述

**Latch**是PostgreSQL高效的进程间唤醒机制，优于传统的信号+sleep。

```
特点:
✅ 比信号更高效 (基于eventfd或pipe)
✅ 支持超时等待
✅ 可以同时等待多个事件
✅ 避免信号丢失问题
```

### 4.2 Latch使用示例

```c
// src/backend/storage/ipc/latch.c

// 等待端
void
WaitForWork(void)
{
    int rc;
    
    /* 等待latch被设置或超时 */
    rc = WaitLatch(MyLatch,
                   WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH,
                   1000L,  // 超时1秒
                   WAIT_EVENT_BACKGROUND_WORKER_MAIN);
    
    /* 重置latch */
    ResetLatch(MyLatch);
    
    if (rc & WL_LATCH_SET)
    {
        /* 被其他进程唤醒，处理工作 */
        DoSomeWork();
    }
    else if (rc & WL_TIMEOUT)
    {
        /* 超时，执行定期任务 */
        DoPeriodicMaintenance();
    }
}

// 唤醒端
void
WakeupWorker(PGPROC *proc)
{
    /* 设置latch，唤醒等待的进程 */
    SetLatch(&proc->procLatch);
}
```

### 4.3 Latch实现原理

```
Linux: 使用 eventfd
  ├─ eventfd_create() 创建文件描述符
  ├─ write() 设置latch
  └─ epoll_wait() 等待

其他Unix: 使用 self-pipe
  ├─ pipe() 创建管道
  ├─ write(pipe[1]) 设置latch
  └─ select(pipe[0]) 等待

Windows: 使用 Event对象
  ├─ CreateEvent() 创建事件
  ├─ SetEvent() 设置latch
  └─ WaitForMultipleObjects() 等待
```

---

## 5. LWLock (轻量级锁)

### 5.1 LWLock特点

```
LWLock (Lightweight Lock):
✅ 比常规锁 (LOCK表) 轻量
✅ 用于保护共享内存结构
✅ 支持共享模式和排他模式
✅ 基于自旋锁 + 信号量实现
```

### 5.2 常见LWLock

```c
// src/include/storage/lwlock.h

typedef enum BuiltinTrancheIds
{
    LWTRANCHE_BUFFER_MAPPING,      // Buffer映射哈希表
    LWTRANCHE_BUFFER_CONTENT,      // Buffer内容
    LWTRANCHE_LOCK_MANAGER,        // 锁管理器
    LWTRANCHE_WAL_INSERT,          // WAL插入
    LWTRANCHE_CLOG_CONTROL,        // CLOG访问
    LWTRANCHE_PROC,                // PGPROC数组
    // ...
} BuiltinTrancheIds;
```

### 5.3 LWLock使用示例

```c
// 获取buffer映射锁 (排他模式)
LWLockAcquire(BufMappingPartitionLock(hashcode), LW_EXCLUSIVE);

/* 操作共享哈希表 */
entry = hash_search(BufferMappingTable, &tag, HASH_ENTER, &found);

LWLockRelease(BufMappingPartitionLock(hashcode));

// 获取buffer内容锁 (共享模式)
LWLockAcquire(BufferDescriptorGetContentLock(bufHdr), LW_SHARED);

/* 读取页面内容 */
memcpy(dest, BufferGetPage(buffer), BLCKSZ);

LWLockRelease(BufferDescriptorGetContentLock(bufHdr));
```

---

## 6. PGPROC结构

### 6.1 PGPROC定义

```c
// src/include/storage/proc.h
struct PGPROC
{
    /* 进程标识 */
    int pid;
    Oid databaseId;
    Oid roleId;
    
    /* 锁信息 */
    LOCK *waitLock;          // 正在等待的锁
    LOCKMODE waitLockMode;
    dlist_head myProcLocks;  // 持有的锁链表
    
    /* 事务信息 */
    TransactionId xid;       // 当前事务ID
    TransactionId xmin;      // 快照xmin
    
    /* Latch */
    Latch procLatch;         // 进程latch
    
    /* 信号标志 */
    sig_atomic_t procSignalFlags[NUM_PROCSIGNALS];
    
    /* 其他状态 */
    // ...
};
```

### 6.2 PGPROC数组

```
ProcGlobal->allProcs[] (PGPROC数组):
┌────────────────────────────────────────────────────┐
│ [0]: Postmaster的PGPROC (虚拟)                     │
│ [1]: Backend进程1                                  │
│ [2]: Backend进程2                                  │
│ ...                                                 │
│ [max_connections]: Backend进程N                    │
│ [max_connections+1]: Autovacuum worker 1           │
│ ...                                                 │
│ [total]: Checkpointer                              │
│ [total+1]: BGWriter                                │
│ [total+2]: WAL Writer                              │
│ ...                                                 │
└────────────────────────────────────────────────────┘

每个进程在fork时分配一个PGPROC槽位
```

---

## 7. 监控共享内存

### 7.1 查看共享内存使用

```sql
-- 查看共享内存参数
SHOW shared_buffers;
SHOW max_connections;

-- 查看实际使用情况
SELECT 
    name,
    setting,
    unit,
    category
FROM pg_settings
WHERE name IN ('shared_buffers', 'wal_buffers', 'max_connections');

-- 查看buffer使用统计
SELECT * FROM pg_stat_bgwriter;
```

### 7.2 系统层面监控

```bash
# 查看共享内存段
ipcs -m

# 示例输出:
# ------ Shared Memory Segments --------
# key        shmid      owner      perms      bytes      nattch     status
# 0x0052e2c1 0          postgres   600        4294967296 10

# 查看PostgreSQL进程附加情况
ipcs -m -p

# 删除共享内存段 (危险! 仅在PostgreSQL未运行时)
# ipcrm -m <shmid>
```

---

## 8. 架构总图

```
PostgreSQL 进程间通信架构:

┌──────────────────────────────────────────────────────┐
│               Operating System                        │
│                                                       │
│  ┌────────────────────────────────────────────────┐ │
│  │          Shared Memory Segment                  │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │ │
│  │  │ Buffers  │  │ WAL Buf  │  │  Lock Tables │ │ │
│  │  └──────────┘  └──────────┘  └──────────────┘ │ │
│  │  ┌──────────┐  ┌──────────┐                   │ │
│  │  │ PGPROC[] │  │ LWLocks  │  ...              │ │
│  │  └──────────┘  └──────────┘                   │ │
│  └───────┬────────────────────────────────────────┘ │
│          │ 直接访问 (mmap)                           │
│          │                                           │
│  ┌───────┼─────────────┬──────────────┬──────────┐ │
│  │       │             │              │          │ │
│  │  ┌────▼────┐   ┌───▼──────┐  ┌───▼─────┐   │ │
│  │  │Backend 1│   │Backend 2 │  │Backend N│ ... │ │
│  │  └────┬────┘   └───┬──────┘  └────┬────┘   │ │
│  │       │            │               │        │ │
│  │       └────────────┼───────────────┘        │ │
│  │                    │ Signal/Latch通信       │ │
│  │       ┌────────────▼────────────┐           │ │
│  │       │     Postmaster          │           │ │
│  │       └─────────────────────────┘           │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

---

## 总结

### 进程间通信方式

1. **共享内存**: 高速数据共享 (Buffers、锁表)
2. **信号**: 进程通知和唤醒
3. **Latch**: 高效等待/唤醒机制
4. **LWLock**: 保护共享数据结构

### 关键特性

- ✅ **高效**: 共享内存直接访问
- ✅ **安全**: LWLock保护并发
- ✅ **可扩展**: 支持大量并发连接

---

**完成**: PostgreSQL进程架构分析模块已完成！

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

