# 第 1 章 协程

协程（Coroutine）是Python中处理高并发场景的强大工具，尤其在 I/O 密集型任务中表现出色。协程是一种用户态的轻量级线程，它不像线程那样由操作系统调度，而是由程序自身控制，所以也叫微线程。

## 1.1 线程、进程和协程的对比

| **机制**        | **调度者**            | **切换开销**     | **资源共享**        | **适用场景**          | **核心优势**          | 局限性                                    |
| --------------- | --------------------- | ---------------- | ------------------- | --------------------- | --------------------- | ----------------------------------------- |
| 进程(process)   | 操作系统              | 极大  (内核级)   | 完全隔离            | CPU密集型             | 利用多核              | 资源消耗大，数量有限。                    |
| 线程 (thread)   | 操作系统              | 较大  (内核级)   | 共享内存 (需加锁)   | I/O 密集型 (低并发)   | 共享地址空间          | 受GIL限制，且数量过多时调度和同步开销大。 |
| 协程(coroutine) | 程序自身   Event Loop | 极小(函数调用级) | 共享内存 (无需加锁) | I/O 密集型 (极高并发) | 极高并发，高性能I/O。 | 需要与其配套的异步库或者线程池处理阻塞    |

## 1.2 线程处理IO密集操作的不足

在处理大量的 I/O 密集型任务（如Web服务要同时处理数千个用户连接）时，线程模型会遇到以下瓶颈：

- 切换开销大： 操作系统在数千个线程之间切换需要消耗大量的 CPU 时间。
- 同步复杂： 线程共享内存，需要使用锁（Lock）来保护共享数据，这增加了程序的复杂性，并可能导致死锁。
- 阻塞浪费： 当一个线程执行到网络等待时，它会阻塞并被挂起。虽然操作系统会切换到另一个线程，但线程自身的切换成本仍然很高，而且往往无法创建足够多的线程来应对极高并发。

## 1.3 协程的优势

- 调度者：协程由程序自身进行调用，主动让出CPU给其他协程（通过 await 关键字）
- 轻量级：创建成本极低，一个线程程可以轻松创建数万个协程
- 无锁机制：同一时间只有一个协程运行，无需锁保护共享资源
- 与线程相比，协程避免了操作系统级别的上下文切换开销，只有极小的函数调用开销，在处理大量 I/O 操作时效率更高

## 1.4 定义协程函数：async def

使用`async def`关键字（async全称 **asynchronous** /eɪˈsɪŋkrənəs/（异步的），日常编程里都简写读 **async** /əˈsɪŋk/ ）定义的函数就是协程函数，它返回一个协程对象。

```python
import asyncio

# 这是一个协程函数
async def hello():
    print("Hello")
    print("World")

# 调用 hello() 不会立即执行，只会返回一个协程对象
coro_obj = hello()
print(type(coro_obj)) # 输出: <class 'coroutine'>
```

## 1.5 事件循环(Event Loop) 详述

事件循环是协程并发模型的核心，它是负责调度、管理和执行所有协程的“心脏”。其工作流程如下：

### 1.5.1 注册任务 

注册后，协程就被注册到事件循环中等待执行，如下3种情况都会实现注册。

**（1）`asyncio.run(main_coro)`(顶级注册，启动程序)**

目的：启动整个异步程序。

机制：asyncio.run()负责创建EventLoop，并将传入的顶级协程main_coro隐式地包装成一个Task并运行。这个Task成为整个EventLoop的主要任务。

用途：作为程序的入口点。

**（2）asyncio.create_task(coro)(最常用，实现并发)**

目的：实现并发执行。

机制：将协程对象（coroutine object）立即包装成一个Task，并将其添加到EventLoop的队列中，使其可以立即开始在后台运行，而调用者无需等待。

用途：启动多个任务，然后通过asyncio.gather()等待它们的结果。

**（3）await coro(隐式注册，实现串行)**

目的：实现串行等待。

机制：当EventLoop正在运行一个协程A，协程A遇到await coro_B时，EventLoop内部会隐式地将coro_B注册为一个任务，并开始执行它。但此时EventLoop必须等待coro_B完成，才能让A协程恢复执行。

用途：确保代码按顺序执行。

### 1.5.2 启动协程

事件循环开始运行，选择一个就绪的协程开始执行。

一般是在程序的同步部分（通常是`if__name__=="__main__":`）调用`asyncio.run(主协程对象)`。asyncio.run()负责实例化一个EventLoop对象，并将传入的`主协程对象`包装成一个Task，并将其添加到EventLoop中。然后调用EventLoop对象的run_until_complete()方法。此时，EventLoop进入一个永不停止的whileTrue循环，选择第一批就绪任务（即主协程任务）开始执行。

### 1.5.3 遇到 await

协程执行到 await 关键字，表示它即将开始一个耗时的 I/O 等待。让出控制权： 协程主动暂停（挂起），将 I/O 操作注册到事件循环的监控中，然后将控制权交还给事件循环。

### 1.5.4 切换任务

事件循环发现当前协程暂停了，它会立即选择下一个处于“就绪”状态的协程继续执行。

### 1.5.5 I/O完成通知

当 I/O 操作（例如网络连接等待耗时3秒）完成后，操作系统会通知事件循环：“这个 I/O 已经准备好了！”

### 1.5.6 恢复协程

事件循环将先前挂起的协程重新标记为“就绪”，并在合适的时机让它从上次 await 的地方恢复执行。

### 1.5.7 终止与清理

EventLoop持续运行上述1-6步。

如果EventLoop发现它的顶级Task(main)已经完成，它立即触发退出机制。

此时EventLoop强制取消所有仍在运行的子Task，然后关闭EventLoop，程序退出。

### 1.5.8 事件循环伪代码模型

```python
# 全局数据结构
所有任务 = []       # 存放所有需要运行的协程任务
就绪队列 = []       # 可以马上执行的任务
等待IO的任务 = {}   # 正在等网络/文件的任务 {IO事件: 任务}

while True:
    # ==========================================
    # 第一步：检查有没有 IO 完成了
    # ==========================================
    已完成的IO事件 = 操作系统监听IO()  # 比如：socket收到数据了

    # 把“刚完成IO”的任务，从等待 → 放回就绪队列
    for 事件 in 已完成的IO事件:
        任务 = 等待IO的任务.pop(事件)
        就绪队列.append(任务)

    # ==========================================
    # 第二步：执行所有【就绪】的任务
    # ==========================================
    for 任务 in 就绪队列:
        结果 = 任务.run()  # 执行，直到遇到 await 停下来

        if 任务.完成了:
            所有任务.remove(任务)  # 任务结束，删掉
        else:
            # 遇到 await，任务需要等 IO → 移到等待区
            等待的IO事件 = 任务.当前等待的IO()
            等待IO的任务[等待的IO事件] = 任务

    # 执行完一轮，清空就绪队列
    就绪队列.clear()

    # ==========================================
    # 第三步：结束循环的条件
    # ==========================================
    if not 所有任务:
        break
```

## 1.6 运行协程

### 1.6.1 多协程任务串行

```python
import asyncio
import time

async def my_func(name, delay):
    print(f"任务 {name} 开始")
    await asyncio.sleep(delay)
    print(f"任务 {name} 完成")
    return f"任务 {name} 结果"

async def main_sync():
    print("--- 串行执行---")
    await my_func("A", 1)
    await my_func("B", 2)

if __name__ == "__main__":
    start_time = time.time()
    asyncio.run(main_sync())
    end_time = time.time()
    print(f"串行耗时: {end_time - start_time:.2f}秒")

```

```
--- 串行执行---
注册任务A
任务 A 开始
任务 A 完成
注册任务B
任务 B 开始
任务 B 完成
串行耗时: 3.02秒
```



### 1.6.2 多协程任务并发

```python
import asyncio
import time

async def my_func(name, delay):
    print(f"任务 {name} 开始...")
    await asyncio.sleep(delay)
    print(f"任务 {name} 完成.")
    return f"结果 of {name}"

async def main_concurrent():
    print("--- 并发执行---")
    print("注册任务C")
    asyncio.create_task(my_func("C", 1))
    print("注册任务D")
    asyncio.create_task(my_func("D", 2))

    #必须加这个，否则如果EventLoop发现它的顶级Task(main_concurrent)完成了，它立即触发退出机制。
    #此时EventLoop强制取消所有仍在运行的子Task，然后关闭EventLoop，程序退出。
    await asyncio.sleep(2)#后面换成其他写法

if __name__ == "__main__":
    start_time = time.time()
    asyncio.run(main_concurrent())
    print(f"并发总耗时: {time.time() - start_time:.2f}秒")

```

```
--- 并发执行---
注册任务C
注册任务D
任务 C 开始...
任务 D 开始...
任务 C 完成.
任务 D 完成.
并发总耗时: 2.01秒
```



### 1.6.3 顶级Task退出，事件循环终止

```python
import asyncio
import time

async def my_func(name, delay):
    print(f"任务 {name} 开始...")
    await asyncio.sleep(delay)
    print(f"任务 {name} 完成.")
    return f"结果 of {name}"

async def main_concurrent():
    print("--- 并发执行---")
    print("注册任务C")
    asyncio.create_task(my_func("C", 1))
    print("注册任务D")
    asyncio.create_task(my_func("D", 2))
    print("顶级Task任务完成，退出")

if __name__ == "__main__":
    start_time = time.time()
    asyncio.run(main_concurrent())

    print(f"并发总耗时: {time.time() - start_time:.2f}秒")

```

```
--- 并发执行---
注册任务C
注册任务D
顶级Task任务完成，退出
任务 C 开始...
任务 D 开始...
并发总耗时: 0.00秒
```



### 1.6.4 批量并发执行：Task与asyncio.gather()

协程本身只是一个“可暂停的函数”，要实现并发，必须将其包装成任务(Task)并提交给事件循环。Task是对协程的封装，它自动将协程注册到事件循环中，使其具备被调度和执行的能力。

- 创建：使用asyncio.create_task(协程对象)。
- 作用：一旦Task被创建，它就立即开始在后台运行（事件循环会调度它）

asyncio.gather()是用于并发执行多个协程或任务的常用工具。它等待所有传入的任务都完成后，返回它们的结果。

```python
import asyncio
import time

async def my_func(name, delay):
    print(f"任务 {name} 开始...")
    await asyncio.sleep(delay)
    print(f"任务 {name} 完成.")
    return f"结果 of {name}"

async def main_concurrent():
    print("--- 并发执行---")
    print("注册任务C")
    task_c = asyncio.create_task(my_func("C", 1))
    print("注册任务D")
    task_d= asyncio.create_task(my_func("D", 2))

    #await会让main_concurrent协程挂起，直到所有子协程任务完成，并返回结果
    #等待所有传入的任务都完成后，返回它们的结果。
    results = await asyncio.gather(task_c, task_d)
    print(f"所有结果: {results}")

if __name__ == "__main__":
    start_time = time.time()
    asyncio.run(main_concurrent())

    print(f"并发总耗时: {time.time() - start_time:.2f}秒")

```

