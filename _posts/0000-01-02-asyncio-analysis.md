---
layout: slide
title: "Python asyncio 异步编程：优缺点分析与最佳实践"
---

Python asyncio 异步编程

优缺点分析与最佳实践

---

## 什么是 asyncio？

- Python 3.4+ 引入的**异步 I/O 框架**
- 基于**事件循环 (Event Loop)** 的并发模型
- 使用 `async/await` 语法（Python 3.5+）
- 单线程实现并发，适合 **I/O 密集型** 任务

```python
import asyncio

async def main():
    print("Hello")
    await asyncio.sleep(1)
    print("World")

asyncio.run(main())
```

---

## 核心概念

| 概念 | 说明 |
|------|------|
| **协程 (Coroutine)** | 用 `async def` 定义的异步函数 |
| **事件循环 (Event Loop)** | 调度和执行协程的核心引擎 |
| **Task** | 对协程的封装，可并发执行 |
| **Future** | 表示异步操作的最终结果 |
| **await** | 挂起当前协程，等待异步操作完成 |

---

## 优点一：高并发性能

- 单线程处理**数千甚至数万**并发连接
- 无线程切换开销，**内存占用低**
- 适合网络服务器、爬虫、API 网关等场景

```python
async def fetch_all(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        # 同时发起所有请求
        return await asyncio.gather(*tasks)
```

> 对比线程方案：1000 个并发连接仅需 ~10MB 内存，线程方案需要 ~1GB

---

## 优点二：代码可读性

- `async/await` 语法让异步代码**接近同步风格**
- 避免了回调地狱 (Callback Hell)
- 控制流清晰，异常处理直观

```python
# 回调风格（难以维护）
def callback_style():
    fetch(url1, lambda r1:
        fetch(url2, lambda r2:
            process(r1, r2, lambda result:
                save(result, on_done))))

# async/await 风格（清晰易读）
async def async_style():
    r1 = await fetch(url1)
    r2 = await fetch(url2)
    result = await process(r1, r2)
    await save(result)
```

---

## 优点三：资源效率高

- **无 GIL 竞争问题**（单线程模型）
- 没有锁、信号量等复杂同步原语的负担
- 上下文切换由程序员控制（在 `await` 处切换）
- 更少的系统资源消耗

```python
# 不需要锁！协程在 await 之间不会被打断
shared_state = {}

async def update_state(key, value):
    # 在 await 之前的代码是原子性的
    old = shared_state.get(key)
    shared_state[key] = value  # 安全！
    await notify_change(key, old, value)
```

---

## 优点四：生态系统成熟

主流异步库：

- **aiohttp** — 异步 HTTP 客户端/服务器
- **FastAPI** — 高性能异步 Web 框架
- **asyncpg** — 异步 PostgreSQL 驱动
- **aioredis** — 异步 Redis 客户端
- **aiomysql** — 异步 MySQL 驱动
- **Uvicorn** — ASGI 服务器
- **httpx** — 同步/异步兼容的 HTTP 客户端

---

## 缺点一：学习曲线陡峭

- 需要理解事件循环、协程调度等概念
- **调试困难**：堆栈跟踪不直观
- 常见陷阱多，新手容易出错

```python
# 陷阱 1：忘记 await
async def bad_example():
    result = some_coroutine()  # 没有 await！
    # result 是一个协程对象，不是结果值

# 陷阱 2：在协程中调用阻塞代码
async def blocking_mistake():
    time.sleep(5)  # 阻塞整个事件循环！
    # 应该用 await asyncio.sleep(5)
```

---

## 缺点二：生态兼容性问题

- **不能直接使用同步库**（如 requests, psycopg2）
- 需要所有 I/O 操作都是异步的（**传染性**）
- 混合同步/异步代码需要特殊处理

```python
# 错误：直接在异步函数中使用同步库
async def wrong():
    response = requests.get(url)  # 阻塞事件循环！

# 正确：使用 run_in_executor 包装
async def correct():
    loop = asyncio.get_event_loop()
    response = await loop.run_in_executor(
        None, requests.get, url
    )
```

---

## 缺点三：不适合 CPU 密集型任务

- 单线程模型无法利用多核 CPU
- CPU 密集计算会阻塞事件循环
- 需要配合**多进程**来解决

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

async def cpu_bound_task(data):
    loop = asyncio.get_event_loop()
    # 将 CPU 密集任务放到进程池
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(
            pool, heavy_computation, data
        )
    return result
```

---

## 缺点四：错误处理复杂

- 未被 await 的异常可能被**静默吞掉**
- Task 异常需要显式处理
- 取消操作需要仔细设计

```python
async def risky():
    task = asyncio.create_task(might_fail())
    # 如果 might_fail 抛出异常且 task 未被 await
    # 异常会被静默丢弃（仅打印警告）

