---
layout: post
title: "PostgreSQL 查询性能优化实战指南"
date: 2026-07-31 10:00:00 +0800
categories: [开发]
tags: [postgresql, database, performance, optimization, sql]
---

## 为什么需要学会看执行计划

很多开发者在写 SQL 时，习惯性地认为"能跑就行"——只要查询结果正确，性能问题交给 DBA 或者加索引解决。但在实际生产环境中，一个慢查询就能拖垮整个数据库，甚至引发级联故障。

PostgreSQL 提供了强大的查询优化器和执行计划分析工具。学会读懂执行计划，是数据库性能调优的第一步，也是最重要的一步。

这篇指南从实操出发，不讲空泛的理论，只给能直接用的方法和案例。

## 准备实验环境

为了让所有示例可复现，先创建一个测试表并填充数据：

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    amount NUMERIC(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 插入 100 万行测试数据
INSERT INTO orders (user_id, product_id, amount, status, created_at)
SELECT
    (random() * 10000)::int + 1,
    (random() * 5000)::int + 1,
    (random() * 1000)::numeric(10,2) + 0.01,
    CASE WHEN random() < 0.6 THEN 'completed'
         WHEN random() < 0.8 THEN 'pending'
         WHEN random() < 0.95 THEN 'cancelled'
         ELSE 'refunded' END,
    NOW() - (random() * 365 || ' days')::interval
FROM generate_series(1, 1000000);
```

同样创建一张关联的用户表：

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    tier VARCHAR(20) NOT NULL DEFAULT 'free',
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

INSERT INTO users (id, email, tier)
SELECT
    g,
    'user' || g || '@example.com',
    CASE WHEN random() < 0.5 THEN 'free'
         WHEN random() < 0.8 THEN 'pro'
         ELSE 'enterprise' END
FROM generate_series(1, 10000) g;
```

分析前先更新统计信息，确保优化器有准确的数据分布：

```sql
ANALYZE orders;
ANALYZE users;
```

## EXPLAIN ANALYZE：你的第一诊断工具

### 基础用法

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING) 
SELECT * FROM orders WHERE status = 'cancelled';
```

输出示例：

```
Gather  (cost=1000.00..12976.43 rows=48500 width=44)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on orders
        Filter: ((status)::text = 'cancelled'::text)
        Rows Removed by Filter: 317167
        Buffers: shared hit=8850
Planning Time: 0.123 ms
Execution Time: 45.672 ms
```

关键信息解读：
- **cost**：第一个数字是启动成本，第二个是总成本。单位是 PostgreSQL 的抽象成本单位，不是毫秒
- **rows**：优化器预估的行数。与真实行数差异过大说明统计信息过时
- **Buffers: shared hit**：从共享缓冲区读取的页面数。8850 页 × 8KB ≈ 69MB 数据被扫描
- **Execution Time**：实际执行时间，这是最直接的性能指标

### 看懂三种常见扫描方式

**Seq Scan（顺序扫描）**——全表逐行扫描，数据量大时极慢。

```
Seq Scan on orders  (cost=0.00..19834.00 rows=485000 width=44)
  Filter: ((status)::text = 'cancelled'::text)
```

**Index Scan（索引扫描）**——通过索引定位行，返回少量行时高效。

```
Index Scan using idx_orders_status on orders
  Index Cond: ((status)::text = 'cancelled'::text)
```

**Index Only Scan（仅索引扫描）**——所有需要的列都在索引中，无需回表，最快。

```
Index Only Scan using idx_orders_status_amount on orders
  Index Cond: ((status)::text = 'cancelled'::text)
