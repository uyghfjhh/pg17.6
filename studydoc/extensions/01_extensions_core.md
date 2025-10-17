# Extensions 核心分析 (精简版)

> PostgreSQL扩展系统的核心原理、开发和使用

**源码**: `src/backend/commands/extension.c`  
**版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

---

## 1. 扩展系统架构

### 1.1 扩展机制

```
PostgreSQL扩展系统:

[优势]
  ✅ 模块化 - 功能解耦
  ✅ 版本管理 - 独立升级
  ✅ 依赖管理 - 自动处理依赖
  ✅ 一键安装 - CREATE EXTENSION
  ✅ Schema隔离 - 避免命名冲突

[核心组件]
  • Extension Control File (.control)
  • SQL Script Files (.sql)
  • Shared Library (.so)
  • Extension Catalog (pg_extension)
```

---

## 2. 常用扩展

### 2.1 官方扩展

```sql
-- pg_stat_statements (查询统计)
CREATE EXTENSION pg_stat_statements;
SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;

-- pgcrypto (加密函数)
CREATE EXTENSION pgcrypto;
SELECT crypt('password', gen_salt('bf'));

-- hstore (键值存储)
CREATE EXTENSION hstore;
SELECT 'a=>1,b=>2'::hstore;

-- uuid-ossp (UUID生成)
CREATE EXTENSION "uuid-ossp";
SELECT uuid_generate_v4();

-- pg_trgm (模糊搜索)
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_name_trgm ON users USING gin(name gin_trgm_ops);
SELECT * FROM users WHERE name % 'Alice';  -- 相似度搜索

-- btree_gin/btree_gist (复合索引)
CREATE EXTENSION btree_gin;
CREATE INDEX idx_multi ON table USING gin(col1, col2);
```

### 2.2 第三方扩展

```sql
-- PostGIS (地理信息)
CREATE EXTENSION postgis;
SELECT ST_Distance(
    ST_MakePoint(-73.9857, 40.7484),  -- NYC
    ST_MakePoint(-0.1276, 51.5074)    -- London
);

-- TimescaleDB (时序数据)
CREATE EXTENSION timescaledb;
SELECT create_hypertable('metrics', 'time');

-- Citus (分布式)
CREATE EXTENSION citus;
SELECT create_distributed_table('events', 'user_id');

-- pg_partman (分区管理)
CREATE EXTENSION pg_partman;

-- pgvector (向量相似度)
CREATE EXTENSION vector;
CREATE TABLE items (embedding vector(3));
SELECT * FROM items ORDER BY embedding <-> '[1,2,3]' LIMIT 5;
```

---

## 3. 扩展开发

### 3.1 创建简单SQL扩展

```sql
-- [1] 创建control文件: my_extension.control
/*
comment = 'My custom extension'
default_version = '1.0'
relocatable = true
schema = my_ext
*/

-- [2] 创建SQL脚本: my_extension--1.0.sql
CREATE FUNCTION hello(name text)
RETURNS text AS $$
    SELECT 'Hello, ' || name || '!';
$$ LANGUAGE SQL IMMUTABLE;

CREATE TABLE my_table (
    id serial PRIMARY KEY,
    value text
);

-- [3] 安装
-- 将文件放到 $SHAREDIR/extension/
-- 然后:
CREATE EXTENSION my_extension;

-- [4] 使用
SELECT hello('World');  -- 'Hello, World!'
```

### 3.2 扩展升级

```sql
-- 创建升级脚本: my_extension--1.0--1.1.sql
ALTER TABLE my_table ADD COLUMN created_at timestamp DEFAULT now();

CREATE FUNCTION goodbye(name text)
RETURNS text AS $$
    SELECT 'Goodbye, ' || name || '!';
$$ LANGUAGE SQL IMMUTABLE;

-- 升级扩展
ALTER EXTENSION my_extension UPDATE TO '1.1';
```

---

## 4. C扩展开发

### 4.1 基本结构

