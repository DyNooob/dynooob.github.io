---
layout: post
title: "Python 异步编程：asyncio 从基础到生产实践"
date: 2026-09-09 09:00:00 +0800
categories: [开发]
tags: [Python, asyncio, 异步编程, 并发, 性能优化]
---

## 1. 为什么需要异步

Python 的 GIL 让多线程在 CPU 密集场景下形同虚设，而多进程又太重。大部分 Web 服务、API 调用、数据库查询的瓶颈在于 **I/O 等待**——程序花大量时间等网络响应、等磁盘读写、等数据库返回结果。asyncio 让这些等待期间 CPU 可以切去做别的事，用单线程实现高并发。

一个直观对比：顺序请求 10 个 HTTP API，每个耗时 200ms，总耗时 2s。用 asyncio 并发请求，总耗时约 200ms——接近 10 倍提升，且不依赖多线程。

asyncio 的核心模型是**协作式多任务**：协程主动让出控制权（await），事件循环调度下一个就绪的协程。没有抢占、没有 race condition、没有锁的烦恼——**前提是你只用 async 代码**。

## 2. 基础概念

### 2.1 协程与 await

```python
import asyncio

async def fetch_data(url: str) -> dict:
    print(f"开始请求: {url}")
    await asyncio.sleep(1)  # 模拟 I/O 等待
    return {"url": url, "status": 200}

# 协程对象需要事件循环驱动
coro = fetch_data("https://api.example.com/data")
# 直接运行
result = asyncio.run(coro)
print(result)
```

`async def` 定义协程函数，调用后返回协程对象（不执行）。`await` 挂起当前协程，把控制权交还给事件循环。`asyncio.run()` 创建事件循环、运行入口协程、最后关闭循环——**一个程序只调一次**。

### 2.2 awaitable 对象

三种 awaitable：

| 类型 | 说明 | 示例 |
|------|------|------|
| 协程 (coroutine) | async def 返回的对象 | `await my_coro()` |
| Task | 包装协程为调度单元 | `await asyncio.create_task(coro())` |
| Future | 底层 awaitable，Task 继承自它 | 通常不直接使用 |

关键区别：协程被 `await` 时直接执行；Task 被创建时**立即加入事件循环调度队列**，不等你 `await` 就开始跑。

### 2.3 事件循环

事件循环是 asyncio 的核心调度器。`asyncio.run()` 帮你管理它，但了解以下 API 有助于调试和高级场景：

```python
# 获取当前事件循环
loop = asyncio.get_running_loop()

# 查看所有等待中的 Task
tasks = asyncio.all_tasks(loop)

# 调试模式（会打印更多调度信息）
asyncio.run(main(), debug=True)
```

## 3. 核心并发 API

### 3.1 create_task —— 最基本的并发单元

```python
async def worker(n: int):
    await asyncio.sleep(n)
    return f"worker-{n} done"

async def main():
    # Task 创建即开始运行，不阻塞
    t1 = asyncio.create_task(worker(2))
    t2 = asyncio.create_task(worker(1))
    
    # 在 await 之前，两个 worker 都在跑
    r1 = await t1  # 等 t1 完成（约 2s）
    r2 = await t2  # t2 早就完成了（1s），这句几乎不阻塞
    print(r1, r2)  # 总耗时 ~2s，不是 3s
```

### 3.2 gather —— 批量等待一组协程

```python
async def main():
    results = await asyncio.gather(
        worker(3),
        worker(1),
        worker(2),
        return_exceptions=True  # 遇到异常不中断其他任务
    )
    print(results)  # ['worker-3 done', 'worker-1 done', 'worker-2 done']
```

`gather` 按传入顺序返回结果，不是按完成顺序。`return_exceptions=True` 时，异常会被作为返回值返回而不冒泡。

### 3.3 as_completed —— 谁先完成处理谁

当你想按完成顺序处理结果时很有用：

```python
async def main():
    coros = [worker(i) for i in [3, 1, 2]]
    for task in asyncio.as_completed(coros):
        result = await task
        print(f"完成: {result}")  # 按 1s → 2s → 3s 顺序打印
```

### 3.4 wait —— 精细控制等待策略

```python
async def main():
    tasks = {asyncio.create_task(worker(i)) for i in [1, 2, 3]}
    done, pending = await asyncio.wait(
        tasks,
        timeout=2.0,               # 最多等 2 秒
        return_when=asyncio.FIRST_COMPLETED  # 第一个完成就返回
    )
    print(f"已完成: {len(done)}, 待处理: {len(pending)}")
    # 取消未完成的任务
    for t in pending:
        t.cancel()
```

## 4. 实战案例

### 4.1 并发 HTTP 请求（生产级）

