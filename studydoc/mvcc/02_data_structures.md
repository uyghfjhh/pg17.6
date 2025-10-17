# MVCC 核心数据结构详解

> 深入分析 MVCC 实现的核心数据结构，包括元组头、快照、事务状态等关键组件。

---

## 目录

1. [元组头结构](#一元组头结构)
2. [快照结构](#二快照结构)
3. [事务状态管理](#三事务状态管理)
4. [可见性映射](#四可见性映射)
5. [数据结构关系图](#五数据结构关系图)

---

## 一、元组头结构

### 1.1 HeapTupleHeaderData

**源码**: `src/include/access/htup_details.h:116`

```c
/*
 * HeapTupleHeaderData - 堆元组头部
 * 
 * 大小: 23 字节 (不包括可选的 OID)
 * 作用: 存储 MVCC 元数据和元组信息
 */
typedef struct HeapTupleHeaderData
{
    union
    {
        HeapTupleFields t_heap;    // 普通元组字段
        DatumTupleFields t_datum;  // 系统列字段
    } t_choice;

    ItemPointerData t_ctid;        // 当前或新元组的 TID

    /* 以下字段在单个 16 位字中 */
    uint16      t_infomask2;       // 字段数和标志位
    uint16      t_infomask;        // 各种标志位
    uint8       t_hoff;            // 头部大小（含对齐）

    /* 后面跟着 NULL bitmap 和用户数据 */
} HeapTupleHeaderData;
```

### 1.2 HeapTupleFields (MVCC 关键字段)

```c
/*
 * t_heap: 普通元组的关键 MVCC 字段
 */
typedef struct HeapTupleFields
{
    TransactionId t_xmin;    // 插入事务 ID (创建者)
    TransactionId t_xmax;    // 删除/更新事务 ID (删除者)

    union
    {
        CommandId   t_cid;   // 插入和删除命令 ID
        TransactionId t_xvac;// VACUUM 事务 ID
    } t_field3;
} HeapTupleFields;
```

### 1.3 元组头详细布局

```
HeapTupleHeaderData 完整布局 (23 字节):

Offset  Field           Size    说明
───────────────────────────────────────────────────────
0       t_xmin          4       插入该元组的事务 ID
4       t_xmax          4       删除该元组的事务 ID
8       t_cid/t_xvac    4       命令 ID 或 VACUUM 事务 ID
12      t_ctid          6       元组链指针 (块号+项号)
18      t_infomask2     2       字段数和标志位
20      t_infomask      2       状态标志位
22      t_hoff          1       头部大小
───────────────────────────────────────────────────────
23      [NULL bitmap]   变长    NULL 值位图 (可选)
        [user data]     变长    实际用户数据

示例元组 (users 表):
┌─────────────────────────────────────────────────────┐
│ HeapTupleHeader (23 字节)                           │
├─────────────────────────────────────────────────────┤
│ t_xmin:     1000    ← 事务 1000 插入               │
│ t_xmax:     0       ← 未被删除                     │
│ t_cid:      5       ← 命令 5 创建                  │
│ t_ctid:     (0, 12) ← 当前元组位置                 │
│ t_infomask2: 0x0003 ← 3 个字段                     │
│ t_infomask:  0x0900 ← HEAP_XMIN_COMMITTED + ...    │
│ t_hoff:     24      ← 头部 24 字节 (含对齐)        │
├─────────────────────────────────────────────────────┤
│ NULL bitmap (1 字节,因为 3 个字段)                  │
│ 0b00000000  ← 无 NULL 值                            │
├─────────────────────────────────────────────────────┤
│ User Data:                                          │
│  id:    INT     = 100                               │
│  name:  VARCHAR = 'Alice'                           │
│  age:   INT     = 25                                │
└─────────────────────────────────────────────────────┘
```

### 1.4 t_infomask 标志位 (16 位)

```c
/* t_infomask 标志位定义 */

/* 事务状态标志 */
#define HEAP_XMIN_COMMITTED    0x0100  // xmin 已提交
#define HEAP_XMIN_INVALID      0x0200  // xmin 失效(已回滚)
#define HEAP_XMAX_COMMITTED    0x0400  // xmax 已提交
#define HEAP_XMAX_INVALID      0x0800  // xmax 失效

/* 锁相关标志 */
#define HEAP_XMAX_LOCK_ONLY    0x0080  // xmax 仅是锁,非删除
#define HEAP_XMAX_KEYSHR_LOCK  0x0010  // Key Share Lock
#define HEAP_XMAX_EXCL_LOCK    0x0040  // Exclusive Lock

/* 数据标志 */
#define HEAP_HASNULL           0x0001  // 有 NULL 值
#define HEAP_HASVARWIDTH       0x0002  // 有变长字段
#define HEAP_HASEXTERNAL       0x0004  // 有 TOAST 数据
#define HEAP_HASOID            0x0008  // 有 OID (已废弃)

/* HOT 相关 */
#define HEAP_HOT_UPDATED       0x4000  // HOT 更新
#define HEAP_ONLY_TUPLE        0x8000  // HOT 链中的元组

/* 标志位组合示例 */

1. 新插入的元组:
   t_infomask = HEAP_XMIN_COMMITTED | HEAP_HASVARWIDTH
   = 0x0100 | 0x0002 = 0x0102

2. 已删除的元组:
   t_infomask = HEAP_XMIN_COMMITTED | HEAP_XMAX_COMMITTED
   = 0x0100 | 0x0400 = 0x0500

3. 被锁定的元组:
   t_infomask = HEAP_XMIN_COMMITTED | HEAP_XMAX_LOCK_ONLY
   = 0x0100 | 0x0080 = 0x0180

4. HOT 更新的元组:
   t_infomask = HEAP_XMIN_COMMITTED | HEAP_HOT_UPDATED
   = 0x0100 | 0x4000 = 0x4100
```

### 1.5 t_ctid 的作用 (元组链)

```
t_ctid (Item Pointer) 结构:
┌────────────────────────────────┐
│ BlockNumber: 4 字节 (块号)     │
│ OffsetNumber: 2 字节 (项号)    │
└────────────────────────────────┘

用途 1: 指向当前元组自己
┌──────────────────────────┐
│ Tuple at (Block 5, Item 3)│
│ t_ctid = (5, 3)           │  ← 指向自己
└──────────────────────────┘

用途 2: 指向更新后的新版本 (UPDATE 链)
┌──────────────────────────┐
│ 旧版本 Tuple (5, 3)       │
│ t_xmin = 1000            │
│ t_xmax = 1005            │  ← 被事务 1005 删除
│ t_ctid = (5, 7)          │  ← 指向新版本
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│ 新版本 Tuple (5, 7)       │
│ t_xmin = 1005            │  ← 事务 1005 创建
│ t_xmax = 0               │
│ t_ctid = (5, 7)          │  ← 指向自己
└──────────────────────────┘

用途 3: HOT (Heap Only Tuple) 链
┌──────────────────────────┐
│ 原始 Tuple (5, 3)        │
│ t_ctid = (5, 4)          │  ← 指向 HOT 更新
└──────┬───────────────────┘
       │ (同一页内更新)
       ▼
┌──────────────────────────┐
│ HOT Tuple (5, 4)         │
│ t_infomask = HEAP_ONLY_TUPLE
│ t_ctid = (5, 4)          │
└──────────────────────────┘
```

---

## 二、快照结构

### 2.1 SnapshotData

**源码**: `src/include/utils/snapshot.h:113`

```c
/*
 * SnapshotData - 快照数据结构
 * 
 * 作用: 定义事务的"可见性视图"
 * 用途: 判断哪些元组版本对当前事务可见
 */
typedef struct SnapshotData
{
    SnapshotType snapshot_type;  // 快照类型

    /*
     * 核心字段 - 定义可见性
     */
    TransactionId xmin;          // 所有 < xmin 的事务已提交
    TransactionId xmax;          // 所有 >= xmax 的事务不可见

    /*
     * 活跃事务列表
     * - 在 [xmin, xmax) 范围内
     * - 快照创建时正在运行
     * - 数组按升序排列
     */
    TransactionId *xip;          // 活跃事务 ID 数组
    uint32      xcnt;            // 数组长度

    /*
     * 子事务支持
     */
    TransactionId *subxip;       // 活跃子事务数组
    int32       subxcnt;         // 子事务数量
    bool        suboverflowed;   // 子事务是否溢出

    /*
     * MVCC 快照创建时间
     */
    CommandId   curcid;          // 当前命令 ID
    
    /*
     * 其他字段
     */
    uint32      active_count;    // 使用计数
    uint32      regd_count;      // 注册计数
    ...
} SnapshotData;
```

### 2.2 快照类型

```c
typedef enum SnapshotType
{
    SNAPSHOT_MVCC = 0,     // 标准 MVCC 快照
    SNAPSHOT_SELF,         // 只看当前事务的修改
    SNAPSHOT_ANY,          // 看所有版本 (用于 VACUUM)
    SNAPSHOT_TOAST,        // TOAST 数据访问
    SNAPSHOT_DIRTY,        // 脏读 (看未提交修改)
    SNAPSHOT_HISTORIC_MVCC // 历史快照 (逻辑解码)
} SnapshotType;
```

### 2.3 快照示例

```
场景: 数据库中有以下活跃事务

时刻 T1:
活跃事务: 1000, 1002, 1005, 1008
已提交事务: < 1000 的所有事务
未开始事务: >= 1009

创建快照 S1:
┌────────────────────────────────────────┐
│ Snapshot S1                            │
├────────────────────────────────────────┤
│ xmin:  1000                            │
│ xmax:  1009                            │
│ xip:   [1000, 1002, 1005, 1008]        │
│ xcnt:  4                               │
└────────────────────────────────────────┘

可见性规则:
┌────────────────────────────────────────┐
│ 事务 ID      状态          可见性       │
├────────────────────────────────────────┤
│ < 1000       已提交        可见 ✓       │
│ 1000         活跃(xip中)   不可见 ✗     │
│ 1001         已提交        可见 ✓       │
│ 1002         活跃(xip中)   不可见 ✗     │
│ 1003         已提交        可见 ✓       │
│ 1004         已提交        可见 ✓       │
│ 1005         活跃(xip中)   不可见 ✗     │
│ 1006         已提交        可见 ✓       │
│ 1007         已提交        可见 ✓       │
│ 1008         活跃(xip中)   不可见 ✗     │
│ >= 1009      未开始        不可见 ✗     │
└────────────────────────────────────────┘

示例查询:
SELECT * FROM users WHERE id = 100;

元组版本:
┌──────────────────────────────────────┐
│ Version 1: t_xmin=999, t_xmax=0     │
│ → xmin < 1000 → 可见 ✓              │
├──────────────────────────────────────┤
│ Version 2: t_xmin=1002, t_xmax=0    │
│ → xmin 在 xip 中 → 不可见 ✗         │
├──────────────────────────────────────┤
│ Version 3: t_xmin=1001, t_xmax=0    │
│ → xmin=1001 < xmax,不在 xip         │
│ → 已提交 → 可见 ✓                   │
└──────────────────────────────────────┘

返回 Version 1 或 Version 3 (取决于哪个有效)
```

---

## 三、事务状态管理

### 3.1 CLOG (Commit Log)

**源码**: `src/backend/access/transam/clog.c`

```
CLOG 结构:
┌─────────────────────────────────────────────────────┐
│               CLOG (事务提交日志)                    │
├─────────────────────────────────────────────────────┤
│ 作用: 存储每个事务的提交状态                         │
│ 组织: 按 TransactionId 顺序,每个事务 2 位           │
│ 位置: $PGDATA/pg_xact/                              │
└─────────────────────────────────────────────────────┘

每个事务的状态 (2 位):
00: IN_PROGRESS  (进行中)
01: COMMITTED    (已提交)
10: ABORTED      (已回滚)
11: SUB_COMMITTED (子事务提交)

CLOG 页面布局 (8KB):
┌─────────────────────────────────────────────┐
│ Page Header                                 │
├─────────────────────────────────────────────┤
│ 事务状态位 (每个 2 位)                       │
│                                             │
│ XID    State                                │
│ 0      01 (COMMITTED)                       │
│ 1      10 (ABORTED)                         │
│ 2      01 (COMMITTED)                       │
│ 3      00 (IN_PROGRESS)                     │
│ ...                                         │
│ 32767  01                                   │
└─────────────────────────────────────────────┘

容量: 8KB = 8192 字节 = 32768 位 = 16384 个事务

CLOG 文件:
pg_xact/
├── 0000  (XID 0 - 16383)
├── 0001  (XID 16384 - 32767)
├── 0002  (XID 32768 - 49151)
└── ...

查询事务状态:
┌────────────────────────────────────────┐
│ TransactionIdGetStatus(xid)            │
│  ├─ 计算页号: xid / 16384              │
│  ├─ 计算偏移: (xid % 16384) * 2 bits  │
│  ├─ 读取 2 位                          │
│  └─ 返回状态                           │
└────────────────────────────────────────┘
```

### 3.2 事务 ID 分配

```c
/*
 * TransactionId 分配策略
 */
typedef uint32 TransactionId;

/* 特殊事务 ID */
#define InvalidTransactionId  0      // 无效 ID
#define BootstrapTransactionId 1     // 启动时
#define FrozenTransactionId   2      // 冻结标记
#define FirstNormalTransactionId 3   // 第一个正常 ID

/*
 * 事务 ID 循环问题
 * 
 * 32 位 TransactionId 范围: 0 ~ 4,294,967,295
 * 约 42 亿个事务
 * 
 * 问题: ID 用尽后回绕 (wraparound)
 * 解决: VACUUM FREEZE
 */

事务 ID 比较 (考虑回绕):
┌────────────────────────────────────────┐
│ 当前 XID: 1000                         │
│                                        │
│ XID   相对关系     说明                 │
│ 500   < 1000      过去                 │
│ 1000  = 1000      当前                 │
│ 1500  > 1000      未来                 │
│                                        │
│ 回绕场景:                              │
│ 当前 XID: 4,000,000,000               │
│                                        │
│ 100   > 当前      未来 (回绕)          │
│ 2,000,000,000 < 当前 过去              │
└────────────────────────────────────────┘
```

### 3.3 提示位 (Hint Bits)

```
提示位优化 CLOG 访问:

问题: 每次检查元组可见性都要查 CLOG → 慢
解决: 将事务状态缓存在元组头的 t_infomask 中

提示位标志:
- HEAP_XMIN_COMMITTED: xmin 已提交
- HEAP_XMIN_INVALID:   xmin 已回滚
- HEAP_XMAX_COMMITTED: xmax 已提交
- HEAP_XMAX_INVALID:   xmax 已回滚

设置流程:
┌────────────────────────────────────────┐
│ 1. 首次访问元组                         │
│    检查 t_infomask                     │
│    没有提示位 → 查询 CLOG               │
│                                        │
│ 2. 从 CLOG 获取事务状态                 │
│    状态: COMMITTED                     │
│                                        │
│ 3. 设置提示位                          │
│    t_infomask |= HEAP_XMIN_COMMITTED  │
│                                        │
│ 4. 后续访问                            │
│    直接读取提示位,无需查 CLOG ✓        │
└────────────────────────────────────────┘

性能提升:
- 首次访问: 需查 CLOG (~1-5 μs)
- 后续访问: 只读提示位 (~0.1 μs)
- 50x 性能提升!
```

---

## 四、可见性映射

### 4.1 Visibility Map (VM)

**源码**: `src/backend/access/heap/visibilitymap.c`

```
VM 文件结构:
┌─────────────────────────────────────────────┐
│ 文件名: <relfilenode>_vm                    │
│ 位置: $PGDATA/base/<dboid>/                │
│ 每页: 2 位 (all-visible + all-frozen)      │
└─────────────────────────────────────────────┘

位含义:
Bit 0 (all-visible):  所有元组对所有事务可见
Bit 1 (all-frozen):   所有元组已冻结

┌─────────────────────────────────────────────┐
│ VM 页面 (8KB)                               │
├─────────────────────────────────────────────┤
│ Page  Bits    说明                          │
│ 0     11      all-visible + all-frozen     │
│ 1     10      all-visible only             │
│ 2     00      neither                      │
│ 3     01      all-frozen only (罕见)       │
│ ...                                         │
│ 32767 11                                    │
└─────────────────────────────────────────────┘

容量: 8KB = 65536 位 = 32768 页 = 256MB 数据

优化效果:
┌────────────────────────────────────────┐
│ Index-Only Scan 优化:                  │
│  - 如果 VM 标记 all-visible            │
│  - 索引扫描无需访问堆表                │
│  - 大幅减少 I/O                        │
│                                        │
│ VACUUM 优化:                           │
│  - 跳过 all-visible 页面               │
│  - 只处理有死元组的页面                │
│  - 加快 VACUUM 速度                    │
└────────────────────────────────────────┘
```

### 4.2 VM 更新时机

```
VM 位设置:
┌────────────────────────────────────────┐
│ 1. VACUUM 扫描页面                     │
│    检查所有元组                        │
│     ├─ 所有 t_xmin 已冻结?            │
│     ├─ 所有元组对所有事务可见?         │
│     └─ 无死元组?                       │
│                                        │
│ 2. 设置相应位                          │
│    if (all_frozen)                     │
│      set bit 1                         │
│    if (all_visible)                    │
│      set bit 0                         │
└────────────────────────────────────────┘

VM 位清除:
┌────────────────────────────────────────┐
│ 触发条件:                              │
│  - INSERT (新元组)                     │
│  - UPDATE (修改元组)                   │
│  - DELETE (删除元组)                   │
│                                        │
│ 操作:                                  │
│  clear bits for affected page         │
└────────────────────────────────────────┘
```

---

## 五、数据结构关系图

### 5.1 MVCC 数据结构全景

```
┌────────────────────────────────────────────────────────┐
│                 MVCC 核心数据结构                       │
└────────────────────────────────────────────────────────┘

Backend 进程
   │
   │ 创建快照
   ▼
┌──────────────────────────────┐
│ SnapshotData                 │
│  xmin: 1000                  │
│  xmax: 1010                  │
│  xip: [1002, 1005, 1008]     │
└──────────────┬───────────────┘
               │
               │ 检查可见性
               ▼
┌──────────────────────────────────────────────┐
│ 数据页 (Data Page)                           │
├──────────────────────────────────────────────┤
│ HeapTupleHeader                              │
│ ┌──────────────────────────────────────────┐│
│ │ t_xmin: 999                              ││
│ │ t_xmax: 0                                ││
│ │ t_infomask: HEAP_XMIN_COMMITTED          ││
│ │ t_ctid: (5, 3)                           ││
│ └──────────────────────────────────────────┘│
│           ▲                                  │
│           │ 检查事务状态                     │
│           ▼                                  │
│ ┌──────────────────────────────────────────┐│
│ │ 提示位 (Hint Bits)                       ││
│ │  - HEAP_XMIN_COMMITTED: 已提交           ││
│ │  - 无需查 CLOG ✓                        ││
│ └──────────────────────────────────────────┘│
└──────────────────┬───────────────────────────┘
                   │ (如需要)
                   │ 查询 CLOG
                   ▼
┌──────────────────────────────────────────────┐
│ CLOG (pg_xact/)                              │
│ ┌──────────────────────────────────────────┐│
│ │ XID    Status                            ││
│ │ 999    01 (COMMITTED)                    ││
│ │ 1000   00 (IN_PROGRESS)                  ││
│ │ 1001   01 (COMMITTED)                    ││
│ │ ...                                      ││
│ └──────────────────────────────────────────┘│
└──────────────────────────────────────────────┘

优化层:
┌──────────────────────────────────────────────┐
│ Visibility Map (_vm)                         │
│ ┌──────────────────────────────────────────┐│
│ │ Page   all-visible  all-frozen          ││
│ │ 5      1            0                    ││
│ │ 6      1            1                    ││
│ │ 7      0            0                    ││
│ └──────────────────────────────────────────┘│
│ 作用: 加速 VACUUM 和 Index-Only Scan         │
└──────────────────────────────────────────────┘
```

### 5.2 元组版本链

```
UPDATE 操作创建的元组版本链:

原始元组:
┌──────────────────────────────┐
│ TID: (5, 3)                  │
│ t_xmin: 1000                 │
│ t_xmax: 1005                 │ ← 被事务 1005 更新
│ t_ctid: (5, 7)               │ ← 指向新版本
│ data: {name: 'Alice'}        │
└──────────┬───────────────────┘
           │
           ▼
新版本:
┌──────────────────────────────┐
│ TID: (5, 7)                  │
│ t_xmin: 1005                 │ ← 事务 1005 创建
│ t_xmax: 0                    │ ← 尚未被删除
│ t_ctid: (5, 7)               │ ← 指向自己
│ data: {name: 'Bob'}          │
└──────────────────────────────┘

不同快照的可见性:
┌────────────────────────────────────────┐
│ 快照 A (xmin=1000, xmax=1005)          │
│  看到: (5, 3) 版本 'Alice' ✓           │
│                                        │
│ 快照 B (xmin=1006, xmax=1010)          │
│  看到: (5, 7) 版本 'Bob' ✓             │
└────────────────────────────────────────┘
```

---

## 总结

### MVCC 数据结构的精妙设计

1. **HeapTupleHeader**: 23 字节存储关键 MVCC 元数据
2. **SnapshotData**: 定义事务的"可见性视图"
3. **CLOG**: 紧凑存储事务状态 (每事务 2 位)
4. **Visibility Map**: 优化 VACUUM 和索引扫描
5. **提示位**: 缓存事务状态,避免重复查 CLOG

### 关键特点

- ✅ 空间高效 (23 字节元组头)
- ✅ 时间高效 (提示位优化)
- ✅ 可扩展 (支持子事务)
- ✅ 并发友好 (无锁读取)

---

**文档版本**: 1.0
**相关源码**: PostgreSQL 17.5
**创建日期**: 2025-01-16

**下一篇**: [03_implementation_flow.md](03_implementation_flow.md) - MVCC 实现流程分析


