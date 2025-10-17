# PostgreSQL Checkpoint 实现深度分析 - 概览

## 文档导航

本系列文档深入分析 PostgreSQL 17.5 的 Checkpoint 实现机制：

1. **01_checkpoint_overview.md** (本文档) - 概览和核心功能
2. **02_data_structures.md** - 核心数据结构详解
3. **03_implementation_flow.md** - 实现流程详解
4. **04_key_algorithms.md** - 关键算法分析
5. **05_performance_optimization.md** - 性能优化技术
6. **06_verification_testcases.md** - 验证用例和实践
7. **07_diagrams.md** - 流程图和架构图

---

## 一、Checkpoint 是什么

Checkpoint（检查点）是 PostgreSQL 确保数据持久性和加速崩溃恢复的关键机制。它通过定期将内存中的脏数据刷写到磁盘，建立一个数据库一致性的快照点。

### 1.1 核心概念

```
WAL 日志流:  [WAL1] [WAL2] [WAL3] [Checkpoint] [WAL4] [WAL5] [WAL6]
                                      ↑
                                   REDO点

如果系统在 WAL6 时崩溃：
- 从 Checkpoint 的 REDO 点开始恢复
- 重放 WAL4、WAL5、WAL6
- 不需要从 WAL1 开始，大大加速恢复
```

### 1.2 为什么需要 Checkpoint

1. **数据持久性保证**
   - 事务提交时只需将 WAL 刷到磁盘（轻量级操作）
   - Checkpoint 负责将数据页刷到磁盘（重量级操作，异步进行）
   - 分离关键路径和后台任务，提升性能

2. **快速崩溃恢复**
   - 限制需要重放的 WAL 日志量
   - 恢复时间可预测和控制

3. **WAL 文件管理**
   - 回收不再需要的 WAL 段文件
   - 防止磁盘空间耗尽

4. **MVCC 清理基础**
   - 为 VACUUM 提供安全的最小恢复点
   - 确定哪些旧版本数据可以清理

---

## 二、Checkpoint 核心功能

### 2.1 功能清单

Checkpoint 执行时会完成以下 7 大类任务：

| 功能分类 | 具体操作 | 源码位置 |
|---------|---------|---------|
| **1. 刷写脏页** | 将共享缓冲池中的所有脏页写入磁盘 | bufmgr.c:2901 BufferSync() |
| **2. 记录检查点** | 在 WAL 日志中写入 checkpoint 记录 | xlog.c:7204 XLogInsert() |
| **3. 更新控制文件** | 更新 pg_control 文件，记录 checkpoint 位置 | xlog.c:7244 UpdateControlFile() |
| **4. 同步 SLRU** | 刷写 CLOG、CommitTs、SubTrans、MultiXact | xlog.c:7511-7515 CheckPoint*() |
| **5. 管理 WAL 文件** | 清理、回收和预分配 WAL 段文件 | xlog.c:7323 RemoveOldXlogFiles() |
| **6. 维护 REDO 点** | 确定崩溃恢复的起始位置 | xlog.c:7016 或 7061 |
| **7. 两阶段事务** | 检查点两阶段提交状态 | xlog.c:7526 CheckPointTwoPhase() |

### 2.2 详细功能说明

#### 2.2.1 刷写脏页 (BufferSync)

**目的**: 将内存中修改过的数据页持久化到磁盘。

**实现要点**:
- 扫描整个共享缓冲池（默认 128MB，包含 16384 个 8KB 页）
- 标记所有脏的、永久的页面（排除临时表）
- 排序后批量写入，减少随机 I/O
- 平衡多个表空间的写入负载
- 根据 `checkpoint_completion_target` 限流

**示例**:
```
假设缓冲池有 10,000 个脏页：
1. 扫描标记: 10,000 个页面被标记为 BM_CHECKPOINT_NEEDED
2. 排序: 按 (表空间, 关系, fork, 块号) 排序
3. 分组: 3 个表空间，分别有 4000, 3000, 3000 个脏页
4. 平衡写入: 使用堆算法交替写入各表空间
5. 限流: 每写 100 个页面检查进度，必要时休眠
```

#### 2.2.2 记录检查点 (Checkpoint Record)

**目的**: 在 WAL 中标记一个一致性点。