```
--- 并发执行---
注册任务C
注册任务D
任务 C 开始...
任务 D 开始...
任务 C 完成.
任务 D 完成.
所有结果: ['结果 of C', '结果 of D']
并发总耗时: 2.01秒
```

### 1.6.5 高级控制与协程的限制

#### 1、隔离阻塞操作

如果协程代码中包含阻塞 I/O（如 `time.sleep()`、`requests.get()` 等同步库调用），整个事件循环和所有其他协程都会被卡住，并发能力将失效。

解决方案：隔离阻塞操作

使用 `asyncio.to_thread()`（Python 3.9+）或 `loop.run_in_executor()` 将阻塞代码交给线程池或进程池去执行，从而隔离阻塞，避免卡住主事件循环。

```python
import asyncio
import time

async def my_func(name, delay):
    print(f"任务 {name} 开始...")
    # await time.sleep(10)
    # await asyncio.to_thread(time.sleep, 10)
    loop = asyncio.get_running_loop() #获取事件循环
    await loop.run_in_executor(None,time.sleep,10) #None=默认线程池
    print(f"任务 {name} 完成.")
    return f"结果 of {name}"

async def main_concurrent():
    print("--- 并发执行---")
    print("注册任务C")
    task_c = asyncio.create_task(my_func("C", 1))
    print("注册任务D")
    task_d= asyncio.create_task(my_func("D", 2))

    #await会让main_concurrent协程挂起，直到所有子协程任务完成，并返回结果
    #等待所有传入的任务都完成后，返回它们的结果。
    results = await asyncio.gather(task_c, task_d)
    print(f"所有结果: {results}")

if __name__ == "__main__":
    start_time = time.time()
    asyncio.run(main_concurrent())

    print(f"并发总耗时: {time.time() - start_time:.2f}秒")

```

#### 2、超时处理：asyncio.wait_for()

用于为单个可等待对象设置明确的超时时间。

```python
import asyncio
import time

async def my_func(name, delay):
    print(f"任务 {name} 开始...")
    await asyncio.sleep(delay)
    print(f"任务 {name} 完成.")
    return f"结果 of {name}"

async def main_concurrent():
    print("--- 并发执行---")
    print("注册任务C")
    task_c = asyncio.wait_for(my_func("C", 3), 2) #任务必须在2秒内完成
    print("注册任务D")
    task_d= asyncio.wait_for(my_func("D", 2),2)#任务必须在2秒内完成
    try:
        #await会让main_concurrent协程挂起，直到所有子协程任务完成，并返回结果
        #等待所有传入的任务都完成后，返回它们的结果。
        results = await asyncio.gather(task_c, task_d)
        print(f"所有结果: {results}")
    except asyncio.TimeoutError:
        print("任务超时！")

if __name__ == "__main__":
    start_time = time.time()
    asyncio.run(main_concurrent())

    print(f"并发总耗时: {time.time() - start_time:.2f}秒")
```

### 3、任务取消：Task.cancel()

用于请求取消一个正在运行的Task。被取消的Task会抛出asyncio.CancelledError异常，协程应捕获此异常进行资源清理。

```python
import asyncio
import time

async def my_func(name, delay):
    try:
        print(f"任务 {name} 开始...")
        await asyncio.sleep(delay)
        print(f"任务 {name} 完成.")
        return f"结果 of {name}"
    except asyncio.CancelledError as e:
        print(e)

async def main_concurrent():
    print("--- 并发执行---")
    print("注册任务C")
    task_c = asyncio.create_task(my_func("C", 10))
    print("注册任务D")
    task_d= asyncio.create_task(my_func("D", 2))

    await asyncio.sleep(5)
    task_c.cancel("任务C被取消了")

    results = await asyncio.gather(task_c, task_d)
    print(f"所有结果: {results}")

if __name__ == "__main__":
    start_time = time.time()
    asyncio.run(main_concurrent())

    print(f"并发总耗时: {time.time() - start_time:.2f}秒")
```



## 1.7 实战案例：异步网络请求

协程最适合的场景是 I/O 密集型任务，网络请求是典型代表。aiohttp（异步 HTTP 客户端）

注意：with和async with

```python
#之前读取文件代码
with open("file.txt",encoding="UTF-8") as f:     # 打开文件是同步操作
    content = f.read()           # 读取也是同步
```

而以下操作**都是异步操作**，需要配合 `await` 使用，而 普通`with` 无法支持异步上下文管理。

- aiohttp.ClientSession()：创建连接池需要异步
- session.get()：网络请求是异步
- response.text()：网络读取响应内容是异步

`async with` 允许在进入/退出上下文时执行异步操作（如连接、清理资源），这是 `with` 无法做到的。在异步 I/O 密集型的网络请求中，必须使用 `async with`。如果强行使用 `with` 会抛出 `TypeError`，因为 `ClientResponse` 只实现了异步上下文协议（`__aenter__/__aexit__`），没有同步版本。

```python
import asyncio
import aiohttp
import time

# 需要安装 aiohttp: pip install aiohttp
async def fetch_url(session, url):
    """异步请求一个URL并返回结果"""
    start_time = time.time()
    # async with 也是一种 awaitable 对象
    async with session.get(url) as response:
        # 等待响应体内容
        content = await response.text()
        response_time = time.time() - start_time
        # 返回：URL、耗时、状态码、响应内容前200字符（避免太长）
        return {
            "url": url,
            "time": f"{response_time:.2f}秒",
            "status": response.status,  # 状态码 200=成功
            "content_preview": content[:200] + "..."  # 只展示前200个字符
        }

async def main():
    urls = [
        "https://www.baidu.com",
        "https://www.atguigu.com"
    ]

    async with aiohttp.ClientSession() as session:
        # 并发请求：总耗时约为单个最慢请求的时间
        print("\n--- 并发请求 ---")
        start_time = time.time()
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        for result in results:
            print(result)
        end_time = time.time()
        print(f"并发总耗时: {end_time - start_time:.2f}秒")


if __name__ == "__main__":
    asyncio.run(main())
```

# 第 2章 WSGI和ASGI

WSGI和ASGI是Python Web开发中两个重要的接口规范，用于定义Web服务器与 Python Web应用之间的通信规则，二者的核心区别在于对异步的支持能力。

## 2.1 核心定位与目标

WSGI（Web Server Gateway Interface）是 Python 最早的 Web 服务器与应用接口规范（2003 年提出），**仅支持同步操作，**主要解决早期 Python Web 框架（如 Flask、Django 旧版本）与服务器的兼容性问题，让同一应用可以运行在不同的 WSGI 服务器上（如 Gunicorn、uWSGI）。

ASGI（Asynchronous Server Gateway Interface）是 WSGI 的异步升级版（2018 年提出），原生支持异步操作，同时兼容 WSGI。设计目标是解决 WSGI 无法高效处理异步任务（如 WebSocket、长轮询）的问题，为 FastAPI、Starlette 等异步框架提供标准接口。

## 2.2 工作流程差异

1、WSGI 工作流程：

客户端发送请求 → WSGI 服务器接收 → 同步调用应用的 application(environ, start_response) 函数 → 应用处理后通过 start_response 返回响应 → 服务器转发响应。

整个过程是同步阻塞的，一个请求未处理完时，对应的线程 / 进程无法处理其他请求。

2、ASGI 工作流程：

客户端发送请求 → ASGI 服务器接收 → 将请求封装为事件（如 http.request） → 通过事件循环异步传递给应用 → 应用处理后返回事件（如 http.response） → 服务器转发响应。

等待 I/O 操作（如数据库查询）时，事件循环会切换到其他请求，实现非阻塞处理。

## 1.3 如何选择

- 若使用同步框架（如 Flask、Django 3.0 之前版本），需用WSGI服务器（如 Gunicorn）。
- 若使用异步框架（如 FastAPI、Starlette、Django 3.1+ 异步模式），需用 ASGI 服务器（如 Uvicorn）以发挥异步性能。
- 对于需要 实时通信（如 WebSocket 聊天、实时数据推送）的场景，必须使用 ASGI。

# 第3章 FastAPI

## 3.1 FastAPI介绍

FastAPI 是一个现代、快速（高性能）的Web框架，用于构建API。是建立在Starlette 和Pydantic基础上的。它基于Python 3.7 +的类型提示（type hints）和异步编程（asyncio）能力，使得代码易于编写、阅读和维护。FastAPI 具有自动交互式文档（基于 OpenAPI 规范和 JSON Schema）、数据验证、依赖注入（Dependency Injection）等功能，这些功能使得 API 的开发速度更快、更可靠。

文档： https://fastapi.tiangolo.com 

源码： https://github.com/fastapi/fastapi

## 3.2 第一个FastAPI程序

### 3.2.1 创建新的项目

这里选择直接创建FastAPI项目。如果创建的是纯Python项目，就手动安装fastapi和uvicorn。

```
pip install fastapi #直接创建FastAPI项目的已经安装过了
pip install uvicorn #直接创建FastAPI项目的已经安装过了
```



![image-20260526204610120](images/image-20260526204610120.png)

### 3.2.2 编辑main.py 文件

如果创建的是纯Python项目，需要手动创建main.py。

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def root():
    return {"message": "Hello World"}


@app.get("/hello/{name}")
async def say_hello(name: str):
    return {"message": f"Hello {name}"}

