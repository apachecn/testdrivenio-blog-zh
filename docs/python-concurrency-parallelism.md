# Python 中的并行性、并发性和异步性——示例

> 原文：<https://testdriven.io/blog/python-concurrency-parallelism/>

本教程着眼于如何通过多处理、线程和异步来加速 CPU 绑定和 IO 绑定操作。

## 并发性与并行性

并发和并行是相似的术语，但它们不是一回事。

并发是指在 CPU 上同时运行多个任务的能力。任务可以在重叠的时间段内开始、运行和完成。在单个 CPU 的情况下，多个任务在[上下文切换](https://en.wikipedia.org/wiki/Context_switch)的帮助下运行，其中存储了一个进程的状态，以便稍后调用和执行。

同时，并行性是指在多个 CPU 内核上同时运行多个任务的能力。

虽然它们可以提高应用程序的速度，但是并发和并行不应该到处使用。用例取决于任务是 CPU 受限还是 IO 受限。

受 CPU 限制的任务是受 CPU 限制的。例如，数学计算是受 CPU 限制的，因为计算能力随着计算机处理器数量的增加而增加。并行性适用于 CPU 受限的任务。理论上，如果一个任务被分成 n 个子任务，这 n 个任务中的每一个都可以并行运行，以有效地将时间减少到原来非并行任务的 1/n。并发是 IO 绑定任务的首选，因为您可以在获取 IO 资源的同时做其他事情。

CPU 密集型任务的最好例子是在数据科学中。数据科学家处理大量的数据。对于数据预处理，他们可以将数据分成多个批处理并并行运行，从而有效地减少处理的总时间。增加内核数量可以提高处理速度。

网页抓取是 IO 绑定的。因为该任务对 CPU 的影响很小，因为大部分时间都花在从网络读取数据和向网络写入数据上。其他常见的 IO 绑定任务包括数据库调用以及向磁盘读写文件。像 Django 和 Flask 这样的 Web 应用程序都是 IO 绑定的应用程序。

> 如果您有兴趣了解更多关于 Python 中线程、多处理和异步的区别，请查看文章[用并发、并行和异步加速 Python。](/blog/concurrency-parallelism-asyncio/)

## 方案

至此，让我们来看看如何加速以下任务:

```py
`# tasks.py

import os
from multiprocessing import current_process
from threading import current_thread

import requests

def make_request(num):
    # io-bound

    pid = os.getpid()
    thread_name = current_thread().name
    process_name = current_process().name
    print(f"{pid} - {process_name} - {thread_name}")

    requests.get("https://httpbin.org/ip")

async def make_request_async(num, client):
    # io-bound

    pid = os.getpid()
    thread_name = current_thread().name
    process_name = current_process().name
    print(f"{pid} - {process_name} - {thread_name}")

    await client.get("https://httpbin.org/ip")

def get_prime_numbers(num):
    # cpu-bound

    pid = os.getpid()
    thread_name = current_thread().name
    process_name = current_process().name
    print(f"{pid} - {process_name} - {thread_name}")

    numbers = []

    prime = [True for i in range(num + 1)]
    p = 2

    while p * p <= num:
        if prime[p]:
            for i in range(p * 2, num + 1, p):
                prime[i] = False
        p += 1

    prime[0] = False
    prime[1] = False

    for p in range(num + 1):
        if prime[p]:
            numbers.append(p)

    return numbers` 
```

> 本教程中的所有代码示例都可以在[parallel-concurrent-examples-python](https://github.com/testdrivenio/parallel-concurrent-examples-python)repo 中找到。

注意事项:

*   `make_request`向[https://httpbin.org/ip](https://httpbin.org/ip)发出 X 次 HTTP 请求。
*   `make_request_async`与 [HTTPX](https://www.python-httpx.org/) 异步发出相同的 HTTP 请求。
*   `get_prime_numbers`通过厄拉多塞方法的[筛子，计算从 2 到给定极限的素数。](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes)

我们将使用标准库中的以下库来加速上述任务:

## IO 绑定操作

同样，IO 绑定任务在 IO 上花费的时间比在 CPU 上花费的时间多。

由于 web 抓取是 IO 绑定的，我们应该使用线程来加快处理速度，因为检索 HTML (IO)比解析它(CPU)要慢。

场景:*如何加速一个基于 Python 的网页抓取和爬取脚本？*

### 同步示例

先说一个基准。

```py
`# io-bound_sync.py

import time

from tasks import make_request

def main():
    for num in range(1, 101):
        make_request(num)

if __name__ == "__main__":
    start_time = time.perf_counter()

    main()

    end_time = time.perf_counter()
    print(f"Elapsed run time: {end_time - start_time} seconds.")` 
```

这里，我们使用`make_request`函数发出了 100 个 HTTP 请求。因为请求是同步发生的，所以每个任务都是按顺序执行的。

```py
`Elapsed run time: 15.710984757 seconds.` 
```

因此，每个请求大约需要 0.16 秒。

### 线程示例

```py
`# io-bound_concurrent_1.py

import threading
import time

from tasks import make_request

def main():
    tasks = []

    for num in range(1, 101):
        tasks.append(threading.Thread(target=make_request, args=(num,)))
        tasks[-1].start()

    for task in tasks:
        task.join()

if __name__ == "__main__":
    start_time = time.perf_counter()

    main()

    end_time = time.perf_counter()
    print(f"Elapsed run time: {end_time - start_time} seconds.")` 
```

在这里，同一个`make_request`函数被调用 100 次。这次使用`threading`库为每个请求创建一个线程。

```py
`Elapsed run time: 1.020112515 seconds.` 
```

总时间从大约 16s 减少到大约 1s。

由于我们对每个请求使用单独的线程，您可能会奇怪为什么整个过程没有花 0.16 秒就完成了。这些额外的时间是管理线程的开销。Python 中的[全局解释器锁](https://wiki.python.org/moin/GlobalInterpreterLock) (GIL)确保一次只有一个线程使用 Python 字节码。

### 并发.未来示例

```py
`# io-bound_concurrent_2.py

import time
from concurrent.futures import ThreadPoolExecutor, wait

from tasks import make_request

def main():
    futures = []

    with ThreadPoolExecutor() as executor:
        for num in range(1, 101):
            futures.append(executor.submit(make_request, num))

    wait(futures)

if __name__ == "__main__":
    start_time = time.perf_counter()

    main()

    end_time = time.perf_counter()
    print(f"Elapsed run time: {end_time - start_time} seconds.")` 
```

这里我们使用了`concurrent.futures.ThreadPoolExecutor`来实现多线程。在创建了所有的未来/承诺之后，我们使用`wait`来等待它们全部完成。

```py
`Elapsed run time: 1.340592231 seconds` 
```

`concurrent.futures.ThreadPoolExecutor`实际上是围绕`multithreading`库的一个抽象，这使得它更容易使用。在前面的例子中，我们将每个请求分配给一个线程，总共使用了 100 个线程。但是`ThreadPoolExecutor`默认工作线程的数量为`min(32, os.cpu_count() + 4)`。ThreadPoolExecutor 的存在是为了简化实现多线程的过程。如果你想对多线程有更多的控制，请使用`multithreading`库。

### AsyncIO 示例

```py
`# io-bound_concurrent_3.py

import asyncio
import time

import httpx

from tasks import make_request_async

async def main():
    async with httpx.AsyncClient() as client:
        return await asyncio.gather(
            *[make_request_async(num, client) for num in range(1, 101)]
        )

if __name__ == "__main__":
    start_time = time.perf_counter()

    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())

    end_time = time.perf_counter()
    elapsed_time = end_time - start_time
    print(f"Elapsed run time: {elapsed_time} seconds")` 
```

> 此处使用`httpx`，因为`requests`不支持异步操作。

这里，我们使用了`asyncio`来实现并发。

```py
`Elapsed run time: 0.553961068 seconds` 
```

`asyncio`比其他方法更快，因为`threading`利用了 OS(操作系统)线程。所以线程由操作系统管理，线程切换由操作系统抢占。`asyncio`使用由 Python 解释器定义的协程。使用协程，程序决定何时以最佳方式切换任务。这由 asyncio 中的`even_loop`处理。

## CPU 限制的操作

场景:*如何加速一个简单的数据处理脚本？*

### 同步示例

同样，让我们从一个基准开始。

```py
`# cpu-bound_sync.py

import time

from tasks import get_prime_numbers

def main():
    for num in range(1000, 16000):
        get_prime_numbers(num)

if __name__ == "__main__":
    start_time = time.perf_counter()

    main()

    end_time = time.perf_counter()
    print(f"Elapsed run time: {end_time - start_time} seconds.")` 
```

这里，我们对从 1000 到 16000 的数字执行了`get_prime_numbers`函数。

```py
`Elapsed run time: 17.863046316 seconds.` 
```

### 多重处理示例

```py
`# cpu-bound_parallel_1.py

import time
from multiprocessing import Pool, cpu_count

from tasks import get_prime_numbers

def main():
    with Pool(cpu_count() - 1) as p:
        p.starmap(get_prime_numbers, zip(range(1000, 16000)))
        p.close()
        p.join()

if __name__ == "__main__":
    start_time = time.perf_counter()

    main()

    end_time = time.perf_counter()
    print(f"Elapsed run time: {end_time - start_time} seconds.")` 
```

在这里，我们使用`multiprocessing`来计算质数。

```py
`Elapsed run time: 2.9848740599999997 seconds.` 
```

### 并发.未来示例

```py
`# cpu-bound_parallel_2.py

import time
from concurrent.futures import ProcessPoolExecutor, wait
from multiprocessing import cpu_count

from tasks import get_prime_numbers

def main():
    futures = []

    with ProcessPoolExecutor(cpu_count() - 1) as executor:
        for num in range(1000, 16000):
            futures.append(executor.submit(get_prime_numbers, num))

    wait(futures)

if __name__ == "__main__":
    start_time = time.perf_counter()

    main()

    end_time = time.perf_counter()
    print(f"Elapsed run time: {end_time - start_time} seconds.")` 
```

这里，我们使用`concurrent.futures.ProcessPoolExecutor`实现了多重处理。一旦作业被添加到 futures 中，`wait(futures)`就会等待它们完成。

```py
`Elapsed run time: 4.452427557 seconds.` 
```

`concurrent.futures.ProcessPoolExecutor`是围绕`multiprocessing.Pool`的包装器。它和`ThreadPoolExecutor`有同样的局限性。如果你想对多重处理有更多的控制，使用`multiprocessing.Pool`。`concurrent.futures`提供了对多处理和线程的抽象，使得两者之间的切换变得容易。

## 结论

值得注意的是，使用多重处理来执行`make_request`函数将比线程方式慢得多，因为进程需要等待 IO。不过，多处理方法将比同步方法更快。

类似地，与并行性相比，对 CPU 受限的任务使用并发性是不值得的。

也就是说，使用并发性或并行性来执行脚本会增加复杂性。您的代码通常更难阅读、测试和调试，所以只有在长时间运行的脚本绝对需要时才使用它们。

我通常从这里开始，因为-

1.  在并发和并行之间来回切换很容易
2.  从属库不需要支持 asyncio ( `requests` vs `httpx`)
3.  与其他方法相比，它更清晰、更容易阅读

从 GitHub 的[parallel-concurrent-examples-python](https://github.com/testdrivenio/parallel-concurrent-examples-python)repo 中抓取代码。