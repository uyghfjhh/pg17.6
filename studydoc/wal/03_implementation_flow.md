# WAL 实现流程分析

> 深入分析 WAL 系统的完整实现流程，包括记录插入、刷写、恢复等关键路径。

---

## 目录

1. [WAL 记录插入流程](#一wal-记录插入流程)
2. [WAL 刷写入磁盘流程](#二wal-刷写入磁盘流程)
3. [WAL 恢复和重放流程](#三wal-恢复和重放流程)
4. [WAL 清理和回收流程](#四wal-清理和回收流程)
5. [WAL 复制和传输流程](#五wal-复制和传输流程)
6. [关键并发控制机制](#六关键并发控制机制)
7. [错误处理和异常流程](#七错误处理和异常流程)

---

## 一、WAL 记录插入流程

### 1.1 XLogInsert 核心流程

**源码位置**: `src/backend/access/transam/xloginsert.c:330 XLogInsertRecord()`

**流程图**:
```
Backend Process                    Shared Memory
─────────────────┐          ┌──────────────────────────────┐
XLogInsert()      │          │                              │
                  │          │                              │
┌─────────────────┤          │                              │
│ 1. 构造记录数据  │          │                              │
│    - 准备主数据  │          │                              │
│    - 准备块数据  │          │                              │
│    - 计算总大小  │          │                              │
└─────────────────┘          │                              │
                  │          │                              │
┌─────────────────┤          │                              │
│ 2. 获取插入锁    ├──────────►│ LWLockAcquire(WALInsertLock)│
│    (独占锁)      │          │                              │
└─────────────────┘          │                              │
                  │          │                              │
┌─────────────────┤          │               ┌──────────────┐
│ 3. 保留空间     ├──────────►│ ReserveLogSpace │              │
│    - 计算位置    │          │               │              │
│    - 检查边界    │          │               ▼              │
│    - 更新指针    │          │        CurrBytePos += size   │
└─────────────────┘          │                              │
                  │          │                              │
┌─────────────────┤          │               ┌──────────────┐
│ 4. 复制数据     ├──────────►│ CopyToWALBuffer│              │
│    - 复制头部    │          │               │              │
│    - 复制数据    │          │               ▼              │
│    - 更新偏移    │          │        memcpy() to buffers   │
└─────────────────┘          │                              │
                  │          │                              │
┌─────────────────┤          │               ┌──────────────┐
│ 5. 释放插入锁    ├──────────►│ LWLockRelease  │              │
│    (可以并发)    │          │               │              │
└─────────────────┘          └───────────────┘              │
                  │                                          │
┌─────────────────┤                                          │
│ 6. 标记页脏     ├──────────────────────────────────────────►│
│    MarkBufferDirty │            │                            │
└─────────────────┘          │                            │
                              │                            │
┌─────────────────┤          │                            │
│ 7. 返回 LSN     │          │                            │
│    给调用者     │          │                            │
└─────────────────┘          └────────────────────────────┘
```

### 1.2 详细代码分析

#### 1.2.1 记录数据准备

```c
// src/backend/access/transam/xloginsert.c:200
XLogRecPtr
XLogInsertRecord(RmgrId rmid, uint8 info,
                 XLogRecData *rdata,
                 XLogRecPtr RedoRecPtr, bool fpw)
{
    XLogRecData rd;
    XLogRecPtr  EndPos;
    uint32      len;
    
    // 1. 计算记录总大小
    len = calculate_total_size(rdata);
    
    // 2. 准备 XLogRecData 链表
    // 头部数据
    rd.data = (char *) &constructed_hdr;
    rd.len = hdr_len;
    rdata = &rd;
    rd.next = rdata;
    
    // 3. 保留 WAL 空间 (原子操作)
    START_CRIT_SECTION();
    
    // 获取插入锁
    LWLockAcquire(WALInsertLock, LW_EXCLUSIVE);
    
    // 保留空间
    EndPos = ReserveLogSpace(len);
    
    // 4. 复制数据到 WAL 缓冲区
    CopyDataToWALBuffer(rdata, EndPos);
    
    // 5. 更新控制信息
    XLogCtl->Insert.PrevBytePos = EndPos - len;
    
    LWLockRelease(WALInsertLock);
    
    END_CRIT_SECTION();
    
    return EndPos - len;
}
```

#### 1.2.2 空间保留算法

```c
// src/backend/access/transam/xlog.c:800
static XLogRecPtr
ReserveLogSpace(uint32 size)
{
    XLogRecPtr  startBytePos;
    XLogRecPtr  endBytePos;
    
    // 原子获取当前插入位置
    startBytePos = pg_atomic_read_u64(&XLogCtl->Insert.CurrBytePos);
    
    // 计算结束位置
    endBytePos = startBytePos + size;
    
    // 检查是否跨越边界
    if (endBytePos / XLOG_BLCKSZ != startBytePos / XLOG_BLCKSZ)
    {
        // 需要填充到下一页
        uint32 align = XLOG_BLCKSZ - (startBytePos % XLOG_BLCKSZ);
        size += align;
        endBytePos = startBytePos + size;
        
        // 填充当前位置到页边界
        FillCurrentPageWithZeros(align);
        startBytePos = pg_atomic_read_u64(&XLogCtl->Insert.CurrBytePos);
    }
    
    // 原子更新插入位置
    while (true)
    {
        XLogRecPtr  currPos = pg_atomic_read_u64(&XLogCtl->Insert.CurrBytePos);
        
        if (pg_atomic_compare_exchange_u64(&XLogCtl->Insert.CurrBytePos,
                                           &currPos,
                                           endBytePos))
            break;
        
        // CAS 失败，重试
    }
    
    return startBytePos;
}
```

#### 1.2.3 数据复制优化

```c
// src/backend/access/transam/xloginsert.c:400
static void
CopyDataToWALBuffer(XLogRecData *rdata, XLogRecPtr startPos)
{
    XLogRecPtr  currPos = startPos;
    uint32      currPage = currPos / XLOG_BLCKSZ;
    uint32      currOffset = currPos % XLOG_BLCKSZ;
    char       *pagePtr;
    
    // 遍历 rdata 链表
    for (; rdata != NULL; rdata = rdata->next)
    {
        const char *data = rdata->data;
        uint32      len = rdata->len;
        uint32      copied = 0;
        
        while (len > 0)
        {
            // 获取当前页面指针
            pagePtr = XLogCtl->pages + (currPage * XLOG_BLCKSZ);
            
            // 计算本页可复制的大小
            uint32 available = XLOG_BLCKSZ - currOffset;
            uint32 tocopy = Min(len, available);
            
            // 复制数据
            memcpy(pagePtr + currOffset, data + copied, tocopy);
            
            // 更新位置
            copied += tocopy;
            len -= tocopy;
            currPos += tocopy;
            currOffset += tocopy;
            
            // 跨页了？
            if (currOffset >= XLOG_BLCKSZ)
            {
                currPage++;
                currOffset = 0;
            }
        }
    }
}
```

### 1.3 常见 WAL 记录插入场景

#### 1.3.1 Heap Insert

```c
// src/backend/access/heap/heapam.c:2100
void
heap_insert(Relation relation, HeapTuple tup, CommandId cid,
            int options, BulkInsertState bistate)
{
    // ... 插入逻辑 ...
    
    // 生成 WAL 记录
    if (RelationNeedsWAL(relation))
    {
        xl_heap_insert xlrec;
        XLogRecPtr  recptr;
        
        // 1. 填充 xlrec
        xlrec.offnum = newoff;
        xlrec.flags = 0;
        
        // 2. 准备 rdata
        XLogBeginInsert();
        XLogRegisterData((char *) &xlrec, SizeOfHeapInsert);
        
        // 3. 注册块数据
        XLogRegisterBuffer(0, buffer, REGBUF_STANDARD);
        
        // 4. 插入记录
        recptr = XLogInsert(RM_HEAP_ID, XLOG_HEAP_INSERT);
        
        // 5. 更新页面 LSN
        PageSetLSN(PageGetItemId(page, newoff), recptr);
    }
}
```

#### 1.3.2 B-Tree Split

```c
// src/backend/access/nbtree/nbtpage.c:800
void
_bt_split(Relation rel, Buffer buf, Page rpage, OffsetNumber firstright)
{
    xl_btree_split xlrec;
    XLogRecPtr  recptr;
    
    // 填充分裂记录
    xlrec.leftblk = BufferGetBlockNumber(buf);
    xlrec.rightblk = BufferGetBlockNumber(rbuf);
    xlrec.level = level;
    xlrec.firstrightoff = firstright;
    
    // 注册 WAL 数据
    XLogBeginInsert();
    XLogRegisterData((char *) &xlrec, SizeOfBtreeSplit);
    
    // 注册左右页面
    XLogRegisterBuffer(0, buf, REGBUF_STANDARD);
    XLogRegisterBuffer(1, rbuf, REGBUF_WILL_INIT);
    
    // 如果是根分裂，注册元页
    if (isroot)
        XLogRegisterBuffer(2, metabuf, REGBUF_STANDARD);
    
    // 插入 WAL
    recptr = XLogInsert(RM_BTREE_ID, isroot ? XLOG_BTREE_NEWROOT : XLOG_BTREE_SPLIT_L);
    
    return recptr;
}
```

---

## 二、WAL 刷写入磁盘流程

### 2.1 XLogFlush 核心流程

**源码位置**: `src/backend/access/transam/xlog.c:2300 XLogFlush()`

**两种刷写模式**:
1. **同步刷写**: 事务提交时调用，保证持久性
2. **异步刷写**: WAL Writer 周期性刷写，优化性能

**同步刷写流程**:
```
Backend Process                     WAL Writer                Storage
─────────────────┐          ┌──────────────────┐          ┌──────────────┐
XLogFlush()       │          │                  │          │              │
                  │          │                  │          │              │
┌─────────────────┤          │                  │          │              │
│ 1. 检查是否需要  │          │                  │          │              │
│    刷写         │          │                  │          │              │
│    * already   │          │                  │          │              │
│      flushed?  │          │                  │          │              │
└─────────────────┘          │                  │          │              │
         是/否                │                  │          │              │
         │                   │                  │          │              │
         └─是─────────────────┤                  │          │              │
                             │                  │          │              │
         └─否─────────────────┼────────────────►│          │              │
                             │                  │          │              │
┌─────────────────┤          │                  │          │              │
│ 2. 请求刷写     ├──────────►│ RequestXLogFlush │          │              │
│    - 设置目标    │          │   - LogwrtRqst   │          │              │
│      LSN        │          │   - 唤醒 Writer   │          │              │
└─────────────────┘          │                  │          │              │
                             │                  │          │              │
┌─────────────────┤          │   ┌────────────┐  │          │              │
│ 3. 等待刷写完成 ├──────────►│   │          │  ▼          │              │
│    * 等待条件变量│          │   │    WAL    │  │          │              │
│      回答       │          │   │  Writer   │  │          │              │
│    * 或超时     │          │   │   Run()   │  │          │              │
└─────────────────┘          │   │            │──┼────────►│              │
                             │   │  - 读取    │  │          │              │
                             │   │    缓冲区  │  │          │              │
                             │   │  -打开文件  │  │          │              │
                             │   │  -pwrite() │  │          │              │
                             │   │  -fsync()  │  │          │              │
                             │   │            │  │          │              │
                             │   └────┬──────┘  │          │              │
                             │        │         │          │              │
                             │        ▼         │          │              │
                             │   刷写完成      │          │              │
                             │        │         │          │              │
                             │        └─────────┤          │              │
                             │                  │          │              │
┌─────────────────┤          │                  │          │              │
│ 4. 返回         │          │                  │          │              │
└─────────────────┘          └──────────────────┘          └──────────────┘
```

### 2.2 WAL Writer 异步流程

**源码位置**: `src/backend/postmaster/walwriter.c:150 WalWriterMain()`

```c
void
WalWriterMain(void)
{
    XLogRecPtr  WriteRqstPtr;
    XLogRecPtr  FlushRqstPtr;
    pg_time_t   now;
    long        udelay;
    
    for (;;)
    {
        // 1. 计算下次唤醒时间
        now = time(NULL);
        udelay = 1000000 * WalWriterDelay;
        
        // 2. 检查是否有未刷写的 WAL
        WriteRqstPtr = XLogCtl->LogwrtRqst.Write;
        if (WriteRqstPtr <= XLogCtl->LogwrtResult.Write)
        {
            // 没有新数据，休眠
            pg_usleep(udelay);
            continue;
        }
        
        // 3. 写入 WAL 到磁盘
        if (WriteRqstPtr > XLogCtl->LogwrtResult.Write)
            XLogWrite(0, false);
        
        // 4. 批量刷写条件检查
        FlushRqstPtr = XLogCtl->LogwrtRqst.Flush;
        if (FlushRqstPtr > XLogCtl->LogwrtResult.Flush)
        {
            // 累积了足够的数据或超时了
            if ((now - XLogCtl->LastFlushTime) * 1000 >= WalWriterFlushAfter ||
                WalSegmentSize - (XLogCtl->LogwrtResult.Write % WalSegmentSize) < 8192)
            {
                XLogWrite(0, true); // 强制刷写
            }
        }
        
        // 5. 处理信号
        ProcessConfigFile(PGC_SIGHUP);
    }
}
```

### 2.3 批量刷写优化

**组提交机制**:
```c
// src/backend/access/transam/xlog.c:2500
static void
XLogWrite(XLogRecPtr WriteRqstPtr, bool flexible)
{
    XLogRecPtr  LastRecPtr;
    XLogRecPtr  EndPtr;
    int         nbytes;
    XLogSegNo   segno;
    int         fd = -1;
    
    // 1. 获取写入范围
    LastRecPtr = XLogCtl->LSN;
    EndPtr = WriteRqstPtr;
    
    // 2. 循环写入每个段文件
    while (LastRecPtr < EndPtr)
    {
        // 计算本段写入范围
        XLogRecPtr  seg_start = LastRecPtr;
        XLogRecPtr  seg_end = Min(EndPtr,
                                  ((LastRecPtr / XLOG_SEG_SIZE) + 1) * XLOG_SEG_SIZE);
        
        // 3. 打开段文件 (如有需要)
        segno = LastRecPtr / XLOG_SEG_SIZE;
        if (fd < 0 || XLogGetCurrentSegNo() != segno)
        {
            if (fd >= 0)
                close(fd);
            fd = XLogFileOpen(segno, false);
        }
        
        // 4. 写入数据
        nbytes = seg_end - seg_start;
        write_offset = LastRecPtr % XLOG_SEG_SIZE;
        
        // 从缓冲区复制到文件
        char *srcptr = XLogCtl->pages + (seg_start % XLogSegSize);
        
        while (nbytes > 0)
        {
            int this_write = Min(nbytes, WRITE_CHUNK);
            
            if (write(fd, srcptr, this_write) != this_write)
                ereport(ERROR, 
                        (errcode_for_file_access(),
                         errmsg("could not write to log file %s: %m",
                                XLogFileName)));
            
            srcptr += this_write;
            nbytes -= this_write;
        }
        
        // 5. 更新写入位置
        LastRecPtr = seg_end;
        XLogCtl->LogwrtResult.Write = LastRecPtr;
        pg_memory_barrier();
    }
    
    // 6. 强制刷写到磁盘
    if (flexible || WriteRqstPtr > XLogCtl->LogwrtResult.Flush)
    {
        if (fsync(fd) != 0)
            ereport(ERROR, 
                    (errcode_for_file_access(),
                     errmsg("could not fsync log file %s: %m",
                            XLogFileName)));
        
        XLogCtl->LogwrtResult.Flush = WriteRqstPtr;
        pg_memory_barrier();
    }
}
```

### 2.4 性能优化技术

#### 2.4.1 延迟组提交

```postgresql
-- 配置组提交
-- commit_delay: 等待同伙的时间 (微秒)
-- commit_siblings: 触发组提交的并发事务数

-- 示例配置
commit_delay = 1000      -- 1ms 等待
commit_siblings = 5      -- 5个并发事务即触发组提交
```

**算法实现**:
```c
// src/backend/access/transam/xlog.c:2700
static void
commit_delay(void)
{
    static int  commitDelay = 0;
    struct timeval tv;
    
    // 检查同伙数
    if (ProcGlobal->xactCommittors < commit_siblings)
        return;
    
    // 延迟等待
    tv.tv_sec = 0;
    tv.tv_usec = commit_delay + 
                (random() % commitDelayRange);
    
    select(0, NULL, NULL, NULL, &tv);
}
```

---

## 三、WAL 恢复和重放流程

### 3.1 Startup Process 恢复流程

**源码位置**: `src/backend/access/transam/xlogrecovery.c:1000 StartupXLOG()`

**完整恢复流程图**:
```
Startup Process                 Storage Manager               Checkpointer
─────────────────┐          ┌─────────────────┐          ┌──────────────────┐
StartupXLOG()     │          │                 │          │                  │
                  │          │                 │          │                  │
┌─────────────────┤          │                 │          │                  │
│ 1. 读取控制文件  ├──────────►│ ReadControlFile│          │                  │
│    - 检查状态    │          │                 │          │                  │
│    - 获取REDO点  │          │                 │          │                  │
└─────────────────┘          └─────────────────┘          │                  │
         │                                                  │
         ▼                                                  │
┌─────────────────┐                                        │
│ 2. 确定恢复起点  │                                        │
│    * 正常关闭    │                                        │
│      - 无需恢复  │                                        │
│    * 异常关闭    │                                        │
│      - 从REDO   │                                        │
│        开始恢复  │                                        │
│    * 备份恢复    │                                        │
│      - 从backup  │                                        │
│        开始恢复  │                                        │
└─────────────────┘                                        │
         │                                                  │
         ▼                                                  │
┌─────────────────┐          ┌─────────────────┐          │
│ 3. 进入Redo循环 ├──────────►│   WAL Replay   │          │
│    - 读取段文件  │          │   - 解析记录    │          │
│    - 解析记录    │          │   - 调用redo    │          │
│    - 调用redo    │          │     函数        │          │
│      函数       │          │   - 更新缓冲区  │          │
└─────────────────┘          └─────────────────┘          │
         │                                                  │
         ▼                                                  │
┌─────────────────┐                                        │
│ 4. 处理Timelines│                                        │
│    - 读取.history│                                        │
│    - 确定TLI    │                                        │
└─────────────────┘                                        │
         │                                                  │
         ▼                                                  │
┌─────────────────┐                                        │
│ 5. 清理资源     ├────────────────────────────────────────►│
│    - 清理临时文件│          │          │                  │
│    - 删除restart │          │          │                  │
│      标签       │          │          │                  │
└─────────────────┘          │          │                  │
         │                     │          │                  │
         ▼                     │          │                  │
┌─────────────────┐          │          │                  │
│ 6. 创建End-LV  ├──────────►│  CreateCheckPoint │          │
│    Checkpoint   │          │  (END_OF_RECOVERY)│          │
└─────────────────┘          └─────────────────┘          │
         │                                                  │
         ▼                                                  │
┌─────────────────┐          ┌─────────────────┐          │
│ 7. 进入运行状态 ├──────────►│ TransitionToNormal│         │
│    - 启动全局   │          │    State        │         │
│      统计       │          │                 │         │
│    - 初始化子   │          │                 │         │
│      系统       │          │                 │         │
└─────────────────┘          └─────────────────┘         └──────────────────┘
```

### 3.2 详细恢复实现

#### 3.2.1 恢复起点确定

```c
// src/backend/access/transam/xlogrecovery.c:1100
static void
FindRecoveryStartPoint(XLogRecPtr *RecPtr, TimeLineID *TLI)
{
    XLogRecPtr  checkpointRecPtr;
    XLogRecPtr  backupStartRecPtr;
    
    // 1. 检查是否有 backup_label
    if (read_backup_label(&backupStartRecPtr, TLI))
    {
        // 从备份标签确定恢复起点
        *RecPtr = backupStartRecPtr;
        elog(LOG, "starting point from backup_label: %X/%X",
             LSN_FORMAT_ARGS(*RecPtr));
        return;
    }
    
    // 2. 从控制文件获取最近检查点
    checkpointRecPtr = ControlFile->checkPoint;
    
    // 3. 验证检查点
    if (XRecOffIsValid(checkpointRecPtr))
    {
        ReadCheckpointRecord(checkpointRecPtr, &checkPoint, TLI);
        *RecPtr = checkpoint.redo;
        elog(LOG, "starting point from checkpoint: %X/%X",
             LSN_FORMAT_ARGS(*RecPtr));
    }
    else
    {
        // 损坏的检查点，退回到下一个段文件
        *RecPtr = XLogSegSize * (checkpointRecPtr / XLogSegSize + 1);
        *TLI = ControlFile->ThisTimeLineID;
        
        ereport(LOG,
                (errmsg("invalid checkpoint in control file")));
    }
}
```

#### 3.2.2 WAL 重放主循环

```c
// src/backend/access/transam/xlogrecovery.c:1500
static void
RecoveryLoop(void)
{
    XLogRecord *record;
    XLogReaderState *xlogreader;
    ErrorContextCallback errcontext;
    
    xlogreader = XLogReaderAllocate(wal_segment_size, XL_ROUTINE());
    
    // 设置错误上下文
    errcontext.callback = recovery_error_callback;
    error_context_stack = &errcontext;
    
    for (;;)
    {
        char *errormsg;
        
        // 1. 读取日志记录
        record = XLogReadRecord(xlogreader, &errormsg);
        
        // 2. 检查恢复结束条件
        if (record == NULL)
        {
            if (RecoveryIsPaused())
            {
                // 恢复暂停，等待
                RecoveryPauseInfo();
                continue;
            }
            
            // 恢复完成
            break;
        }
        
        // 3. 调用相应的 redo 函数
        RmgrTable[record->xl_rmid].rm_redo(xlogreader);
        
        // 4. 更新最小恢复点
        if (XLogRecPtrIsInvalid(minRecoveryPoint) ||
            XLogRecPtrIsInvalid(ControlFile->minRecoveryPoint))
        {
            UpdateMinRecoveryPoint(XLogReader_getCurrentRecPtr(xlogreader), false);
        }
        
        // 5. 处理资源管理器
        RmgrTable[record->xl_rmid].rm_end(xlogreader);
    }
    
    // 清理
    XLogReaderFree(xlogreader);
}
```

#### 3.2.3 资源管理器 Redo 函数

**Heap Redo 示例**:
```c
// src/backend/access heap/heapam.c:3000
void
heap_redo(XLogReaderState *record)
{
    XLogRecPtr  lsn = record->EndRecPtr;
    
    switch (XLogRecGetInfo(record) & XLOG_HEAP_OPMASK)
    {
        case XLOG_HEAP_INSERT:
            {
                xl_heap_insert *xlrec = (xl_heap_insert *) XLogRecGetData(record);
                Buffer      buffer;
                Page        page;
                OffsetNumber offnum;
                
                // 1. 读取块
                buffer = XLogReadBuffer(xlrec->target.node, 
                                       xlrec->target.block,
                                       true);
                page = BufferGetPage(buffer);
                
                // 2. 插入元组
                offnum = xlrec->offnum;
                if (PageAddItem(page, (Item) XLogRecGetBlockData(record, 0),
                               XLogRecGetBlockLen(record, 0),
                               offnum, false, false) == InvalidOffsetNumber)
                    elog(PANIC, "failed to add tuple");
                
                // 3. 更新页面
                PageSetLSN(page, lsn);
                MarkBufferDirty(buffer);
                UnlockReleaseBuffer(buffer);
                break;
            }
            
        case XLOG_HEAP_DELETE:
            // 处理删除
            break;
            
        case XLOG_HEAP_UPDATE:
            // 处理更新
            break;
            
        default:
            elog(PANIC, "heap_redo: unknown op code %u",
                 XLogRecGetInfo(record));
    }
}
```

### 3.3 Timeline 管理

**History 文件示例**:
```bash
# pg_wal/00000002.history
1        0/8000000
2        0/A000000

# pg_wal/00000003.history  
1        0/8000000
2        0/A000000
3        0/C000000
```

**Timeline 切换逻辑**:
```c
进入了3个函数：CreateTimeLineHistory
1. 标示切换：$pg_resetwal -D /opt/pgsql_data -c 10
2. 提升备库：pg_ctl promote
3. 创建新timeline时：不在当前3个场景内的情况？
```

---broken---
**Timeline 切换逻辑**:
```c
// src/backend/access/transam/timeline.c:400
static void
CreateTimeLineHistory(TimeLineID newTLI, TimeLineID parentTLI,
                      XLogRecPtr switchpoint, bool isTempTLI)
{
    char        path[MAXPGPATH];
    char        tmppath[MAXPGPATH];
    FILE       *fd;
    int         fd0;
    
    // 1. 生成历史文件路径
    TLHistoryFileName(path, newTLI);
    
    // 2. 读取父历史
    List *history = readTimeLineHistory(parentTLI);
    
    // 3. 添加切换点
    snprintf(buf, sizeof(buf), "%u\t%X/%X\n",
             newTLI, LSN_FORMAT_ARGS(switchpoint));
    
    history = lappend(history, pstrdup(buf));
    
    // 4. 写入临时文件
    snprintf(tmppath, sizeof(tmppath), "%s.tmp", path);
    fd = AllocateFile(tmppath, "w");
    
    // 写入历史条目
    ListCell *cell;
    foreach(cell, history)
    {
        fputs((char *) lfirst(cell), fd);
    }
    
    FreeFile(fd);
    
    // 5. 原子性重命名
    durable_rename(tmppath, path);
    
    // 6. 更新共享内存
    XLogCtl->ThisTimeLineID = newTLI;
    XLogCtl->PrevTimeLineID = parentTLI;
}
```

---

## 四、WAL 清理和回收流程

### 4.1 WAL 段文件回收机制

**回收条件检查**:
```c
// src/backend/access/transam/xlog.c:3600
static void
RemoveOldXlogFiles(XLogRecPtr EndRecPtr)
{
    XLogSegNo   segno;
    XLogRecPtr  keep;
    int         maxSegs;
    
    // 1. 计算需要保留的文件数
    maxSegs = CheckPointSegments;
    
    // 2. 确定删除点
    segno = XLogGetOldestSegno();
    keep = EndRecPtr - (maxSegs * XLOG_SEG_SIZE);
    
    // 3. 检查恢复需求
    if (InArchiveRecovery && !PromoteIsTriggered())
    {
        // 备库需要保留更多 WAL
        keep = Min(keep, RecoveryTargetLSN);
    }
    
    // 4. 检查复制槽需求
    keep = CheckReplicationSlotRetention(keep);
    
    // 5. 删除过期文件
    while (segno < keep / XLOG_SEG_SIZE)
    {
        RemoveXlogFile(segno);
        segno++;
    }
}
```

### 4.2 归档处理流程

**Archive Manager 流程**:
```
Archiver Process                   WAL Writer/Promote
─────────────────┐          ┌─────────────────────────┐
pgarch.c        │          │                         │
                  │          │                         │
┌─────────────────┤          │                         │
│ 1. 检查待归档   ├──────────►│ 创建 .ready 文件        │
│    文件 (.ready)│          │                         │
└─────────────────┘          └─────────────────────────┘
         │                                                   
         ▼                                                   
┌─────────────────┐                                        
│ 2. 获取文件名    │                                        
│    - 从.ready   │                                        
│      读取LSN     │                                        
└─────────────────┘                                        
         │                                                   
         ▼                                                   
┌─────────────────┐                                        
│ 3. 执行归档命令   ├────────────────────────────────────────►│
│    archive_command                      │
│    - %p: 文件路径                          │
│    - %f: 文件名                           │
└─────────────────┘          │                           │
         │                    │                           │
         ▼                    │                           │
┌─────────────────┤          │                           │
│ 4. 检查返回值    ├───成功────►│                          │
│    - 0: 成功     │          │                          │
│    - 非0: 失败   │          │                          │
└─────────────────┤          │                          │
         │失败              │                          │
         ▼                  │                          │
┌─────────────────┤          │                          │
│ 5. 重命名文件     ├───┐      │                          │
│    .ready→.done   │    │      │                          │
│    .ready→.failed │    │      │                          │
└─────────────────┘    │      │                          │
         │             │      │                          │
         └─────────────┘      │                          │
                              │                          │
         ┌─────────────────────┘                          │
         ▼                                                │
┌─────────────────┐                                        │
│ 6. 等待间隔     │                                        │
│    archive_timeout                                        │
└─────────────────┘                                        │
```

**归档命令示例**:
```bash
# 简单复制到备份目录
archive_command = 'cp %p /backup/wal/%f'

# 压缩归档
archive_command = 'gzip < %p > /backup/wal/%f.gz && rm %p'

# 远程归档到 S3
archive_command = 'aws s3 cp %p s3://my-backup/wal/%f'

# 带验证的归档
archive_command = '
  cp %p /backup/wal/%f &&
  md5sum /backup/wal/%f > /backup/wal/%f.md5
'
```

### 4.3 自动清理优化

**策略配置**:
```postgresql
# WAL 保留策略
wal_keep_size = 0              # 0=不限制，单位 MB
max_slot_wal_keep_size = 0     # 复制槽最大保留

# 周期清理
checkpoint_completion_target = 0.9  # 减少检查点干扰
```

---

## 五、WAL 复制和传输流程

### 5.1 流复制架构

**协议流程**:
```
Primary Server                                  Standby Server
─────────────────┐                        ┌──────────────────┐
WAL Sender        ──── TCP(5432) ──────────┤ WAL Receiver     │
                  <---- 检查响应 ---->>  │                  │
                  <---- 确认延迟 ---->>  │                  │
                                                      Store WAL in cache and apply to database filesystem
                                                  <- Later -> Recovery Process
```

### 5.2 物理复制实现

**位置计算与进度**:
```c
// src/backend/replication/walreceiver.c
static void WalReceiverMain(void)
{
    // Get last received WAL location
    XLogRecPtr startpoint = Record.location;

    // Initialize replication origin
    replication_origin_create("pg_walreceiver");
    
    // Accept replication start
    if (ReplicationSlotCreateSlot())
    { replication_start(); }

    // Manage apply in multiple attempts update applied location
    while (WalReceiverStop())
    { 
        // Get response
        WalRcvMessage response;    
        
        // Parse status
        if (MessageParse(response))
        { WalDataWrite(*Data);
        progress_update(); } }
}
```

---

## 六、关键并发控制机制

### 6.1 锁机制

**插入锁控制**:
```c
// 插入锁保证了 WAL 记录的顺序性
LWLock *WALInsertLock;

// 获取锁
LWLockAcquire(WALInsertLock, LW_EXCLUSIVE);

// 释放锁
LWLockRelease(WALInsertLock);

// 避免死锁的获取顺序
1. 先获取 WALInsertLock
2. 再获取 BufferLock
3. 最后获取 CheckpointLock
```

### 6.2 原子操作

**位置更新**:
```c
// 使用 CAS 避免锁竞争
while (!pg_atomic_compare_exchange_u64(&XLogCtl->Insert.CurrBytePos,
                                       &expected,
                                       new_value))
{
    // 失败，重试
    expected = pg_atomic_read_u64(&XLogCtl->Insert.CurrBytePos);
    new_value = expected + size;
}
```

---

## 七、错误处理和异常流程

### 7.1 常见故障类型

| 故障类型 | 检测方法 | 恢复策略 |
|----------|----------|----------|
| **WAL 文件损坏** | `pg_waldump` 校验失败 | 从备份点恢复 |
| **写入失败** | `fsync` 错误 | 检查磁盘空间 |
| **复制中断** | 连接超时 | 重启 WAL Sender/Receiver |

### 7.2 故障恢复

**自动恢复逻辑**:
```c
// 恢复模式决策：
// 1. 判断是否需要归档根位置
bool needs_archive = RecoveryRequiresArchive();

// 2. 获取可用 WAL 段
List *available_wals = FindAvailableWALSegments(RequiredLogSegments);

// 3. 根据数据段计算可恢复范围
CalculateRecoverableRange(available_wals);

// 4. 如果可恢复，创建归档所需参数
if (is_recoverable)
{
      // 启用归档恢复模式
      enable_archive_recovery();
}
else
{
      // 请求管理员干预
      request_admin_help();
}
```

**操作示例**:
```bash
# 损坏 WAL 时
pg_resetwal -D /data -f

# 验证文件
pg_verifybackup /backup

# 检查和备份
pg_dump -d db > backup.sql
pg_controldata $PGDATA | grep -i wal
```

---

## 总结

WAL 实现机制体现的关键设计：

1. **性能优化**：批量刷写、组提交
2. **并发控制**：最少锁使用、原子操作
3. **可靠性**：双写、校验、恢复
4. **扩展性**：支持多种复制

**下一步**：关键算法和复杂度 → [04_key_algorithms.md](04_key_algorithms.md)

---

**相关函数**:
- `XLogInsertRecord()`: WAL 记录插入
- `XLogFlush()`: WAL 同步刷写
- `StartupXLOG()`: 系统启动和恢复
- `WalWriterMain()`: 异步写循环
- `WalSndLoop()`: 复制发送
- `pg_waldump`: WAL 文件解析工具
