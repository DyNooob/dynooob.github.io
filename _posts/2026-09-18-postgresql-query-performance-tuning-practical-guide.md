---
layout: post
title: "PostgreSQL 查询性能调优实战指南"
date: 2026-09-18 09:00:00 +0800
categories: [开发]
tags: [postgresql, database, performance, sql, query-tuning, indexing, explain, backend, optimization]
---

## 为什么需要查询性能调优

PostgreSQL 是功能最强大的开源关系型数据库，但"强大"的另一面是"复杂"。一个写得不经心的查询，在数据量从万级增长到百万级时，执行时间可能从 10ms 膨胀到 10s 甚至更久。绝大多数性能问题不是硬件不够，而是查询写得不对、索引没建好、配置没调优。

本文从实战出发，覆盖 EXPLAIN 解读、索引策略、查询重写、配置调优四个核心环节，每一个环节都有真实可复现的命令和输出。

## 一、读懂 EXPLAIN 是调优的起点

在优化任何查询之前，必须先知道 PostgreSQL 打算怎么执行它。`EXPLAIN` 和 `EXPLAIN ANALYZE` 是你的第一把手术刀。

### 基础用法

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
```

输出是一棵计划节点树，从叶子到根描述了执行步骤。不使用 `ANALYZE` 时，输出的是预估成本，不是真实执行时间。

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING) 
SELECT * FROM orders WHERE user_id = 42;
```

**参数说明：**
- `ANALYZE`：实际执行查询并报告真实耗时和行数
- `BUFFERS`：显示缓存命中情况（shared hit / read）
- `TIMING`：显示每个节点的实际耗时（默认开启）

### 读懂一个典型的执行计划

```
                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..15431.20 rows=150 width=42)  (actual time=0.35..128.42 rows=2000 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on orders  (cost=0.00..14416.20 rows=62 width=42)  (actual time=62.18..96.33 rows=667 loops=3)
         Filter: (user_id = 42)
         Rows Removed by Filter: 333333
 Planning Time: 0.12 ms
 Execution Time: 128.58 ms
```

**关键信息解读：**

| 字段 | 含义 |
|------|------|
| `cost` | 启动成本..总成本（任意单位，相对值） |
| `rows` | 预估返回行数 |
| `actual time` | 实际耗时（启动..结束） |
| `loops` | 该节点执行次数 |
| `Rows Removed by Filter` | 扫描中过滤掉的行数 |

上面这个计划的问题是：虽然只查 2000 行，但走了**全表并行扫描**，过滤掉了 100 万行。这就是缺少索引的典型信号。

### 三种最常遇到的计划节点

**Seq Scan（顺序扫描）：** 全表逐行扫描。小表没问题，大表超过几千行就危险。

**Index Scan（索引扫描）：** 通过索引定位到少量行，再回表取完整数据。这是你想要的。

**Index Only Scan（仅索引扫描）：** 所需列全部在索引中，无需回表。性能最佳。

判断标准很简单：`EXPLAIN ANALYZE` 后看 `actual time` 和 `rows` 是否匹配预期——如果 `Seq Scan` 过滤掉了 99% 的行，说明该建索引了。

## 二、索引策略：选对类型比建得多更重要

很多人遇到慢查询的第一反应是"加索引"，但加什么样的索引、加在哪些列上、什么时候不该加，这些才是关键。

### B-tree 索引（默认类型）

适用场景：等值查询（`=`）、范围查询（`<`、`>`、`BETWEEN`）、排序（`ORDER BY`）、前缀匹配（`LIKE 'abc%'`）。

```sql
CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_orders_created_at ON orders (created_at);
```

**复合索引的列顺序至关重要：**

```sql
-- 查询：WHERE user_id = 42 AND status = 'paid' ORDER BY created_at DESC
CREATE INDEX idx_orders_user_status_created 
  ON orders (user_id, status, created_at DESC);
```

**列顺序原则：** 等值条件的列放前面，范围条件（`<`、`>`）的列放中间，排序列放最后。PostgreSQL 在复合索引上只能使用最左前缀——如果你的查询条件不包含最左列，索引不会被使用。

### 部分索引