**Checkpoint 记录包含的信息**:
```c
CheckPoint {
    redo:           0/1A000028    // 恢复起点
    ThisTimeLineID: 1             // 当前时间线
    nextXid:        1000          // 下一个事务 ID
    nextOid:        16384         // 下一个对象 ID
    oldestXid:      500           // 最老未冻结 XID
    time:           2025-01-15 10:30:00
    // ... 更多元数据
}
```

**两种类型**:
- **Online Checkpoint**: 系统运行时的常规检查点
- **Shutdown Checkpoint**: 数据库关闭时的检查点

#### 2.2.3 更新控制文件 (pg_control)

**目的**: 持久化 checkpoint 信息，即使 WAL 损坏也能恢复。

**pg_control 文件结构**:
```
大小: 8192 字节 (只使用前 512 字节，确保原子写入)
位置: $PGDATA/global/pg_control

关键字段:
- system_identifier: 集群唯一 ID
- checkPoint: 最新 checkpoint 记录的 LSN
- checkPointCopy: checkpoint 记录的完整副本
- state: DB_IN_PRODUCTION / DB_SHUTDOWNED / ...
- minRecoveryPoint: 恢复必须达到的最小点
- wal_level, MaxConnections 等配置参数
```

**原子性保证**:
```c
// xlog.c:7244
LWLockAcquire(ControlFileLock, LW_EXCLUSIVE);
ControlFile->checkPoint = ProcLastRecPtr;
ControlFile->checkPointCopy = checkPoint;
UpdateControlFile();  // 单个 512 字节扇区写入
LWLockRelease(ControlFileLock);
```

#### 2.2.4 同步 SLRU (简单 LRU 缓存)

**目的**: 刷写事务状态相关的元数据。

**SLRU 类型**:
1. **CLOG (pg_xact)**: 事务提交状态 (committed/aborted/in-progress)
2. **CommitTs (pg_commit_ts)**: 事务提交时间戳
3. **SubTrans (pg_subtrans)**: 子事务信息
4. **MultiXact (pg_multixact)**: 多事务 ID 映射

**实现**:
```c
// xlog.c:7508-7515
CheckPointCLOG();      // 刷写 CLOG 页面
CheckPointCommitTs();  // 刷写提交时间戳
CheckPointSUBTRANS();  // 刷写子事务信息
CheckPointMultiXact(); // 刷写 MultiXact
CheckPointPredicate(); // 刷写谓词锁信息
```

#### 2.2.5 管理 WAL 文件

**目的**: 保持 WAL 目录整洁，回收磁盘空间。

**操作类型**:

1. **删除旧 WAL 段**:
```c
// xlog.c:7323
RemoveOldXlogFiles(_logSegNo, RedoRecPtr, recptr, ThisTimeLineID);

删除条件:
- 段号 < _logSegNo (redo 点之前的段)
- 不被复制槽、备份、归档需要
```

2. **回收 WAL 段**:
```c
// 不直接删除，而是重命名为未来的段名
// 例如: 000000010000000000000005 -> 00000001000000000000000A
// 优点: 避免重新创建文件的开销
```

3. **预分配 WAL 段**:
```c
// xlog.c:7331
PreallocXlogFiles(recptr, ThisTimeLineID);

预分配数量基于:
- CheckPointSegments (从 max_wal_size 计算)
- 最近的 WAL 生成速率
- 公式: 保留 (3 + 2 * CheckPointSegments) 个段
```

**示例**:
```bash
# WAL 目录变化
Before Checkpoint:
pg_wal/
  000000010000000000000001  (24 MB, 已归档)
  000000010000000000000002  (24 MB, 已归档)
  000000010000000000000003  (24 MB, 当前 redo 点在此)
  000000010000000000000004  (24 MB, 当前写入)
  000000010000000000000005  (24 MB, 预分配)

After Checkpoint:
pg_wal/
  000000010000000000000003  (保留，包含 redo 点)
  000000010000000000000004
  000000010000000000000005
  000000010000000000000006  (新预分配)
  000000010000000000000007  (新预分配)

# 段 1 和 2 被删除或回收
```

#### 2.2.6 维护 REDO 点

**目的**: 确定崩溃恢复应从哪里开始重放 WAL。

**REDO 点的确定**:

**Online Checkpoint**:
```
时间线:
  [事务A修改] [事务B修改] [插入REDO记录] [Checkpoint开始] [刷写脏页] [插入Checkpoint记录]
                              ↑
                          REDO 点

- 在开始刷写脏页前，先插入 XLOG_CHECKPOINT_REDO 记录
- 该记录的 LSN 成为 redo 点
- 确保 redo 点之后的所有修改都被包含在 checkpoint 中
```