```

你已经创建了一个具有以下功能的 API：

通过 路径 / 和 /hello/{name} 接受 HTTP 请求。以上 路径 都接受 GET 操作（也被称为 HTTP 方法）。/hello/{name} 路径 有一个 路径参数 name用于接收一个str类型的名字。

### 3.2.3 同步或异步

1、同步

如果函数前没有加async，本质上是定义一个普通的同步函数，不能在内部使用 await，所有操作都会阻塞当前线程直到完成。

适用场景：函数内部是纯计算逻辑（无 I/O 操作），或使用的库只有同步实现（如某些旧的数据库驱动）。

执行机制：FastAPI 会将同步函数自动放入线程池执行（默认线程池大小为 os.cpu_count() * 5），因此多个同步请求会被分配到不同线程并行处理，不会完全阻塞所有请求，也就是说，可以同时处理多个请求并发，但是在单个线程上，还是会阻塞的。另外当同步操作耗时过长（如秒级）时，还会因线程池耗尽导致后续请求排队。



2、异步

如果函数前加async def（异步函数），本质上定义一个异步协程（coroutine），支持在函数内部使用 await 调用其他异步操作（如异步数据库查询、异步 HTTP 请求等）。

适用场景：函数内部包含I/O 密集型操作（如网络请求、文件读写、数据库操作等），且这些操作有对应的异步实现（如 asyncpg 异步数据库、aiohttp 异步 HTTP 库）。

执行机制：FastAPI 会通过 Python 的 asyncio 库处理这些异步请求，这种情况下，请求的处理是“真正的异步”。当遇到 await 时，会暂时让出 CPU 控制权，控制权会被交还给事件循环，允许 FastAPI 在当前请求的 I/O 操作（例如，异步数据库查询或 HTTP 请求）等待期间，继续处理其他请求，从而提高并发效率。

 

3、异步和同步函数的选择

异步函数（async def）：FastAPI 会直接在事件循环中运行，遇到 await 时自动切换任务，充分利用异步性能。

同步函数（def）：FastAPI 会自动将其放入一个线程池中执行，避免阻塞事件循环（但本质仍是同步执行，无法并行处理）。

总结：

只要涉及 I/O 操作（如数据库、网络请求），优先用 async def 并配合异步库，发挥 FastAPI的异步优势。

纯计算逻辑或无异步库可用时，用 def 即可（FastAPI 会自动处理线程池，无需手动管理）。



### 3.2.4 运行服务器

#### 1、在pycharm终端中启动uvicorn服务器

通过以下命令运行uvicorn服务器：

```
uvicorn main:app --reload
```

- main：main.py 文件（一个 Python "模块"）。
- app：在 main.py 文件中通过 app = FastAPI() 创建的对象。
- --reload：让服务器在更新代码后重新启动。仅在开发时使用该选项

![image-20260526205235777](images/image-20260526205235777.png)

#### 2、直接通过程序启动uvicorn服务器

在main.py文件中加入如下代码

```python
if __name__ == "__main__":
    # 直接在代码中启动uvicorn服务器
    uvicorn.run(
        app="main:app",       # 指定要运行的FastAPI应用实例
        host="0.0.0.0",  # 允许外部访问（本地可通过127.0.0.1或localhost访问）
        port=8000,     # 端口号
        reload=True    # 开发模式：代码修改后自动重启（生产环境需去掉）
)

```

![image-20260526211523609](images/image-20260526211523609.png)

### 3.2.5 浏览器查看效果

使用浏览器访问 http://127.0.0.1:8000/ 。

你将会看到如下 JSON 响应：

![image-20260526205410551](images/image-20260526205410551.png)

![image-20260526205458666](images/image-20260526205458666.png)

### 3.2.6 交互式API文档

![image-20260526210716759](images/image-20260526210716759.png)

### 3.2.7 程序说明

#### 1、创建FastAPI实例

```
app = FastAPI()
```

这里的变量 `app` 会是 `FastAPI` 类的一个「实例」。

这个实例将是创建你所有 API 的主要交互对象。

#### 2、定义一个路径操作装饰器

```
@app.get("/")
@app.get("/hello/{name}")
```

1）路径

这里的「路径」指的是 URL 中从第一个 / 起的后半部分。例如：https://example.com/items/foo中路径是：/items/foo。「路径」也通常被称为「端点」或「路由」。开发 API 时，「路径」是用来分离「关注点」和「资源」的主要手段。

2）操作

这里的「操作」指的是一种 HTTP「方法」。

最常用的是：POST、GET、PUT、DELETE

其他比较少见的操作有：OPTIONS、HEAD、PATCH、TRACE

在 HTTP 协议中，你可以使用以上的其中一种（或多种）「方法」与每个路径进行通信。在开发 API 时，你通常使用特定的 HTTP 方法去执行特定的行为。通常使用：

- POST：创建数据。
- GET：读取数据。
- PUT：更新数据。
- DELETE：删除数据。

因此，在 OpenAPI 中，每一个 HTTP 方法都被称为「操作」。

3）定义一个路径操作装饰器

告诉 FastAPI 在它下方的函数负责处理如下访问请求操作：@app.get()表示使用 get 操作

你也可以使用其他的操作：@app.post()、@app.put()、@app.delete()

以及更少见的：@app.options()、@app.head()、@app.patch()、@app.trace()

#### 3、定义路径操作函数

以这个函数为例

```python
@app.get("/")
def root():
    return {"message": "Hello World"}

@app.get("/hello/{name}")
def say_hello(name: str):
    return {"message": f"Hello {name}"}
```

路径：/  和 /hello/{name} 

操作： get

函数：是位于装饰器下方（位于 @app.get("/") 和@app.get("/hello/{name}")下方）的函数root和say_hello。

- 每当 FastAPI接收一个使用GET方法访问/的请求时root函数会被调用。
- 每当 FastAPI接收一个使用GET方法访问/hello/name值的请求时say_hello函数会被调用。

![image-20260527120031398](images/image-20260527120031398.png)

#### 4、返回内容

```python
return {"message": "Hello World"}
```

你可以返回一个 dict、list，像 str、int 一样的单个值等，会自动转换为 JSON。



## 3.3 路径参数

### 3.3.1 声明路径参数

FastAPI 支持使用Python字符串格式化语法声明路径参数（变量）

```python
@app.get("/items/{item_id}")
async def read_item_id(item_id):
    return {"item_id": item_id, "id_type": str(type(item_id))} #注意加str()
```

![image-20260528154111389](images/image-20260528154111389.png)

查看API文档：http://127.0.0.1:8000/docs

![image-20260528154313633](images/image-20260528154313633.png)

### 3.3.2 声明路径参数的类型

使用 Python 标准类型注释，声明路径操作函数中路径参数的类型。类型声明将为函数提供错误检查、代码补全等编辑器支持。

```python
@app.get("/items/{item_id}")
async def read_item_int_id(item_id:int):
    return {"item_id": item_id, "id_type": str(type(item_id))}
```

注意，函数接收并返回的值是 3（ int），不是 "3"（str）。

#### 1、类型转换

浏览器中传递的数据都是以字符串的形式进行传递的，FastAPI 通过类型声明自动解析请求中的数据，会自动的将路径参数3转换为函数中声明的int类型。

![image-20260528154809910](images/image-20260528154809910.png)

#### 2、类型校验

通过浏览器访问 http://127.0.0.1:8000/items/atguigu，接收如下 HTTP 错误信息：

![image-20260528154804503](images/image-20260528154804503.png)

这是因为路径参数 item_id 的值 （"atguigu"）的类型不是 int。

值的类型不是int而是浮点数（float）时也会显示同样的错误，比如： http://127.0.0.1:8000/items/4.2 ，可以通过api文档看类型约定。

查看API文档：http://127.0.0.1:8000/docs

![image-20260528154704359](images/image-20260528154704359.png)

### 3.3.3 参数顺序

有时，路径操作中的路径是写死的，比如要使用 `/users/me` 获取当前用户的数据，然后还要使用 `/users/{user_id}`，通过用户 ID 获取指定用户的数据。由于*路径操作*是按顺序依次运行的，因此，一定要在 `/users/{user_id}` 之前声明 `/users/me` 。否则，/users/{user_id} 将匹配 /users/me，FastAPI 会认为正在接收值为 "me" 的 user_id 参数。

```python
@app.get("/users/me")
async def read_user_me():
    return {"user_id": "the current user"}

@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}
```

![image-20260528160820269](images/image-20260528160820269.png)

![image-20260528160837136](images/image-20260528160837136.png)

## 3.4 查询参数

### 3.4.1 url带查询参数

```python
items_list = [{"item1": "Foo"}, {"item2": "Bar"}, {"item3": "Baz"}] #用于数据查询

@app.get("/items/")
async def read_item(start: int, limit: int):
    return items_list[start : start + limit]

```

![image-20260528163340708](images/image-20260528163340708.png)

声明的参数不是路径参数时，路径操作函数会把该参数自动解释为查询参数,按照参数的名字进行匹配。查询字符串是键值对的集合，这些键值对位于URL的?之后，以&分隔。

例如，以下 URL 中：[http://127.0.0.1:8000/items/?start=0&limit=2](http://127.0.0.1:8000/items/?start=0&limit=20)，查询参数为：start：值为 0 ，limit：值为 2

这些值都是URL的组成部分，因此，它们的类型本应是字符串。但声明 Python 类型（上例中为 int）之后，这些值就会转换为声明的类型，并进行类型校验。

如果查询参数没有设置默认值，就表示必选的。如果使用URL[http://127.0.0.1:8000/items](http://127.0.0.1:8000/items)，则会报缺失start和limit的查询参数。

![image-20260528181313835](images/image-20260528181313835.png)

### 3.4.2 查询参数设置默认值

如果查询参数设置了默认值，就表示它是可选的。

```python
@app.get("/items/")
async def read_item(start: int=0, limit: int=10):
    return items_list[start : start + limit]
```

![image-20260528163548383](images/image-20260528163548383.png)

访问 URL：http://127.0.0.1:8000/items/ 与访问以下地址相同：http://127.0.0.1:8000/items/?start=0&limit=10

但如果访问：http://127.0.0.1:8000/items/?start=20，查询参数的值就是：

start=20：在 URL 中设定的值

limit=10：使用默认值



### 3.4.3 可选的查询参数

如果想要表示查询参数是可选的，但是又不知道具体设置什么默认值，那么可以设置为None。

```python
@app.get("/items/{item_id}")
async def read_item(item_id: int, q: str| None = None):
    return {"item_id": item_id, "q": q}
```

![image-20260528164111652](images/image-20260528164111652.png)

![image-20260528164100409](images/image-20260528164100409.png)

本例中，查询参数q是可选的，默认值为 None。FastAPI 可以识别出item_id 是路径参数，q 不是路径参数，而是查询参数。

### 3.4.4 查询参数bool类型转换说明

参数还可以声明为 bool 类型，FastAPI 会自动转换参数类型

```python
@app.get("/items/{item_id}")
async def read_item(item_id: str, q: str | None = None, short: bool = False):
    item = {"item_id": item_id}
    if q:
        item.update({"q": q})
    if not short:
        item.update({"description": "这是描述信息"})
    return item

