# 增量备份 - 节省存储和时间

> PostgreSQL 17的备份革命性改进

**特性**: 增量备份 (Incremental Backup)  
**重要程度**: ⭐⭐⭐⭐ (P1)  
**性能提升**: 存储节省83%+，备份时间节省90%+  
**源码位置**: `src/bin/pg_basebackup/`, `src/backend/backup/`

---

## 📋 问题背景

### PostgreSQL 16的备份方式

```
【全量备份 - 每次备份所有数据】
┌────────────────────────────────────────┐
│ 每次pg_basebackup:                     │
│                                        │
│ Day 1: 备份1TB → 1TB                   │
│ Day 2: 备份1TB → 1TB (即使只改了1GB)  │
│ Day 3: 备份1TB → 1TB (即使只改了2GB)  │
│ Day 4: 备份1TB → 1TB                   │
│ Day 5: 备份1TB → 1TB                   │
│                                        │
│ 5天备份: 5TB存储空间                   │
│ 每次备份: ~4小时 (假设SSD 70MB/s)     │
└────────────────────────────────────────┘

【问题分析】
┌──────────────────────────────────────────┐
│ 问题1: 存储浪费严重                      │
│ ─────────────────────────────            │
│ 数据库: 1TB                              │
│ 每日变化: ~2% (20GB)                     │
│ 但每次备份1TB                            │
│ 98%的数据是重复的!                       │
│                                          │
│ 7天备份保留:                             │
│ - 7 * 1TB = 7TB存储                     │
│ - 实际只需: 1TB + 6*20GB = 1.12TB      │
│ - 浪费: 83%                             │
│                                          │
│ 问题2: 备份时间长                        │
│ ─────────────────────────────            │
│ 每次需要读取1TB数据                      │
│ 网络传输1TB                              │
│ 影响生产系统I/O                          │
│                                          │
│ 问题3: 备份窗口限制                      │
│ ─────────────────────────────            │
│ 大型数据库(10TB+):                       │
│ - 备份需要24小时+                        │
│ - 无法每日备份                           │
│ - RPO/RTO难以满足                        │
│                                          │
│ 问题4: 恢复复杂                          │
│ ─────────────────────────────            │
│ 只能恢复到某个全量备份点                 │
│ 无法灵活选择恢复点                       │
└──────────────────────────────────────────┘
```

### 真实场景影响

```bash
# 生产环境示例
数据库大小: 5TB
每日变化: 1% (50GB)

# PG 16备份方案
每日全量备份:
  - 备份时间: 20小时 (SSD RAID)
  - 存储空间: 7天 = 35TB
  - 月度成本: $3,500 (云存储$0.1/GB)
  - 网络带宽: 70MB/s 持续20小时

# 问题:
1. 备份窗口太长，影响业务
2. 存储成本高昂
3. 无法频繁备份 (RPO=24小时)
```

---

## 核心改进

### PostgreSQL 17增量备份

```
【增量备份 - 只备份变化数据】
┌────────────────────────────────────────┐
│ Day 1: 全量备份 → 1TB                  │
│        创建WAL Summary                 │
│        ↓                               │
│ Day 2: 增量备份 → 20GB (只备份变化)   │
│        基于Day 1                       │
│        ↓                               │
│ Day 3: 增量备份 → 18GB                 │
│        基于Day 1                       │
│        ↓                               │
│ Day 4: 增量备份 → 22GB                 │
│        基于Day 1                       │
│        ↓                               │
│ Day 5: 增量备份 → 19GB                 │
│        基于Day 1                       │
│                                        │
│ 5天备份: 1TB + 79GB = 1.08TB          │
│ 节省: 79% 存储空间!                    │
│ Day 2-5备份时间: ~15分钟/次            │
└────────────────────────────────────────┘

【架构设计】
┌──────────────────────────────────────────┐
│                                          │
│ ┌──────────────┐                        │
│ │ Full Backup  │ ← Day 1                │
│ │   1TB        │                        │
│ └──────┬───────┘                        │
│        │                                │
│        ├──→ WAL Summary (LSN范围)      │
│        │    记录哪些block被修改         │
│        │                                │
│        ├──→ Incremental 1 (Day 2)      │
│        │    只包含修改的blocks          │
│        │    20GB                        │
│        │                                │
│        ├──→ Incremental 2 (Day 3)      │
│        │    18GB                        │
│        │                                │
│        └──→ Incremental 3 (Day 4)      │
│             22GB                        │
│                                          │
│ 恢复到Day 4:                            │
│   Full Backup + Incremental 3           │
│   = 1TB + 22GB                          │
└──────────────────────────────────────────┘
```

