# HEAP 测试用例

本文档提供堆表相关的测试用例。

---

## 1. 基础操作测试

### 测试1.1: INSERT

```sql
CREATE TABLE test_heap (id INT, data TEXT);

INSERT INTO test_heap VALUES (1, 'test');
SELECT * FROM test_heap;
-- 预期: (1, 'test')

SELECT ctid, * FROM test_heap;
-- 预期: ctid = (0,1)
```

### 测试1.2: UPDATE

```sql
UPDATE test_heap SET data = 'updated' WHERE id = 1;
SELECT ctid, * FROM test_heap;
-- 预期: ctid 改变 (如 (0,2))
```

### 测试1.3: DELETE

```sql
DELETE FROM test_heap WHERE id = 1;
SELECT * FROM test_heap;
-- 预期: 空结果
```

---

## 2. HOT Update 测试

### 测试2.1: HOT Update成功

```sql
CREATE TABLE hot_test (
    id SERIAL PRIMARY KEY,
    status TEXT,
    data TEXT
) WITH (fillfactor = 70);

INSERT INTO hot_test (status, data) 
SELECT 'active', 'data' || i FROM generate_series(1, 100) i;

-- 重置统计
SELECT pg_stat_reset_single_table_counters('hot_test'::regclass);

-- 更新非索引列
UPDATE hot_test SET data = 'updated' WHERE id = 1;

-- 检查HOT Update
SELECT n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables WHERE relname = 'hot_test';
-- 预期: n_tup_hot_upd = 1
```

### 测试2.2: HOT Update失败 (改变索引列)

```sql
UPDATE hot_test SET id = 999 WHERE id = 1;

SELECT n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables WHERE relname = 'hot_test';
-- 预期: n_tup_upd = 2, n_tup_hot_upd = 1 (未增加)
```

---

## 3. TOAST测试

### 测试3.1: TOAST触发

```sql
CREATE TABLE toast_test (id INT, large_text TEXT);

-- 插入大数据 (>2KB)
INSERT INTO toast_test VALUES (1, repeat('x', 3000));

-- 查看TOAST表
SELECT relname, reltoastrelid::regclass 
FROM pg_class WHERE relname = 'toast_test';

-- 检查TOAST表大小
SELECT pg_size_pretty(pg_relation_size(reltoastrelid)) 
FROM pg_class WHERE relname = 'toast_test';
-- 预期: > 0
```

---

## 4. Fillfactor测试

### 测试4.1: Fillfactor影响

```sql
-- 低fillfactor
CREATE TABLE ff_70 (id INT, data TEXT) WITH (fillfactor = 70);
-- 高fillfactor
CREATE TABLE ff_100 (id INT, data TEXT) WITH (fillfactor = 100);

-- 插入相同数据
INSERT INTO ff_70 SELECT i, 'data' FROM generate_series(1, 1000) i;
INSERT INTO ff_100 SELECT i, 'data' FROM generate_series(1, 1000) i;

-- 比较大小
SELECT 
    'ff_70' AS table_name, pg_size_pretty(pg_relation_size('ff_70')) AS size
UNION ALL
SELECT 
    'ff_100', pg_size_pretty(pg_relation_size('ff_100'));
    
-- 预期: ff_70 > ff_100 (预留了空间)
```

---

## 5. 表膨胀测试

### 测试5.1: 制造表膨胀

```sql
CREATE TABLE bloat_test (id INT, data TEXT);
INSERT INTO bloat_test SELECT i, 'data' FROM generate_series(1, 10000) i;

-- 大量UPDATE (制造死元组)
UPDATE bloat_test SET data = 'updated' || random();

-- 检查死元组
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'bloat_test';
-- 预期: n_dead_tup = 10000

-- VACUUM清理
VACUUM bloat_test;

-- 再次检查
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'bloat_test';
-- 预期: n_dead_tup ≈ 0
```

---

## 6. 页面检查测试

### 测试6.1: pageinspect

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

CREATE TABLE page_test (id INT, data TEXT);
INSERT INTO page_test VALUES (1, 'test');

-- 查看页头
SELECT * FROM page_header(get_raw_page('page_test', 0));

-- 查看页内容
SELECT lp, lp_off, lp_flags, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('page_test', 0));
```

---

## 总结

测试覆盖:
- ✅ 基础CRUD操作
- ✅ HOT Update机制
- ✅ TOAST功能
- ✅ Fillfactor效果
- ✅ 表膨胀和VACUUM
- ✅ 页面内部结构

**下一步**: 阅读 [07_diagrams.md](07_diagrams.md)

**源码版本**: PostgreSQL 17.6  
**最后更新**: 2025-10-16

