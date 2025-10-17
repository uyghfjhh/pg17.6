# Replication 复制系统核心分析 (精简版)

> PostgreSQL流复制和逻辑复制的核心原理、配置和监控

**源码**: `src/backend/replication/`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. 流复制 (Streaming Replication)

### 1.1 核心原理

```
流复制 = WAL复制 + 持续恢复

Primary (主库):
  [1] 执行SQL → 生成WAL
  [2] WAL Sender进程读取WAL
  [3] 通过TCP发送到Standby
  [4] 等待Standby确认 (同步模式)

Standby (备库):
  [1] WAL Receiver接收WAL
  [2] 写入本地WAL文件
  [3] Startup Process应用WAL
  [4] 更新数据文件
  [5] 发送确认给Primary
```

### 1.2 复制模式

```
异步复制 (Asynchronous):
  • synchronous_commit = off
  • Primary不等待Standby确认
  • 性能最好，可能丢数据

同步复制 (Synchronous):
  • synchronous_commit = on
  • Primary等待Standby写入WAL
  • 零数据丢失

远程应用 (Remote Apply):
  • synchronous_commit = remote_apply
  • Primary等待Standby应用WAL
  • 最强一致性
```

---

## 2. 配置流复制

### 2.1 Primary配置

```bash
# postgresql.conf
wal_level = replica                 # 启用复制日志级别
max_wal_senders = 10               # 最多10个WAL发送进程
wal_keep_size = 1GB                # 保留1GB的WAL
synchronous_commit = on            # 同步复制
synchronous_standby_names = 'standby1'  # 同步备库名称

# pg_hba.conf - 允许复制连接
host    replication    replicator    192.168.1.0/24    md5
```

### 2.2 Standby配置

```bash
# postgresql.conf
primary_conninfo = 'host=primary port=5432 user=replicator password=xxx'
primary_slot_name = 'standby1'     # 使用复制槽
hot_standby = on                   # 允许只读查询

# standby.signal - 标记为备库
touch /path/to/data/standby.signal
```

### 2.3 创建复制槽

```sql
-- Primary上创建复制槽
SELECT pg_create_physical_replication_slot('standby1');

-- 查看复制槽
SELECT * FROM pg_replication_slots;

-- 删除复制槽
SELECT pg_drop_replication_slot('standby1');
```

---

## 3. 逻辑复制 (Logical Replication)

### 3.1 核心原理

```
逻辑复制 = 发布/订阅模式

Publisher (发布端):
  [1] 创建PUBLICATION
  [2] Logical Decoding解码WAL
  [3] 转换为逻辑变更 (INSERT/UPDATE/DELETE)
  [4] 发送到Subscriber

Subscriber (订阅端):
  [1] 创建SUBSCRIPTION
  [2] 接收逻辑变更
  [3] Apply Worker应用变更
  [4] 写入本地表

特点:
  • 表级别复制 (不是整个数据库)
  • 可以过滤数据
  • 可以跨版本复制
  • 双向复制可能冲突
```

### 3.2 配置逻辑复制

```sql
-- Publisher配置
-- postgresql.conf
wal_level = logical                # 逻辑复制日志级别
max_replication_slots = 10
max_wal_senders = 10

-- 创建发布
CREATE PUBLICATION pub_users FOR TABLE users;

-- 发布多个表
CREATE PUBLICATION pub_all FOR TABLE users, orders;

-- 发布所有表
CREATE PUBLICATION pub_all_tables FOR ALL TABLES;

-- 查看发布
\dRp+
SELECT * FROM pg_publication;

---

-- Subscriber配置
-- 创建订阅
CREATE SUBSCRIPTION sub_users
    CONNECTION 'host=publisher port=5432 dbname=mydb user=replicator password=xxx'
    PUBLICATION pub_users;

-- 查看订阅
\dRs+
SELECT * FROM pg_subscription;

-- 禁用/启用订阅
ALTER SUBSCRIPTION sub_users DISABLE;
ALTER SUBSCRIPTION sub_users ENABLE;

-- 删除订阅
DROP SUBSCRIPTION sub_users;
```

---

## 4. 复制槽 (Replication Slots)

### 4.1 作用