```

![image-20260528164813818](images/image-20260528164813818.png)

![image-20260528164959624](images/image-20260528164959624.png)

```
short=True
short=on
short=yes
short=1
True,on,yes不区分大小写，TRUE,tRUe等都是True
```

```
short=False
short=off
short=no
short=0
short=   #意味着空字符串
False,off,no不区分大小写形式
```

### 3.4.5 多个路径和查询参数

FastAPI 可以识别同时声明的多个路径参数和查询参数。

- 路径参数：
  - "/users/{user_id}/items/{item_id}" 与 函数参数列表中顺序可以不一致。但是顺序一致，更符合阅读习惯。
  - 甚至路径参数在可以定义在查询参数后面，但是通常都是路径参数在前，查询参数在后。因为路径参数通常没有默认值，查询参数有默认值。
  - 但是在浏览器客户端请求时，路径参数顺序必须与"/users/{user_id}/items/{item_id}" 顺序一致，否则找不到
- 查询参数：
  - 查询参数之间声明顺序随意
  - 在浏览器客户端请求时，查询参数顺序随意，因为FastAPI 通过参数名进行检测。

```python
@app.get("/users/{user_id}/items/{item_id}")
async def read_user_item(user_id: int, item_id: str, q: str | None = None, short: bool = False):
    item = {"item_id": item_id, "owner_id": user_id}
    if q:
        item.update({"q": q})
    if not short:
        item.update({"description": "这是描述信息"})
    return item
```

![image-20260528172157131](images/image-20260528172157131.png)

![image-20260528172633072](images/image-20260528172633072.png)

![image-20260528174344642](images/image-20260528174344642.png)

## 3.5 请求传参

FastAPI 使用请求体从客户端（例如浏览器）向 API 发送数据。使用 [Pydantic](https://docs.pydantic.dev/) 模型声明请求体，能充分利用它的功能和优点。

发送数据使用 POST（最常用）、PUT、DELETE、PATCH 等操作。

```python
# 定义数据模型类，需要继承 BaseModel 的类。
class Item(BaseModel):
    name: str
    desc: str | None = None
    price: float

@app.post("/items/") #注意请求方式是post（post方式请求的数据在地址栏不会显示）
async def create_item(item: Item):
    return item

```

可以通过apidoc查看，请求方式：

![image-20260528182715336](images/image-20260528182715336.png)

![image-20260528182731509](images/image-20260528182731509.png)

![image-20260528182618028](images/image-20260528182618028.png)

![image-20260528182643016](images/image-20260528182643016.png)

如果你使用ApiPost或 Postman 等进行接口测试，要注意FastAPI 默认**只接收 JSON**，如果你用表单格式 form-data 或 x-www-form，会直接 报422错误。

![image-20260528185013891](images/image-20260528185013891.png)

## 3.6 路由分发

当项目规模扩大时，将所有路由写在一个文件中会导致代码臃肿、难以维护。路由分发（也叫路由拆分）是将不同功能模块的路由拆分到不同文件，再通过路由注册的方式整合到主应用中，实现代码模块化。

FastAPI 中实现路由分发的核心工具是 APIRouter，它允许你在子模块中定义路由，再将其挂载到主应用

### 3.6.1 核心组件：APIRouter

- 作用：在子模块中创建独立的路由集合，类似一个小型 FastAPI 应用。
- 用法：先实例化 APIRouter，用它定义路由，最后通过 app.include_router() 挂载到主应用。

### 3.6.2 案例：用户模块与商品模块的路由拆分

#### 1、项目结构

![image-20260528185409890](images/image-20260528185409890.png)

#### 2、用户模块路由定义（routers/user.py）

```python
from fastapi import APIRouter

# 实例化一个 APIRouter，可指定前缀（所有路由自动加上 /users）
router = APIRouter(
    prefix="/users",
    tags=["用户管理"]  # 文档中归类为「用户管理」
)

# 定义用户相关路由
@router.get("/")
def get_all_users():
    return {"message": "获取所有用户列表"}

@router.get("/{user_id}")
def get_user(user_id: int):
    return {"message": f"获取 ID 为 {user_id} 的用户信息"}

```

#### 3、商品模块路由定义（routers/item.py）

```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/items",
    tags=["商品管理"]  # 文档中归类为「商品管理」
)

# 定义商品相关路由
@router.get("/")
def get_all_items():
    return {"message": "获取所有商品列表"}

@router.get("/{item_id}")
def get_item(item_id: int):
    return {"message": f"获取 ID 为 {item_id} 的商品信息"}

```

#### 4、主应用整合路由

```python
from fastapi import FastAPI
from routers import user, item  # 导入子模块路由
import uvicorn

app = FastAPI(title="路由分发示例")

# 挂载用户路由：所有 /users 开头的请求由 user.router 处理
app.include_router(user.router)

# 挂载商品路由：所有 /items 开头的请求由 item.router 处理
app.include_router(item.router)

# 主应用自身也可以定义路由
@app.get("/")
def root():
    return {"message": "欢迎访问主页面"}

if __name__ == "__main__":
    # 直接在代码中启动uvicorn服务器
    uvicorn.run(
        app="main:app",       # 指定要运行的FastAPI应用实例
        host="0.0.0.0",  # 允许外部访问（本地可通过127.0.0.1或localhost访问）
        port=8000,     # 端口号
        reload=True    # 开发模式：代码修改后自动重启（生产环境需去掉）
    )
```

#### 5、运行与测试

- 启动主服务
- 主页面：http://127.0.0.1:8000 → 返回主页面信息
- 用户列表：http://127.0.0.1:8000/users → 返回用户列表信息
- 商品详情：http://127.0.0.1:8000/items/100 → 返回 ID=100 的商品信息
- 查看自动文档：http://127.0.0.1:8000/docs，可看到路由按 tags 分类（用户管理、商品管理）。

![image-20260529074603468](images/image-20260529074603468.png)

# 第 4 章 SQLAlchemy

（SQL /ˈæl.kə.mi/）

## 4.1 ORM介绍

### 4.1.1 什么是 ORM

ORM（Object-Relational Mapping，对象关系映射）是一种编程技术，它将数据库中的表结构映射为编程语言中的类，将表中的行映射为类中的对象，让开发者可以用面向对象的方式操作数据库，而无需直接编写 SQL 语句。

### 4.1.2 ORM的核心价值

- 屏蔽数据库差异：同一套代码可适配多种数据库（MySQL、PostgreSQL 等），无需修改核心逻辑。

- 简化开发流程：用类、对象、方法替代 SQL 语句，降低数据库操作的学习成本。

- 提高代码可读性：将数据库操作与业务逻辑融合，代码更符合面向对象思维。

- 自动处理类型转换：无需手动转换数据库字段与Python类型（如MySQL的INT与Python的int）。

### 4.1.3 主流ORM产品对比

Python生态中有多个成熟的ORM工具，各有侧重，以下是常见产品的对比：

| ORM工具          | 特点                                                         | 适用场景                                       |
| ---------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| **SQLAlchemy**   | 功能全面，支持ORM和原生SQL，灵活度极高，文档丰富，生态完善。 | 中大型项目、复杂查询场景、需要跨数据库兼容。   |
| **Django ORM**   | 与 Django 框架深度绑定，开箱即用，简化 CRUD 操作，但灵活性较低。 | Django 框架开发的 Web 应用。                   |
| **Tortoise-ORM** | 异步 ORM，支持  async/await，与 FastAPI 等异步框架契合度高。 | 异步 Web 应用（如 FastAPI + 异步数据库驱动）。 |
| **SQLModel**     | 基于 SQLAlchemy 和 Pydantic，简化模型定义，兼顾 ORM 和数据验证。 | FastAPI 项目，追求模型定义简洁性。             |

SQLAlchemy（SQL /ˈæl.kə.mi/）是Python中最成熟的ORM 之一，既支持简单的CRUD 操作，也能应对复杂的多表关联、事务管理等场景，且与FastAPI兼容性极佳，是生产环境的首选。

## 4.2 SQLAlchemy基本架构

![image-20260529075309289](images/image-20260529075309289.png)

SQLAlchemy的架构分层，从下到上可分为DBAPI 层、SQLAlchemy Core（核心层）、SQLAlchemy ORM（对象关系映射层），各组件作用如下

### 4.2.1 DBAPI（数据库通信层）

DBAPI（Database API）是Python数据库接口规范，是SQLAlchemy与底层数据库通信的 “桥梁”。

不同数据库（如 MySQL、PostgreSQL）有各自的 DBAPI 实现（如 pymysql 是 MySQL 的 DBAPI，psycopg2 是 PostgreSQL 的 DBAPI）。SQLAlchemy 通过适配这些 DBAPI，实现对多种数据库的兼容。

### 4.2.2 SQLAlchemy Core（核心层）

Core是SQLAlchemy的 “基础工具集”，提供了SQL表达式语言、数据库连接管理等核心能力，即使不使用 ORM，也能通过 Core 操作数据库。

1）Schema / Types

定义数据库的模式（Schema）和数据类型（Types）。

Schema对应数据库的表、列、约束等结构（比如定义一张表有哪些字段、字段类型是什么）。Types封装了数据库支持的数据类型（如 Integer、String、DateTime 等），并提供 Python 类型与数据库类型的映射。

2）SQL Expression Language

用Python代码生成SQL语句的 “表达式语言”。

它允许你用面向对象的方式编写 SQL（比如用 table.c.column == value 表示 WHERE column = value），既保留了 SQL 的灵活性，又能避免手写 SQL 带来的语法错误和安全问题（如 SQL 注入）。

3）Engine

管理数据库连接的 “引擎”，是与数据库交互的 “入口”。

Engine 负责创建和维护数据库连接，还会集成连接池（Connection Pooling）和方言（Dialect）。

4）Connection Pooling

管理数据库连接池，提升数据库操作性能。

连接池会预先创建一批数据库连接并复用，避免频繁创建 / 销毁连接的开销，尤其在高并发场景下能显著提高效率。

5）Dialect

处理 “方言” 差异，适配不同数据库的 SQL 语法和特性。

不同数据库（如 MySQL 和 PostgreSQL）的 SQL 语法、函数可能有差异（比如 MySQL 的 LIMIT 和 PostgreSQL 的 LIMIT/OFFSET 用法不同）。Dialect 会对这些差异做 “翻译”，让上层代码能以统一的方式操作不同数据库。

### 4.2.3 SQLAlchemy ORM（对象关系映射层）

在Core的基础上，提供象关系映射（ORM）能力，让开发者可以用 “类和对象” 的方式操作数据库（比如用 User 类对应 users 表，用 user = User(name="Alice") 表示新增一条用户记录）。

ORM本质是对Core的 “封装”—— 它会把面向对象的操作（如创建对象、查询对象）自动转换为Core能理解的 SQL 表达式通过，最终 Engine 执行。这样开发者可以更聚焦于业务逻辑，而无需关注底层SQL实现。

### 4.2.4 核心组件

- 引擎（Engine）：`Engine` 是与数据库的连接入口，负责管理连接池和执行 SQL 语句。

- 基类（Base）：所有 ORM 模型的父类，用于统一管理数据库表结构。

- 会话（Session）：用于执行数据库操作的会话对象，负责暂存、提交、回滚数据操作。

## 4.3 环境准备

### 4.3.1 安装依赖

```
# 安装 SQLAlchemy 核心库
pip install sqlalchemy

