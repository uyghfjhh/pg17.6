# PostgreSQL 后台进程分析

> Checkpointer、BGWriter、WAL Writer、Autovacuum等后台进程的核心源码分析

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. Checkpointer (检查点进程)

### 1.1 核心功能

**源码**: `src/backend/postmaster/checkpointer.c`

```
✅ 定期执行检查点
✅ 刷新所有脏页到磁盘
✅ 推进WAL重放点
✅ 管理WAL文件回收
✅ 通知其他进程检查点完成
```

### 1.2 主循环流程

```
CheckpointerMain()
│
while (true) {
  │
  ├─→ [1] 等待唤醒或超时
  │    └─ WaitLatch(checkpoint_timeout)
  │
  ├─→ [2] 检查是否需要检查点
  │    ├─ 达到checkpoint_timeout?
  │    ├─ WAL使用量超过max_wal_size?
  │    └─ 收到强制检查点请求?
  │
  ├─→ [3] 执行检查点
  │    └─ CreateCheckPoint(flags)
  │        │
  │        ├─ [a] 获取CheckpointLock
  │        ├─ [b] Redo point推进
  │        ├─ [c] BufferSync() 刷新所有脏页
  │        ├─ [d] FlushDatabaseBuffers()
  │        ├─ [e] CheckPointGuts() 其他清理
  │        ├─ [f] 写入检查点记录到WAL
  │        ├─ [g] 更新pg_control文件
  │        └─ [h] 释放CheckpointLock
  │
  └─→ [4] 通知其他进程
       └─ elog(LOG, "checkpoint complete")
}
```

### 1.3 BufferSync 核心算法

```c
// src/backend/storage/buffer/bufmgr.c:3200
static void
BufferSync(int flags)
{
    int num_to_scan = NBuffers;
    int num_written = 0;
    
    /* [1] 扫描所有buffer */
    for (int buf_id = 0; buf_id < num_to_scan; buf_id++)
    {
        BufferDesc *bufHdr = GetBufferDescriptor(buf_id);
        
        /* [2] 检查是否需要刷新 */
        if (!(bufHdr->flags & BM_DIRTY))
            continue;  // 跳过干净页
        
        /* [3] 添加到待刷新列表 */
        pending_writes[num_written++] = buf_id;
        
        /* [4] 批量刷新 (每次128个) */
        if (num_written >= BUF_WRITE_SIZE)
        {
            FlushBuffers(pending_writes, num_written);
            num_written = 0;
        }
    }
    
    /* [5] 刷新剩余的buffer */
    if (num_written > 0)
        FlushBuffers(pending_writes, num_written);
}
```

### 1.4 检查点触发条件

```sql
-- 参数配置
checkpoint_timeout = 5min      -- 时间触发
max_wal_size = 1GB             -- WAL大小触发
checkpoint_completion_target = 0.9  -- 分散I/O (90%时间内完成)

-- 手动触发
CHECKPOINT;
```

---

## 2. Background Writer (后台写入器)

### 2.1 核心功能

**源码**: `src/backend/postmaster/bgwriter.c`

```
✅ 持续扫描缓冲池
✅ 主动刷新脏页
✅ 分散I/O压力 (避免检查点峰值)
✅ 为新页面腾出空间
```

### 2.2 主循环流程

```
BackgroundWriterMain()
│
while (true) {
  │
  ├─→ [1] 睡眠一段时间
  │    └─ sleep(bgwriter_delay)  // 默认200ms
  │
  ├─→ [2] 扫描缓冲池
  │    └─ BgBufferSync()
  │        │
  │        ├─ 从Clock扫描位置开始
  │        ├─ 查找脏页
  │        ├─ 刷新少量脏页 (每次~100个)
  │        └─ 更新扫描位置
  │
  └─→ [3] 统计和休眠
       ├─ 记录写入数量
       └─ 进入下一轮
}
```

### 2.3 BGWriter vs Checkpointer

```
┌─────────────────────┬──────────────────┬──────────────────┐
│       特性          │   BGWriter       │  Checkpointer    │
├─────────────────────┼──────────────────┼──────────────────┤
│ 触发频率            │ 持续 (200ms间隔) │ 周期 (5分钟)     │
│ 写入量              │ 少量 (~100页/次) │ 全部脏页         │
│ I/O特性             │ 分散、平滑       │ 集中、峰值       │
│ 目的                │ 预防性清理       │ 保证持久化       │
│ 是否推进Redo Point  │ 否               │ 是               │
└─────────────────────┴──────────────────┴──────────────────┘
```