```
复制槽确保:
  ✅ 备库需要的WAL不会被删除
  ✅ 即使备库断开，也能重新连接继续复制
  ✅ 防止WAL被vacuum清理

风险:
  ⚠️ 如果备库长时间断开，WAL会无限积累
  ⚠️ 可能导致磁盘空间耗尽
```

### 4.2 管理复制槽

```sql
-- 查看复制槽状态
SELECT 
    slot_name,
    slot_type,              -- physical / logical
    active,                 -- 是否活跃
    restart_lsn,            -- 需要保留的最早WAL位置
    confirmed_flush_lsn,    -- 已确认应用的位置
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS retained_wal       -- 保留的WAL大小
FROM pg_replication_slots;

-- 删除不用的槽 (危险操作!)
SELECT pg_drop_replication_slot('old_slot');
```

---

## 5. 监控复制状态

### 5.1 Primary监控

```sql
-- 查看WAL发送进程
SELECT 
    pid,
    usename,
    application_name,       -- 备库名称
    client_addr,            -- 备库IP
    state,                  -- streaming / catchup
    sync_state,             -- async / sync
    sent_lsn,               -- 已发送的LSN
    write_lsn,              -- 备库已写入的LSN
    flush_lsn,              -- 备库已刷盘的LSN
    replay_lsn,             -- 备库已应用的LSN
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes,
    write_lag,              -- 写入延迟
    flush_lag,              -- 刷盘延迟
    replay_lag              -- 应用延迟
FROM pg_stat_replication;
```

### 5.2 Standby监控

```sql
-- 查看接收状态
SELECT 
    pid,
    status,                 -- streaming / starting
    receive_start_lsn,
    receive_start_tli,
    received_lsn,
    received_tli,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn,
    slot_name
FROM pg_stat_wal_receiver;

-- 查看恢复进度
SELECT 
    pg_is_in_recovery(),                        -- 是否在恢复中
    pg_last_wal_receive_lsn(),                  -- 最后接收的LSN
    pg_last_wal_replay_lsn(),                   -- 最后应用的LSN
    pg_last_xact_replay_timestamp(),            -- 最后事务时间
    now() - pg_last_xact_replay_timestamp() AS replication_delay;
```

---

## 6. 故障切换

### 6.1 提升Standby为Primary

```bash
# 方法1: 使用pg_ctl
pg_ctl promote -D /path/to/data

# 方法2: 创建触发文件
touch /path/to/data/promote

# 方法3: 使用SQL函数
SELECT pg_promote();
```

### 6.2 级联复制

```
Primary
  │
  ├─→ Standby1 (中继)
  │     │
  │     ├─→ Standby2
  │     └─→ Standby3
  │
  └─→ Standby4 (直连)

好处:
  • 减轻Primary压力
  • 地理分布式部署
  • 分层架构
```

---

## 7. 性能优化

### 7.1 复制参数调优

```bash
# Primary
wal_sender_timeout = 60s           # 发送超时
wal_keep_size = 2GB                # 保留WAL大小
max_slot_wal_keep_size = 10GB      # 复制槽最大保留

# Standby
wal_receiver_timeout = 60s         # 接收超时
wal_retrieve_retry_interval = 5s   # 重试间隔
hot_standby_feedback = on          # 反馈给Primary，防止冲突
max_standby_streaming_delay = 30s  # 最大应用延迟
```

### 7.2 减少复制延迟

```sql
-- 检查延迟原因
-- 1. 网络带宽
-- 2. Standby硬件性能
-- 3. 大事务应用慢
-- 4. 长查询阻塞应用

-- 优化措施:
-- ✅ 增加网络带宽
-- ✅ 使用SSD加速Standby
-- ✅ 减少大事务
-- ✅ 设置max_standby_streaming_delay
```

---

## 8. 常见问题

### 8.1 复制中断

```sql
-- 检查Primary
SELECT * FROM pg_stat_replication;  -- 是否有Standby连接

-- 检查Standby日志
tail -f /path/to/log/postgresql.log

-- 常见原因:
--   • 网络断开
--   • 认证失败 (pg_hba.conf)
--   • Primary WAL被删除 (没用复制槽)
--   • max_wal_senders不够

-- 解决:
--   1. 检查网络连接
--   2. 检查pg_hba.conf配置
--   3. 使用复制槽
--   4. 增加max_wal_senders
```