```

## 索引设计：从最常用的模式开始

### 单列索引——最基础的加速

```sql
CREATE INDEX idx_orders_status ON orders (status);
```

适用于等值查询 `WHERE status = 'completed'`。但要注意，如果查询返回的行数超过表总量的 5-10%，优化器会直接跳过索引走顺序扫描——因为随机 I/O 比顺序 I/O 更慢。

### 复合索引——多条件查询的关键

```sql
-- 为最常见的查询模式创建复合索引
CREATE INDEX idx_orders_user_status_date ON orders (user_id, status, created_at DESC);
```

这个索引直接覆盖以下查询：

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 42
  AND status = 'completed'
ORDER BY created_at DESC
LIMIT 10;
```

复合索引的**列顺序**至关重要。规则很简单：等值条件放前面，范围条件放后面。`user_id` 和 `status` 是等值查询，`created_at` 是排序/范围条件，所以放在最后。

### 部分索引——只索引你需要的行

很多场景下，你只关心某一部分数据。部分索引能大幅减小索引体积，提升写入性能：

```sql
-- 只索引活跃订单，忽略已完结的
CREATE INDEX idx_orders_active ON orders (created_at DESC)
WHERE status IN ('pending', 'processing');
```

这个索引只有约 20% 的行，查询活跃订单时速度更快，维护成本更低。

### 覆盖索引——避免回表

```sql
-- 如果查询只需要 amount 和 status，包含它们到索引中
CREATE INDEX idx_orders_status_covering ON orders (status)
INCLUDE (amount, created_at);
```

查询 `SELECT status, amount, created_at FROM orders WHERE status = 'completed'` 会触发 Index Only Scan，完全不需要访问堆表。

## 查询优化实战案例

### 案例 1：分页查询性能优化

**反模式——OFFSET 深分页：**

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
ORDER BY created_at DESC
OFFSET 100000 LIMIT 20;
```

```
Limit  (cost=10086.39..10086.44 rows=20 width=44)
  ->  Sort  (cost=10086.39..10386.39 rows=120000 width=44)
        Sort Key: created_at DESC
        ->  Seq Scan on orders  (cost=0.00..19834.00 rows=1000000 width=44)