```c
/* my_cfunc.c */
#include "postgres.h"
#include "fmgr.h"
#include "utils/builtins.h"

PG_MODULE_MAGIC;  /* 必须 */

/* 函数声明 */
PG_FUNCTION_INFO_V1(add_one);

Datum
add_one(PG_FUNCTION_ARGS)
{
    int32 arg = PG_GETARG_INT32(0);
    PG_RETURN_INT32(arg + 1);
}
```

### 4.2 编译和安装

```makefile
# Makefile
MODULES = my_cfunc
EXTENSION = my_cfunc
DATA = my_cfunc--1.0.sql

PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
```

```sql
-- my_cfunc--1.0.sql
CREATE FUNCTION add_one(integer)
RETURNS integer
AS 'MODULE_PATHNAME'
LANGUAGE C IMMUTABLE STRICT;
```

```bash
# 编译安装
make
sudo make install

# 使用
CREATE EXTENSION my_cfunc;
SELECT add_one(41);  -- 42
```

---

## 5. 扩展管理

### 5.1 查询扩展

```sql
-- 查看已安装扩展
SELECT * FROM pg_extension;

-- 查看可用扩展
SELECT * FROM pg_available_extensions;

-- 查看扩展详情
\dx+ pg_stat_statements

-- 查看扩展对象
SELECT 
    e.extname,
    n.nspname as schema,
    p.proname as function
FROM pg_extension e
JOIN pg_depend d ON d.refobjid = e.oid
JOIN pg_proc p ON p.oid = d.objid
JOIN pg_namespace n ON n.oid = p.pronamespace
WHERE e.extname = 'pg_stat_statements';
```

### 5.2 扩展操作

```sql
-- 创建扩展
CREATE EXTENSION pg_trgm;

-- 指定Schema
CREATE EXTENSION pg_trgm SCHEMA extensions;

-- 指定版本
CREATE EXTENSION my_ext VERSION '1.0';

-- 如果不存在才创建
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- 更新扩展
ALTER EXTENSION pg_trgm UPDATE TO '1.5';

-- 删除扩展
DROP EXTENSION pg_trgm;

-- 级联删除 (删除依赖对象)
DROP EXTENSION pg_trgm CASCADE;
```

---

## 6. 扩展依赖

### 6.1 声明依赖

```
-- my_extension.control
requires = 'hstore, postgis'
```

### 6.2 查看依赖

```sql
-- 查看扩展依赖关系
SELECT 
    e1.extname as extension,
    e2.extname as depends_on
FROM pg_extension e1
JOIN pg_depend d ON d.objid = e1.oid
JOIN pg_extension e2 ON e2.oid = d.refobjid
WHERE d.deptype = 'e';
```

---

## 7. 高级特性

### 7.1 自定义数据类型

```c
/* complex.c - 复数类型 */
typedef struct Complex {
    double x;  /* 实部 */
    double y;  /* 虚部 */
} Complex;

/* 输入函数 */
PG_FUNCTION_INFO_V1(complex_in);
Datum
complex_in(PG_FUNCTION_ARGS)
{
    char *str = PG_GETARG_CSTRING(0);
    Complex *result = (Complex *) palloc(sizeof(Complex));
    
    if (sscanf(str, " ( %lf , %lf )", &result->x, &result->y) != 2)
        ereport(ERROR, ...);
    
    PG_RETURN_POINTER(result);
}

/* 输出函数 */
PG_FUNCTION_INFO_V1(complex_out);
Datum
complex_out(PG_FUNCTION_ARGS)
{
    Complex *complex = (Complex *) PG_GETARG_POINTER(0);
    char *result = psprintf("(%g,%g)", complex->x, complex->y);
    PG_RETURN_CSTRING(result);
}
```

```sql
-- complex--1.0.sql
CREATE TYPE complex (
    internallength = 16,
    input = complex_in,
    output = complex_out,
    alignment = double
);

-- 使用
SELECT '(1.0, 2.0)'::complex;
```