---

## 技术实现

### WAL Summary - 变化跟踪

```c
/*
 * WAL Summary - 跟踪数据块变化
 * 文件: src/backend/backup/walsummary.c
 */

/*
 * WAL Summary结构
 * 记录在特定LSN范围内哪些blocks被修改
 */
typedef struct WALSummary
{
    /* LSN范围 */
    XLogRecPtr start_lsn;   // 起始LSN
    XLogRecPtr end_lsn;     // 结束LSN
    
    /* 修改的blocks位图 */
    /* 使用位图高效存储修改信息 */
    BlockNumberBitmap *modified_blocks;
    
    /* 每个表空间/数据库/关系 */
    RelFileNode rnode;
    
    /* 统计信息 */
    uint64 num_blocks_modified;
    uint64 total_blocks;
} WALSummary;

/*
 * 创建WAL Summary
 * 在全量备份时调用
 */
void
CreateWALSummary(XLogRecPtr start_lsn)
{
    WALSummaryContext *ctx;
    
    /* 创建summary上下文 */
    ctx = palloc0(sizeof(WALSummaryContext));
    ctx->start_lsn = start_lsn;
    ctx->end_lsn = InvalidXLogRecPtr;
    
    /*
     * 初始化位图
     * 每个数据文件一个位图
     */
    ctx->block_bitmaps = hash_create("WAL Summary Bitmaps",
                                     1024,  // 初始大小
                                     HASH_ELEM | HASH_BLOBS);
    
    /* 保存到磁盘 */
    /* 文件路径: pg_wal/summaries/00000001.summary */
    WriteWALSummaryFile(ctx);
}

/*
 * 更新WAL Summary
 * 在WAL回放时调用，记录修改的blocks
 */
void
UpdateWALSummary(XLogReaderState *record)
{
    uint8 info = XLogRecGetInfo(record) & ~XLR_INFO_MASK;
    
    /* 解析WAL记录 */
    if (info == XLOG_HEAP_INSERT ||
        info == XLOG_HEAP_UPDATE ||
        info == XLOG_HEAP_DELETE)
    {
        /* 提取被修改的block */
        RelFileNode rnode;
        BlockNumber blkno;
        
        XLogRecGetBlockTag(record, 0, &rnode, NULL, &blkno);
        
        /*
         * 在位图中标记这个block被修改
         * 位图操作: O(1) 时间复杂度
         */
        MarkBlockModified(rnode, blkno);
    }
    
    /* 类似处理B-tree, GIN等索引的WAL记录 */
}

/*
 * 读取WAL Summary
 * 在增量备份时调用
 */
BlockNumberBitmap *
ReadWALSummary(XLogRecPtr start_lsn, 
               XLogRecPtr end_lsn,
               RelFileNode rnode)
{
    WALSummaryFile *file;
    BlockNumberBitmap *bitmap;
    
    /* 打开summary文件 */
    file = OpenWALSummaryFile(start_lsn);
    
    /* 读取指定关系的位图 */
    bitmap = ReadBlockBitmap(file, rnode);
    
    /*
     * 位图示例:
     * Block 0: 1 (修改)
     * Block 1: 0
     * Block 2: 1 (修改)
     * Block 3: 0
     * ...
     */
    
    return bitmap;
}
```

### 增量备份执行