**Shutdown Checkpoint**:
```
时间线:
  [事务A修改] [等待所有事务完成] [Checkpoint记录]
                                      ↑
                                  REDO 点

- 没有并发事务，checkpoint 记录本身就是 redo 点
- 恢复时无需重放任何 WAL
```

**代码实现**:
```c
// xlog.c:6998-7062
if (shutdown) {
    // Shutdown: redo 点是当前位置
    checkPoint.redo = XLogBytePosToRecPtr(Insert->CurrBytePos);
} else {
    // Online: 先插入 REDO 记录
    XLogBeginInsert();
    XLogRegisterData((char *) &wal_level, sizeof(wal_level));
    XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT_REDO);
    // 插入后的 LSN 成为 redo 点
    checkPoint.redo = RedoRecPtr;
}
```

#### 2.2.7 两阶段事务

**目的**: 确保分布式事务的一致性。

**实现**:
```c
// xlog.c:7526
CheckPointTwoPhase(checkPointRedo);

处理:
- 扫描所有 prepared 状态的事务
- 将状态持久化到磁盘 (pg_twophase/)
- 在 checkpoint 记录中记录这些事务
- 恢复时能够正确 COMMIT/ROLLBACK
```

---

## 三、Checkpoint 进程架构

### 3.1 进程模型

```
┌─────────────────────────────────────────────────────────────┐
│                    Postmaster (主进程)                       │
└────────────┬────────────────────────────────────────────────┘
             │
             ├─ fork ──> Backend 进程 1 (处理客户端连接)
             ├─ fork ──> Backend 进程 2
             ├─ fork ──> Backend 进程 N
             │
             ├─ fork ──> WAL Writer (专门写 WAL 缓冲区)
             ├─ fork ──> Checkpointer (本文主角)
             ├─ fork ──> Bgwriter (后台写进程)
             ├─ fork ──> Autovacuum Launcher
             └─ fork ──> Stats Collector
```

### 3.2 Checkpointer 进程职责

**主循环** (checkpointer.c:173 CheckpointerMain):

```c
for (;;) {
    bool do_checkpoint = false;

    // 1. 检查触发条件
    if (CheckpointerShmem->ckpt_flags)
        do_checkpoint = true;  // 后端请求

    if (now - last_checkpoint_time >= CheckPointTimeout)
        do_checkpoint = true;  // 时间触发

    // 2. 执行 checkpoint
    if (do_checkpoint) {
        if (RecoveryInProgress())
            CreateRestartPoint(flags);  // 恢复中
        else
            CreateCheckPoint(flags);    // 正常模式

        last_checkpoint_time = now;
    }

    // 3. 检查归档超时
    CheckArchiveTimeout();

    // 4. 休眠等待下次触发
    WaitLatch(MyLatch, timeout);
}
```

**进程特点**:
- **单一职责**: 只负责 checkpoint，不处理查询
- **长期运行**: 从数据库启动到关闭
- **信号驱动**: 通过 SIGINT 信号唤醒
- **低优先级**: 后台运行，不阻塞正常查询

### 3.3 与其他进程的交互

```
Backend 进程                 Checkpointer 进程
    │                              │
    │ 1. 写脏页到共享缓冲池           │
    ├──────────────────────────────>│
    │                              │
    │ 2. 发送 fsync 请求到队列       │
    ├──────────────────────────────>│
    │                              │
    │ 3. 调用 RequestCheckpoint()   │
    ├──────────────────────────────>│
    │    设置标志，发送 SIGINT       │
    │                              │
    │                              ├─ 4. 醒来检查标志
    │                              │
    │                              ├─ 5. 执行 CreateCheckPoint()
    │                              │    - BufferSync 刷写脏页
    │                              │    - ProcessSyncRequests 处理 fsync
    │                              │    - 写 WAL 记录
    │                              │    - 更新 pg_control
    │                              │
    │ 6. 等待完成                   │
    │<─────────────────────────────┤
    │    ConditionVariable 通知     │
    │                              │
```

---

## 四、Checkpoint 触发机制

### 4.1 触发条件