# 正确的异常处理
async def safe():
    task = asyncio.create_task(might_fail())
    try:
        result = await task
    except Exception as e:
        logger.error(f"Task failed: {e}")
```

---

## 最佳实践 1：正确启动事件循环

```python
# ✅ 推荐：使用 asyncio.run()（Python 3.7+）
async def main():
    await do_something()

asyncio.run(main())

# ❌ 避免：手动管理事件循环
loop = asyncio.get_event_loop()
loop.run_until_complete(main())
loop.close()
```

- `asyncio.run()` 自动创建、运行并关闭事件循环
- 每个线程只能有一个运行中的事件循环

---

## 最佳实践 2：合理使用并发原语

```python
# ✅ 使用 asyncio.gather 并发执行多个任务
results = await asyncio.gather(
    fetch(url1), fetch(url2), fetch(url3),
    return_exceptions=True  # 不因单个失败而中断
)

# ✅ 使用 asyncio.Semaphore 限制并发数
semaphore = asyncio.Semaphore(10)

async def limited_fetch(url):
    async with semaphore:
        return await fetch(url)

# ✅ 使用 TaskGroup（Python 3.11+）
async with asyncio.TaskGroup() as tg:
    task1 = tg.create_task(coro1())
    task2 = tg.create_task(coro2())
# 所有任务完成后自动退出，异常处理更安全
```

---

## 最佳实践 3：优雅的超时与取消

```python
# ✅ 使用 asyncio.timeout（Python 3.11+）
async def fetch_with_timeout():
    async with asyncio.timeout(10):
        return await fetch(url)

# ✅ 使用 wait_for（兼容旧版本）
try:
    result = await asyncio.wait_for(
        fetch(url), timeout=10.0
    )
except asyncio.TimeoutError:
    logger.warning("Request timed out")

# ✅ 优雅处理取消
async def cancellable_task():
    try:
        while True:
            await do_work()
    except asyncio.CancelledError:
        await cleanup()  # 清理资源
        raise  # 重新抛出，让调用者知道已取消
```

---

## 最佳实践 4：处理阻塞操作

```python
import asyncio
from functools import partial

# ✅ 使用 to_thread（Python 3.9+）
async def read_file():
    content = await asyncio.to_thread(
        Path("large_file.txt").read_text
    )
    return content

# ✅ 使用 run_in_executor（兼容旧版本）
async def legacy_sync_call():
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(
        None,  # 使用默认线程池
        partial(sync_function, arg1, arg2)
    )
    return result
```

**原则：永远不要在协程中直接调用阻塞函数**

---

## 最佳实践 5：结构化并发与资源管理

```python
# ✅ 使用 async with 管理异步资源
async def best_practice():
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            data = await response.json()

# ✅ 使用 async for 处理异步迭代
async def stream_data():
    async for chunk in response.content.iter_chunked(1024):
        process(chunk)

# ✅ 优雅关闭
async def graceful_shutdown(tasks):
    for task in tasks:
        task.cancel()
    results = await asyncio.gather(
        *tasks, return_exceptions=True
    )
```

---

## 最佳实践 6：调试与监控

```python
# ✅ 开启 debug 模式
asyncio.run(main(), debug=True)

# ✅ 设置环境变量
# PYTHONASYNCIODEBUG=1

# ✅ 监控慢回调
import logging
logging.basicConfig(level=logging.DEBUG)
# debug 模式会警告超过 100ms 的回调

# ✅ 使用 asyncio 的内省 API
for task in asyncio.all_tasks():
    print(f"Task: {task.get_name()}, "
          f"State: {task.done()}")
```

---

## 何时使用 asyncio？

| 场景 | 推荐方案 |
|------|---------|
| 大量网络 I/O（爬虫、API） | ✅ asyncio |
| Web 服务器/微服务 | ✅ asyncio (FastAPI) |
| 数据库大量查询 | ✅ asyncio |
| WebSocket 服务 | ✅ asyncio |
| CPU 密集计算 | ❌ 多进程 |
| 简单脚本 | ❌ 同步代码 |
| 少量并发 I/O | ⚠️ 线程可能更简单 |
| 已有大量同步代码 | ⚠️ 渐进迁移 |

---

## 总结

**asyncio 的核心价值：**

1. 高效处理 **I/O 密集型** 并发任务
2. `async/await` 语法优雅，代码可维护
3. 资源消耗低，性能优秀

**需要注意的关键点：**

1. 避免在协程中调用阻塞代码
2. 合理使用 Semaphore 控制并发度
3. 正确处理异常和取消
4. 善用 `asyncio.gather` 和 `TaskGroup`
5. 利用 debug 模式排查问题

> "异步编程不是银弹，但在正确的场景下，它是最佳选择。"
