# HEAP 核心数据结构

本文档详细说明堆表的核心数据结构。

---

## 1. PageHeaderData (页头)

### 1.1 结构定义

```c
// src/include/storage/bufpage.h:178
typedef struct PageHeaderData
{
    PageXLogRecPtr pd_lsn;        // LSN: 最后修改此页的WAL记录位置 (8B)
    uint16      pd_checksum;      // 页面校验和 (2B)
    uint16      pd_flags;         // 标志位 (2B)
    LocationIndex pd_lower;       // 空闲空间起始偏移 (2B)
    LocationIndex pd_upper;       // 空闲空间结束偏移 (2B)
    LocationIndex pd_special;     // 特殊空间起始偏移 (2B)
    uint16      pd_pagesize_version; // 页大小和版本 (2B)
    TransactionId pd_prune_xid;   // 最老的可能需要剪枝的XID (4B)
    ItemIdData  pd_linp[FLEXIBLE_ARRAY_MEMBER]; // 行指针数组
} PageHeaderData;

// 总计: 24字节固定部分
```

### 1.2 关键字段

**pd_lsn**: WAL日志序列号
```
作用: 崩溃恢复时判断页面是否需要重放WAL
逻辑: if (wal_record_lsn > page_lsn) → 需要重放
```

**pd_lower / pd_upper**: 空闲空间边界
```
┌─────────────────────────────────────┐  0
│ PageHeader (24B)                    │
├─────────────────────────────────────┤  24
│ ItemId[0..N]                        │
│             ↓ pd_lower              │  ← ItemId数组末尾
├─────────────────────────────────────┤
│                                     │
│       Free Space                    │  ← pd_upper - pd_lower
│                                     │
│             ↑ pd_upper              │  ← 元组数据起始
├─────────────────────────────────────┤
│ Tuples (bottom-up)                  │
└─────────────────────────────────────┘  8192

空闲空间 = pd_upper - pd_lower
页满条件: pd_lower >= pd_upper (无空闲空间)
```

**pd_flags**: 页面标志
```c
#define PD_HAS_FREE_LINES  0x0001  // 有空闲的ItemId可重用
#define PD_PAGE_FULL       0x0002  // 页面满（FSM应标记为0空闲）
#define PD_ALL_VISIBLE     0x0004  // 所有元组对所有事务可见
```

**pd_prune_xid**: 剪枝优化
```
记录页面中最老的可能需要剪枝的事务ID
快速判断: if (current_xid < pd_prune_xid) → 跳过剪枝
```

---

## 2. ItemIdData (行指针)

### 2.1 结构定义

```c
// src/include/storage/itemid.h:26
typedef struct ItemIdData
{
    unsigned    lp_off:15,      // 元组在页面中的偏移 (15位)
                lp_flags:2,     // 状态标志 (2位)
                lp_len:15;      // 元组长度 (15位)
} ItemIdData;

// 总计: 4字节 (32位紧凑存储)
```

### 2.2 lp_flags 状态

```c
#define LP_UNUSED   0  // 未使用
#define LP_NORMAL   1  // 正常使用中
#define LP_REDIRECT 2  // 重定向到另一个ItemId (HOT链)
#define LP_DEAD     3  // 死元组，可重用
```

### 2.3 示例

```
页面中的3个元组:

ItemId[0]: {lp_off: 8000, lp_flags: LP_NORMAL, lp_len: 64}
  → 指向偏移8000处的64字节元组

ItemId[1]: {lp_off: 5, lp_flags: LP_REDIRECT, lp_len: 0}
  → 重定向到ItemId[5] (HOT链)

ItemId[2]: {lp_off: 0, lp_flags: LP_DEAD, lp_len: 0}
  → 死元组，ItemId可重用
```

---

## 3. HeapTupleHeaderData (元组头)

### 3.1 完整结构

```c
// src/include/access/htup_details.h:116
typedef struct HeapTupleHeaderData
{
    union {
        HeapTupleFields t_heap;
        DatumTupleFields t_datum;
    } t_choice;

    ItemPointerData t_ctid;     // 当前或新版本的TID (6B)

    uint16  t_infomask2;        // 列数和标志位 (2B)
    uint16  t_infomask;         // 各种标志位 (2B)
    uint8   t_hoff;             // 头部长度 (1B)

    bits8   t_bits[FLEXIBLE_ARRAY_MEMBER];  // NULL位图

    /* 之后是用户数据 */
} HeapTupleHeaderData;
```

### 3.2 HeapTupleFields (常规元组)

