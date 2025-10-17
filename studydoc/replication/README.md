# Replication 复制系统 (精简版)

> PostgreSQL流复制、逻辑复制核心原理

**源码**: `src/backend/replication/`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 📚 核心概念

**Replication** 是PostgreSQL的**高可用和容灾**核心机制，提供数据的实时复制。

### 复制类型

```
✅ 流复制 (Streaming Replication) - 物理复制
   • 复制整个数据库集群
   • 二进制WAL传输
   • 用于HA/DR

✅ 逻辑复制 (Logical Replication) - 逻辑复制
   • 表级别复制
   • 行级别复制
   • 用于数据分发、升级
```

---

## 🎯 复制架构

```
Primary (主库)
  │
  ├─ WAL Writer → 写入WAL
  │
  ├─ WAL Sender → 发送WAL到Standby
  │   ├─ 同步模式 (synchronous_commit = on)
  │   └─ 异步模式 (synchronous_commit = off)
  │
  └─ Replication Slots → 保留WAL不被删除

         ↓ (WAL Stream)

Standby (备库)
  │
  ├─ WAL Receiver → 接收WAL
  │
  ├─ Startup Process → 应用WAL (恢复)
  │
  └─ Hot Standby → 支持只读查询
```

---

**详细内容**: 阅读 [01_replication_core.md](01_replication_core.md)

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

