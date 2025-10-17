# Backup/Recovery 备份恢复 (精简版)

> PostgreSQL备份恢复机制核心原理

**源码**: `src/backend/backup/`, `src/backend/access/transam/xlog.c`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 📚 核心概念

**Backup/Recovery** 是PostgreSQL的**数据保护和灾难恢复**核心机制。

### 备份类型

```
✅ 逻辑备份 (Logical Backup)
   • pg_dump / pg_dumpall
   • SQL文件
   • 跨版本恢复

✅ 物理备份 (Physical Backup)
   • pg_basebackup
   • 文件系统快照
   • 同版本恢复

✅ 连续归档 (WAL Archiving)
   • archive_mode = on
   • 归档WAL文件
   • PITR (时间点恢复)
```

---

## 🎯 备份架构

```
┌─────────────────────────────────────────┐
│         PostgreSQL 备份恢复架构         │
└─────────────────────────────────────────┘

[1] 逻辑备份:
    pg_dump → SQL文件 → psql恢复

[2] 物理备份:
    pg_basebackup → 数据目录 → 直接启动

[3] PITR:
    基础备份 + WAL归档 → 恢复到任意时间点
    
    Primary:
      ├─ WAL Writer
      ├─ Archiver → archive_command
      └─ WAL归档目录
      
    Recovery:
      ├─ 恢复基础备份
      ├─ 应用WAL归档
      └─ 恢复到指定时间点
```

---

**详细内容**: 阅读 [01_backup_core.md](01_backup_core.md)

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

