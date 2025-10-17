# HEAP 实现流程

本文档说明堆表的INSERT、UPDATE、DELETE、SELECT等操作流程。

---

## 1. INSERT 流程

### 1.1 完整流程图

```
heap_insert(relation, tuple)
│
├─ [1] 查找有空间的页面
│   └─ RelationGetBufferForTuple()
│       ├─ 检查FSM (Free Space Map)
│       ├─ 找到有足够空间的页
│       └─ 如果没有 → 扩展表 (新增页面)
│
├─ [2] 锁定缓冲区
│   └─ LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE)
│
├─ [3] 准备元组
│   ├─ 设置 t_xmin = 当前事务ID
│   ├─ 设置 t_xmax = 0 (未删除)
│   ├─ 设置 t_ctid = (blockNum, offsetNum)
│   └─ 设置 t_infomask 标志
│
├─ [4] 在页面中插入元组
│   └─ PageAddItem()
│       ├─ 分配新的ItemId
│       ├─ 将元组数据写入页底部
│       ├─ 更新 pd_lower += 4 (ItemId)
│       └─ 更新 pd_upper -= tuple_size
│
├─ [5] 记录WAL日志
│   └─ XLogInsert(RM_HEAP_ID, XLOG_HEAP_INSERT)
│
├─ [6] 标记缓冲区为脏
│   └─ MarkBufferDirty(buffer)
│
├─ [7] 释放锁
│   └─ UnlockReleaseBuffer(buffer)
│
└─ [8] 返回新元组的TID
```

### 1.2 FSM查找流程

```c
// src/backend/access/heap/hio.c:RelationGetBufferForTuple

步骤:
1. 查询FSM: GetPageWithFreeSpace(relation, tuple_size)
   └─ 返回有足够空间的页号

2. 如果找到页面:
   └─ ReadBuffer(relation, page_num)
   
3. 如果FSM返回InvalidBlockNumber:
   ├─ 尝试最后一页
   └─ 如果仍不够 → 扩展表 (RelationAddExtraBlocks)
```

---

## 2. UPDATE 流程

### 2.1 标准UPDATE (非HOT)

```
heap_update(relation, oldTuple, newTuple)
│
├─ [1] 锁定旧元组
│   └─ heap_lock_tuple() → 设置 t_xmax = 当前XID
│
├─ [2] 检查更新冲突
│   └─ 如果旧元组已被其他事务修改 → 返回冲突
│
├─ [3] 准备新元组
│   ├─ t_xmin = 当前XID
│   ├─ t_xmax = 0
│   ├─ t_ctid = (newBlock, newOffset)
│   └─ t_infomask |= HEAP_UPDATED
│
├─ [4] 查找新元组的存储位置
│   └─ RelationGetBufferForTuple()
│       (可能在同一页或不同页)
│
├─ [5] 插入新元组
│   └─ PageAddItem(newPage, newTuple)
│
├─ [6] 更新旧元组的t_ctid
│   └─ oldTuple->t_ctid = newTID
│
├─ [7] 更新所有索引
│   └─ 对每个索引:
│       ├─ 删除旧TID
│       └─ 插入新TID (如果键值改变)
│
├─ [8] 记录WAL
│   └─ XLOG_HEAP_UPDATE
│
└─ [9] 标记缓冲区为脏并释放
```

### 2.2 HOT UPDATE 流程

```
heap_update() with HOT optimization
│
├─ [1] 检查HOT Update条件
│   ├─ 更新未改变索引列? (t_infomask2 & HEAP_KEYS_UPDATED == 0)
│   ├─ 同一页有足够空间?
│   └─ 页面未满?
│
├─ [2] 如果满足HOT条件:
│   ├─ 在同一页插入新元组
│   ├─ 旧元组: t_infomask |= HEAP_HOT_UPDATED
│   ├─ 新元组: t_infomask |= HEAP_ONLY_TUPLE
│   ├─ 旧元组 t_ctid → 新元组TID
│   └─ 跳过索引更新 ✓ (巨大性能提升!)
│
└─ [3] 如果不满足:
    └─ 降级到标准UPDATE (更新所有索引)
```

### 2.3 HOT链示例