```

OFFSET 100000 意味着数据库需要排序全部 100 万行，然后丢弃前 100000 行。越往后翻越慢。

**优化方案——Keyset Pagination（游标分页）：**

```sql
-- 第一页
SELECT * FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- 后续页：记住上一页最后一条的 created_at 和 id
SELECT * FROM orders
WHERE (created_at, id) < ('2026-01-15 10:00:00', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

```
Limit  (cost=0.43..12.45 rows=20 width=44)
  ->  Index Scan Backward using idx_orders_date_id on orders
        Index Cond: (ROW(created_at, id) < ROW('2026-01-15 10:00:00'::timestamp without time zone, 12345))
```

无论翻到第几页，性能始终不变。这是生产环境必须使用的分页方式。

### 案例 2：JOIN 优化

**慢查询：**

```sql
EXPLAIN ANALYZE
SELECT u.email, o.amount, o.status
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.tier = 'enterprise'
  AND o.created_at >= NOW() - INTERVAL '30 days';
```

如果 orders 表没有 `user_id` 索引，会触发 Hash Join 或 Nested Loop 扫描大量数据。

**优化方案：**

```sql
-- 确保 JOIN 键上有索引
CREATE INDEX idx_orders_user_id ON orders (user_id);

-- 考虑在 orders 上创建复合索引覆盖查询
CREATE INDEX idx_orders_user_date ON orders (user_id, created_at DESC)
WHERE created_at >= NOW() - INTERVAL '30 days';
```

### 案例 3：聚合查询优化

**慢查询——扫描全表计算统计：**

```sql
EXPLAIN ANALYZE
SELECT
    user_id,
    COUNT(*) as order_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount
FROM orders
WHERE created_at >= '2026-01-01'
GROUP BY user_id
HAVING COUNT(*) > 10;
```

**优化方案——物化视图：**

```sql
CREATE MATERIALIZED VIEW user_order_stats AS
SELECT
    user_id,
    COUNT(*) as order_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount,
    MAX(created_at) as last_order_date
FROM orders
GROUP BY user_id;

CREATE UNIQUE INDEX idx_stats_user ON user_order_stats (user_id);

-- 定时刷新（例如每天凌晨）
REFRESH MATERIALIZED VIEW CONCURRENTLY user_order_stats;
```

查询变成：

```sql
SELECT * FROM user_order_stats
WHERE order_count > 10;
```

毫秒级返回，不需要扫描百万行数据。

## 配置调优：不重启也能生效

### 确认当前配置

```sql
SHOW shared_buffers;
SHOW work_mem;
SHOW effective_cache_size;
SHOW random_page_cost;
```

### 关键参数推荐值

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `shared_buffers` | 内存的 25% | 共享缓冲区大小，不是越大越好 |
| `work_mem` | 32-256MB | 排序和哈希操作的内存，注意是 per-operation |
| `effective_cache_size` | 内存的 50-75% | 让优化器知道操作系统能缓存多少 |
| `random_page_cost` | 1.1（SSD） | SSD 设为 1.1，HDD 保持 4.0 |
| `maintenance_work_mem` | 1GB | VACUUM、CREATE INDEX 等维护操作 |

在 session 级别临时调整：

```sql
SET work_mem = '128MB';
SET random_page_cost = 1.1;
```

## 慢查询定位三板斧

### 1. pg_stat_statements——全局慢查询视图

```sql
-- 需要先启用扩展
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT
    queryid,
    calls,
    mean_exec_time::numeric(10,2) as avg_ms,
    total_exec_time::numeric(10,2) as total_ms,
    rows,
    SUBSTRING(query, 1, 80) as query_preview
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

这个视图能告诉你：哪些查询执行最频繁、哪些最耗时、哪些返回行数最多。

### 2. 启用慢查询日志

```ini
# postgresql.conf
log_min_duration_statement = 1000  # 记录超过 1 秒的查询
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
```

### 3. 使用 auto_explain 自动记录慢查询的执行计划

```ini
# postgresql.conf 或 session 级别
LOAD 'auto_explain';
SET auto_explain.log_min_duration = '500ms';  -- 记录超过 500ms 的查询计划
SET auto_explain.log_analyze = on;
SET auto_explain.log_buffers = on;
SET auto_explain.log_nested_statements = on;
```

这样每个慢查询都会自动附带完整的执行计划到日志，不需要手动去 EXPLAIN。

## 一个完整的调优流程

假设你收到告警：某个页面加载需要 30 秒。

**第一步：确认慢查询**

```sql
-- 从 pg_stat_activity 抓取正在运行的查询
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC
LIMIT 5;
```

**第二步：获取执行计划**

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING)
-- 粘贴慢查询 SQL
```

**第三步：分析瓶颈点**

- 是否全表扫描？→ 加索引
- 是否预估行数严重偏差？→ ANALYZE 更新统计信息
- 是否有临时文件写入？→ work_mem 不足
- 是否大量 Buffer 命中？→ 缓存有效，考虑索引优化

**第四步：优化后验证**

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING)
-- 优化后的查询
```

对比优化前后的 Execution Time 和 Buffers，确认改善幅度。

## 总结

PostgreSQL 查询优化的核心就三件事：

1. **读懂执行计划**——EXPLAIN ANALYZE 是你最常用的工具，不要凭感觉猜
2. **设计对口的索引**——复合索引的列顺序、部分索引、覆盖索引，比单列索引有效得多
3. **让统计信息准确**——定期的 ANALYZE 和合理的配置参数，是优化器做对决策的前提

把这三点做到位，95% 的慢查询问题都能解决。剩下的 5%，通常涉及数据模型设计本身的缺陷，那已经不是"查询优化"能解决的了。

下次遇到慢查询，别急着加索引。先跑一遍 EXPLAIN ANALYZE，看清楚数据库到底在干什么。