当查询只关心某个子集时，部分索引可以大幅缩小索引体积：

```sql
-- 只索引已支付且金额大于 100 的订单
CREATE INDEX idx_orders_large_paid 
  ON orders (user_id, created_at) 
  WHERE status = 'paid' AND amount > 100;
```

这个索引比全表索引小得多，写入维护成本也更低。

### 覆盖索引

如果查询只需要少数几列，把它们都包含在索引里，就可以走 Index Only Scan：

```sql
CREATE INDEX idx_orders_user_cover 
  ON orders (user_id) 
  INCLUDE (amount, status, created_at);
```

现在 `SELECT amount, status FROM orders WHERE user_id = 42` 不需要回表。

### 不适合索引的场景

- **小表**（< 1000 行）：Seq Scan 比 Index Scan 更快，因为索引的随机 I/O 开销超过了顺序扫描
- **频繁更新的高基数列**：索引维护成本可能超过查询收益
- **不会出现在 WHERE / JOIN / ORDER BY 中的列**

### 用 pg_stat_user_indexes 检查索引使用率

```sql
SELECT 
  schemaname, tablename, indexname, 
  idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

`idx_scan = 0` 的索引就是**从未被使用过的死索引**，可以安全删除。

## 三、查询重写：三条最有效的优化法则

写对 SQL 只是第一步，写高效的 SQL 需要理解优化器的工作原理。

### 法则一：用 EXISTS 代替 COUNT(*) 做存在性检查

**反例：**

```sql
IF (SELECT COUNT(*) FROM orders WHERE user_id = 42) > 0 THEN
  -- 做某事
END IF;
```

这会在找到匹配后继续扫描全表计数。正确写法：

```sql
IF EXISTS (SELECT 1 FROM orders WHERE user_id = 42) THEN
  -- 做某事
END IF;
```

`EXISTS` 在找到**第一条**匹配记录后就停止扫描。

### 法则二：避免在 WHERE 中对列使用函数或计算

**反例（索引无效）：**

```sql
SELECT * FROM orders 
WHERE DATE(created_at) = '2026-09-18';
```

即使 `created_at` 上有索引，`DATE()` 函数包装后优化器也无法使用它。改写为范围查询：

```sql
SELECT * FROM orders 
WHERE created_at >= '2026-09-18 00:00:00' 
  AND created_at <  '2026-09-19 00:00:00';
```

**另一个常见陷阱（隐式类型转换）：**

```sql
-- 假设 user_id 是 integer 类型
SELECT * FROM orders WHERE user_id = '42';  -- 隐式转换，索引仍可用
SELECT * FROM orders WHERE CAST(user_id AS text) = '42';  -- 索引不可用
```

### 法则三：用 LATERAL JOIN 替代分组子查询

当需要取每个分组的前 N 条记录时，`LATERAL JOIN` 比窗口函数 + 子查询更高效：

```sql
-- 查每个用户最近 3 笔订单
SELECT u.id, u.name, o.amount, o.created_at
FROM users u
CROSS JOIN LATERAL (
  SELECT amount, created_at
  FROM orders
  WHERE user_id = u.id
  ORDER BY created_at DESC
  LIMIT 3
) o;
```

这个查询对每个用户只扫描索引的前 3 行，而非全表分组排序。

## 四、配置调优：不需要重启也能改善

PostgreSQL 的默认配置针对的是资源有限的开发环境。生产环境需要调整以下参数，其中大部分无需重启即可生效。

### shared_buffers（共享缓冲区大小）

最关键的缓存参数。假设机器有 16 GB 内存：

```ini
shared_buffers = 4GB      # 物理内存的 25%
```

```sql
-- 在线查看当前值
SHOW shared_buffers;