### 2.4 参数调优

```sql
-- BGWriter参数
bgwriter_delay = 200ms              -- 扫描间隔
bgwriter_lru_maxpages = 100         -- 每轮最多写入页数
bgwriter_lru_multiplier = 2.0       -- 写入页数 = 最近使用量 × 乘数
```

---

## 3. WAL Writer (WAL写入器)

### 3.1 核心功能

**源码**: `src/backend/postmaster/walwriter.c`

```
✅ 定期刷新WAL缓冲区到磁盘
✅ 减少Backend进程的刷盘延迟
✅ 提高事务提交吞吐量
✅ 异步提交的保证
```

### 3.2 主循环流程

```
WalWriterMain()
│
while (true) {
  │
  ├─→ [1] 等待唤醒或超时
  │    └─ WaitLatch(wal_writer_delay)  // 默认200ms
  │
  ├─→ [2] 检查是否有待写入WAL
  │    └─ GetFlushRecPtr() vs GetInsertRecPtr()
  │
  ├─→ [3] 刷新WAL
  │    └─ XLogBackgroundFlush()
  │        │
  │        ├─ 获取WAL插入锁
  │        ├─ 计算需要刷新的LSN
  │        ├─ write() 写入OS缓冲
  │        ├─ fdatasync() 强制刷盘
  │        └─ 更新刷新位置
  │
  └─→ [4] 统计和休眠
       └─ 进入下一轮
}
```

### 3.3 WAL刷新策略

```
Backend提交事务时:
  ├─ 同步提交 (synchronous_commit = on)
  │   └─ Backend自己调用XLogFlush()
  │       └─ 等待刷盘完成 (慢但安全)
  │
  └─ 异步提交 (synchronous_commit = off)
      └─ Backend只写入WAL缓冲
          └─ 由WAL Writer后台刷盘
              └─ 快速返回 (可能丢失<wal_writer_delay的事务)
```

### 3.4 参数配置

```sql
-- WAL Writer参数
wal_writer_delay = 200ms           -- 刷新间隔
wal_writer_flush_after = 1MB       -- 累积多少后强制刷新

-- 提交模式
synchronous_commit = on            -- 同步提交 (默认)
                   = off           -- 异步提交 (快但可能丢数据)
                   = local         -- 本地刷盘
                   = remote_write  -- 备库接收
                   = remote_apply  -- 备库应用
```

---

## 4. Autovacuum (自动清理)

### 4.1 核心功能

**源码**: `src/backend/postmaster/autovacuum.c`

```
✅ Launcher: 定期检查表统计信息
✅ Launcher: 决定哪些表需要VACUUM
✅ Launcher: Fork worker进程执行清理
✅ Workers: 执行VACUUM和ANALYZE
✅ 防止事务ID回卷
```

### 4.2 Launcher主循环

```
AutoVacLauncherMain()
│
while (true) {
  │
  ├─→ [1] 睡眠一段时间
  │    └─ sleep(autovacuum_naptime)  // 默认1分钟
  │
  ├─→ [2] 扫描所有数据库
  │    for (each database) {
  │      │
  │      ├─ 查询pg_stat_user_tables
  │      ├─ 计算死元组比例
  │      └─ 判断是否需要VACUUM
  │    }
  │
  ├─→ [3] 选择需要清理的表
  │    └─ 按优先级排序:
  │        1. 接近事务ID回卷的表 (最高优先级)
  │        2. 死元组最多的表
  │        3. 统计信息过期的表
  │
  ├─→ [4] Fork worker进程
  │    ├─ 检查worker数量限制
  │    │   └─ autovacuum_max_workers (默认3)
  │    └─ fork() AutoVacWorker
  │
  └─→ [5] 进入下一轮
}
```

### 4.3 Worker执行流程