# 安装 MySQL 驱动（推荐 pymysql，兼容 Python 3.x）
pip install pymysql
```

### 4.3.2 准备 MySQL 环境

确保本地或远程 MySQL 服务已启动（默认端口 3306）。

创建一个测试数据库（如 `fastapi_db`）

## 4.4 基本操作案例

### 4.4.1 在models.py中定义ORM模型类（映射MySQL表）

模型类在定义的时候需要继承Base基类。

```python
from sqlalchemy.ext.declarative import declarative_base

# 生成基类，所有模型需继承该类
Base = declarative_base()
```



模型类对应MySQL中的表，类属性对应表字段，通过 Column 定义字段类型和约束。

| SQLAlchemy 类型 | 数据库类型     | 说明                     |
| --------------- | -------------- | ------------------------ |
| **Integer**     | INT / INTEGER  | 整数         |
| **SmallInteger ** | SMALLINT | 短整数（小范围数字） |
| **BigInteger**  | BIGINT         | 长整数                   |
| **Float**       | FLOAT / DOUBLE | 浮点数                   |
| **DECIMAL**     | DECIMAL        | 精确小数 |
| **String(N)**   | VARCHAR(N) | 可变长度字符串            |
| **CHAR(N) ** | CHAR(N) | 固定长度字符串（手机号、邮编） |
| **Text**        | TEXT       | 长文本                    |
| **MediumText** | MEDIUMTEXT | 中等长文本 |
| **LongText** | LONGTEXT | 超长文本 |
| **LargeBinary** | BLOB       | 二进制文件（图片 / 文件） |
| **Boolean** | BOOLEAN / TINYINT(1) | 真 / 假 |
| **Date**        | DATE       | 日期：2025-01-01                 |
| **DateTime**    | DATETIME   | 日期 + 时间：2025-01-01 12:00:00 |
| **Time**        | TIME       | 时间：12:00:00                   |
| **Timestamp** | TIMESTAMP | 时间戳（自动更新） |
| **Enum** | ENUM | 单选枚举（MySQL 枚举） |
| **Enum(native_enum=False) 或 SET（新版）** | SET | 多选集合（MySQL 集合） |
| **JSON** | JSON | JSON 类型（MySQL 5.7+） |

字段约束说明：

- primary_key=True：设为主键。
- autoincrement=True：MySQL 自增（仅整数类型可用）。
- unique=True：唯一索引，避免重复值。
- nullable=False：非空约束（字段必须有值）。
- default = 值：默认值约束
- ForeignKey(主表.关联字典,  ondelete="XX", onupdate="XX")：外键约束， **XX4 个值**：
  1. **RESTRICT**（禁止删除，默认 ✅）
  2. **CASCADE**（级联删除 ⚠️）
  3. **SET NULL**（设为空 ✅）
  4. **NO ACTION**（同 RESTRICT）

```python
from sqlalchemy import Column, Integer, String, ForeignKey, Date, Float,DECIMAL,Enum,CHAR
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.mysql import SET

class Department(Base): #部门模型类
    """部门模型（一对多：一个部门包含多个员工）"""
    __tablename__ = "t_department"   #对应表名

    did = Column(Integer, primary_key=True, autoincrement=True) #部门编号
    dname = Column(String(20), nullable=False, unique=True)  # 部门名称
    description = Column(String(200))  # 部门简介


    def __str__(self):
        return f"{self.did},{self.dname},{self.description}"

class Employee(Base): #员工模型类
    """员工模型（多对一：多个员工属于一个部门）"""
    __tablename__ = "t_employee"  #对应表名

    eid = Column(Integer, primary_key=True, autoincrement=True) #员工编号
    ename = Column(String(20), nullable=False)  # 姓名
    salary = Column(Float,nullable=False) #薪资
    commission_pct = Column(DECIMAL) #奖金比例
    birthday = Column(Date,nullable=False) #出生日期
    gender = Column(Enum('男','女'),default='男',nullable=False) #性别
    tel = Column(CHAR(11),nullable=False) #电话
    email = Column(String(32),nullable=False)#邮箱
    address = Column(String(150))#地址
    work_place = Column(SET('北京','深圳','上海','武汉','成都','西安'),default="北京",nullable=False) #工作地点
    hiredate = Column(Date,nullable=False)  # 入职日期
    job_id=Column(Integer,ForeignKey("t_job.jid", ondelete="SET NULL", onupdate="CASCADE")) #职位编号
    mid=Column(Integer,ForeignKey("t_employee.eid", ondelete="SET NULL", onupdate="CASCADE")) #领导编号
    did = Column(Integer,ForeignKey("t_department.did", ondelete="SET NULL", onupdate="CASCADE"))#部门编号

    def __str__(self):
        return (f"{self.eid},{self.ename},{self.salary},{self.commission_pct},{self.birthday},"
                f"{self.gender},{self.tel},{self.email},{self.address},{self.work_place},"
                f"{self.hiredate},{self.job_id},{self.mid},{self.did}")

class Job(Base): #职位模型类
    """职位模型（一对多：一个职位包含多个员工）"""
    __tablename__ = "t_job"  #对应表名

    jid = Column(Integer, primary_key=True, autoincrement=True)#职位编号
    jname = Column(String(20), nullable=False, unique=True)  # 职位名称
    description = Column(String(200))  # 职位简介

    def __str__(self):
        return f"{self.jid},{self.jname},{self.description}"
```



### 4.4.2 在database.py创建引擎以及会话

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# MySQL 连接格式：mysql+pymysql://用户名:密码@主机地址:端口/数据库名
DATABASE_URL = "mysql+pymysql://root:123456@localhost:3306/fastapi_db"

# 创建引擎（echo=True 会打印执行的 SQL，方便调试）
engine = create_engine(
    DATABASE_URL,
    echo=True,  # 是否启用日志输出 开发环境启用，生产环境关闭
    pool_pre_ping=True  # 连接前检查有效性，避免连接失效
)

# 创建会话工厂（绑定引擎）
SessionLocal = sessionmaker(
    autocommit=False,  # 关闭自动提交，需手动 commit()
    bind=engine
)
```

- **autoflush=False ：**SQLAlchemy 的会话（Session）内部维护了一个 “身份映射”（Identity Map），用于缓存当前会话中操作的对象。当你执行 db.add(new_user) 时，new_user 仅被添加到这个会话内存的缓存中，并未同步到数据库（无论是缓冲区还是物理表，你就算马上 query，也查不到刚 add 的对象），必须手动`db.flush()` 或 `db.commit()` 才会发给数据库。
- **autoflush=True（默认）：**只要你执行 **query /execute 查询**，SQLAlchemy 会**自动先 flush**，把你 `add/update/delete` 的内容从 **Python 内存 → 数据库事务缓冲区**。这样你本次查询能看到自己刚加的数据，但**没 commit，别人还是看不到**。
- **autocommit=False（默认，最常用）**：Session 会**一直开着一个事务**，所有 `add/delete/update` 都在这个事务里，**必须手动 commit ()** 才会**真正写入数据库、别人才能看到**，可以 rollback ()。
- **autocommit=True（极少用，不推荐）**：没有长事务，每条语句自己开、自己提交，**无法回滚**，一般只用于执行 DDL（建表）

### 4.4.3 新增数据（Create）

#### 1、添加部门

```python
from models import Department

def insert_department_data():
    # 获取数据库会话
    session = SessionLocal()

    try:
        # =========== 第一步：新增部门 =============
        new_dept = Department(dname="安保部", description="负责公司安保工作")
        session.add(new_dept)  # 将部门对象加入会话
        session.commit()  # 提交到数据库（执行 INSERT 语句）
        session.refresh(new_dept)  # 刷新对象，获取自增的 id 等字段
        # ===================== 输出结果 =====================
        print(f"新增部门：ID={new_dept.did}，名称={new_dept.dname}，描述={new_dept.description}")

    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"新增失败：{e}")
    finally:
        session.close()  # 关闭会话
```

#### 2、添加员工

```python
def insert_employee_data():
    # 获取数据库会话
    session = SessionLocal()

    try:
        # =========== 第二步：新增关联的员工 ===========
        # 员工1：关联上面创建的安保部
        emp1 = Employee(
            ename="张三",
            salary = 15000,
            birthday = date(1995,5,1),
            gender = "男",
            tel = "18396587548",
            email = "zhangsan@atguigu.com",
            hiredate=date(2023, 1, 1),
            did=8
        )
        # 员工2：同部门的另一个员工
        emp2 = Employee(
            ename="李四",
            salary = 15000,
            birthday = date(1996,6,1),
            gender = "男",
            tel = "18396587546",
            email = "lisi@atguigu.com",
            work_place="北京,深圳",
            hiredate=date(2023, 1, 1),
            did=8
        )

        # 员工3：其他部门的另一个员工
        emp3 = Employee(
            ename="王五",
            salary = 15000,
            birthday = date(1996,6,1),
            gender = "男",
            tel = "18396587545",
            email = "wangwu@atguigu.com",
            work_place="北京,深圳",
            hiredate=date(2023, 1, 1),
            did=1
        )

        # 批量添加员工（也可逐个 add）
        session.add_all([emp1, emp2,emp3])
        session.commit()  # 提交员工数据
        # 刷新员工对象，获取自增 ID
        session.refresh(emp1)
        session.refresh(emp2)
        session.refresh(emp3)

        print(f"新增员工1：ID={emp1.eid}，姓名={emp1.ename}，所属部门={emp1.did}")
        print(f"新增员工2：ID={emp2.eid}，姓名={emp2.ename}，所属部门={emp2.did}")
        print(f"新增员工3：ID={emp3.eid}，姓名={emp3.ename}，所属部门={emp3.did}")


    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"新增失败：{e}")
    finally:
        session.close()  # 关闭会话
```

### 4.4.4 删除数据（Delete）

#### 1、删除员工

- 先查询实体对象
- session会话.delete(实体对象)

```python
def delete_employee_data():
    # 获取会话
    session = SessionLocal()
    try:
        # 删除单个员工
        emp = session.query(Employee).filter(Employee.ename == "王五").first()
        if emp:
            session.delete(emp)
            session.commit()
            print(f"已删除员工：{emp.ename}")

    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"刪除失败：{e}")
    finally:
        session.close()  # 关闭会话
```

#### 2、删除部门

```python
def delete_department_data():
    # 获取会话
    session = SessionLocal()

    try:
        # 删除部门
        dept = session.query(Department).filter(Department.dname == "安保部").first()
        if dept:
            session.delete(dept)
            session.commit()
            print(f"已删除部门：{dept.dname}")
            #查看员工表，安保部门的员工的did此时为NULL了

    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"刪除失败：{e}")
    finally:
        session.close()  # 关闭会话

```

### 4.4.5 修改数据（Update）

- 先查询实体对象
- 然后直接修改实体对象的属性