```c
/*
 * 执行增量备份
 * 文件: src/bin/pg_basebackup/pg_basebackup.c
 */

/*
 * 增量备份主函数
 */
void
PerformIncrementalBackup(char *basedir,
                        char *incremental_from,
                        XLogRecPtr startptr)
{
    WALSummaryContext *summary;
    DIR *dir;
    
    /*
     * 步骤1: 读取基础备份的LSN
     */
    XLogRecPtr base_lsn = ReadBackupLabel(incremental_from);
    
    /*
     * 步骤2: 读取WAL Summary
     * 获取从base_lsn到startptr之间修改的blocks
     */
    summary = ReadWALSummary(base_lsn, startptr, NULL);
    
    /*
     * 步骤3: 遍历所有数据文件
     */
    dir = AllocateDir(basedir);
    while ((entry = ReadDir(dir, basedir)) != NULL)
    {
        RelFileNode rnode;
        BlockNumberBitmap *modified;
        
        /* 解析文件名获取RelFileNode */
        if (!ParseRelFileName(entry->d_name, &rnode))
            continue;
        
        /*
         * 步骤4: 获取这个文件的修改位图
         */
        modified = GetModifiedBlocks(summary, rnode);
        
        if (modified == NULL || bitmap_is_empty(modified))
        {
            /* 这个文件没有修改，跳过 */
            continue;
        }
        
        /*
         * 步骤5: 只复制修改的blocks
         */
        CopyModifiedBlocks(entry->d_name, 
                          basedir,
                          modified,
                          outputdir);
    }
    
    /*
     * 步骤6: 记录备份元数据
     */
    WriteIncrementalBackupLabel(outputdir, 
                               base_lsn,
                               startptr,
                               incremental_from);
}

/*
 * 复制修改的blocks
 */
static void
CopyModifiedBlocks(char *filename,
                  char *datadir,
                  BlockNumberBitmap *modified,
                  char *outputdir)
{
    FILE *infile, *outfile;
    BlockNumber blkno;
    char buffer[BLCKSZ];
    
    infile = fopen(filename, "rb");
    outfile = fopen(outputfile, "wb");
    
    /*
     * 遍历位图中所有修改的blocks
     */
    for (blkno = 0; blkno < MaxBlockNumber; blkno++)
    {
        if (!bitmap_is_set(modified, blkno))
            continue;  // 这个block没修改，跳过
        
        /*
         * 读取这个block
         * 位置: blkno * BLCKSZ
         */
        fseek(infile, blkno * BLCKSZ, SEEK_SET);
        fread(buffer, BLCKSZ, 1, infile);
        
        /*
         * 写入增量备份文件
         * 格式: [BlockNumber][Block Data]
         */
        fwrite(&blkno, sizeof(BlockNumber), 1, outfile);
        fwrite(buffer, BLCKSZ, 1, outfile);
    }
    
    fclose(infile);
    fclose(outfile);
}
```

### 增量备份恢复

```c
/*
 * 从增量备份恢复
 * 文件: src/bin/pg_combinebackup/pg_combinebackup.c
 */

/*
 * pg_combinebackup - 合并全量+增量备份
 * 
 * 用法:
 *   pg_combinebackup -o /path/to/restored \\
 *     /path/to/full_backup \\
 *     /path/to/incremental_backup
 */
int
main(int argc, char **argv)
{
    char *full_backup_path;
    char *incremental_backup_path;
    char *output_path;
    
    /* 解析参数... */
    
    /*
     * 步骤1: 复制全量备份到输出目录
     */
    CopyDirectory(full_backup_path, output_path);
    
    /*
     * 步骤2: 应用增量备份
     */
    ApplyIncrementalBackup(incremental_backup_path,
                          output_path);
    
    /*
     * 步骤3: 更新backup_label
     */
    UpdateBackupLabel(output_path);
    
    return 0;
}

/*
 * 应用增量备份
 */
static void
ApplyIncrementalBackup(char *incr_path,
                      char *target_path)
{
    DIR *dir;
    struct dirent *entry;
    
    /* 遍历增量备份中的所有文件 */
    dir = opendir(incr_path);
    while ((entry = readdir(dir)) != NULL)
    {
        if (!IsIncrementalFile(entry->d_name))
            continue;
        
        /*
         * 读取增量文件
         * 格式: [BlockNumber][Block Data]
         */
        FILE *incr_file = fopen(entry->d_name, "rb");
        
        /* 打开目标数据文件 */
        char target_file[MAXPGPATH];
        snprintf(target_file, MAXPGPATH, 
                "%s/%s", target_path, entry->d_name);
        FILE *data_file = fopen(target_file, "r+b");
        
        /* 应用每个修改的block */
        while (!feof(incr_file))
        {
            BlockNumber blkno;
            char buffer[BLCKSZ];
            
            /* 读取block号和数据 */
            fread(&blkno, sizeof(BlockNumber), 1, incr_file);
            fread(buffer, BLCKSZ, 1, incr_file);
            
            /*
             * 覆盖目标文件中的对应block
             */
            fseek(data_file, blkno * BLCKSZ, SEEK_SET);
            fwrite(buffer, BLCKSZ, 1, data_file);
        }
        
        fclose(incr_file);
        fclose(data_file);
    }
    
    closedir(dir);
}
```

---

## 使用示例

### 创建增量备份

