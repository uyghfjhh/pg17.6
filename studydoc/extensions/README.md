# Extensions 扩展系统 (精简版)

> PostgreSQL扩展机制核心原理

**源码**: `src/backend/commands/extension.c`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 📚 核心概念

**Extensions** 是PostgreSQL的**可扩展性核心**，允许动态添加功能而不修改核心代码。

### 扩展类型

```
✅ SQL扩展 - 纯SQL实现
✅ C扩展 - 共享库 (.so)
✅ PL扩展 - 过程语言 (PL/Python, PL/Perl)
✅ 数据类型扩展 - 自定义类型
✅ 索引扩展 - 自定义索引方法
```

---

## 🎯 扩展架构

```
PostgreSQL Extension架构:

[1] 扩展文件结构:
    my_extension/
    ├── my_extension.control  (元数据)
    ├── my_extension--1.0.sql (安装脚本)
    ├── my_extension--1.0--1.1.sql (升级脚本)
    └── my_extension.so       (C共享库)

[2] 安装位置:
    $SHAREDIR/extension/      (SQL和control文件)
    $LIBDIR/                  (共享库)

[3] 使用:
    CREATE EXTENSION my_extension;
    DROP EXTENSION my_extension;
```

---

**详细内容**: 阅读 [01_extensions_core.md](01_extensions_core.md)

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

