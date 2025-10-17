# 软件工程实践

> 从PostgreSQL学习工业级代码组织和工程实践

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-17

---

## 目录

1. [代码组织与模块化](#1-代码组织与模块化)
2. [错误处理机制](#2-错误处理机制)
3. [内存管理](#3-内存管理)
4. [资源清理与RAII](#4-资源清理与raii)
5. [测试策略](#5-测试策略)
6. [文档与注释规范](#6-文档与注释规范)

---

## 1. 代码组织与模块化

### 1.1 目录结构

```
src/backend/
├── access/          # 存储访问层
│   ├── common/      # 通用代码
│   ├── heap/        # 堆表
│   ├── nbtree/      # B树索引
│   └── transam/     # 事务管理
│
├── executor/        # 查询执行器
│   ├── execMain.c
│   ├── nodeSeqscan.c
│   └── nodeIndexscan.c
│
├── optimizer/       # 查询优化器
│   ├── path/
│   ├── plan/
│   └── util/
│
├── storage/         # 存储管理
│   ├── buffer/      # 缓冲池
│   ├── smgr/        # 存储管理器
│   └── lmgr/        # 锁管理器
│
└── utils/           # 工具函数
    ├── adt/         # 数据类型
    ├── cache/       # 系统缓存
    └── mmgr/        # 内存管理

原则: 按功能模块组织，清晰的层级关系
```

### 1.2 头文件组织

```c
/* ❌ 不好: 循环依赖 */
// a.h includes b.h
// b.h includes a.h

/* ✅ 好: 前向声明 */
// a.h
typedef struct BufferDesc BufferDesc;
void ProcessBuffer(BufferDesc *buf);

// a.c
#include "storage/bufmgr.h"  // 完整定义

原则:
✅ 头文件只包含必要的声明
✅ 实现文件包含完整定义
✅ 使用前向声明避免循环依赖
```

---

## 2. 错误处理机制

### 2.1 elog/ereport

```c
/* 错误级别 */
DEBUG    - 调试信息
LOG      - 正常日志
INFO     - 提示信息
NOTICE   - 注意事项
WARNING  - 警告
ERROR    - 错误 (回滚当前事务)
FATAL    - 严重错误 (终止当前会话)
PANIC    - 致命错误 (停止数据库)

/* 使用示例 */
if (buffer == NULL)
    elog(ERROR, "invalid buffer pointer");

if (!TransactionIdIsValid(xid))
    ereport(ERROR,
            (errcode(ERRCODE_INVALID_PARAMETER_VALUE),
             errmsg("invalid transaction ID: %u", xid),
             errhint("Check the transaction log.")));
```

### 2.2 异常安全

```c
/* PG_TRY/PG_CATCH机制 */
PG_TRY();
{
    /* 可能出错的操作 */
    result = risky_operation();
}
PG_CATCH();
{
    /* 错误处理 */
    cleanup_resources();
    PG_RE_THROW();  // 重新抛出异常
}
PG_END_TRY();

原理: 基于setjmp/longjmp实现
类似: C++的try/catch
```

---

## 3. 内存管理

### 3.1 内存上下文

```
内存上下文层级:
┌──────────────────────────┐
│ TopMemoryContext         │ ← 永久存在
└────────┬─────────────────┘
         │
    ┌────┴────┐
    │         │
┌───┴───┐ ┌──┴────┐
│Transaction│ │Query  │ ← 事务/查询结束时释放
│Context    │ │Context│
└───────────┘ └───────┘

创建和使用:
MemoryContext old_context;
MemoryContext my_context;

my_context = AllocSetContextCreate(
    CurrentMemoryContext,
    "My Context",
    ALLOCSET_DEFAULT_SIZES);

old_context = MemoryContextSwitchTo(my_context);
ptr = palloc(size);  // 在my_context中分配
MemoryContextSwitchTo(old_context);

// 释放整个context
MemoryContextDelete(my_context);
```

### 3.2 内存管理优点

```
优点:
✅ 自动生命周期管理
✅ 避免内存泄漏
✅ 异常安全 (context自动清理)
✅ 性能好 (批量释放)

对比:
malloc/free:
  - 需要手动配对
  - 容易泄漏
  - 异常时难以清理

MemoryContext:
  - 自动管理
  - 不会泄漏
  - 异常安全
```

---

## 4. 资源清理与RAII

### 4.1 资源清理模式

```c
/* Resource Owner */
ResourceOwner owner = ResourceOwnerCreate(NULL, "My Owner");

/* 注册资源 */
buffer = ReadBuffer(relation, blocknum);
ResourceOwnerEnlargeBuffers(owner);
ResourceOwnerRememberBuffer(owner, buffer);

/* 自动清理 */
PG_TRY();
{
    /* 使用buffer */
}
PG_CATCH();
{
    /* 发生异常，自动释放buffer */
}
PG_END_TRY();

ResourceOwnerRelease(owner, ...);
```

### 4.2 RAII思想

```
RAII (Resource Acquisition Is Initialization):
资源获取即初始化

PostgreSQL实现:
┌──────────────────────────────┐
│ 1. 获取资源时注册到Owner      │
│    (ResourceOwnerRemember*)  │
└────────┬─────────────────────┘
         ↓
┌──────────────────────────────┐
│ 2. 正常使用资源               │
└────────┬─────────────────────┘
         ↓
┌──────────────────────────────┐
│ 3. 退出时自动释放             │
│    (ResourceOwnerRelease)    │
└──────────────────────────────┘

优点: 即使发生异常，资源也会被正确释放
```

---

## 5. 测试策略

### 5.1 测试金字塔

```
测试金字塔:
            ┌──────┐
            │ E2E  │ ← 端到端测试 (少量)
            └──┬───┘
          ┌────┴────┐
          │ 集成测试 │ ← 中等数量
          └────┬────┘
        ┌──────┴──────┐
        │   单元测试   │ ← 大量
        └─────────────┘

PostgreSQL测试:
- 回归测试 (Regression Tests)
- TAP测试 (Test Anything Protocol)
- 隔离测试 (Isolation Tests)
- pg_regress框架
```

### 5.2 回归测试

```bash
# 运行回归测试
cd src/test/regress
make check

# 测试文件组织
sql/           # SQL测试脚本
expected/      # 期望输出
results/       # 实际输出

# 测试示例
-- test: select
SELECT * FROM test_table;

-- test: join
SELECT a.*, b.* FROM a JOIN b ON a.id = b.id;
```

### 5.3 隔离测试

```
测试并发场景:
session1:
BEGIN;
SELECT * FROM accounts WHERE id = 1;

session2:
BEGIN;
UPDATE accounts SET balance = 100 WHERE id = 1;

session1:
SELECT * FROM accounts WHERE id = 1;
COMMIT;

session2:
COMMIT;

验证: 不同隔离级别下的行为
```

---

## 6. 文档与注释规范

### 6.1 函数注释

```c
/*
 * BufferAlloc - 分配一个buffer
 *
 * 从空闲列表或通过淘汰算法获取一个buffer。
 * 如果buffer是脏的，先写回磁盘。
 *
 * 参数:
 *   tag - buffer的标识
 *   strategy - 缓冲策略
 *   found - 返回是否在缓冲池中找到
 *
 * 返回:
 *   Buffer descriptor指针
 *
 * 注意:
 *   调用者必须持有BufMappingLock
 */
static BufferDesc *
BufferAlloc(BufferTag *tag,
            BufferAccessStrategy strategy,
            bool *found)
{
    // 实现...
}
```

### 6.2 代码注释风格

```c
/* ✅ 好: 解释"为什么" */
/*
 * 我们在这里使用双重检查锁定模式。
 * 第一次检查避免昂贵的锁获取，
 * 第二次检查确保正确性。
 */
if (!buffer->valid)  // 快速路径
{
    LockBuffer(buffer);
    if (!buffer->valid)  // 慢路径
    {
        LoadBuffer(buffer);
    }
    UnlockBuffer(buffer);
}

/* ❌ 不好: 重复代码 */
// 检查buffer是否有效
if (!buffer->valid)
{
    // 加锁
    LockBuffer(buffer);
}
```

### 6.3 文档组织

```
docs/
├── README          # 项目概述
├── INSTALL         # 安装指南
├── TODO            # 待办事项
└── src/
    └── backend/
        └── storage/
            └── README  # 模块文档

模块README内容:
1. 模块概述
2. 核心概念
3. 数据结构
4. 关键算法
5. 使用示例
6. 注意事项
```

---

## 总结

### 软件工程实践

1. **模块化** - 清晰的目录结构和模块划分
2. **错误处理** - elog/ereport统一机制
3. **内存管理** - MemoryContext自动管理
4. **资源清理** - ResourceOwner+RAII
5. **测试** - 完善的测试框架
6. **文档** - 详细的注释和文档

### 设计原则

- ✅ 代码组织清晰
- ✅ 错误处理统一
- ✅ 资源自动管理
- ✅ 异常安全
- ✅ 充分测试
- ✅ 文档完善

---

**完成**: 所有设计学习系列文档已完成！

**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-17