```python
def update_employee_data():
    # 获取会话
    session = SessionLocal()
    try:
        # 修改员工薪资
        emp = session.query(Employee).filter(Employee.ename == "张三").first()
        if emp:
            print(f"修改前{emp.ename}薪资：{emp.salary}")
            emp.salary += 2000  # 直接修改属性
            print(f"修改后{emp.ename}薪资：{emp.salary}")4
            session.commit()  # 提交更新

    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"修改失败：{e}")
    finally:
        session.close()  # 关闭会话
```

### 4.4.6 查询数据（Read）

#### 1、根据主键查询

- session会话.get(类，主键值)

```python
def read_data_by_primary_key():
    # 获取会话
    session = SessionLocal()
    try:
        # ======按主键查询 get========
        #  查询id=1的部门
        dept = session.get(Department, 1)
        print(f"部门 ID=1：{dept.dname}（{dept.description}）")
    except Exception as e:
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话
```

#### 2、根据条件筛选filter

- session会话.query(类).filter(条件).first()：first()返回第一条结果（适合唯一查询）
- session会话.query(类).filter(条件).all()：all()返回所有结果（列表）

```python
def read_data_by_filter():
    # 获取会话
    session = SessionLocal()
    try:
        # ======过滤（filter）查询========
        emp = session.query(Employee).filter(Employee.eid == 1).first()
        print("员工编号为1的员工：",emp)

        # 查询研发部的所有员工 ,按部门ID过滤
        employees = session.query(Employee).filter(Employee.did == 1).all()
        print("研发部员工：")
        for e in employees:
            print(e)

        # 查询薪资大于15000的员工
        employees = session.query(Employee).filter(Employee.salary > 15000).all()
        print("薪资>15000的员工：")
        for e in employees:
            print(e)
    except Exception as e:
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话
```

#### 3、多条件结合查询

- session会话.query(类).filter(and_(条件1,条件2)).all() 或 first()
- session会话.query(类).filter(or_(条件1,条件2)).all() 或 first()

```python
from sqlalchemy import and_, or_

def read_data_and_or():
    # 获取会话
    session = SessionLocal()
    try:
        # ======逻辑运算（and_/or_）查询========
        # 薪资[10000,15000]且属于研发部的员工（and_）
        employees = session.query(Employee).filter(and_(Employee.salary.between(10000, 15000), Employee.did == 1)).all()
        print("符合[10000,15000]且属于研发部条件的员工：")
        for e in employees:
            print(e)

        # 属于2号部门或薪资高于20000的员工（or_）
        employees = session.query(Employee).filter(or_(Employee.did == 2, Employee.salary > 20000)).all()
        print("符合属于2号部门或薪资高于20000条件的员工：")
        for e in employees:
            print(e)
    except Exception as e:
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话
```

#### 4、去重

- session会话.query(类.属性).distinct().all()：distinct() 去重

```python
def read_data_distinct():
    # 获取会话
    session = SessionLocal()
    try:
        # ======去重（distinct）========
        # 查询所有有员工的部门编号（去重）
        dids = session.query(Employee.did).distinct().all()  # distinct() 去重
        print("部门编号：", dids)

    except Exception as e:
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话
```

#### 5、分组查询

- session会话.query(类.分组字段, func.分组函数(类.字段), func.分组函数(类.字段)).group_by(类.分组字段)

```python
def read_data_group_by():
    # 获取会话
    session = SessionLocal()
    try:
        # ======分组查询========
        results = session.query(Employee.did,
                                func.max(Employee.salary).label("max_salary"),
                                   func.avg(Employee.salary).label("avg_salary")
                                   ).group_by(Employee.did)

        # 获取所有字段信息列表
        # cols = results.column_descriptions
        # field_names = [col['name'] for col in cols]
        # print(field_names)
        for row in results:
            print(row)
    except Exception as e:
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话
```



#### 6、子查询

- subquery()：转为子查询对象
- 获取子查询对象的字段：子查询对象.c.字段名

```python
from sqlalchemy import func

def read_data_subquery():
    # 获取会话
    session = SessionLocal()
    try:
        # ======子查询（subquery）========
        # 步骤1：子查询：查询最高薪资值
        max_salary = session.query(func.max(Employee.salary).label("max_salary")).subquery()  # 转为子查询
        # 步骤2：主查询  ，子查询对象.c.字段名
        emps = session.query(Employee).filter(Employee.salary == max_salary.c.max_salary).all() #筛选薪资最高的员工

        print("薪资最高的员工：")
        for emp in emps:
            print(emp)
    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话

```

#### 7、关联查询

（1）内连接：session会话.query(A表, B表).join(B表, 关联条件).all()

（2）左外连接：session会话.query(A表，B表).outerjoin(B表, 关联条件).all()

（3）右外连接：session会话.query(A表，B表).join(A表, 关联条件, isouter=True) 

​						 session会话.query(A表，B表).select_from(B).outerjoin(A表, 关联条件) 

（4）全外连接：session会话.query(A表，B表).join(B表, 关联条件, full=True)，但是MySQL不支持

（5）合并查询结果：

​						session会话.query(A表，B表).outerjoin(B表, 关联条件).union(session会话.query(A表，B表).select_from(B).outerjoin(A表, 关联条件))

说明：

- query(A表，B表)：等价于联合查询的`select A表.*, B表.`*
- join(B表, 关联条件) ：等价于 `A表 inner join B表 on 关联条件`，此时join()中写A或B表都一样
- outerjoin(B表, 关联条件)：等价于`A表 left join B表 on 关联条件`
- outerjoin(A表, 关联条件)：等价于`B表 left join A表 on 关联条件`
- select_from(B).outerjoin(A表, 关联条件) ：等价于 `A表 right join B表 on 关联条件`
- select_from(A).outerjoin(B表, 关联条件) ：等价于`B表 right join A表 on 关联条件`

> 内连接：
>

```python
def read_data_inner_join():
    # 获取会话
    session = SessionLocal()
    try:
        # ======表连接（join）查询========
        # 内连接：查询员工及其所属部门名称
        # query(A表，B表).join(A或B表, 关联条件)
        result = session.query(Employee, Department)\
                        .join(Department, Employee.did == Department.did)\
                        .all()
        for emp, dept in result:
            print(f"{emp.eid:<5}\t{emp.ename:<5}\t{emp.salary:<8}\t{dept.did:<5}\t{dept.dname:<5}")
    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话
```

> 左连接：

```python
def read_data_left_join():
    # 获取会话
    session = SessionLocal()
    try:
        # ======表连接（join）查询========
        # 左连接：查询员工及其所属部门名称
        # query(A表，B表).outerjoin(B表, 关联条件) 查询A
        # 左连接 outerjoin()里面写B，就表示 A left join B 查询A
        # 左连接 outerjoin()里面写A，就表示 B left join A 查询B
        left_result = session.query(Employee, Department)\
                        .outerjoin(Department, Employee.did == Department.did)\
                        .all()

        # left_result = session.query(Employee, Department)\
        #                 .outerjoin(Employee, Employee.did == Department.did)\
        #                 .all()
        print_emp_dept(left_result)
    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话
```



> 右连接：

```python
def read_data_right_join():
    # 获取会话
    session = SessionLocal()
    try:
        # ======表连接（join）查询========
        # 右连接：查询员工及其所属部门名称
        # query(A表，B表).join(A表, 关联条件, isouter=True) 查询B   A right join B
        # query(A表，B表).join(B表, 关联条件, isouter=True) 查询A  B right join A
        # right_result = session.query(Employee, Department)\
        #                 .join(Employee, Employee.did == Department.did,isouter=True)\
        #                 .all()
        # right_result = session.query(Employee, Department) \
        #     .join(Department, Employee.did == Department.did, isouter=True) \
        #     .all()

        # 或 query(A表，B表).select_from(B).outerjoin(A表, 关联条件) 查询B   A right join B
        # 或 query(A表，B表).select_from(A).outerjoin(B表, 关联条件) 查询A   B right join A
        right_result = session.query(Employee, Department) \
                            .select_from(Department) \
                            .outerjoin(Employee, Employee.did == Department.did) \
                            .all()
        # right_result = session.query(Employee, Department) \
        #     .select_from(Employee) \
        #     .outerjoin(Department, Employee.did == Department.did) \
        #     .all()
        print_emp_dept(right_result)
    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话
```



> 合并左右连接：

```python
def read_data_full_join():
    # 获取会话
    session = SessionLocal()
    try:
        # ======表连接（join）查询========
        # 全连接：查询员工及其所属部门名称
        # query(A表，B表).join(B表, 关联条件, full=True) A表 full outer join B表
        # full_result = session.query(Employee, Department) \
        #     .join(Department, Employee.did == Department.did, full=True) \
        #     .all()
        # 但是MySQL不支持，所以需要使用union
        full_result = session.query(Employee, Department)\
                        .outerjoin(Department, Employee.did == Department.did)\
                        .union(
                            session.query(Employee, Department)\
                                .select_from(Department)\
                                .outerjoin(Employee, Employee.did == Department.did)
                        ).all()
        print_emp_dept(full_result)
    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话
```



#### 8、原生SQL查询

```python
def read_data_by_sql():
    # 获取会话
    session = SessionLocal()
    try:
        # ======基于原生SQL查询========
        # 查询每一个部门的人数
        result = session.execute(
                text("""
                        SELECT dname,count(eid)
                        FROM t_employee e 
                        RIGHT JOIN t_department d ON e.did = d.did
                        group by dname
                    """)
                    ).all()
        print("查询结果是：",result)
    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话
```







## 4.5 关联关系

在 SQLAlchemy 中，关联关系（Relationship） 用于定义不同模型（表）之间的业务关联（如一对一、一对多、多对多），通过 relationship 函数实现，配合字段定义（如外键）可实现对象化的关联查询和操作。

### 4.5.1 常见的关联关系

关联关系本质是映射数据库中表与表的关系，SQLAlchemy 提供了 4 种基础关联类型：

| 关联类型               | 场景示例                 | 数据库实现                        |
| ---------------------- | ------------------------ | --------------------------------- |
| 一对多（One-to-Many）  | 一个用户拥有多个商品     | 子表通过外键关联主表              |
| 多对一（Many-to-One）  | 多个商品属于一个用户     | 同上（一对多的反向视角）          |
| 一对一（One-to-One）   | 一个用户对应一个个人资料 | 子表外键设为唯一（`unique=True`） |
| 多对多（Many-to-Many） | 多个学生选修多个课程     | 通过中间表关联两个表              |

### 4.5.2 核心配置参数（relationship 函数）

relationship 函数是定义关联关系的核心，常用参数如下