```
AutoVacWorkerMain()
│
├─→ [1] 连接到目标数据库
│
├─→ [2] 对分配的表执行VACUUM
│    └─ vacuum()
│        │
│        ├─ 扫描堆表，标记死元组
│        ├─ 清理索引中的死元组指针
│        ├─ 回收堆表页面空间
│        ├─ 更新FSM (空闲空间映射)
│        ├─ 更新VM (可见性映射)
│        └─ 更新统计信息
│
├─→ [3] 如果需要，执行ANALYZE
│    └─ do_analyze_rel()
│        └─ 采样统计，更新pg_statistic
│
└─→ [4] 退出
```

### 4.4 触发条件

```sql
-- 表级阈值
autovacuum_vacuum_threshold = 50              -- 基础阈值
autovacuum_vacuum_scale_factor = 0.2          -- 20% 死元组触发

-- 触发条件:
-- n_dead_tup > threshold + (n_live_tup * scale_factor)

-- 示例: 1000行的表
-- 死元组 > 50 + (1000 * 0.2) = 250 时触发

-- 调整单表参数
ALTER TABLE hot_table SET (
    autovacuum_vacuum_scale_factor = 0.05,   -- 5% 触发
    autovacuum_vacuum_threshold = 100
);
```

### 4.5 事务ID回卷预防

```
事务ID空间: 2^32 = 4,294,967,296

PostgreSQL每200M事务强制VACUUM FREEZE:
  └─ autovacuum_freeze_max_age = 200,000,000

如果表长时间不VACUUM:
  ├─ XID接近回卷点
  ├─ 触发强制VACUUM (优先级最高)
  └─ 防止数据"消失" (XID回卷导致可见性错误)

监控:
SELECT datname, age(datfrozenxid) 
FROM pg_database 
ORDER BY age(datfrozenxid) DESC;
```

---

## 5. 辅助进程总览

### 5.1 进程协作图

```
                    WAL日志流
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│  Backend ─→ 写入WAL Buffer ─→ 写入Shared Buffer │
└──┬────────────────────────────────────────┬─────┘
   │                                        │
   │ 定期刷新                                │ 定期刷新
   ▼                                        ▼
┌──────────────┐                     ┌──────────────┐
│ WAL Writer   │                     │  BGWriter    │
│ 200ms刷WAL   │                     │  200ms扫描   │
└──────────────┘                     └──────────────┘
                                            │
                                            │ 分散I/O
                                            ▼
                                     ┌──────────────┐
                                     │ Checkpointer │
                                     │ 5min全刷     │
                                     └──────┬───────┘
                                            │
                                            │ 推进Redo Point
                                            ▼
                                     [WAL文件回收]
```

### 5.2 进程参数对比

| 进程 | 默认间隔 | 主要参数 | I/O特性 |
|------|---------|---------|---------|
| Checkpointer | 5分钟 | checkpoint_timeout | 峰值、全量 |
| BGWriter | 200ms | bgwriter_delay | 平滑、少量 |
| WAL Writer | 200ms | wal_writer_delay | 顺序写 |
| Autovacuum | 1分钟 | autovacuum_naptime | 后台、低优先级 |

---

## 6. 监控查询

```sql
-- 查看后台进程状态
SELECT pid, backend_type, backend_start 
FROM pg_stat_activity 
WHERE backend_type IN ('checkpointer', 'background writer', 'walwriter', 'autovacuum launcher');

-- 查看autovacuum worker
SELECT pid, datname, usename, state, query 
FROM pg_stat_activity 
WHERE backend_type = 'autovacuum worker';

-- 查看检查点统计
SELECT * FROM pg_stat_bgwriter;

-- 查看autovacuum统计
SELECT relname, last_vacuum, last_autovacuum, n_dead_tup 
FROM pg_stat_user_tables 
ORDER BY n_dead_tup DESC;
```

---

## 总结

### 后台进程职责

1. **Checkpointer**: 周期性检查点，保证持久化
2. **BGWriter**: 持续清理脏页，平滑I/O
3. **WAL Writer**: 异步刷WAL，提升吞吐
4. **Autovacuum**: 自动清理死元组，防止膨胀

### 协同工作

- ✅ **Checkpointer + BGWriter**: 分散I/O负载
- ✅ **WAL Writer**: 支持异步提交
- ✅ **Autovacuum**: 维护表健康度

---

**下一步**: 阅读 [04_shared_memory.md](04_shared_memory.md) 了解共享内存和进程间通信

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