| 触发类型 | 条件 | 标志位 | 优先级 |
|---------|------|--------|--------|
| **时间触发** | `checkpoint_timeout` 到期 | CHECKPOINT_CAUSE_TIME | 低 |
| **WAL 大小触发** | WAL 累积达到 `max_wal_size` | CHECKPOINT_CAUSE_XLOG | 中 |
| **手动触发** | 执行 `CHECKPOINT` 命令 | CHECKPOINT_IMMEDIATE + CHECKPOINT_FORCE | 高 |
| **关闭触发** | 数据库 shutdown | CHECKPOINT_IS_SHUTDOWN | 最高 |
| **恢复结束触发** | WAL 恢复完成 | CHECKPOINT_END_OF_RECOVERY | 最高 |

### 4.2 时间触发

**配置参数**:
```sql
-- 默认 5 分钟
checkpoint_timeout = 300

-- 检查逻辑 (checkpointer.c:371-379)
now = time(NULL);
elapsed_secs = now - last_checkpoint_time;
if (elapsed_secs >= CheckPointTimeout) {
    do_checkpoint = true;
    flags |= CHECKPOINT_CAUSE_TIME;
}
```

**特点**:
- 定期执行，保证恢复时间可预测
- 即使系统空闲也会触发（但会被跳过）
- 适合低负载系统

### 4.3 WAL 大小触发

**配置参数**:
```sql
-- 默认 1GB
max_wal_size = 1024MB

-- 计算 CheckPointSegments (xlog.c:157)
CheckPointSegments = max_wal_size / wal_segment_size;
// 例如: 1024MB / 16MB = 64 个段
```

**触发逻辑** (在 XLogInsert 中检查):
```c
// 每次插入 WAL 记录时检查
if (Insert->CurrBytePos >=
    XLogCtl->PrevSegRecPtr + CheckPointSegments * wal_segment_size) {
    // WAL 累积量超过阈值
    RequestCheckpoint(CHECKPOINT_CAUSE_XLOG | CHECKPOINT_WAIT);
}
```

**警告机制**:
```c
// xlog.c:437-445
if (elapsed_secs < CheckPointWarning) {
    ereport(LOG,
        "checkpoints are occurring too frequently (%d seconds apart)",
        elapsed_secs);
    errhint("Consider increasing max_wal_size.");
}
```

**特点**:
- 适应负载变化（高负载时更频繁）
- 防止 WAL 目录撑爆磁盘
- 高负载系统的主要触发方式

### 4.4 手动触发

**SQL 命令**:
```sql
-- 同步 checkpoint (等待完成)
CHECKPOINT;

-- 等价于
SELECT pg_checkpoint();
```

**实现**:
```c
// tcop/utility.c 处理 CHECKPOINT 命令
case T_CheckPointStmt:
    RequestCheckpoint(CHECKPOINT_IMMEDIATE |
                     CHECKPOINT_FORCE |
                     CHECKPOINT_WAIT);
    break;
```

**标志含义**:
- `CHECKPOINT_IMMEDIATE`: 立即完成，不限流
- `CHECKPOINT_FORCE`: 强制执行，即使系统空闲
- `CHECKPOINT_WAIT`: 等待完成后返回

**使用场景**:
- 大批量数据加载后
- 重要维护操作前
- 性能测试前建立一致性点

### 4.5 关闭触发

**触发时机**:
```c
// xlog.c:6638
ShutdownXLOG(int code, Datum arg)
{
    // 等待 WAL sender 停止
    WalSndWaitStopping();

    // 执行 shutdown checkpoint
    CreateCheckPoint(CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_IMMEDIATE);
}
```

**特点**:
- 必须等待所有事务完成
- redo 点就是 checkpoint 记录本身
- 恢复时无需重放 WAL
- 启动速度最快

---

## 五、关键配置参数

### 5.1 Checkpoint 频率控制

```sql
-- 时间间隔 (默认 5 分钟)
checkpoint_timeout = 300  -- 秒

-- WAL 大小阈值 (默认 1GB)
max_wal_size = 1024MB

-- 最小 WAL 保留 (默认 80MB)
min_wal_size = 80MB
```

**调优建议**:
- OLTP 系统: 增大 `checkpoint_timeout` 和 `max_wal_size`，减少频率
- OLAP 系统: 默认值通常够用
- 磁盘空间充足: 可以大幅增加 `max_wal_size` (如 8GB)

### 5.2 Checkpoint 完成时间控制

```sql
-- 完成目标 (默认 0.9，即 90% 的时间内完成)
checkpoint_completion_target = 0.9
```