```
表: users (id INT PRIMARY KEY, status TEXT, last_login TIMESTAMP)
索引: PRIMARY KEY on id

UPDATE users SET last_login = now() WHERE id = 1;
(不改变索引列id)

页面状态:
┌────────────────────────────────────────────────┐
│ ItemId[1] → Tuple v1 [t_infomask: HOT_UPDATED] │
│              t_ctid = (0, 5)                   │
│                                                 │
│ ItemId[5] → Tuple v2 [t_infomask: ONLY_TUPLE]  │
│              t_ctid = (0, 5) (指向自己)        │
└────────────────────────────────────────────────┘

索引不变:
  id=1 → TID (0, 1)  (仍指向ItemId[1])

查询流程:
  1. 索引查找: id=1 → TID (0, 1)
  2. 访问 ItemId[1] → 发现 HOT_UPDATED
  3. 沿着 t_ctid 链: (0,1) → (0,5)
  4. 返回 ItemId[5] 的数据 ✓
```

---

## 3. DELETE 流程

### 3.1 逻辑删除

```
heap_delete(relation, tid)
│
├─ [1] 锁定元组
│   └─ ReadBuffer() + LockBuffer()
│
├─ [2] 标记删除
│   ├─ t_xmax = 当前XID
│   └─ t_infomask 不设置 XMAX_INVALID
│
├─ [3] 记录WAL
│   └─ XLOG_HEAP_DELETE
│
├─ [4] 标记缓冲区为脏
│
└─ [5] 索引不变 (保留TID)
    └─ 索引清理由VACUUM处理

注意: PostgreSQL DELETE是逻辑删除，不立即回收空间
```

### 3.2 可见性判断

```
SELECT后如何判断元组已删除?

HeapTupleSatisfiesMVCC():
├─ 检查 t_xmax != 0
├─ 检查 t_xmax 是否已提交
│   └─ 如果 t_xmax已提交且 < 当前快照的xmin
│       → 元组对当前事务不可见 (已删除)
└─ 返回可见性结果
```

---

## 4. SELECT 流程

### 4.1 顺序扫描

```
heap_beginscan(relation)
│
├─ [1] 初始化扫描状态
│   └─ HeapScanDesc
│
├─ [2] 获取快照
│   └─ GetTransactionSnapshot()
│
└─ [3] 扫描循环
    while (more_pages) {
      ├─ heap_getnext()
      │   ├─ ReadBuffer(next_page)
      │   ├─ 遍历页面中的所有ItemId
      │   ├─ 对每个元组:
      │   │   ├─ 检查可见性: HeapTupleSatisfiesMVCC()
      │   │   └─ 如果可见 → 返回元组
      │   └─ 释放Buffer
      │
      └─ 移到下一页
    }
```

### 4.2 TID扫描 (通过索引)

```
IndexScan → heap_fetch(tid)
│
├─ [1] 根据TID定位
│   ├─ blockNum = BlockIdGetBlockNumber(tid)
│   ├─ offset = ItemPointerGetOffsetNumber(tid)
│   └─ ReadBuffer(relation, blockNum)
│
├─ [2] 获取ItemId
│   └─ itemId = PageGetItemId(page, offset)
│
├─ [3] 处理ItemId状态
│   ├─ LP_NORMAL: 直接获取元组
│   ├─ LP_REDIRECT: 跟随重定向 (HOT链)
│   │   └─ newOffset = ItemIdGetRedirect(itemId)
│   │       → 递归查找最新版本
│   ├─ LP_DEAD: 元组已死 → 返回NULL
│   └─ LP_UNUSED: 无效 → 返回NULL
│
└─ [4] 检查可见性
    └─ HeapTupleSatisfiesMVCC(tuple, snapshot)
```

---

## 5. 页面剪枝流程

### 5.1 触发时机

```
heap_page_prune_opt()
│
├─ 触发条件:
│   1. UPDATE/SELECT 访问页面
│   2. pd_prune_xid < 当前XID (有可能的死元组)
│   3. 启用剪枝 (vacuum_defer_cleanup_age)
│
└─ 执行剪枝:
    └─ heap_page_prune()
```

### 5.2 剪枝算法