| 参数                          | 作用                                                     | 示例                                     |
| ----------------------------- | -------------------------------------------------------- | ---------------------------------------- |
| `argument`                    | 必选，指定关联的目标模型（类或字符串）                   | `relationship("Employee")`               |
| `back_populates`              | 双向关联时，指定反向关联的字段名（显式定义双向关系）     | `back_populates="department"`            |
| `backref`                     | 简化双向关联，自动为目标模型添加反向关联字段（隐式定义） | `backref="department"`                   |
| `foreign_keys`                | 显式指定关联的外键字段（多外键场景下必用）               | `foreign_keys=[Item.owner_id]`           |
| `cascade`                     | 级联操作规则（如保存、删除关联数据）                     | `cascade="all, delete-orphan"`           |
| `lazy`                        | 关联数据的加载方式（控制查询性能）                       | `lazy="selectin"`（一次性加载）          |
| `uselist`                     | 控制是否为集合（`True` 表示一对多，`False` 表示一对一）  | `uselist=False`（一对一）                |
| `secondary`                   | 多对多关系中，指定中间表                                 | `secondary=user_course`                  |
| `primaryjoin`/`secondaryjoin` | 复杂关联时，显式定义表连接条件（默认自动生成）           | `primaryjoin=(User.id == Item.owner_id)` |

### 4.5.3 具体关联类型的配置方式

#### 1、一对多（One-to-Many）

一对多（One-to-Many）与多对一（Many-to-One）

使用示例：

```python
from sqlalchemy import Column, Integer, String, ForeignKey, Date, Float,DECIMAL,Enum,CHAR
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.mysql import SET
from base import Base

class Department(Base):
    """部门模型（一对多：一个部门包含多个员工）"""
    __tablename__ = "t_department"

    did = Column(Integer, primary_key=True, autoincrement=True)
    dname = Column(String(20), nullable=False, unique=True)  # 部门名称
    description = Column(String(200))  # 部门简介

    # 一对多关联员工：返回员工列表
    employees = relationship("Employee", back_populates="department",lazy="selectin")

    def __str__(self):
        return f"{self.did},{self.dname},{self.description}"

class Employee(Base):
    """员工模型（多对一：多个员工属于一个部门）"""
    __tablename__ = "t_employee"

    eid = Column(Integer, primary_key=True, autoincrement=True)
    ename = Column(String(20), nullable=False)  # 姓名
    salary = Column(Float,nullable=False) #薪资
    commission_pct = Column(DECIMAL) #奖金比例
    birthday = Column(Date,nullable=False) #出生日期
    gender = Column(Enum('男','女'),default='男',nullable=False) #性别
    tel = Column(CHAR(11),nullable=False) #电话
    email = Column(String(32),nullable=False)#邮箱
    address = Column(String(150))#地址
    work_place = Column(SET('北京','深圳','上海','武汉','成都','西安'),default="北京",nullable=False) #工作地点
    hiredate = Column(Date,nullable=False)  # 入职日期
    job_id=Column(Integer,ForeignKey("t_job.jid", ondelete="SET NULL", onupdate="CASCADE")) #职位编号
    mid=Column(Integer,ForeignKey("t_employee.eid", ondelete="SET NULL", onupdate="CASCADE")) #领导编号
    did = Column(Integer,ForeignKey("t_department.did", ondelete="SET NULL", onupdate="CASCADE"))#部门编号

    # 多对一关联
    department = relationship("Department",back_populates="employees",lazy="joined")
    job = relationship("Job",backref="employees",lazy="joined" )

    def __str__(self):
        return (f"{self.eid},{self.ename},{self.salary},{self.commission_pct},{self.birthday},"
                f"{self.gender},{self.tel},{self.email},{self.address},{self.work_place},"
                f"{self.hiredate},{self.job_id},{self.mid},{self.did}")

class Job(Base):
    """职位模型（一对多：一个职位包含多个员工）"""
    __tablename__ = "t_job"

    jid = Column(Integer, primary_key=True, autoincrement=True)#职位编号
    jname = Column(String(20), nullable=False, unique=True)  # 职位名称
    description = Column(String(200))  # 职位简介

    def __str__(self):
        return f"{self.jid},{self.jname},{self.description}"
```

```python
def read_data_relationship():
    # 获取会话
    session = SessionLocal()
    try:
        # ======查询所有员工========
        # 加载员工时同时加载部门信息
        employees = session.query(Employee).options(joinedload(Employee.department)).all()
        for emp in employees:
            print(f"{emp.ename}"
                  f",部门：{emp.department.dname if emp.department else "无部门"}，"
                  f",职位：{emp.job.jname if emp.job else "无职位"}")

        # ======查询所有部门========
        departments = session.query(Department).all()
        for dept in departments:
            print(f"{dept.dname}")
            employees = dept.employees
            print(f"\t{[emp.ename for emp in employees]}")

        # ======查询所有职位========
        jobs = session.query(Job).all()
        for job in jobs:
            print(f"{job.jname}")
            employees = job.employees
            print(f"\t{[emp.ename for emp in employees]}")
    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话

```



#### 2、一对一（One-to-One）

在一对多基础上，通过 uselist=False 限制关联为单个对象，以 “用户（User）- 个人资料（Profile）” 为例：

> 表结构SQL：

```sql
-- ----------------------------
-- Table structure for t_profiles
-- ----------------------------
DROP TABLE IF EXISTS `t_profiles`;
CREATE TABLE `t_profiles`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `bio` varchar(200) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NULL DEFAULT NULL,
  `user_id` int NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `user_id`(`user_id` ASC) USING BTREE,
  CONSTRAINT `t_profiles_ibfk_1` FOREIGN KEY (`user_id`) REFERENCES `t_users` (`id`) ON DELETE RESTRICT ON UPDATE RESTRICT
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Table structure for t_users
-- ----------------------------
DROP TABLE IF EXISTS `t_users`;
CREATE TABLE `t_users`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `username` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci ROW_FORMAT = Dynamic;

```

> ORM模型类：

```python
class User(Base):
    __tablename__ = "t_users"
    id = Column(Integer, primary_key=True,autoincrement=True)
    username = Column(String(50))

    # 一对一：用户对应一个资料（uselist=False 表示非集合）
    profile = relationship(
        "Profile",
        back_populates="user",
        uselist=False,  # 关键：关联结果为单个对象（非列表）
        cascade="all, delete-orphan"  # 删除用户时删除资料
        # cascade="all, delete"  # 删除用户时删除资料
    )

class Profile(Base):
    __tablename__ = "t_profiles"
    id = Column(Integer, primary_key=True,autoincrement=True)
    bio = Column(String(200))  # 个人简介
    user_id = Column(Integer, ForeignKey("t_users.id"), unique=True)  # 外键唯一

    # 反向关联用户
    user = relationship("User", back_populates="profile")
```

关键点：

- 子表（Profile）的外键需设 unique=True，确保一个用户只对应一个资料；
- 主表（User）的 relationship 需设 uselist=False，表示关联结果是单个对象（而非列表）。

> 使用示例
>

```python
def one_by_one():
    # 获取会话
    session = SessionLocal()
    try:
        # 方式1：先创建用户，再创建资料并关联
        user1 = User(username="alice")
        session.add(user1)
        session.commit()  # 先提交用户，获取 ID
        session.refresh(user1)

        profile1 = Profile(bio="喜欢读书", user_id=user1.id)  # 通过 user_id 关联
        session.add(profile1)
        session.commit()

        # 方式2：直接通过 relationship 关联（更简洁）
        user2 = User(
            username="bob",
            profile=Profile(bio="热爱运动")  # 直接嵌套 Profile 对象
        )
        session.add(user2)
        session.commit()  # 自动同步 user_id

        user3 = User(
            username="Lucy",
            profile=Profile(bio="喜欢跳舞")  # 直接嵌套 Profile 对象
        )
        session.add(user3)
        session.commit()  # 自动同步 user_id
    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话

def delete_user():
    # 获取会话
    session = SessionLocal()
    try:
        user = session.query(User).filter(User.username=="alice").first()
        # user = session.query(User).filter(User.username=="Lucy").first()
        print(f"用户名：{user.username}，爱好：{user.profile.bio}")
        session.delete(user)
        session.commit()

    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"执行失败：{e}")
    finally:
        session.close()  # 关闭会话

def delete_orphan():
    # 获取会话
    session = SessionLocal()
    try:
        user = session.query(User).filter(User.username=="bob").first()
        print(f"用户名：{user.username}，爱好：{user.profile.bio}")
        user.profile = None
        print(f"解除关系用户名：{user.username}，爱好：{user.profile}")
        session.refresh(user)
        session.commit()

    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"执行失败：{e}")
    finally:
        session.close()  # 关闭会话
```

#### 3、多对多（Many-to-Many）

需要通过中间表关联两个模型，以 “学生（Student）- 课程（Course）” 为例：

> 表SQL：

```sql
-- ----------------------------
-- Table structure for students
-- ----------------------------
DROP TABLE IF EXISTS `students`;
CREATE TABLE `students`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci ROW_FORMAT = Dynamic;


-- ----------------------------
-- Table structure for courses
-- ----------------------------
DROP TABLE IF EXISTS `courses`;
CREATE TABLE `courses`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Table structure for student_course
-- ----------------------------
DROP TABLE IF EXISTS `student_course`;
CREATE TABLE `student_course`  (
  `student_id` int NOT NULL,
  `course_id` int NOT NULL,
  PRIMARY KEY (`student_id`, `course_id`) USING BTREE,
  INDEX `course_id`(`course_id` ASC) USING BTREE,
  CONSTRAINT `student_course_ibfk_1` FOREIGN KEY (`student_id`) REFERENCES `students` (`id`) ON DELETE RESTRICT ON UPDATE RESTRICT,
  CONSTRAINT `student_course_ibfk_2` FOREIGN KEY (`course_id`) REFERENCES `courses` (`id`) ON DELETE RESTRICT ON UPDATE RESTRICT
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci ROW_FORMAT = Dynamic;


```

> ORM模型类

```python
# 1. 定义中间表（无需模型类，直接用 Table 定义）
student_course = Table(
    "student_course",  # 中间表名
    Base.metadata,
     Column("student_id", Integer, ForeignKey("students.id"), primary_key=True),
    Column("course_id", Integer, ForeignKey("courses.id"), primary_key=True)
)

class Student(Base):
    __tablename__ = "students"
    id = Column(Integer, primary_key=True,autoincrement=True)
    name = Column(String(50))

    # 多对多：学生选修多个课程（通过 secondary 指定中间表）
    courses = relationship(
        "Course",
        secondary=student_course,  # 关联中间表
        back_populates="students",
        lazy="selectin"
)

class Course(Base):
    __tablename__ = "courses"
    id = Column(Integer, primary_key=True,autoincrement=True)
    name = Column(String(100))

    # 反向关联：课程包含多个学生
    students = relationship(
        "Student",
        secondary=student_course,
        back_populates="courses"
    )

```