**效果示例**:
```
checkpoint_timeout = 300s
checkpoint_completion_target = 0.9

目标完成时间 = 300 * 0.9 = 270 秒

假设有 10000 个脏页需要写入：
- 如果 100 秒内写完 5000 个页面（50%）
- 进度: 50% * 0.9 = 45%
- 时间进度: 100 / 300 = 33%
- 45% > 33%，进度超前，可以休眠降低速度
- 平滑 I/O，减少对查询的影响
```

**调优建议**:
- 一般系统: 保持默认 0.9
- I/O 敏感系统: 可降至 0.5-0.7，加快完成
- 设置过小: checkpoint 完成过快，I/O 突发
- 设置过大: 接近 1.0，恢复时间变长

### 5.3 Checkpoint 日志

```sql
-- 记录 checkpoint 统计信息 (默认 on)
log_checkpoints = on
```

**日志示例**:
```
LOG:  checkpoint starting: time
LOG:  checkpoint complete: wrote 1234 buffers (7.5%);
      0 WAL file(s) added, 2 removed, 1 recycled;
      write=25.123 s, sync=1.456 s, total=26.789 s;
      sync files=78, longest=0.234 s, average=0.018 s;
      distance=24567 kB, estimate=25000 kB
```

### 5.4 警告阈值

```sql
-- Checkpoint 频率警告阈值 (默认 30 秒)
checkpoint_warning = 30
```

**作用**:
- 如果两次 checkpoint 间隔小于此值，发出警告
- 提示 `max_wal_size` 可能太小

---

## 六、源码文件导航

### 6.1 核心文件

| 文件 | 路径 | 主要内容 |
|------|------|---------|
| **checkpointer.c** | src/backend/postmaster/ | Checkpointer 进程主循环、请求处理 |
| **xlog.c** | src/backend/access/transam/ | CreateCheckPoint 主函数、WAL 管理 |
| **bufmgr.c** | src/backend/storage/buffer/ | BufferSync 缓冲区刷写算法 |
| **pg_control.h** | src/include/catalog/ | CheckPoint 结构定义 |
| **bgwriter.h** | src/include/postmaster/ | Checkpointer 接口定义 |

### 6.2 关键函数定位

```c
// Checkpointer 进程入口
checkpointer.c:173   CheckpointerMain()

// Checkpoint 主函数
xlog.c:6881          CreateCheckPoint(int flags)

// Checkpoint 核心逻辑
xlog.c:7500          CheckPointGuts(XLogRecPtr checkPointRedo, int flags)

// 缓冲区同步
bufmgr.c:2901        BufferSync(int flags)

// 处理 fsync 请求
sync.c:183           ProcessSyncRequests(void)

// 后端请求 checkpoint
checkpointer.c:941   RequestCheckpoint(int flags)

// 转发 fsync 请求
checkpointer.c:1093  ForwardSyncRequest(const FileTag *ftag, SyncRequestType type)
```

---

## 七、下一步阅读

建议按以下顺序阅读本系列文档：

1. ✅ **01_checkpoint_overview.md** (当前) - 建立整体认知
2. 📖 **02_data_structures.md** - 理解数据结构
3. 📖 **03_implementation_flow.md** - 掌握实现流程
4. 📖 **04_key_algorithms.md** - 深入算法细节
5. 📖 **05_performance_optimization.md** - 学习优化技术
6. 📖 **06_verification_testcases.md** - 实践验证
7. 📖 **07_diagrams.md** - 查阅流程图

---

## 附录：术语表

| 术语 | 英文 | 解释 |
|------|------|------|
| **Checkpoint** | Checkpoint | 检查点，数据库一致性快照点 |
| **REDO 点** | REDO Point | 崩溃恢复的起始位置 |
| **脏页** | Dirty Page | 内存中已修改但未写入磁盘的页面 |
| **WAL** | Write-Ahead Log | 预写式日志 |
| **LSN** | Log Sequence Number | 日志序列号，WAL 中的位置 |
| **SLRU** | Simple LRU | 简单 LRU 缓存，用于事务状态等 |
| **Shared Buffer Pool** | - | 共享缓冲池，PostgreSQL 的主内存缓存 |
| **Fsync** | File Sync | 将文件数据刷写到磁盘的系统调用 |
| **Timeline** | Timeline | 时间线，用于 PITR 和主备切换 |
| **Restartpoint** | Restartpoint | 恢复过程中的检查点 |

---

**文档版本**: 1.0
**基于源码**: PostgreSQL 17.5
**最后更新**: 2025-01-15