```c
heap_page_prune(relation, buffer, OldestXmin)
│
├─ [1] 扫描所有ItemId
│   for (i = FirstOffsetNumber; i <= maxoff; i++)
│
├─ [2] 对每个元组:
│   ├─ 检查是否可剪枝
│   │   └─ t_xmax < OldestXmin && t_xmax已提交
│   │
│   ├─ 如果是HOT链的头:
│   │   └─ 剪枝整个HOT链
│   │       ├─ 保留链头的ItemId (设为LP_REDIRECT)
│   │       ├─ 删除中间版本 (设为LP_DEAD)
│   │       └─ 保留最新版本
│   │
│   └─ 如果是独立死元组:
│       └─ 设为LP_DEAD
│
├─ [3] 压实空间 (可选)
│   └─ PageRepairFragmentation()
│       ├─ 移动元组消除碎片
│       └─ 更新 pd_lower/pd_upper
│
└─ [4] 记录WAL
    └─ XLOG_HEAP2_PRUNE
```

### 5.3 HOT链剪枝示例

```
剪枝前:
  ItemId[1] → Tuple v1 [dead, t_ctid=(0,5)]
  ItemId[5] → Tuple v2 [dead, t_ctid=(0,9)]
  ItemId[9] → Tuple v3 [current]

剪枝后:
  ItemId[1] → LP_REDIRECT to 9
  ItemId[5] → LP_DEAD
  ItemId[9] → Tuple v3 [current]

效果:
  - 回收v1, v2占用的空间
  - 索引仍然有效 (TID (0,1) → 重定向到 (0,9))
  - 减少HOT链长度 (加速查找)
```

---

## 6. TOAST 流程

### 6.1 TOAST 触发

```
heap_insert/update()
│
└─ toast_insert_or_update()
    │
    ├─ [1] 检查元组大小
    │   └─ if (tuple_size > TOAST_TUPLE_THRESHOLD ~2KB)
    │
    ├─ [2] 尝试压缩
    │   └─ 对EXTENDED策略的列进行LZ压缩
    │
    ├─ [3] 如果仍太大
    │   └─ 移动到TOAST表
    │       ├─ 将大字段切分为2KB块
    │       ├─ 插入到pg_toast.pg_toast_<oid>
    │       │   INSERT (chunk_id, chunk_seq, chunk_data)
    │       └─ 主表存储TOAST指针 (18B)
    │
    └─ [4] 返回修改后的元组
```

### 6.2 TOAST 读取

```
heap_getnext() → 解TOAST
│
└─ toast_flatten_tuple()
    ├─ 检测TOAST字段: t_infomask & HEAP_HASEXTERNAL
    ├─ 对每个TOAST字段:
    │   ├─ 读取TOAST指针
    │   ├─ 从TOAST表获取所有chunk
    │   │   SELECT chunk_data FROM pg_toast_xxx
    │   │   WHERE chunk_id = X ORDER BY chunk_seq
    │   ├─ 组合所有chunk
    │   └─ 解压缩 (如果需要)
    └─ 返回完整元组
```

---

## 7. 表扩展流程

```
RelationAddExtraBlocks()
│
├─ [1] 计算扩展大小
│   └─ extend_by_pages = Min(512, 已有页数/512)
│       (默认最多512页 = 4MB)
│
├─ [2] 扩展文件
│   └─ mdextend()
│       └─ lseek() + write() 填充0
│
├─ [3] 初始化新页面
│   └─ PageInit() 对每个新页
│       ├─ 设置PageHeader
│       ├─ pd_lower = sizeof(PageHeader)
│       └─ pd_upper = BLCKSZ
│
└─ [4] 更新FSM
    └─ RecordPageWithFreeSpace()
```

---

## 总结

### 关键流程对比

| 操作 | 是否需要WAL | 索引更新 | 空间回收 |
|------|-------------|----------|----------|
| INSERT | ✅ | ✅ (新TID) | N/A |
| UPDATE (标准) | ✅ | ✅ (新TID) | 否 (留给VACUUM) |
| UPDATE (HOT) | ✅ | ✗ (无需!) | 页面剪枝 |
| DELETE | ✅ | ✗ (延迟) | 否 (留给VACUUM) |
| SELECT | ✗ | N/A | N/A |
| 页面剪枝 | ✅ | ✗ | 部分 (同页) |

### 性能优化点

- ✅ **HOT Update**: 避免索引更新 (80%+ UPDATE可优化)
- ✅ **页面剪枝**: 轻量级空间回收
- ✅ **FSM**: 快速找到有空间的页
- ✅ **TOAST**: 大字段自动外部存储

---

**下一步**: 阅读 [04_key_algorithms.md](04_key_algorithms.md) 了解关键算法

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