用 `aiohttp` 实现，但思路通用（`httpx` 也类似）：

```python
import asyncio
import aiohttp
from dataclasses import dataclass
from typing import Optional

@dataclass
class FetchResult:
    url: str
    status: int
    data: Optional[bytes] = None
    error: Optional[str] = None

async def fetch_one(session: aiohttp.ClientSession, url: str, 
                    timeout: float = 10.0) -> FetchResult:
    try:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=timeout)) as resp:
            data = await resp.read()
            return FetchResult(url=url, status=resp.status, data=data)
    except asyncio.TimeoutError:
        return FetchResult(url=url, status=0, error="timeout")
    except Exception as e:
        return FetchResult(url=url, status=0, error=str(e))

async def fetch_many(urls: list[str], concurrency: int = 10) -> list[FetchResult]:
    sem = asyncio.Semaphore(concurrency)
    
    async def bounded_fetch(session, url):
        async with sem:  # 控制并发上限
            return await fetch_one(session, url)
    
    connector = aiohttp.TCPConnector(limit=concurrency, ttl_dns_cache=300)
    async with aiohttp.ClientSession(connector=connector) as session:
        tasks = [bounded_fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)

# 使用
urls = [f"https://httpbin.org/delay/{i}" for i in range(1, 11)]
results = asyncio.run(fetch_many(urls, concurrency=5))
for r in results:
    print(f"{r.url} → {r.status}" + (f" [{r.error}]" if r.error else ""))
```

关键点：
- **信号量限制并发**：避免打爆目标服务器或本地连接池
- **连接器复用**：`TCPConnector` 管理连接池和 DNS 缓存
- **超时兜底**：每个请求都有独立超时，防止一个慢请求拖垮全部
- **错误隔离**：异常被捕获为 `FetchResult`，不中断其他请求

### 4.2 异步数据库查询（asyncpg 示例）

```python
import asyncpg
import asyncio

async def query_users(conn, user_ids: list[int]) -> list[dict]:
    # 并发查询多条数据
    async def fetch_one(uid):
        row = await conn.fetchrow(
            "SELECT id, name, email FROM users WHERE id = $1", uid
        )
        return dict(row) if row else None
    
    tasks = [fetch_one(uid) for uid in user_ids]
    return await asyncio.gather(*tasks)

async def main():
    conn = await asyncpg.connect(
        "postgresql://user:pass@localhost:5432/db",
        min_size=5, max_size=20  # 连接池配置
    )
    try:
        users = await query_users(conn, [1, 2, 3, 4, 5])
        for u in users:
            print(u)
    finally:
        await conn.close()

asyncio.run(main())
```

`asyncpg` 是目前最快的 Python PostgreSQL 驱动，比 psycopg2 快 3-10 倍。但注意：异步只是让数据库调用不阻塞主线程，真正的查询性能取决于 SQL 本身的优化。

### 4.3 超时与优雅关闭

生产环境中，必须处理以下场景：
- 某个协程卡死不返回
- 收到 SIGTERM 需要平滑退出
- 部分结果可用时不必等全部完成

```python
import asyncio
import signal

async def main():
    # 超时保护
    try:
        result = await asyncio.wait_for(
            worker(100),  # 假设要跑 100 秒
            timeout=5.0   # 最多等 5 秒
        )
    except asyncio.TimeoutError:
        print("任务超时，已取消")
    
    # 优雅关闭
    stop_event = asyncio.Event()
    
    def handle_signal():
        print("收到停止信号，开始优雅关闭...")
        stop_event.set()
    
    # 注册信号处理
    loop = asyncio.get_running_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, handle_signal)
    
    # 主循环
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_one(session, url) for url in urls]
        # as_completed + stop_event 实现优雅退出
        for coro in asyncio.as_completed(tasks):
            if stop_event.is_set():
                break
            result = await coro
            process(result)
    
    print("所有占用资源已释放")

asyncio.run(main())
```

## 5. 常见陷阱

### 5.1 阻塞代码混入事件循环

**最致命的错误**：在协程里调用同步阻塞函数。

```python
# 错误的做法 —— 会阻塞整个事件循环
async def bad_coro():
    time.sleep(5)           # ❌ 同步 sleep，阻塞所有协程
    requests.get(url)       # ❌ 同步 HTTP，阻塞直到返回
    json.loads(big_data)    # ⚠️ CPU 密集操作，也会阻塞

# 正确的做法
async def good_coro():
    await asyncio.sleep(5)                       # ✓ 异步 sleep
    async with aiohttp.ClientSession() as s:     # ✓ 异步 HTTP
        async with s.get(url) as resp: ...
    data = await asyncio.to_thread(json.loads, big_data)  # ✓ 丢到线程池
```

