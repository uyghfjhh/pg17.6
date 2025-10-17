# HEAP 架构图表

本文档提供堆表的核心架构图和流程图。

---

## 1. 整体架构图

```
┌────────────────────────────────────────────────────────────┐
│                PostgreSQL Heap Storage                      │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              Heap Access Method (heapam)              │ │
│  │  heap_insert() | heap_update() | heap_delete()       │ │
│  └──────────────┬───────────────────────────────────────┘ │
│                 │                                           │
│  ┌──────────────▼───────────────────────────────────────┐ │
│  │          Page Management (bufpage)                    │ │
│  │  PageAddItem() | PageRepairFragmentation()           │ │
│  └──────────────┬───────────────────────────────────────┘ │
│                 │                                           │
│  ┌──────────────▼───────────────────────────────────────┐ │
│  │         Buffer Manager (8KB pages)                    │ │
│  │  ReadBuffer() | MarkBufferDirty()                    │ │
│  └──────────────┬───────────────────────────────────────┘ │
│                 │                                           │
│  ┌──────────────▼───────────────────────────────────────┐ │
│  │          Storage Manager (smgr)                       │ │
│  │  md.c (磁盘文件I/O)                                   │ │
│  └──────────────┬───────────────────────────────────────┘ │
│                 │                                           │
│  ┌──────────────▼───────────────────────────────────────┐ │
│  │          Disk Files                                   │ │
│  │  base/<dboid>/<reloid>  - 主表                       │ │
│  │  base/<dboid>/<reloid>_fsm  - 空闲空间映射           │ │
│  │  base/<dboid>/<reloid>_vm   - 可见性映射             │ │
│  │  pg_toast/<toastoid>  - TOAST表                     │ │
│  └──────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

---

## 2. 页面结构图

```
8KB Page Structure:
┌─────────────────────────────────────────────────────────┐  0
│ PageHeaderData (24 Bytes)                               │
│ ┌───────────────────────────────────────────────────┐  │
│ │ pd_lsn:      8B  │ WAL LSN                        │  │
│ │ pd_checksum: 2B  │ 校验和                         │  │
│ │ pd_flags:    2B  │ 标志                           │  │
│ │ pd_lower:    2B  │ 空闲空间起始 ─────┐           │  │
│ │ pd_upper:    2B  │ 空闲空间结束 ─────┼─┐        │  │
│ │ pd_special:  2B  │ 特殊空间偏移       │ │        │  │
│ │ pd_pagesize: 2B  │ 8192               │ │        │  │
│ │ pd_prune_xid:4B  │ 剪枝XID            │ │        │  │
│ └───────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤  24
│ Item Pointers (ItemIdData[])                            │
│ ┌───────────┬───────────┬───────────┬─────┬─────────┐ │
│ │ ItemId[0] │ ItemId[1] │ ItemId[2] │ ... │ItemId[N]│ │
│ │  4 Bytes  │  4 Bytes  │  4 Bytes  │     │ 4 Bytes │ │
│ └───────────┴───────────┴───────────┴─────┴─────────┘ │
│                            ↓ pd_lower                   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│                    Free Space                           │
│           (pd_upper - pd_lower bytes)                   │
│                                                          │
│                            ↑ pd_upper                   │
├─────────────────────────────────────────────────────────┤
│ Tuples (growing from bottom to top)                    │
│ ┌─────────────────────────────────────────────────────┐│
│ │ Tuple N:  HeapTupleHeader + NULL bitmap + Data     ││
│ │ Tuple N-1: ...                                      ││
│ │ ...                                                  ││
│ │ Tuple 1:  HeapTupleHeader + NULL bitmap + Data     ││
│ │ Tuple 0:  HeapTupleHeader + NULL bitmap + Data     ││
│ └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘  8192
```

---

## 3. 元组结构图

```
HeapTuple Layout:
┌───────────────────────────────────────────────────────────┐
│ HeapTupleHeader (23 Bytes fixed)                          │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ t_xmin (4B)      │ 插入事务ID                         │ │
│ │ t_xmax (4B)      │ 删除/锁定事务ID                    │ │
│ │ t_cid (4B)       │ 命令ID                             │ │
│ │ t_ctid (6B)      │ 当前或新版本TID (blockNum, offset)│ │
│ │ t_infomask2 (2B) │ 列数 + 标志                        │ │
│ │ t_infomask (2B)  │ HEAP_HASNULL, HEAP_HOT_UPDATED等  │ │
│ │ t_hoff (1B)      │ 头部总长度                         │ │
│ └───────────────────────────────────────────────────────┘ │
├───────────────────────────────────────────────────────────┤
│ NULL Bitmap (optional, CEIL(ncols/8) bytes)               │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Bit 0: col 0 is NULL? │ Bit 1: col 1? │ ... │Bit N  │ │
│ └───────────────────────────────────────────────────────┘ │
├───────────────────────────────────────────────────────────┤
│ User Data (columns)                                       │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Col 0: Fixed/Varlena data                             │ │
│ │ Col 1: Fixed/Varlena data                             │ │
│ │ ...                                                    │ │
│ │ Col N: Fixed/Varlena data                             │ │
│ └───────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
```

---

## 4. HOT Update 流程图

```
HOT Update Example:

表: users (id INT PRIMARY KEY, status TEXT, last_login TIMESTAMP)
更新: UPDATE users SET last_login = now() WHERE id = 1;

页面状态演化:

初始状态:
┌────────────────────────────────────────────────┐
│ ItemId[1] → Tuple v1                           │
│              {id:1, status:'active', ...}      │
│              t_ctid = (0, 1)  [指向自己]       │
└────────────────────────────────────────────────┘

UPDATE后 (HOT):
┌────────────────────────────────────────────────┐
│ ItemId[1] → Tuple v1 [OLD]                     │
│              t_infomask |= HEAP_HOT_UPDATED    │
│              t_ctid = (0, 5)  [指向新版本]     │
│                                                 │
│ ItemId[5] → Tuple v2 [NEW]                     │
│              t_infomask |= HEAP_ONLY_TUPLE     │
│              t_ctid = (0, 5)  [指向自己]       │
└────────────────────────────────────────────────┘

索引状态 (不变!):
  id=1 → TID (0, 1)

查询流程:
  IndexScan(id=1) → TID(0,1) → ItemId[1] → 检测HOT_UPDATED
    → 沿t_ctid链 → ItemId[5] → 返回 Tuple v2 ✓
```

---

## 5. FSM 三层结构图

```
FSM (Free Space Map) Structure:

表有 1,000,000 页:

Level 2 (Root): 1页
  ├─ 记录 Level 1 中各分支的最大空闲空间
  └─ 每个slot指向 Level 1 的一页

Level 1 (Branch): ~244页
  ├─ 记录 Level 0 中各叶子的最大空闲空间
  └─ 每个slot指向 Level 0 的一页

Level 0 (Leaf): ~4096页
  ├─ 记录实际堆表页的空闲空间
  └─ 每个slot = 1字节 (0-255, 精度~32字节)

查找流程 (需要2KB空间):
  1. Level 2: 找到最大值slot → 进入 Level 1 page X
  2. Level 1: 找到最大值slot → 进入 Level 0 page Y
  3. Level 0: 找到 slot >= 64 (2KB/32) → 返回 heap page Z
  
时间复杂度: O(3) = O(1) (常数层级)
```

---

## 6. TOAST 存储图

```
TOAST (The Oversized-Attribute Storage Technique):

主表 (users):
┌────────────────────────────────────────────────┐
│ id │ name  │ bio (TOAST Pointer, 18B)       │
│ 1  │ Alice │ [va_extsize, va_valueid, ...]  │
└────────────────────────────────────────────────┘
        │
        │ 指向
        ▼
TOAST表 (pg_toast.pg_toast_12345):
┌──────────────────────────────────────────────────┐
│ chunk_id │ chunk_seq │ chunk_data (2KB)          │
├──────────────────────────────────────────────────┤
│   1001   │     0     │ "Alice is a software..." │
│   1001   │     1     │ "...engineer with..."    │
│   1001   │     2     │ "...extensive..."        │
│   1001   │     3     │ "...experience."         │
└──────────────────────────────────────────────────┘

查询时自动组合:
  SELECT bio FROM users WHERE id = 1;
  → 查找 TOAST chunk_id=1001
  → 按 chunk_seq 排序获取所有块
  → 组合成完整数据
  → 返回给用户
```

---

## 总结

核心架构要点:
- ✅ 8KB页面 + 间接寻址 (ItemId)
- ✅ MVCC内嵌 (t_xmin/t_xmax)
- ✅ HOT Update (同页优化)
- ✅ FSM三层B树 (O(1)空间查找)
- ✅ VM位图 (跳过已知可见页)
- ✅ TOAST自动外部存储

---

**完成**: HEAP模块所有文档已完成！

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

