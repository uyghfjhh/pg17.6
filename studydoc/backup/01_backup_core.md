# Backup/Recovery 核心分析 (精简版)

> PostgreSQL备份恢复的核心原理、机制和最佳实践

**源码**: `src/backend/backup/`, `src/backend/access/transam/xlog.c`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. 备份类型对比

### 1.1 三种备份方式

```
┌──────────────┬─────────────┬─────────────┬─────────────┐
│  备份类型    │  工具       │  恢复方式   │  适用场景   │
├──────────────┼─────────────┼─────────────┼─────────────┤
│ 逻辑备份     │ pg_dump     │ SQL导入     │ 单表/小库   │
│              │ pg_dumpall  │             │ 跨版本迁移  │
├──────────────┼─────────────┼─────────────┼─────────────┤
│ 物理备份     │ pg_basebackup│ 直接启动   │ 全库备份    │
│              │ 文件系统快照│             │ 大库快速恢复│
├──────────────┼─────────────┼─────────────┼─────────────┤
│ PITR         │ 基础备份+WAL│ 时间点恢复  │ 数据误删恢复│
│ (连续归档)   │ 归档        │             │ 精确恢复    │
└──────────────┴─────────────┴─────────────┴─────────────┘
```

---

## 2. 逻辑备份 (pg_dump)

### 2.1 基本用法

```bash
# 备份单个数据库
pg_dump -U postgres -d mydb -f mydb.sql

# 备份所有数据库
pg_dumpall -U postgres -f all_databases.sql

# 备份单张表
pg_dump -U postgres -d mydb -t users -f users.sql

# 压缩备份
pg_dump -U postgres -d mydb | gzip > mydb.sql.gz

# 自定义格式 (并行恢复)
pg_dump -U postgres -d mydb -Fc -f mydb.dump

# 恢复
psql -U postgres -d mydb -f mydb.sql
pg_restore -U postgres -d mydb mydb.dump  # 自定义格式
```

### 2.2 工作原理

```
pg_dump工作流程:

[1] 开启事务 (SERIALIZABLE隔离级别)
    └─ BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE

[2] 导出Schema定义
    ├─ CREATE TABLE
    ├─ CREATE INDEX
    └─ CREATE CONSTRAINT

[3] 导出数据 (COPY命令)
    └─ COPY users TO STDOUT

[4] 提交事务
    └─ COMMIT

特点:
  ✅ 一致性快照 (事务隔离保证)
  ✅ 跨版本兼容
  ✅ 可读的SQL文本
  ✗ 速度较慢 (大库)
  ✗ 无法PITR
```

### 2.3 高级选项

```bash
# 只备份Schema (不备份数据)
pg_dump -s -d mydb -f schema.sql

# 只备份数据 (不备份Schema)
pg_dump -a -d mydb -f data.sql

# 排除某些表
pg_dump -d mydb -T 'logs_*' -f mydb.sql

# 并行导出 (自定义格式)
pg_dump -d mydb -Fd -j 4 -f mydb_dump/

# 并行恢复
pg_restore -d mydb -j 4 mydb_dump/
```

---

## 3. 物理备份 (pg_basebackup)

### 3.1 基本用法

```bash
# 基础备份
pg_basebackup -U replication -D /backup/base -Fp -P

参数说明:
  -D: 备份目录
  -Fp: Plain格式 (目录)
  -Ft: Tar格式 (压缩)
  -P: 显示进度
  -X stream: 包含WAL (推荐)
  -c fast: 快速检查点

# 完整示例
pg_basebackup \
  -U replication \
  -D /backup/base_$(date +%Y%m%d) \
  -Fp \
  -X stream \
  -c fast \
  -P
```

### 3.2 工作原理

```
pg_basebackup流程:

Primary (主库):
  │
  [1] 连接到replication用户
  │
  [2] 发起pg_start_backup()
  │    ├─ 创建检查点
  │    └─ 标记备份开始
  │
  [3] 复制数据目录
  │    ├─ 所有数据文件
  │    ├─ 配置文件
  │    └─ WAL文件 (-X stream)
  │
  [4] 调用pg_stop_backup()
  │    └─ 标记备份完成
  │
  [5] 备份完成
       └─ 可直接用于恢复/搭建备库

特点:
  ✅ 快速 (文件级复制)
  ✅ 一致性保证
  ✅ 支持增量备份 (with WAL)
  ✗ 同版本限制
  ✗ 不能跨平台
```