`asyncio.to_thread()` 是 Python 3.9+ 加入的重要 API，把同步阻塞函数交给线程池执行，不阻塞事件循环。

### 5.2 忘记 await 协程

```python
async def main():
    worker(1)  # ❌ 创建了协程对象但没有 await，事件循环不会调度它
    # 正确：t = await worker(1) 或 t = asyncio.create_task(worker(1))
```

更隐蔽的版本：

```python
results = await asyncio.gather(
    worker(1),      # ✓ 传协程
    worker(2)(),    # ❌ 传了返回值，worker(2) 被调用但没 await
)
```

### 5.3 Task 没有引用被立即 GC

```python
# 危险：Task 创建后没有变量引用，可能被垃圾回收
async def main():
    asyncio.create_task(background_worker())  # ❌ 可能被立即 GC
    await asyncio.sleep(10)

# 安全：保持引用
async def main():
    task = asyncio.create_task(background_worker())  # ✓
    await asyncio.sleep(10)
    task.cancel()
```

### 5.4 异常静默丢失

```python
# 错误：Task 内部的异常不会自动冒泡
async def main():
    asyncio.create_task(will_fail())  # ❌ 异常被吞，你永远不会知道
    await asyncio.sleep(1)

# 正确：收集所有 Task 并 await
async def main():
    task = asyncio.create_task(will_fail())
    try:
        await task
    except Exception as e:
        print(f"任务失败: {e}")
```

或者用 `asyncio.gather(return_exceptions=True)` 统一检查异常。

## 6. 性能对比

用 `timeit` 风格测试，但这里用实际代码演示差异：

```python
import time
import asyncio
import aiohttp
import requests

URL = "https://httpbin.org/delay/1"

def sync_fetch(n: int):
    start = time.perf_counter()
    for _ in range(n):
        requests.get(URL)
    return time.perf_counter() - start

async def async_fetch(n: int, concurrency: int):
    sem = asyncio.Semaphore(concurrency)
    async def fetch():
        async with sem:
            async with aiohttp.ClientSession() as s:
                async with s.get(URL) as resp:
                    return resp.status
    start = time.perf_counter()
    await asyncio.gather(*[fetch() for _ in range(n)])
    return time.perf_counter() - start

# 10 个请求，每个 1s 延迟
print(f"同步: {sync_fetch(10):.2f}s")      # 约 10s
print(f"异步(10并发): {asyncio.run(async_fetch(10, 10)):.2f}s")  # 约 1s
print(f"异步(5并发): {asyncio.run(async_fetch(10, 5)):.2f}s")   # 约 2s
```

异步在 I/O 密集场景下提升显著。但**CPU 密集场景不要用 asyncio**——它不会让代码跑得更快，只会增加调度开销。CPU 密集用 `multiprocessing`。

## 7. 生产建议清单

| 实践 | 说明 |
|------|------|
| **控制并发** | 永远用 `Semaphore` 或连接池限制并发数，无限制的 `gather` 会打爆资源 |
| **设置超时** | 每个 I/O 操作都要有超时，用 `asyncio.wait_for` 或 `asyncio.timeout` (3.11+) |
| **优雅关闭** | 注册信号处理器，收到 SIGTERM/SIGINT 时取消所有 Task，释放资源 |
| **日志 trace** | 设置 `debug=True` 或在关键路径打印 `asyncio.current_task()` 的 name |
| **不要混用** | 同步和异步代码通过 `asyncio.to_thread()` / `loop.run_in_executor()` 桥接 |
| **监控 Task** | 定期检查 `asyncio.all_tasks()`，排查被遗忘的 Task |
| **选对库** | HTTP: aiohttp/httpx, DB: asyncpg/aiomysql/aioredis, 文件: aiofiles |
| **版本适配** | asyncio.run() 3.7+, Task 对象名 3.8+, asyncio.to_thread() 3.9+, TaskGroup 3.11+ |

## 8. 总结

asyncio 不是 Python 性能的银弹——它解决的是 I/O 等待的问题，而不是 CPU 计算的问题。在正确的场景（网络请求、数据库查询、文件读写）使用正确的模式（create_task + gather + 限流 + 超时），可以显著提升吞吐量。

最关键的三条原则：
1. **永远不要在协程里放同步阻塞调用**——用 `to_thread()` 桥接
2. **永远给并发操作设限**——信号量 + 超时 + 异常处理
3. **永远保持 Task 引用**——被 GC 回收的 Task 就是静默丢失的作业

从 `async/await` 语法到 `TaskGroup`（Python 3.11+），asyncio 在持续进化。理解它的调度模型和陷阱，才能写出既快又稳的生产级异步代码。