### 7.2 自定义操作符

```sql
-- 复数加法
CREATE FUNCTION complex_add(complex, complex)
RETURNS complex
AS 'MODULE_PATHNAME'
LANGUAGE C IMMUTABLE STRICT;

CREATE OPERATOR + (
    leftarg = complex,
    rightarg = complex,
    procedure = complex_add,
    commutator = +
);

-- 使用
SELECT '(1,2)'::complex + '(3,4)'::complex;  -- (4,6)
```

### 7.3 自定义索引方法

```c
/* 实现AM (Access Method) handler */
PG_FUNCTION_INFO_V1(my_index_handler);
Datum
my_index_handler(PG_FUNCTION_ARGS)
{
    IndexAmRoutine *amroutine = makeNode(IndexAmRoutine);
    
    amroutine->aminsert = my_index_insert;
    amroutine->ambeginscan = my_index_beginscan;
    amroutine->amgettuple = my_index_gettuple;
    amroutine->amendscan = my_index_endscan;
    
    PG_RETURN_POINTER(amroutine);
}
```

```sql
CREATE ACCESS METHOD my_index TYPE INDEX HANDLER my_index_handler;
CREATE INDEX idx_custom ON table USING my_index(column);
```

---

## 8. 扩展最佳实践

### 8.1 开发建议

```
✅ 使用PGXS编译系统
✅ 提供完整的.control文件
✅ 版本化SQL脚本
✅ 提供升级路径
✅ 处理依赖关系
✅ 编写文档和示例
✅ 单元测试 (pg_regress)
✅ Schema隔离 (relocatable=false时)
```

### 8.2 性能考虑

```
✅ C函数标记IMMUTABLE/STABLE
✅ 使用合适的内存上下文
✅ 避免内存泄漏 (pfree)
✅ 批量操作优化
✅ 使用SPI小心 (开销大)
```

### 8.3 安全考虑

```
✅ 输入验证
✅ SQL注入防护
✅ 权限检查
✅ 资源限制
✅ 错误处理
```

---

## 9. 扩展测试

### 9.1 pg_regress测试

```bash
# 创建测试文件
# sql/basic.sql
CREATE EXTENSION my_ext;
SELECT add_one(1);
SELECT add_one(99);

# expected/basic.out (期望输出)
CREATE EXTENSION my_ext;
SELECT add_one(1);
 add_one 
---------
       2
(1 row)

SELECT add_one(99);
 add_one 
---------
     100
(1 row)

# 运行测试
make installcheck
```

---

## 10. 监控和维护

### 10.1 扩展使用统计

```sql
-- 查看扩展大小
SELECT 
    e.extname,
    pg_size_pretty(
        sum(pg_relation_size(d.objid))
    ) as size
FROM pg_extension e
JOIN pg_depend d ON d.refobjid = e.oid
WHERE d.classid = 'pg_class'::regclass
GROUP BY e.extname;

-- 扩展函数调用统计 (需要pg_stat_statements)
SELECT 
    funcname,
    calls,
    total_time,
    mean_time
FROM pg_stat_user_functions
WHERE schemaname = 'my_ext'
ORDER BY total_time DESC;
```

---

## 总结

### 扩展系统核心

1. **模块化**: 功能独立，易于管理
2. **版本化**: SQL脚本版本管理
3. **依赖管理**: 自动处理扩展依赖
4. **C接口**: PGXS简化开发
5. **丰富生态**: 大量第三方扩展

### 常用扩展推荐

- ✅ pg_stat_statements - 查询分析必备
- ✅ pg_trgm - 模糊搜索
- ✅ PostGIS - 地理信息
- ✅ TimescaleDB - 时序数据
- ✅ pgvector - AI/ML应用

### 开发建议

- ✅ 使用PGXS编译系统
- ✅ 提供升级脚本
- ✅ 编写测试用例
- ✅ 注意内存管理
- ✅ Schema隔离

---

**完成**: Extensions核心文档完成！

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

