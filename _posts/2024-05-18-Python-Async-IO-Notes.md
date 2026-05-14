---
layout: post
title: Python Async IO Learning Notes
date:   2024-05-18 16:34:01 +0800
lang: en
permalink: /en/posts/python-async-io-notes/
translation_key: python-async-io-notes
---

> Async is not "faster," but "smarter waiting."

---

## Table of Contents

1. [Why Do We Need Async IO?](#1-why-do-we-need-async-io)
2. [Core Concepts](#2-core-concepts)
3. [asyncio Basics](#3-asyncio-basics)
4. [Tasks and Concurrency](#4-tasks-and-concurrency)
5. [Async Context Managers and Iterators](#5-async-context-managers-and-iterators)
6. [asyncio Synchronization Primitives](#6-asyncio-synchronization-primitives)
7. [Queue: Producer-Consumer Pattern](#7-queue-producer-consumer-pattern)
8. [Async Network IO: aiohttp](#8-async-network-io-aiohttp)
9. [Async File IO: aiofiles](#9-async-file-io-aiofiles)
10. [Async Databases](#10-async-databases)
11. [Calling Async from Sync Code](#11-calling-async-from-sync-code)
12. [Event Loop Internals](#12-event-loop-internals)
13. [Common Pitfalls and Best Practices](#13-common-pitfalls-and-best-practices)
14. [Performance Comparison](#14-performance-comparison)

---

## 1. Why Do We Need Async IO?

### Three Concurrency Models Compared

| Model | Mechanism | Use Case | Drawback |
|-------|-----------|----------|----------|
| **Multiprocessing** | Multiple CPU cores in parallel | CPU-intensive (compute, compression) | High memory, expensive context switches |
| **Multithreading** | OS-level thread scheduling | IO-intensive (with GIL limits) | GIL, race conditions, deadlocks |
| **Async IO** | Single-threaded event loop | IO-intensive (network, files) | Not suitable for CPU-bound work |

### The Core Problem: Wasted Waiting

```
Synchronous mode (one chef cooking one dish):
[Request A sent] → [wait 200ms] → [process] → [Request B sent] → [wait 200ms] → ...
Total time: 200ms × N

Async mode (one chef managing multiple dishes):
[Request A sent] ↘
[Request B sent]  → [waiting...] → [A returns, process] → [B returns, process]
[Request C sent] ↗
Total time: ≈ 200ms (longest single request)
```

**Key insight:** Async IO's essence is **doing other work while waiting for IO**, not true parallel computation.

---

## 2. Core Concepts

### Coroutines

Functions defined with `async def` return a coroutine object when called, **not executing immediately**:

```python
async def say_hello():
    print("Hello")

# Calling doesn't execute! Just creates a coroutine object
coro = say_hello()
print(type(coro))  # <class 'coroutine'>

# Must run via event loop
import asyncio
asyncio.run(say_hello())  # This prints Hello
```

### Event Loop

The "control center" of async programs, responsible for:
- Running coroutines
- Listening for IO events
- Scheduling callbacks

```
┌─────────────────────────────────────┐
│          Event Loop                  │
│                                     │
│  Task A ──► await IO ──► suspended  │
│  Task B ◄── IO done  ◄── wake up    │
│  Task C ──► await IO ──► suspended  │
│  Task A ◄── IO done  ◄── wake up    │
└─────────────────────────────────────┘
```

### The await Keyword

`await` can only be used inside `async def` functions. It:
1. Pauses the current coroutine, yielding control back to the event loop
2. Waits for the awaited object to complete
3. Resumes execution after completion, returning the result

```python
async def fetch():
    # Pauses here; event loop runs other tasks
    result = await some_async_operation()
    # Resumes here after IO completes
    return result
```

### Awaitable Objects

Three types of objects can be awaited:

```python
# 1. Coroutine
async def coro(): ...
await coro()

# 2. Task (created via asyncio.create_task)
task = asyncio.create_task(coro())
await task

# 3. Future (low-level, rarely used directly)
future = asyncio.get_event_loop().create_future()
await future
```

---

## 3. asyncio Basics

### asyncio.run(): Program Entry Point

```python
import asyncio

async def main():
    print("Start")
    await asyncio.sleep(1)  # Simulate IO wait
    print("Done")

# Python 3.7+ standard entry point
asyncio.run(main())
```

### asyncio.sleep()

Async version of `time.sleep()`, **doesn't block the event loop** during wait:

```python
import asyncio
import time

async def task(name, delay):
    print(f"{name} start")
    await asyncio.sleep(delay)  # ✅ Async wait
    print(f"{name} done")

async def main():
    start = time.time()
    # Sequential execution (total time = 1 + 2 = 3s)
    await task("A", 1)
    await task("B", 2)
    print(f"Elapsed: {time.time() - start:.1f}s")

asyncio.run(main())
# Elapsed: 3.0s
```

---

## 4. Tasks and Concurrency

### asyncio.create_task(): True Concurrency

```python
import asyncio
import time

async def task(name, delay):
    print(f"{name} start")
    await asyncio.sleep(delay)
    print(f"{name} done")
    return f"{name}'s result"

async def main():
    start = time.time()

    # Create tasks, scheduled immediately (but not yet running)
    task_a = asyncio.create_task(task("A", 1))
    task_b = asyncio.create_task(task("B", 2))
    task_c = asyncio.create_task(task("C", 1))

    # Wait for all tasks
    result_a = await task_a
    result_b = await task_b
    result_c = await task_c

    print(f"Elapsed: {time.time() - start:.1f}s")
    # Elapsed: 2.0s (concurrent; total = longest single task)

asyncio.run(main())
```

### asyncio.gather(): Bulk Concurrency

Most common concurrent pattern—waits for all coroutines and collects results:

```python
async def fetch(url):
    await asyncio.sleep(1)  # Simulate network request
    return f"Data from {url}"

async def main():
    urls = ["url1", "url2", "url3", "url4", "url5"]

    # Execute all requests concurrently
    results = await asyncio.gather(
        *[fetch(url) for url in urls]
    )

    for r in results:
        print(r)

asyncio.run(main())
# 5 requests sent concurrently; total time ≈ 1s
```

**Exception handling in gather:**

```python
# Default: any exception raises immediately
results = await asyncio.gather(coro1(), coro2(), coro3())

# return_exceptions=True: exceptions returned as results, no interruption
results = await asyncio.gather(
    coro1(), coro2(), coro3(),
    return_exceptions=True
)
for r in results:
    if isinstance(r, Exception):
        print(f"Error: {r}")
    else:
        print(f"Success: {r}")
```

### asyncio.wait(): Fine-Grained Control

```python
import asyncio

async def task(n):
    await asyncio.sleep(n)
    return n

async def main():
    tasks = [asyncio.create_task(task(i)) for i in [3, 1, 2]]

    # Continue when first completes
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )

    for t in done:
        print(f"Done: {t.result()}")

    # Cancel remaining
    for t in pending:
        t.cancel()

asyncio.run(main())
```

| `return_when` Parameter | Meaning |
|--------------------------|---------|
| `ALL_COMPLETED` (default) | Return when all complete |
| `FIRST_COMPLETED` | Return when first completes |
| `FIRST_EXCEPTION` | Return when first fails |

### asyncio.timeout(): Timeout Control (Python 3.11+)

```python
async def main():
    try:
        async with asyncio.timeout(5.0):
            await long_running_task()
    except TimeoutError:
        print("Timed out!")

# Python 3.10 and below: use wait_for
async def main():
    try:
        result = await asyncio.wait_for(
            long_running_task(),
            timeout=5.0
        )
    except asyncio.TimeoutError:
        print("Timed out!")
```

---

## 5. Async Context Managers and Iterators

### Async Context Managers

Implement `__aenter__` and `__aexit__`:

```python
class AsyncDBConnection:
    async def __aenter__(self):
        print("Connecting to database")
        self.conn = await create_connection()
        return self.conn

    async def __aexit__(self, exc_type, exc, tb):
        print("Closing connection")
        await self.conn.close()

async def main():
    async with AsyncDBConnection() as conn:
        await conn.execute("SELECT 1")
```

### Async Iterators

Implement `__aiter__` and `__anext__`:

```python
class AsyncRange:
    def __init__(self, n):
        self.n = n
        self.i = 0

    def __aiter__(self):
        return self

    async def __anext__(self):
        if self.i >= self.n:
            raise StopAsyncIteration
        await asyncio.sleep(0.1)  # Simulate async operation
        val = self.i
        self.i += 1
        return val

async def main():
    async for num in AsyncRange(5):
        print(num)
```

### Async Generators

Cleaner syntax:

```python
async def async_range(n):
    for i in range(n):
        await asyncio.sleep(0.1)
        yield i

async def main():
    async for num in async_range(5):
        print(num)

    # Also supports async list comprehension
    results = [i async for i in async_range(5)]
```

---

## 6. asyncio Synchronization Primitives

When multiple coroutines share resources, synchronization is needed (like locks in multithreading).

### Lock: Mutual Exclusion

```python
import asyncio

lock = asyncio.Lock()
shared_resource = 0

async def worker(name):
    global shared_resource
    async with lock:  # Acquire lock; others wait
        val = shared_resource
        await asyncio.sleep(0.1)  # Simulate work
        shared_resource = val + 1
        print(f"{name}: {shared_resource}")

async def main():
    await asyncio.gather(*[worker(f"W{i}") for i in range(5)])

asyncio.run(main())
```

### Event: Signal Notification

```python
async def waiter(event, name):
    print(f"{name} waiting...")
    await event.wait()
    print(f"{name} received signal, continuing")

async def setter(event):
    await asyncio.sleep(2)
    print("Triggering event!")
    event.set()

async def main():
    event = asyncio.Event()
    await asyncio.gather(
        waiter(event, "W1"),
        waiter(event, "W2"),
        setter(event)
    )
```

### Semaphore: Limit Concurrent Tasks

Most commonly used! Controls max concurrent running coroutines:

```python
async def fetch(session, url, semaphore):
    async with semaphore:  # Max 10 concurrent
        async with session.get(url) as resp:
            return await resp.text()

async def main():
    semaphore = asyncio.Semaphore(10)  # Max concurrency: 10
    urls = [f"https://example.com/{i}" for i in range(100)]

    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url, semaphore) for url in urls]
        results = await asyncio.gather(*tasks)
```

---

## 7. Queue: Producer-Consumer Pattern

```python
import asyncio

async def producer(queue, n):
    for i in range(n):
        await asyncio.sleep(0.5)  # Simulate production
        await queue.put(i)
        print(f"Produced: {i}, queue size: {queue.qsize()}")
    await queue.put(None)  # End signal

async def consumer(queue, name):
    while True:
        item = await queue.get()
        if item is None:
            queue.task_done()
            break
        await asyncio.sleep(1)  # Simulate consumption
        print(f"[{name}] Consumed: {item}")
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=5)  # Max capacity: 5

    await asyncio.gather(
        producer(queue, 10),
        consumer(queue, "Consumer A"),
        consumer(queue, "Consumer B"),
    )

asyncio.run(main())
```

---

## 8. Async Network IO: aiohttp

```bash
pip install aiohttp
```

### Basic GET Request

```python
import asyncio
import aiohttp

async def fetch(session, url):
    async with session.get(url) as response:
        return await response.json()

async def main():
    async with aiohttp.ClientSession() as session:
        data = await fetch(session, "https://api.github.com/users/octocat")
        print(data["name"])

asyncio.run(main())
```

### Concurrent URL Fetching

```python
import asyncio
import aiohttp
import time

async def fetch(session, url):
    try:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as resp:
            return {"url": url, "status": resp.status, "data": await resp.text()}
    except Exception as e:
        return {"url": url, "error": str(e)}

async def main():
    urls = [
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/2",
        "https://httpbin.org/delay/1",
    ]

    start = time.time()
    async with aiohttp.ClientSession() as session:
        results = await asyncio.gather(*[fetch(session, url) for url in urls])

    print(f"Elapsed: {time.time() - start:.1f}s")  # ≈ 2s, not 4s
    for r in results:
        print(r["url"], r.get("status", r.get("error")))

asyncio.run(main())
```

### POST Request

```python
async def post_data(session, url, payload):
    async with session.post(url, json=payload) as resp:
        return await resp.json()

async def main():
    async with aiohttp.ClientSession() as session:
        result = await post_data(
            session,
            "https://httpbin.org/post",
            {"key": "value", "name": "asyncio"}
        )
        print(result)
```

---

## 9. Async File IO: aiofiles

Standard `open()` is synchronous and blocks the event loop. Use `aiofiles`:

```bash
pip install aiofiles
```

```python
import asyncio
import aiofiles

async def read_file(path):
    async with aiofiles.open(path, "r", encoding="utf-8") as f:
        return await f.read()

async def write_file(path, content):
    async with aiofiles.open(path, "w", encoding="utf-8") as f:
        await f.write(content)

async def main():
    # Concurrently read multiple files
    contents = await asyncio.gather(
        read_file("file1.txt"),
        read_file("file2.txt"),
        read_file("file3.txt"),
    )
    for content in contents:
        print(content[:100])

asyncio.run(main())
```

---

## 10. Async Databases

### asyncpg (PostgreSQL)

```bash
pip install asyncpg
```

```python
import asyncio
import asyncpg

async def main():
    # Create connection
    conn = await asyncpg.connect(
        host="localhost", database="mydb",
        user="user", password="password"
    )

    # Query
    rows = await conn.fetch("SELECT id, name FROM users WHERE active = $1", True)
    for row in rows:
        print(row["id"], row["name"])

    # Insert
    await conn.execute(
        "INSERT INTO users(name, email) VALUES($1, $2)",
        "John Doe", "john@example.com"
    )

    await conn.close()

asyncio.run(main())
```

### Connection Pool (Production Required)

```python
async def main():
    pool = await asyncpg.create_pool(
        host="localhost", database="mydb",
        user="user", password="password",
        min_size=5, max_size=20
    )

    async def query(user_id):
        async with pool.acquire() as conn:
            return await conn.fetchrow("SELECT * FROM users WHERE id = $1", user_id)

    # Concurrent queries
    results = await asyncio.gather(*[query(i) for i in range(1, 11)])

    await pool.close()
```

### aiosqlite (SQLite)

```bash
pip install aiosqlite
```

```python
import aiosqlite

async def main():
    async with aiosqlite.connect("test.db") as db:
        await db.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY,
                name TEXT
            )
        """)
        await db.execute("INSERT INTO users(name) VALUES(?)", ("John Doe",))
        await db.commit()

        async with db.execute("SELECT * FROM users") as cursor:
            async for row in cursor:
                print(row)
```

---

## 11. Calling Async from Sync Code

Sometimes you need to call async functions from sync code (regular functions, Django views, etc.):

### asyncio.run() (Simplest)

```python
def sync_function():
    result = asyncio.run(async_function())
    return result
```

> **Note:** `asyncio.run()` creates a new event loop and **cannot be called inside an existing event loop**.

### loop.run_until_complete()

```python
def sync_function():
    loop = asyncio.new_event_loop()
    try:
        result = loop.run_until_complete(async_function())
    finally:
        loop.close()
    return result
```

### Running Sync Blocking Functions in Async

For CPU-intensive or legacy blocking code, use `run_in_executor` with thread pool:

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

def blocking_io():
    import time
    time.sleep(2)      # Sync blocking operation
    return "done"

def cpu_heavy():
    return sum(i * i for i in range(10**7))

async def main():
    loop = asyncio.get_event_loop()

    # Run in thread pool (for IO-blocking)
    result = await loop.run_in_executor(None, blocking_io)

    # Run in process pool (for CPU-heavy)
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy)
```

---

## 12. Event Loop Internals

### Getting the Current Event Loop

```python
async def main():
    loop = asyncio.get_event_loop()       # Get current loop
    loop = asyncio.get_running_loop()     # Inside coroutine (recommended)
```

### Scheduling Callbacks

```python
async def main():
    loop = asyncio.get_running_loop()

    # Execute next iteration (regular function, not coroutine)
    loop.call_soon(print, "Execute immediately")

    # Delayed execution
    loop.call_later(2.0, print, "Execute after 2 seconds")

    # Execute at timestamp
    loop.call_at(loop.time() + 5.0, print, "Execute after 5 seconds")

    await asyncio.sleep(3)
```

### uvloop: High-Performance Event Loop

C-based event loop implementation; 2-4x faster than default:

```bash
pip install uvloop
```

```python
import uvloop

asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
asyncio.run(main())
```

---

## 13. Common Pitfalls and Best Practices

### ❌ Pitfall 1: Using Sync Blocking Calls in Coroutines

```python
import time
import requests  # Sync library

# ❌ Wrong: Blocks entire event loop
async def bad_fetch(url):
    time.sleep(1)              # Blocks!
    return requests.get(url)   # Blocks!

# ✅ Correct: Use async library
async def good_fetch(url):
    await asyncio.sleep(1)
    async with aiohttp.ClientSession() as s:
        async with s.get(url) as r:
            return await r.text()
```

### ❌ Pitfall 2: Forgetting await

```python
# ❌ Wrong: coro() just creates coroutine object, never runs
async def main():
    result = some_async_func()   # Missing await!
    print(result)                # Prints coroutine object

# ✅ Correct
async def main():
    result = await some_async_func()
    print(result)
```

Python will emit: `RuntimeWarning: coroutine 'xxx' was never awaited`

### ❌ Pitfall 3: Sequential await Instead of Concurrent

```python
# ❌ Wrong: Sequential, no concurrency
async def main():
    r1 = await fetch("url1")  # Wait this
    r2 = await fetch("url2")  # Then this
    r3 = await fetch("url3")  # Total time = sum of all
    # Total time: 3×delay

# ✅ Correct: True concurrency
async def main():
    r1, r2, r3 = await asyncio.gather(
        fetch("url1"),
        fetch("url2"),
        fetch("url3"),
    )
    # Total time: max(delays)
```

### ❌ Pitfall 4: Calling asyncio.run() Inside Existing Event Loop

```python
# ❌ Fails in Jupyter Notebook or existing event loop
asyncio.run(main())

# ✅ Jupyter: directly await
await main()

# Or use nest_asyncio
import nest_asyncio
nest_asyncio.apply()
asyncio.run(main())
```

### ✅ Best Practices Summary

```python
# 1. Reuse session; don't create per request
async with aiohttp.ClientSession() as session:
    results = await asyncio.gather(*[fetch(session, url) for url in urls])

# 2. Use Semaphore to limit concurrency, prevent overwhelming server
sem = asyncio.Semaphore(20)
async def safe_fetch(url):
    async with sem:
        return await fetch(url)

# 3. Always handle exceptions; prevent silent task failures
results = await asyncio.gather(*tasks, return_exceptions=True)

# 4. Set timeouts
await asyncio.wait_for(coro(), timeout=30)

# 5. Cancel unwanted tasks
task = asyncio.create_task(coro())
task.cancel()
try:
    await task
except asyncio.CancelledError:
    pass  # Normal cancellation
```

---

## 14. Performance Comparison

### Experiment: Fetching 50 URLs

```python
import asyncio
import aiohttp
import requests
import time

URLS = [f"https://httpbin.org/delay/1" for _ in range(10)]

# ① Sync version
def sync_main():
    start = time.time()
    results = [requests.get(url).status_code for url in URLS]
    print(f"Sync elapsed: {time.time() - start:.1f}s")

# ② Async version
async def async_main():
    start = time.time()
    async with aiohttp.ClientSession() as session:
        async def fetch(url):
            async with session.get(url) as r:
                return r.status
        results = await asyncio.gather(*[fetch(url) for url in URLS])
    print(f"Async elapsed: {time.time() - start:.1f}s")

# ③ Rate-limited async
async def async_limited():
    sem = asyncio.Semaphore(5)
    start = time.time()
    async with aiohttp.ClientSession() as session:
        async def fetch(url):
            async with sem:
                async with session.get(url) as r:
                    return r.status
        results = await asyncio.gather(*[fetch(url) for url in URLS])
    print(f"Rate-limited async elapsed: {time.time() - start:.1f}s")

sync_main()               # Sync elapsed: 10.x s
asyncio.run(async_main()) # Async elapsed: 1.x s
asyncio.run(async_limited()) # Rate-limited: 2.x s
```

| Method | Time (10 requests, 1s delay each) |
|--------|----------------------------------|
| Sync | ≈ 10s |
| Async (unlimited) | ≈ 1s |
| Async (rate-limited 5) | ≈ 2s |

---

## Appendix: Quick Reference

```python
# Run coroutine
asyncio.run(coro())

# Concurrent wait, collect results
await asyncio.gather(coro1(), coro2(), coro3())

# Create task (schedule immediately)
task = asyncio.create_task(coro())

# Timeout
await asyncio.wait_for(coro(), timeout=5)

# Wait for first completion
done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)

# Rate limiting
sem = asyncio.Semaphore(10)
async with sem: ...

# Sync to async (thread pool)
await loop.run_in_executor(None, sync_func, arg1, arg2)

# Get event loop
loop = asyncio.get_running_loop()

# Async sleep
await asyncio.sleep(1)
```

---