-- 需要重启才能修改
ALTER SYSTEM SET shared_buffers = '4GB';
```

### work_mem（每操作内存）

控制排序、哈希连接等操作使用的内存。默认 4MB 对复杂查询来说太小：

```ini
work_mem = 64MB
```

注意：这个值是"每个操作每个节点"的，不是全局的。一个包含三个排序的查询可能消耗 3 × work_mem。保守设置为 32-64MB，监控 `pg_stat_activity` 中的 `temporary file` 使用情况来判断是否需要调大。

```sql
-- 查看是否有查询在写临时文件（说明 work_mem 不够）
SELECT query, temp_files, temp_bytes 
FROM pg_stat_statements 
WHERE temp_files > 0 
ORDER BY temp_bytes DESC 
LIMIT 5;
```

### effective_cache_size

告诉优化器操作系统层面缓存了多少数据。不准确会影响优化器对索引扫描成本的判断：

```ini
effective_cache_size = 12GB  # 物理内存的 75%
```

### random_page_cost

机械硬盘默认 4.0，SSD 上应该降到 1.1-1.5，否则优化器会高估索引扫描的成本：

```ini
random_page_cost = 1.1  # SSD
```

### 实时查看哪些查询最慢

安装 `pg_stat_statements` 扩展后：

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT 
  query,
  calls,
  mean_exec_time::numeric(10,2) AS avg_ms,
  total_exec_time::numeric(10,2) AS total_ms,
  rows,
  shared_blks_hit::numeric / (shared_blks_hit + shared_blks_read + 1)::numeric AS cache_hit_ratio
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat%'
ORDER BY total_exec_time DESC
LIMIT 10;
```

这个查询直接告诉你：哪个 SQL 最耗时、调用了多少次、缓存命中率是多少。**缓存命中率低于 99% 的查询需要重点优化**。

## 五、一个完整的调优案例

假设有一个慢查询：

```sql
SELECT 
  u.name,
  COUNT(o.id) AS order_count,
  SUM(o.amount) AS total_spent
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at >= '2026-01-01'
  AND o.status = 'paid'
  AND o.created_at >= '2026-01-01'
GROUP BY u.id, u.name
HAVING COUNT(o.id) > 5
ORDER BY total_spent DESC
LIMIT 50;
```

数据量：users 50 万行，orders 2000 万行。

### 第一步：EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS) ...
```

发现：orders 走了 Seq Scan（扫描 2000 万行，过滤后剩 500 万行），排序使用了磁盘临时文件。

### 第二步：添加复合索引

```sql
CREATE INDEX idx_orders_user_status_created 
  ON orders (user_id, status, created_at) 
  WHERE status = 'paid';
```

部分索引 + 复合索引，直接缩小扫描范围。

### 第三步：调整 work_mem

发现排序走了磁盘，将 work_mem 从 4MB 提到 64MB。

### 第四步：最终效果

查询时间从 **12.3 秒**下降到 **47 毫秒**，提升了 260 倍。缓存命中率从 87% 提升到 99.7%。

## 六、日常健康检查清单

定期执行以下检查，可以在问题恶化前发现：

```sql
-- 1. 长运行查询
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC
LIMIT 10;

-- 2. 等待事件分布
SELECT wait_event_type, wait_event, count(*) 
FROM pg_stat_activity 
WHERE wait_event IS NOT NULL 
GROUP BY wait_event_type, wait_event 
ORDER BY count(*) DESC;

-- 3. 索引膨胀
SELECT 
  schemaname, tablename, indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  pg_size_pretty(pg_relation_size(indrelid)) AS table_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 10;

-- 4. 表膨胀和死元组
SELECT 
  relname, 
  n_live_tup, n_dead_tup,
  round(n_dead_tup::numeric / (n_live_tup + 1), 2) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

死元组比例超过 20% 的表需要尽快 `VACUUM`。

## 总结

PostgreSQL 查询性能调优没有银弹，但有清晰的方法论：

1. **用 EXPLAIN ANALYZE 量化问题**——不要猜，要测
2. **索引要精准**——复合索引、部分索引、覆盖索引，选对类型比建得多重要
3. **查询重写解决结构性问题**——EXISTS 替代 COUNT、范围查询替代函数包裹、LATERAL 替代分组子查询
4. **配置适配硬件**——shared_buffers、work_mem、effective_cache_size、random_page_cost 四个参数先调好
5. **持续监控**——pg_stat_statements 和 pg_stat_activity 是你最常用的诊断工具

把这些方法变成日常习惯，慢查询会越来越少。下一次遇到查询慢，不要先想着加内存——先 EXPLAIN。