```c
typedef struct HeapTupleFields
{
    TransactionId t_xmin;   // 插入此元组的事务ID (4B)
    TransactionId t_xmax;   // 删除/锁定此元组的事务ID (4B)

    union {
        CommandId t_cid;    // 插入和删除的命令ID (4B)
        TransactionId t_xvac; // 用于VACUUM的事务ID
    } t_field3;
} HeapTupleFields;

// 总计: 12字节
```

### 3.3 t_infomask 标志位

```c
// 事务状态标志
#define HEAP_XMIN_COMMITTED     0x0100  // t_xmin已提交
#define HEAP_XMIN_INVALID       0x0200  // t_xmin无效(已中止)
#define HEAP_XMAX_COMMITTED     0x0400  // t_xmax已提交
#define HEAP_XMAX_INVALID       0x0800  // t_xmax无效

// 锁标志
#define HEAP_XMAX_IS_LOCKED_ONLY 0x0080  // t_xmax只是锁，非删除

// HOT标志
#define HEAP_HOT_UPDATED        0x4000  // 此元组被HOT更新
#define HEAP_ONLY_TUPLE         0x8000  // 此元组是HOT链中间节点

// MVCC标志
#define HEAP_HASVARWIDTH        0x0002  // 有变长字段
#define HEAP_HASEXTERNAL        0x0004  // 有TOAST字段
#define HEAP_HASNULL            0x0001  // 有NULL值
```

### 3.4 t_infomask2 标志位

```c
#define HEAP_NATTS_MASK         0x07FF  // 列数 (0-2047)
#define HEAP_KEYS_UPDATED       0x2000  // 更新了键列(索引列)
#define HEAP_HOT_UPDATED        0x4000  // HOT更新标志(复制)
#define HEAP_ONLY_TUPLE         0x8000  // HOT链标志(复制)
```

---

## 4. ItemPointerData (TID)

### 4.1 结构

```c
// src/include/storage/itemptr.h:28
typedef struct ItemPointerData
{
    BlockIdData ip_blkid;      // 块号 (4B)
    OffsetNumber ip_posid;     // 块内偏移号 (2B)
} ItemPointerData;

typedef struct BlockIdData
{
    uint16  bi_hi;    // 块号高16位
    uint16  bi_lo;    // 块号低16位
} BlockIdData;

// 总计: 6字节
// 块号范围: 0 ~ 2^32-1 (最大16TB，假设8KB页)
// 偏移号范围: 0 ~ 65535
```

### 4.2 TID的作用

```
1. 物理位置: 唯一标识元组的物理位置
   TID (blockNumber, offsetNumber)
   
2. 索引引用: B-Tree索引存储 (key → TID) 映射

3. 元组链: t_ctid 形成更新链
   旧版本 t_ctid → 新版本 TID
   
4. CTID系统列: 每个表都有隐藏的ctid列
   SELECT ctid, * FROM table;
```

---

## 5. 元组布局示例

### 5.1 完整元组

```
表定义:
  CREATE TABLE example (
      a INT,        -- 4B
      b TEXT,       -- 变长
      c TIMESTAMP   -- 8B
  );

元组布局:
┌───────────────────────────────────────────────────────┐
│ HeapTupleHeader (23B fixed)                           │
│ ┌─────────────────────────────────────────────────┐  │
│ │ t_xmin (4B)          = 1000                     │  │
│ │ t_xmax (4B)          = 0 (未删除)               │  │
│ │ t_cid (4B)           = 0                        │  │
│ │ t_ctid (6B)          = (0, 1) 指向自己          │  │
│ │ t_infomask2 (2B)     = 3 (3列)                 │  │
│ │ t_infomask (2B)      = 0x0802 (HASVARWIDTH等)  │  │
│ │ t_hoff (1B)          = 24 (头长度)              │  │
│ └─────────────────────────────────────────────────┘  │
├───────────────────────────────────────────────────────┤
│ NULL位图 (1B): 0b00000111 (没有NULL)                 │
├───────────────────────────────────────────────────────┤
│ 数据部分:                                             │
│ ┌─────────────────────────────────────────────────┐  │
│ │ a (INT4): 42                    (4B)            │  │
│ │ b (TEXT): varhdr(4B) + "hello" (5B) = 9B       │  │
│ │   → 对齐到4B边界，补齐至12B                     │  │
│ │ c (TIMESTAMP): 2024-01-01 00:00 (8B)           │  │
│ └─────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘

总大小: 23 + 1 + 4 + 12 + 8 = 48B
```

### 5.2 有NULL值的元组