### 3.3 配置要求

```sql
-- postgresql.conf
wal_level = replica              -- 启用复制级别WAL
archive_mode = on                -- 启用归档
archive_command = 'cp %p /archive/%f'  -- 归档命令
max_wal_senders = 10             -- 最多10个WAL发送进程
wal_keep_size = 1GB              -- 保留1GB WAL

-- pg_hba.conf (允许replication连接)
host replication replication 0.0.0.0/0 md5

-- 创建replication角色
CREATE ROLE replication WITH REPLICATION LOGIN PASSWORD 'secret';
```

---

## 4. PITR (Point-In-Time Recovery)

### 4.1 连续归档配置

```sql
-- 主库配置 (postgresql.conf)
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
  -- %p: WAL文件完整路径
  -- %f: WAL文件名

-- 推荐: 使用pg_receivewal实时接收WAL
pg_receivewal -D /archive -U replication -h primary
```

### 4.2 PITR恢复流程

```bash
# [步骤1] 停止数据库
pg_ctl stop

# [步骤2] 清空数据目录
rm -rf $PGDATA/*

# [步骤3] 恢复基础备份
cp -r /backup/base/* $PGDATA/

# [步骤4] 创建recovery.signal文件
touch $PGDATA/recovery.signal

# [步骤5] 配置恢复参数 (postgresql.conf)
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2025-10-16 14:30:00'  # 恢复到此时间点
# 或者:
# recovery_target_xid = '1000'      # 恢复到指定事务ID
# recovery_target_lsn = '0/3000000' # 恢复到指定LSN
# recovery_target_name = 'savepoint1'  # 恢复到指定还原点

# [步骤6] 启动数据库
pg_ctl start

# [步骤7] 等待恢复完成
# 数据库会自动应用WAL归档，直到达到目标时间点
# 完成后recovery.signal自动删除
```

### 4.3 创建还原点

```sql
-- 在重要操作前创建还原点
SELECT pg_create_restore_point('before_data_migration');

-- 恢复时指定还原点
recovery_target_name = 'before_data_migration'
```

---

## 5. 归档命令详解

### 5.1 本地归档

```bash
# 简单复制
archive_command = 'cp %p /archive/%f'

# 带错误处理
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'

# 压缩归档
archive_command = 'gzip < %p > /archive/%f.gz'
```

### 5.2 远程归档

```bash
# rsync到远程服务器
archive_command = 'rsync -a %p backup_server:/archive/%f'

# scp到远程服务器
archive_command = 'scp %p user@backup_server:/archive/%f'

# 使用pgBackRest (推荐)
archive_command = 'pgbackrest --stanza=main archive-push %p'
```

### 5.3 监控归档状态

```sql
-- 查看归档统计
SELECT * FROM pg_stat_archiver;
/*
  archived_count: 已归档WAL数量
  last_archived_wal: 最后归档的WAL
  last_archived_time: 最后归档时间
  failed_count: 失败次数
  last_failed_wal: 最后失败的WAL
*/

-- 查看当前WAL文件
SELECT pg_current_wal_lsn();
SELECT pg_walfile_name(pg_current_wal_lsn());

-- 检查WAL堆积
SELECT 
    count(*) as pending_wals
FROM pg_ls_waldir()
WHERE modification > now() - interval '1 hour';
```

---

## 6. 备份策略

### 6.1 3-2-1备份规则

```
3-2-1 备份策略:
  ✅ 3份数据副本 (1原始 + 2备份)
  ✅ 2种不同介质 (磁盘 + 云存储)
  ✅ 1份异地备份 (远程机房)
```

### 6.2 推荐备份方案

```
[小型数据库] (< 100GB)
  • 每天逻辑备份 (pg_dump)
  • 保留7天备份
  • 压缩存储

[中型数据库] (100GB - 1TB)
  • 每周物理全备 (pg_basebackup)
  • 每天WAL归档 (PITR)
  • 保留4周全备 + 所有WAL

[大型数据库] (> 1TB)
  • 每周物理全备
  • 每天增量备份 (使用pgBackRest)
  • 连续WAL归档
  • 保留8周全备 + 所有WAL
```

### 6.3 自动化脚本