### 8.2 WAL堆积

```sql
-- 检查WAL目录大小
SELECT pg_size_pretty(
    sum(size)::bigint
) FROM pg_ls_waldir();

-- 检查复制槽占用
SELECT 
    slot_name,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS wal_retained
FROM pg_replication_slots
WHERE active = false;  -- 不活跃的槽

-- 解决:
-- 1. 修复或删除失效的Standby
-- 2. 删除不用的复制槽
-- 3. 增加磁盘空间
```

---

## 9. 架构图

### 9.1 流复制架构

```
┌─────────────────────────────────────────────┐
│              Primary Server                  │
│                                              │
│  ┌──────────┐      ┌──────────────┐        │
│  │  Client  │──┬──→│   Backend    │        │
│  └──────────┘  │   └──────────────┘        │
│                │           │                 │
│                │           ↓                 │
│                │   ┌──────────────┐         │
│                │   │  WAL Writer  │         │
│                │   └──────────────┘         │
│                │           │                 │
│                │           ↓                 │
│                │   ┌──────────────┐         │
│                └──→│ WAL Sender 1 │──────┐  │
│                    └──────────────┘      │  │
│                    ┌──────────────┐      │  │
│                    │ WAL Sender 2 │──┐   │  │
│                    └──────────────┘  │   │  │
└──────────────────────────────────────┼───┼──┘
                                       │   │
                        TCP Stream     │   │
                                       ↓   ↓
┌─────────────────────────────────────────────┐
│             Standby Server 1                 │
│                                              │
│    ┌──────────────┐      ┌──────────────┐  │
│    │ WAL Receiver │──→   │   Startup    │  │
│    └──────────────┘      │   Process    │  │
│                          └──────────────┘  │
│                                  │          │
│                                  ↓          │
│                          ┌──────────────┐  │
│                          │  Data Files  │  │
│                          └──────────────┘  │
│                                              │
│    ┌──────────────┐                         │
│    │  Read-Only   │  (Hot Standby)         │
│    │   Queries    │                         │
│    └──────────────┘                         │
└─────────────────────────────────────────────┘
```

### 9.2 逻辑复制架构

```
┌─────────────────────────────────────────────┐
│            Publisher Database                │
│                                              │
│  ┌──────────┐                               │
│  │  Table   │  CREATE PUBLICATION           │
│  │  users   │                               │
│  └──────────┘                               │
│       │                                      │
│       ↓                                      │
│  ┌──────────────────┐                       │
│  │ Logical Decoding │                       │
│  │   (pgoutput)     │                       │
│  └──────────────────┘                       │
│       │                                      │
│       ↓                                      │
│  ┌──────────────────┐                       │
│  │  WAL Sender      │                       │
│  │  (Logical)       │                       │
│  └──────────────────┘                       │
└───────────┼────────────────────────────────┘
            │
            │ TCP Stream (Row Changes)
            │
            ↓
┌─────────────────────────────────────────────┐
│           Subscriber Database                │
│                                              │
│  ┌──────────────────┐                       │
│  │ Logical Receiver │                       │
│  └──────────────────┘                       │
│       │                                      │
│       ↓                                      │
│  ┌──────────────────┐                       │
│  │  Apply Worker    │                       │
│  └──────────────────┘                       │
│       │                                      │
│       ↓                                      │
│  ┌──────────┐  CREATE SUBSCRIPTION          │
│  │  Table   │                               │
│  │  users   │                               │
│  └──────────┘                               │
└─────────────────────────────────────────────┘
```

---

## 总结

### 复制核心

1. **流复制**: 物理复制，整库复制，用于HA/DR
2. **逻辑复制**: 表级复制，用于数据分发
3. **复制槽**: 确保WAL不丢失
4. **Hot Standby**: 备库支持只读查询
5. **同步/异步**: 平衡性能和一致性

### 最佳实践

- ✅ 使用复制槽防止WAL丢失
- ✅ 监控复制延迟和WAL堆积
- ✅ 定期演练故障切换
- ✅ 配置合适的超时参数
- ✅ 使用级联复制减轻Primary压力

---

**完成**: Replication复制系统核心文档完成！

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