```
INSERT INTO example VALUES (NULL, 'world', now());

元组布局:
┌───────────────────────────────────────────────────────┐
│ HeapTupleHeader (23B)                                 │
│ t_infomask |= HEAP_HASNULL                            │
├───────────────────────────────────────────────────────┤
│ NULL位图 (1B): 0b00000001                             │
│   → 第0列(a)为NULL, 其他列非NULL                      │
├───────────────────────────────────────────────────────┤
│ 数据部分:                                             │
│ ┌─────────────────────────────────────────────────┐  │
│ │ a: (跳过，NULL)                                 │  │
│ │ b (TEXT): varhdr(4B) + "world" (5B)             │  │
│ │ c (TIMESTAMP): ...                              │  │
│ └─────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
```

---

## 6. TOAST指针结构

### 6.1 varattrib_1b_e (TOAST指针)

```c
// src/include/postgres.h:529
typedef struct varattrib_1b_e
{
    uint8   va_header;     // 标记为外部存储 (1B)
    uint8   va_tag;        // TOAST标签 (1B)
    char    va_data[FLEXIBLE_ARRAY_MEMBER]; // 实际指针数据
} varattrib_1b_e;

// TOAST指针完整结构 (18B)
struct toast_pointer {
    int32   va_rawsize;        // 原始未压缩大小 (4B)
    int32   va_extsize;        // 外部存储大小 (4B)
    Oid     va_valueid;        // TOAST表中的OID (4B)
    Oid     va_toastrelid;     // TOAST表的OID (4B)
};
```

---

## 7. FSM页面结构

```c
// src/backend/storage/freespace/fsmpage.c
// FSM使用3层B树结构

FSM页面 (8KB):
┌─────────────────────────────────────┐
│ FSMPageHeader                       │
├─────────────────────────────────────┤
│ Slots[0..N]                         │
│   每个slot: 1字节表示空闲空间       │
│   0xFF = 全满                       │
│   0x00 = 完全空                     │
│   精度: ~32字节                     │
└─────────────────────────────────────┘

层次结构:
  Level 2 (Root): 汇总Level 1的最大值
  Level 1 (Branch): 汇总Level 0的最大值
  Level 0 (Leaf): 实际页面空闲空间
```

---

## 8. VM页面结构

```c
// src/include/access/visibilitymap.h
// VM使用位图，每个堆表页占2位

VM页面 (8KB):
  可表示: 8KB * 8 / 2 = 32768 个堆表页
  覆盖: 32768 * 8KB = 256MB 堆表数据

每页2位:
  Bit 0: all-visible    (所有元组对所有事务可见)
  Bit 1: all-frozen     (所有元组已冻结)

示例:
  VM[0] = 0b11 → 页0: all-visible + all-frozen
  VM[1] = 0b01 → 页1: all-visible (但未frozen)
  VM[2] = 0b00 → 页2: 有不可见元组
```

---

## 9. 内存中的元组表示

### 9.1 HeapTuple

```c
// src/include/access/htup.h:62
typedef struct HeapTupleData
{
    uint32          t_len;          // 元组总长度 (4B)
    ItemPointerData t_self;         // 元组的TID (6B)
    Oid             t_tableOid;     // 所属表的OID (4B)
    HeapTupleHeader t_data;         // 指向实际数据的指针 (8B)
} HeapTupleData;

// 总计: 22字节 (内存中的元数据)
// 注意: 这只是指针结构，不包含实际元组数据
```

---

## 10. 大小计算

### 10.1 页面容量估算

```
页面大小: 8192B

固定开销:
  PageHeader: 24B
  Special: 0B (堆表)
  
可用空间: 8192 - 24 = 8168B

元组大小 = HeapTupleHeader + NULL位图 + 数据 + 对齐

假设平均元组100B:
  ItemId: 4B
  元组: 100B
  每元组开销: 104B

最大元组数: 8168 / 104 ≈ 78 个元组/页

实际: 留一些空闲空间 → ~70个元组/页
```

---

## 总结

### 核心结构层次

```
页面 (8KB)
  ├─ PageHeaderData (24B)
  ├─ ItemIdData[] (4B each)
  └─ HeapTupleHeaderData (23B+) + 用户数据
      ├─ t_xmin, t_xmax (MVCC)
      ├─ t_ctid (元组链)
      ├─ t_infomask (标志位)
      └─ t_bits[] (NULL位图)
```

### 关键点

- ✅ 间接寻址: ItemId → 元组 (支持元组移动)
- ✅ 紧凑存储: ItemId只4字节
- ✅ MVCC内嵌: 元组头包含事务ID
- ✅ HOT支持: t_infomask标志位
- ✅ TOAST透明: 大字段自动外部存储

---

**下一步**: 阅读 [03_implementation_flow.md](03_implementation_flow.md) 了解操作流程

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16
