# Django 中的异步视图

> 原文：<https://testdriven.io/blog/django-async-views/>

编写异步代码使您能够不费吹灰之力就加快应用程序的速度。Django 版本> = 3.1 [支持](https://docs.djangoproject.com/en/3.2/releases/3.1/#asynchronous-views-and-middleware-support)异步视图、中间件和测试。如果您还没有尝试过异步视图，现在是时候了解一下了。

本教程着眼于如何开始使用 Django 的异步视图。

> 如果您有兴趣了解更多关于异步代码背后的力量以及 Python 中线程、多处理和异步之间的差异，请查看我的文章[用并发、并行和异步加速 Python。](/blog/concurrency-parallelism-asyncio/)

## 目标

学完本教程后，您应该能够:

1.  用 Django 写一个异步视图
2.  在 Django 视图中发出一个非阻塞 HTTP 请求
3.  用 Django 的异步视图简化基本的后台任务
4.  使用`sync_to_async`在异步视图中进行同步调用
5.  解释什么时候应该和不应该使用异步视图

您还应该能够回答以下问题:

1.  如果在异步视图中进行同步调用会怎样？
2.  如果在异步视图中进行同步和异步调用会怎么样？
3.  用 Django 的异步观点芹菜还有必要吗？

## 先决条件

只要您已经熟悉 Django 本身，向非基于类的视图添加异步功能是非常简单的。

### 属国

1.  Python >= 3.8
2.  Django >= 3.1
3.  独角兽企业
4.  HTTPX

### ASGI 是什么？

[ASGI](https://asgi.readthedocs.io/en/latest/) 代表异步服务器网关接口。它是 [WSGI](https://wsgi.readthedocs.io/en/latest/) 的现代异步后续，为创建基于 Python 的异步 web 应用程序提供了标准。

另一件值得一提的事情是，ASGI 向后兼容 WSGI，这是一个很好的借口，可以从像 Gunicorn 或 uWSGI 这样的 WSGI 服务器切换到像 [Uvicorn](https://www.uvicorn.org/) 或 [Daphne](https://github.com/django/daphne) 这样的 ASGI 服务器，即使你*没有*准备好切换到编写异步应用程序。

## 创建应用程序

创建一个新的项目目录和一个新的 Django 项目:

```
`$ mkdir django-async-views && cd django-async-views
$ python3.10 -m venv env
$ source env/bin/activate

(env)$ pip install django
(env)$ django-admin startproject hello_async .` 
```

> 你可以随意把 virtualenv 和 Pip 换成诗歌[或](https://python-poetry.org) [Pipenv](https://github.com/pypa/pipenv) 。更多信息，请查看[现代 Python 环境](/blog/python-environments/)。

如果您使用内置的开发服务器，Django 将运行您的异步视图，但它实际上不会异步运行它们，所以我们将使用 Uvicorn 运行 Django。

安装它:

```
`(env)$ pip install uvicorn` 
```

要使用 Uvicorn 运行您的项目，您可以从项目的根目录使用以下命令:

```
`uvicorn {name of your project}.asgi:application` 
```

在我们的例子中，这将是:

```
`(env)$ uvicorn hello_async.asgi:application` 
```

接下来，让我们创建第一个异步视图。添加一个新文件来保存“hello_async”文件夹中的视图，然后添加以下视图:

```
`# hello_async/views.py

from django.http import HttpResponse

async def index(request):
    return HttpResponse("Hello, async Django!")` 
```

在 Django 中创建异步视图就像创建同步视图一样简单——您所需要做的就是添加关键字`async`。

更新 URL:

```
`# hello_async/urls.py

from django.contrib import admin
from django.urls import path

from hello_async.views import index

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", index),
]` 
```

现在，在终端的根文件夹中，运行:

```
`(env)$ uvicorn hello_async.asgi:application --reload` 
```

> 标志告诉 Uvicorn 观察你的文件是否有变化，如果有变化就重新加载。这可能是不言自明的。

在您最喜欢的网络浏览器中打开 [http://localhost:8000/](http://localhost:8000/) :

不是世界上最令人兴奋的事情，但是，嘿，这是一个开始。值得注意的是，用 Django 的内置开发服务器运行这个视图会产生完全相同的功能和输出。这是因为我们实际上没有在处理程序中做任何异步的事情。

## HTTPX

值得注意的是，异步支持是完全向后兼容的，因此您可以混合异步和同步视图、中间件和测试。Django 将在适当的执行上下文中执行每一个。

为了演示这一点，添加几个新视图:

```
`# hello_async/views.py

import asyncio
from time import sleep

import httpx
from django.http import HttpResponse

# helpers

async def http_call_async():
    for num in range(1, 6):
        await asyncio.sleep(1)
        print(num)
    async with httpx.AsyncClient() as client:
        r = await client.get("https://httpbin.org/")
        print(r)

def http_call_sync():
    for num in range(1, 6):
        sleep(1)
        print(num)
    r = httpx.get("https://httpbin.org/")
    print(r)

# views

async def index(request):
    return HttpResponse("Hello, async Django!")

async def async_view(request):
    loop = asyncio.get_event_loop()
    loop.create_task(http_call_async())
    return HttpResponse("Non-blocking HTTP request")

def sync_view(request):
    http_call_sync()
    return HttpResponse("Blocking HTTP request")` 
```

更新 URL:

```
`# hello_async/urls.py

from django.contrib import admin
from django.urls import path

from hello_async.views import index, async_view, sync_view

urlpatterns = [
    path("admin/", admin.site.urls),
    path("async/", async_view),
    path("sync/", sync_view),
    path("", index),
]` 
```

安装 [HTTPX](https://www.python-httpx.org/) :

服务器运行时，导航到[http://localhost:8000/async/](http://localhost:8000/async/)。您应该会立即看到响应:

```
`Non-blocking HTTP request` 
```

在您的终端中，您应该看到:

```
`INFO:     127.0.0.1:60374 - "GET /async/ HTTP/1.1" 200 OK
1
2
3
4
5
<Response [200 OK]>` 
```

这里，HTTP 响应在第一次睡眠调用之前被发送回。

接下来，导航到[http://localhost:8000/sync/](http://localhost:8000/sync/)。大约需要五秒钟才能得到响应:

转向终端:

```
`1
2
3
4
5
<Response [200 OK]>
INFO:     127.0.0.1:60375 - "GET /sync/ HTTP/1.1" 200 OK` 
```

这里，HTTP 响应是在循环和对`https://httpbin.org/`的请求完成后*发送的。*

## 熏肉

为了更好地模拟如何利用异步的真实场景，让我们看看如何异步运行多个操作，聚合结果，并将它们返回给调用者。

回到你的项目的 URLconf，在 [`smoke_some_meats`](https://www.youtube.com/watch?v=SVyLlFezj2E) 创建一个新路径:

```
`# hello_async/urls.py

from django.contrib import admin
from django.urls import path

from hello_async.views import index, async_view, sync_view, smoke_some_meats

urlpatterns = [
    path("admin/", admin.site.urls),
    path("smoke_some_meats/", smoke_some_meats),
    path("async/", async_view),
    path("sync/", sync_view),
    path("", index),
]` 
```

回到您的视图中，创建一个名为`smoke`的新异步助手函数。这个函数有两个参数:一个名为`smokables`的字符串列表和一个名为`flavor`的字符串。这些分别默认为可吸烟的肉类和“糖宝·雷”的列表。

```
`# hello_async/views.py

async def smoke(smokables: List[str] = None, flavor: str = "Sweet Baby Ray's") -> List[str]:
    """ Smokes some meats and applies the Sweet Baby Ray's """

    for smokable in smokables:
        print(f"Smoking some {smokable}...")
        print(f"Applying the {flavor}...")
        print(f"{smokable.capitalize()} smoked.")

    return len(smokables)` 
```

for 循环异步地将风味(读:糖宝·雷的)应用到烟草(读:熏肉)上。

不要忘记重要的一点:

> `List`用于额外的打字功能。这不是必需的，可以很容易地省略(只需去掉“smokables”参数声明后面的`: List[str]`)。

接下来，再添加两个异步助手:

```
`async def get_smokables():
    print("Getting smokeables...")

    await asyncio.sleep(2)
    async with httpx.AsyncClient() as client:
        await client.get("https://httpbin.org/")

        print("Returning smokeable")
        return [
            "ribs",
            "brisket",
            "lemon chicken",
            "salmon",
            "bison sirloin",
            "sausage",
        ]

async def get_flavor():
    print("Getting flavor...")

    await asyncio.sleep(1)
    async with httpx.AsyncClient() as client:
        await client.get("https://httpbin.org/")

        print("Returning flavor")
        return random.choice(
            [
                "Sweet Baby Ray's",
                "Stubb's Original",
                "Famous Dave's",
            ]
        )` 
```

确保添加导入:

创建使用异步函数的异步视图:

```
`# hello_async/views.py

async def smoke_some_meats(request):
    results = await asyncio.gather(*[get_smokables(), get_flavor()])
    total = await asyncio.gather(*[smoke(results[0], results[1])])
    return HttpResponse(f"Smoked {total[0]} meats with {results[1]}!")` 
```

这个视图同时调用`get_smokables`和`get_flavor`函数。由于`smoke`依赖于来自`get_smokables`和`get_flavor`的结果，我们使用`gather`来等待每个异步任务完成。

**请记住，在常规的同步视图中，`get_smokables`和`get_flavor`将一次处理一个。此外，异步视图将产生执行，并允许在处理异步任务的同时处理其他请求，这允许在特定的时间内由同一进程处理更多的请求。**

最后，返回一个响应，让用户知道他们美味的 BBQ 餐准备好了。

太好了。保存文件，然后返回浏览器，导航到[http://localhost:8000/smoke _ some _ meats/](http://localhost:8000/smoke_some_meats/)。应该需要几秒钟才能得到响应:

```
`Smoked 6 meats with Sweet Baby Ray's!` 
```

在您的控制台中，您应该会看到:

```
`Getting smokeables...
Getting flavor...
Returning flavor
Returning smokeable

Smoking some ribs...
Applying the Stubb's Original...
Ribs smoked.
Smoking some brisket...
Applying the Stubb's Original...
Brisket smoked.
Smoking some lemon chicken...
Applying the Stubb's Original...
Lemon chicken smoked.
Smoking some salmon...
Applying the Stubb's Original...
Salmon smoked.
Smoking some bison sirloin...
Applying the Stubb's Original...
Bison sirloin smoked.
Smoking some sausage...
Applying the Stubb's Original...
Sausage smoked.
INFO:     127.0.0.1:57501 - "GET /smoke_some_meats/ HTTP/1.1" 200 OK` 
```

请注意以下打印语句的顺序:

```
`Getting smokeables...
Getting flavor...
Returning flavor
Returning smokeable` 
```

这就是工作中的异步性:当`get_smokables`函数休眠时，`get_flavor`函数完成处理。

## 烧焦的肉

### 同步呼叫

问:*如果在异步视图中进行同步调用会怎样？*

如果从非异步视图调用非异步函数，也会发生同样的事情。

--

为了说明这一点，在您的 *views.py* 中创建一个名为`oversmoke`的新助手函数:

```
`# hello_async/views.py

def oversmoke() -> None:
    """ If it's not dry, it must be uncooked """
    sleep(5)
    print("Who doesn't love burnt meats?")` 
```

非常简单:我们只是同步等待五秒钟。

创建调用此函数的视图:

```
`# hello_async/views.py

async def burn_some_meats(request):
    oversmoke()
    return HttpResponse(f"Burned some meats.")` 
```

最后，在项目的 URLconf 中连接路线:

```
`# hello_async/urls.py

from django.contrib import admin
from django.urls import path

from hello_async.views import index, async_view, sync_view, smoke_some_meats, burn_some_meats

urlpatterns = [
    path("admin/", admin.site.urls),
    path("smoke_some_meats/", smoke_some_meats),
    path("burn_some_meats/", burn_some_meats),
    path("async/", async_view),
    path("sync/", sync_view),
    path("", index),
]` 
```

在浏览器中访问路线在[http://localhost:8000/burn _ some _ meats](http://localhost:8000/burn_some_meats):

请注意，最终从浏览器得到响应花了五秒钟。您还应该同时收到控制台输出:

```
`Who doesn't love burnt meats?
INFO:     127.0.0.1:40682 - "GET /burn_some_meats HTTP/1.1" 200 OK` 
```

可能值得注意的是，不管您使用的是什么服务器，不管是基于 WSGI 还是 ASGI，都会发生同样的事情。

### 同步和异步呼叫

问:如果在异步视图中进行同步和异步调用会怎么样？

不要这样。

同步和异步视图往往最适合不同的目的。如果在异步视图中有阻塞功能，最好的情况也不会比仅仅使用同步视图更好。

## 同步到异步

如果您需要在一个异步视图中进行同步调用(例如，通过 Django ORM 与数据库交互)，使用 [sync_to_async](https://docs.djangoproject.com/en/4.0/topics/async/#sync-to-async) 作为包装器或装饰器。

示例:

```
`# hello_async/views.py

async def async_with_sync_view(request):
    loop = asyncio.get_event_loop()
    async_function = sync_to_async(http_call_sync, thread_sensitive=False)
    loop.create_task(async_function())
    return HttpResponse("Non-blocking HTTP request (via sync_to_async)")` 
```

> 你注意到我们将`thread_sensitive`参数设置为`False`了吗？这意味着同步函数`http_call_sync`将在一个新的线程中运行。查看[文档](https://docs.djangoproject.com/en/4.0/topics/async/#sync-to-async)了解更多信息。

将导入添加到顶部:

```
`from asgiref.sync import sync_to_async` 
```

添加 URL:

```
`# hello_async/urls.py

from django.contrib import admin
from django.urls import path

from hello_async.views import (
    index,
    async_view,
    sync_view,
    smoke_some_meats,
    burn_some_meats,
    async_with_sync_view
)

urlpatterns = [
    path("admin/", admin.site.urls),
    path("smoke_some_meats/", smoke_some_meats),
    path("burn_some_meats/", burn_some_meats),
    path("sync_to_async/", async_with_sync_view),
    path("async/", async_view),
    path("sync/", sync_view),
    path("", index),
]` 
```

在您的浏览器中进行测试，网址为[http://localhost:8000/sync _ to _ async/](http://localhost:8000/sync_to_async/)。

在您的终端中，您应该看到:

```
`INFO:     127.0.0.1:61365 - "GET /sync_to_async/ HTTP/1.1" 200 OK
1
2
3
4
5
<Response [200 OK]>` 
```

使用`sync_to_async`，阻塞同步调用在后台线程中处理，允许 HTTP 响应在第一个睡眠调用之前被发送回*。*

## 芹菜和异步视图

问:对于 Django 的异步观点，芹菜还有存在的必要吗？

看情况。

Django 的异步视图提供了与任务或消息队列相似的功能，而没有复杂性。如果您正在使用(或者正在考虑使用)Django，并且想做一些简单的事情(并且不关心可靠性)，异步视图是一个快速而简单地实现这一点的好方法。如果您需要执行更繁重、长时间运行的后台进程，您仍然会希望使用 Celery 或 RQ。

应该注意，为了有效地使用异步视图，视图中应该只有异步调用。另一方面，任务队列在不同的进程上使用工作线程，因此能够在多个服务器上的后台运行同步调用。

顺便说一下，您决不需要在异步视图和消息队列之间做出选择——您可以很容易地同时使用它们。例如:您可以使用异步视图发送电子邮件或进行一次性数据库修改，但让 Celery 在每晚的预定时间清理您的数据库或生成并发送客户报告。

## 何时使用

对于绿地项目，如果异步是你的事情，利用异步视图，尽可能以异步方式编写你的 I/O 过程。也就是说，如果您的大多数视图只需要调用数据库并在返回数据之前做一些基本的处理，那么您不会看到比坚持使用同步视图有太多的增加(如果有的话)。

对于棕地项目，如果你有很少或没有 I/O 进程，坚持使用同步视图。如果你有许多 I/O 进程，衡量一下用异步方式重写它们有多容易。将 sync I/O 重写为 async 并不容易，所以在尝试重写为 async 之前，您可能需要优化您的 sync I/O 和视图。另外，将同步过程与异步视图混合在一起从来都不是一个好主意。

在生产中，一定要使用 Gunicorn 来管理 uvicon，以便利用并发性(通过 uvicon)和并行性(通过 Gunicorn workers):

```
`gunicorn -w 3 -k uvicorn.workers.UvicornWorker hello_async.asgi:application` 
```

## 结论

总之，尽管这是一个简单的用例，但它应该让您大致了解 Django 的异步视图所带来的可能性。在异步视图中还可以尝试发送电子邮件、调用第三方 API 和读写文件。

有关 Django 新发现的异步性的更多信息，请参阅这篇[优秀文章](https://wersdoerfer.de/blogs/ephes_blog/django-31-async/)，它涵盖了相同的主题以及多线程和测试。