```bash
# 1. 第一次: 全量备份
pg_basebackup -D /backup/full_20251017 \\
  -Ft -z -P \\
  --wal-method=stream

# 备份时间: 4小时
# 备份大小: 1TB
# 创建WAL Summary: 自动

# 2. 第二天: 增量备份
pg_basebackup -D /backup/incr_20251018 \\
  -Ft -z -P \\
  --incremental=/backup/full_20251017/backup_manifest \\
  --wal-method=stream

# 备份时间: 15分钟 (-94%)
# 备份大小: 20GB (-98%)
# 基于: full_20251017

# 3. 第三天: 另一个增量备份
pg_basebackup -D /backup/incr_20251019 \\
  -Ft -z -P \\
  --incremental=/backup/full_20251017/backup_manifest \\
  --wal-method=stream

# 备份时间: 12分钟
# 备份大小: 18GB
# 基于: full_20251017 (不是incr_20251018!)
```

### 从增量备份恢复

```bash
# 恢复到10月19日的状态
# 需要: 全量备份 + 10月19日的增量备份

# 方法1: 使用pg_combinebackup
pg_combinebackup \\
  -o /restore/recovered_20251019 \\
  /backup/full_20251017 \\
  /backup/incr_20251019

# 输出: 完整的数据目录
# 可以直接启动PostgreSQL

# 方法2: 手动恢复
# 1. 解压全量备份
tar xzf /backup/full_20251017/base.tar.gz -C /restore/data

# 2. 应用增量备份
pg_combinebackup \\
  -o /restore/data \\
  /restore/data \\
  /backup/incr_20251019

# 3. 启动数据库
pg_ctl -D /restore/data start
```

---

## 性能分析

### 存储节省

```
【存储对比】
场景: 1TB数据库，每日2%变化，保留7天

PG 16 (全量备份):
  Day 1: 1.00TB
  Day 2: 1.00TB
  Day 3: 1.00TB
  Day 4: 1.00TB
  Day 5: 1.00TB
  Day 6: 1.00TB
  Day 7: 1.00TB
  ────────────
  总计: 7.00TB

PG 17 (增量备份):
  Day 1: 1.00TB (全量)
  Day 2: 0.02TB (增量)
  Day 3: 0.02TB (增量)
  Day 4: 0.02TB (增量)
  Day 5: 0.02TB (增量)
  Day 6: 0.02TB (增量)
  Day 7: 0.02TB (增量)
  ────────────
  总计: 1.12TB

节省: 83.4%
```

### 时间节省

```
【备份时间对比】
数据库: 5TB
网络/存储: 70MB/s
每日变化: 1% (50GB)

PG 16:
  每次备份: 20小时
  月度总时间: 600小时

PG 17:
  全量备份 (每周): 20小时 * 4 = 80小时
  增量备份 (每日): 0.2小时 * 26 = 5.2小时
  月度总时间: 85.2小时

节省: 85.8%
```

---

## 最佳实践

### 备份策略

```bash
# 推荐策略: 每周全量 + 每日增量

# 周日: 全量备份
0 2 * * 0 pg_basebackup -D /backup/full_$(date +\%Y\%m\%d) ...

# 周一-周六: 增量备份
0 2 * * 1-6 pg_basebackup -D /backup/incr_$(date +\%Y\%m\%d) \\
  --incremental=/backup/full_latest/backup_manifest ...

# 保留策略
# - 保留4周的全量备份
# - 保留最近7天的增量备份
```

### 监控建议

```sql
-- 监控WAL Summary大小
SELECT pg_ls_dir('pg_wal/summaries');

-- 检查增量备份大小
\! du -sh /backup/incr_*

-- 预估下次增量备份大小
SELECT 
    pg_size_pretty(
        sum(pg_relation_size(oid)) * 
        (SELECT pg_current_wal_lsn() - pg_backup_start_lsn)
        / pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0')
    ) as estimated_incremental_size
FROM pg_class
WHERE relkind IN ('r', 'i');
```

---

## 总结

### 核心优势

1. **存储节省**: 83%+ 存储空间
2. **时间节省**: 90%+ 备份时间
3. **灵活恢复**: 任意增量点恢复
4. **降低影响**: 减少对生产的I/O影响

### 技术要点

- ✅ **WAL Summary**: 高效跟踪块变化
- ✅ **位图存储**: O(1)查询和更新
- ✅ **块级增量**: 只复制变化的数据块
- ✅ **链式备份**: 可基于任一全量备份

### 适用场景

- ✅ 大型数据库 (1TB+)
- ✅ 频繁备份需求
- ✅ 存储成本敏感
- ✅ 备份窗口受限

---

**文档版本**: PostgreSQL 17.6  
**创建日期**: 2025-10-17  
**重要程度**: ⭐⭐⭐⭐