```bash
#!/bin/bash
# 每日备份脚本

DATE=$(date +%Y%m%d)
BACKUP_DIR=/backup/$DATE

# 创建备份目录
mkdir -p $BACKUP_DIR

# 物理全备
pg_basebackup \
  -U replication \
  -D $BACKUP_DIR/base \
  -Fp \
  -X stream \
  -c fast \
  -P

# 压缩备份
tar czf $BACKUP_DIR.tar.gz $BACKUP_DIR/

# 删除30天前的备份
find /backup -name "*.tar.gz" -mtime +30 -delete

# 上传到云存储
rclone copy $BACKUP_DIR.tar.gz remote:postgres-backups/

echo "Backup completed: $BACKUP_DIR.tar.gz"
```

---

## 7. 恢复测试

### 7.1 定期恢复演练

```bash
# 每月恢复测试脚本

# 1. 选择最近的备份
LATEST_BACKUP=$(ls -t /backup/*.tar.gz | head -1)

# 2. 解压到测试目录
mkdir /tmp/recovery_test
tar xzf $LATEST_BACKUP -C /tmp/recovery_test

# 3. 创建测试实例
pg_ctl -D /tmp/recovery_test/base start -o "-p 5433"

# 4. 验证数据
psql -p 5433 -c "SELECT count(*) FROM users;"

# 5. 清理
pg_ctl -D /tmp/recovery_test/base stop
rm -rf /tmp/recovery_test

echo "Recovery test completed successfully!"
```

---

## 8. 第三方备份工具

### 8.1 pgBackRest (推荐)

```bash
# 安装
yum install pgbackrest

# 配置 /etc/pgbackrest.conf
[main]
pg1-path=/var/lib/pgsql/17/data

# 创建stanza
pgbackrest --stanza=main stanza-create

# 全备
pgbackrest --stanza=main backup --type=full

# 增量备份
pgbackrest --stanza=main backup --type=incr

# 差异备份
pgbackrest --stanza=main backup --type=diff

# 查看备份
pgbackrest --stanza=main info

# 恢复
pgbackrest --stanza=main restore
```

### 8.2 Barman

```bash
# 安装
yum install barman

# 配置主库
# postgresql.conf
archive_mode = on
archive_command = 'rsync -a %p barman@backup-server:/var/lib/barman/main/incoming/%f'

# Barman服务器
barman check main
barman backup main
barman list-backup main
barman recover main latest /var/lib/pgsql/17/data
```

---

## 9. 监控和告警

### 9.1 监控指标

```sql
-- 备份年龄
SELECT 
    now() - pg_stat_file('base/backup_label')::timestamp as backup_age
WHERE pg_stat_file('base/backup_label') IS NOT NULL;

-- 归档延迟
SELECT 
    EXTRACT(EPOCH FROM (now() - last_archived_time)) as archive_delay_seconds
FROM pg_stat_archiver;

-- WAL归档失败
SELECT 
    failed_count,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver
WHERE failed_count > 0;
```

### 9.2 告警规则

```
告警条件:
  ⚠️ 备份超过24小时未完成
  ⚠️ 归档延迟超过5分钟
  ⚠️ 归档失败次数 > 0
  ⚠️ WAL堆积 > 100个文件
  ⚠️ 备份磁盘空间 < 20%
```

---

## 10. 最佳实践

### 10.1 备份建议

```
✅ 自动化备份 (cron + 脚本)
✅ 异地存储 (云存储/远程机房)
✅ 定期恢复测试 (每月一次)
✅ 监控备份状态
✅ 加密敏感备份
✅ 保留多代备份
✅ 文档化恢复流程
```

### 10.2 常见错误

```
❌ 不测试恢复流程
   → 备份无效时才发现

❌ 只有单个备份
   → 备份损坏时无替代

❌ 备份与生产同存储
   → 存储故障时全部丢失

❌ 未监控归档状态
   → WAL归档失败导致PITR失败

❌ 未加密备份
   → 数据泄露风险
```

---

## 总结

### 备份核心要点

1. **逻辑备份**: pg_dump - 跨版本、小库适用
2. **物理备份**: pg_basebackup - 大库、快速恢复
3. **PITR**: 连续归档 + 时间点恢复 - 精确恢复
4. **自动化**: 脚本 + 监控 + 告警
5. **测试**: 定期恢复演练验证备份有效性

### 最佳实践

- ✅ 3-2-1备份策略
- ✅ 使用pgBackRest增量备份
- ✅ 异地存储备份
- ✅ 每月恢复测试
- ✅ 监控归档状态

---

**完成**: Backup/Recovery核心文档完成！

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