使用示例

```python
def many_to_many():
    # 获取会话
    session = SessionLocal()
    try:
        # 创建学生和课程
        student1 = Student(name="张三")
        student2 = Student(name="李四")
        course1 = Course(name="数学")
        course2 = Course(name="英语")

        # 建立关联
        student1.courses = [course1, course2]
        student2.courses = [course1]

        session.add_all([student1, student2, course1, course2])
        session.commit()

        # 查询：学生张三选修的课程
        print([c.name for c in student1.courses])

        # 查询：数学课程包含的学生
        print([s.name for s in course1.students])
    except Exception as e:
        session.rollback()  # 出错时回滚
        print(f"查询失败：{e}")
    finally:
        session.close()  # 关闭会话

```

### 4.5.4 关键参数详解

1、双向关联：back_populates vs backref

- back_populates（推荐）：显式在两个模型中定义反向关联，逻辑清晰。
  - 例：Department.employees 设 back_populates="department"，同时，Eemloyee.department 设 back_populates="employees"。


- backref：简化写法，在一个模型中定义即可自动为另一个模型生成反向字段。
  - 例：Eemloyee.job设 backref="employees"，则 Job会自动拥有 employees字段。


2、级联操作（cascade）

控制主对象操作对关联对象的影响，常用规则：

- save-update：保存主对象时自动保存关联对象（默认值）；
- delete：删除主对象时自动删除关联对象；
- delete-orphan：关联对象与主对象解除关联时自动删除（仅用于一对多）；
- all：包含 save-update, merge, refresh-expire, expunge, delete。

例：cascade="all, delete-orphan" 表示 “全量级联 + 解除关联时删除”。

3、加载方式（lazy）

控制关联数据的查询时机，影响性能：

- select（默认）：访问关联属性时才查询（可能产生 N+1 问题）；
- selectin：查询主对象时，通过 IN 语句一次性加载所有关联数据（推荐）；
- joined：通过 JOIN 语句与主对象一起加载（适合一对一）；
- subquery：通过子查询加载关联数据；
- dynamic：返回查询对象（可继续添加过滤条件，适合大数据量）。

4、总结

- 关联关系是 SQLAlchemy ORM 的核心，通过 relationship 函数定义，配合外键或中间表实现；
- 按业务场景选择关联类型（一对多、一对一、多对多），并配置双向关联（back_populates）；
- 合理设置 cascade（级联）和 lazy（加载方式），平衡代码简洁性和性能；

多对多关系需通过中间表（Table 实例）实现，无需定义模型类。

## 4.6 根据模型类生成表

### 4.6.1 在models.py定义模型类

```python
from typing import Optional
from sqlalchemy import ForeignKeyConstraint, Index, Integer, String,Column,ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class Departments(Base):
    __tablename__ = 'departments'
    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(20), nullable=False, unique=True)  # 部门名称

    # 一对多关联员工：返回员工列表
    employees = relationship("Employees", back_populates="department",lazy="selectin")


class Employees(Base):
    __tablename__ = 'employees'
    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(20), nullable=False)  # 姓名
    age =Column(Integer) #年龄
    department_id = Column(Integer, ForeignKey("departments.id", ondelete="SET NULL", onupdate="CASCADE"))  # 部门编号

    department = relationship("Departments", back_populates="employees", lazy="joined")
```

### 4.6.2 自动创建表结构

> models_2_table.py文件

```python
from sqlalchemy import create_engine
from models import Base,Employees,Departments #必须要加

# 创建数据库引擎
engine = create_engine("mysql+pymysql://root:123456@localhost:3306/fastapi_db", echo=True)

# 创建表
def create_table():
    print("注册的表名:", Base.metadata.tables.keys())
    # 创建所有模型对应的表
    Base.metadata.create_all(bind=engine)
    print("表创建成功")

if __name__ == '__main__':
    create_table()
```

执行后，MySQL 中会生成 employees 和 departments表，包含定义的字段和约束。

## 4.7 sqlacodegen通过表生成类

sqlacodegen 是一个实用工具，能根据现有数据库表结构（或 SQL 语句）自动生成 SQLAlchemy 模型类，省去手动编写模型的麻烦，尤其适合已有数据库的项目迁移。

它支持多种数据库（MySQL、PostgreSQL、SQLite 等），生成的模型类包含表名、字段类型、主键、外键、索引等完整信息。

### 4.7.1 安装依赖

```
# 直接安装（支持 SQLAlchemy 1.4+ 和 2.0+）
pip install sqlacodegen

# 如果需要连接特定数据库，需安装对应驱动（以 MySQL 为例）
pip install pymysql  # MySQL 驱动
```

### 4.7.2 准备数据库表（SQL）

先在数据库中创建 departments（部门）和 employees（员工）表，包含一对多关系（一个部门有多个员工）：

```sql
-- 部门表
CREATE TABLE departments (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL UNIQUE COMMENT '部门名称'
);

-- 员工表（关联部门）
CREATE TABLE employees (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL COMMENT '员工姓名',
    age INT COMMENT '年龄',
    department_id INT COMMENT '所属部门ID',
    FOREIGN KEY (department_id) REFERENCES departments(id) ON DELETE SET NULL,
    INDEX idx_dept (department_id)  -- 部门索引，优化查询
);

```

### 4.7.3 创建table_2_models.py用于接收创建的模型类

### 4.7.4 在database.py中创建数据库引擎和会话工厂

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

# 创建数据库引擎
url = "mysql+pymysql://root:123456@localhost:3306/fastapi_db"
engine = create_engine(url, echo=True)
# 配置会话工厂
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)
```

### 4.7.5 在gen.py实现表转类

```python
import subprocess
import sys
from database import url

# 生成模型类
def table_2_model(run=False):
    """将数据库表映射为Python类"""
    if not run:
        return
    output_path = "table_2_models.py"

    venv_python = sys.executable  # 若PyCharm使用虚拟环境，这里会返回.venv下的python.exe
    print("当前使用的Python路径：", venv_python)  # 确认输出是.venv/Scripts/python.exe

    cmd = [venv_python,
           "-m","sqlacodegen",
            "--tables", "departments,employees",
           url]
    result = subprocess.run(cmd, capture_output=True, text=True, encoding="utf-8")

    # 打印执行结果（定位问题核心）
    print("=== 命令执行结果 ===")
    print(f"返回码（0=成功，非0=失败）：{result.returncode}")
    print(f"标准输出：\n{result.stdout}")
    print(f"错误输出：\n{result.stderr}")  # 重点看这里，会显示失败原因

    with open(output_path, "w", encoding="utf-8") as f:
        f.write(result.stdout)

if __name__ == "__main__":
    table_2_model(True)

```

### 4.7.5 在test.py测试生成的模型类

```python
from table_2_models import Departments, Employees
from database import SessionLocal

# 向员工和部门表中插入数据
def insert_dept_emp():
    # 1. 创建员工对象
    emp = Employees(name='zs',age=20)

    # 2. 创建部门对象
    dept = Departments(name='研发部',employees=[emp]) # 关联员工

    # 3. 插入数据库
    with SessionLocal() as session:
        try:
            session.add(dept)
            session.commit()
            session.refresh(dept)
            session.refresh(emp)
            print(f"插入成功！部门ID：{dept.id}，员工ID：{emp.id}")
        except Exception as e:
            session.rollback()
            print(f"插入失败：{e}")
if __name__ == "__main__":
    insert_dept_emp()

```





# 第5章 FastAPI与SQLAlchemy结合案例（扩展）

## 5.1 项目结构

myproject/

├── main.py     # FastAPI 主应用（接口定义）

├── database.py    # 数据库配置（引擎、会话）

└── models.py     # SQLAlchemy 模型（映射 MySQL 表）

## 5.2 定义的models.py、database.py

> models.py

```python
from typing import Optional

from sqlalchemy import ForeignKeyConstraint, Index, Integer, String,Column,ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class Departments(Base):
    __tablename__ = 'departments'
    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(20), nullable=False, unique=True)  # 部门名称

    # 一对多关联员工：返回员工列表
    employees = relationship("Employees", back_populates="department",lazy="selectin")


class Employees(Base):
    __tablename__ = 'employees'
    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(20), nullable=False)  # 姓名
    age =Column(Integer) #年龄
    department_id = Column(Integer, ForeignKey("departments.id", ondelete="SET NULL", onupdate="CASCADE"))  # 部门编号

    department = relationship("Departments", back_populates="employees", lazy="selectin")
    #这里lazy不要设置"joined"，因为双方双向引用，FastAPI如果直接返回Employees或Departments对象时，会导致一起加载关联对象，递归死循环
```

> database.py

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# MySQL 连接格式：mysql+pymysql://用户名:密码@主机地址:端口/数据库名
DATABASE_URL = "mysql+pymysql://root:123456@localhost:3306/fastapi_db"

# 创建引擎（echo=True 会打印执行的 SQL，方便调试）
engine = create_engine(
    DATABASE_URL,
    echo=True,  # 是否启用日志输出 开发环境启用，生产环境关闭
    pool_pre_ping=True  # 连接前检查有效性，避免连接失效
)

# 创建会话工厂（绑定引擎）
SessionLocal = sessionmaker(
    # autoflush=False,
    autocommit=False,  # 关闭自动提交，需手动 commit()
    bind=engine
)
```



## 5.3 创建main.py测试

```python
import uvicorn
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session

from models import Departments
from database import SessionLocal

app = FastAPI()

# 依赖项：获取数据库会话
def get_db():
    session = SessionLocal()
    try:
        yield session
    finally:
        session.close()

# 部门相关接口
@app.post("/departments/")
def create_department(name: str, session: Session = Depends(get_db)):
    department = Departments(name=name)
    session.add(department)
    session.commit()
    session.refresh(department)
    return department

@app.get("/departments/")
def read_departments(session: Session = Depends(get_db)):
    return session.query(Departments).all()

@app.get("/departments/{department_id}")
def read_department(department_id: int, session: Session = Depends(get_db)):
    department = session.query(Departments).filter(Departments.id == department_id).first()
    if not department:
        raise HTTPException(status_code=404, detail="Department not found")
    return department

if __name__ == "__main__":
    # 直接在代码中启动uvicorn服务器
    uvicorn.run(
        app="main:app",       # 指定要运行的FastAPI应用实例
        host="0.0.0.0",  # 允许外部访问（本地可通过127.0.0.1或localhost访问）
        port=8000,     # 端口号
        reload=True    # 开发模式：代码修改后自动重启（生产环境需去掉）
)

```

## 5.4 借助apidoc测试接口

![image-20260529183917936](images/image-20260529183917936.png)
