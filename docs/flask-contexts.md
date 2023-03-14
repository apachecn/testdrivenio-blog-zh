# 了解 Flask 中的应用程序和请求上下文

> 原文：<https://testdriven.io/blog/flask-contexts/>

这篇文章的目的是阐明应用程序和请求上下文在 Flask 中是如何工作的。

* * *

这是关于烧瓶环境的两部分系列的第一部分:

1.  **基础知识** : [理解 Flask 中的应用和请求上下文](/blog/flask-contexts/)(本文！)
2.  **高级** : [深入了解 Flask 的应用和请求上下文](/blog/flask-contexts-advanced/)

## 目标

在这篇文章结束时，你应该能够解释:

1.  Flask 如何处理请求对象，以及这与其他 web 框架有何不同
2.  应用程序和请求上下文是什么
3.  哪些数据存储在应用程序和请求上下文中
4.  如何在正确的上下文中使用`current_app`、`test_request_context`和`test_client`

您还应该能够修复以下错误:

```py
`RuntimeError: Working outside of application context.

This typically means that you attempted to use functionality that needed
to interface with the current application object in some way. To solve
this, set up an application context with app.app_context().` 
```

## 烧瓶中的上下文

与 Django 和其他 web 框架不同，Flask view 函数不接受包含 HTTP 请求元数据的请求对象。

Django 示例:

```py
`def users(request):
    if request.method == 'POST':
         # Save the form data to the database
         # Send response
   else:
         # Get all users from the database
         # Send response` 
```

使用 Flask，您可以像这样导入请求对象:

```py
`from flask import request

@app.route('/users', methods=['GET', 'POST'])
def users():
    if request.method == 'POST':
         # Save the form data to the database
         # Send response
    else:
         # Get all users from the database
         # Send response` 
```

在 Flask 示例中，请求对象看起来、感觉起来和行为起来都像一个全局变量，但它不是。

> 如果请求对象是一个全局变量，您将无法运行多线程 Flask 应用程序，因为全局变量不是线程安全的。

相反，Flask 使用 **contexts** 来使许多对象像全局对象一样“行动”,只针对正在使用的特定上下文(线程、进程或协程)。在 Flask 中，这被称为[上下文本地](https://werkzeug.palletsprojects.com/en/2.0.x/local/)。

> 上下文本地类似于 Python 的[线程本地](https://docs.python.org/3/library/threading.html#thread-local-data)实现，用于存储特定于一个线程的数据，但本质上不同。Flask 的实现更加通用，允许工作线程、进程或协程。

## 存储在 Flask 上下文中的数据

当接收到请求时，Flask 提供两个上下文:

| 语境 | 描述 | 可用对象 |
| --- | --- | --- |
| [应用](https://flask.palletsprojects.com/en/2.0.x/appcontext/) | 跟踪应用程序级数据(配置变量、记录器、数据库连接) | `current_app`，`g` |
| [请求](https://flask.palletsprojects.com/en/2.0.x/reqcontext/) | 跟踪请求级数据(URL、HTTP 方法、头、请求数据、会话信息) | `request`，`session` |

> 值得注意的是，上述每个对象通常被称为“代理”。这仅仅意味着它们是对象全局风格的代理。关于这方面的更多信息，请查看本系列的第二篇文章。

当收到请求时，Flask 处理这些上下文的创建。它们会造成混乱，因为根据应用程序所处的状态，您并不总是能够访问特定的对象。

我们来看几个例子。

## 应用程序上下文示例

假设您有以下 Flask 应用程序:

```py
`from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    return 'Welcome!'

if __name__ == '__main__':
    app.run()` 
```

首先，让我们看看如何使用 [current_app](https://flask.palletsprojects.com/en/2.0.x/api/#flask.current_app) 对象来访问应用程序上下文。

在 Python shell 中，如果您试图在视图函数之外访问`current_app.config`对象，您应该会看到以下错误:

```py
`$ python
>>> from flask import current_app
>>> current_app.config

Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "werkzeug/local.py", line 347, in __getattr__
    return getattr(self._get_current_object(), name)
  File "werkzeug/local.py", line 306, in _get_current_object
    return self.__local()
  File "flask/globals.py", line 52, in _find_app
    raise RuntimeError(_app_ctx_err_msg)
RuntimeError: Working outside of application context.

This typically means that you attempted to use functionality that needed
to interface with the current application object in some way. To solve
this, set up an application context with app.app_context().  See the
documentation for more information.` 
```

要访问应用程序公开的对象并请求视图函数之外的上下文，您需要首先创建适当的上下文:

```py
`# without a context manager
$ python

>>> from app import app
>>> from flask import current_app
>>>
>>> app_ctx = app.app_context()
>>> app_ctx.push()
>>>
>>> current_app.config["ENV"]
'production'
>>> app_ctx.pop()
>>>` 
```

```py
`# with a context manager
$ python

>>> from app import app
>>> from flask import current_app
>>>
>>> with app.app_context():
...     current_app.config["ENV"]
...
'production'
>>>` 
```

## 请求上下文示例

您可以使用 [test_request_context](https://flask.palletsprojects.com/en/2.0.x/api/#flask.Flask.test_request_context) 方法来创建一个请求上下文:

```py
`# without a context manager
$ python

>>> from app import app
>>> from flask import request
>>>
>>> request_ctx = app.test_request_context()
>>> request_ctx.push()
>>>
>>> request.method
'GET'
>>>
>>> request.path
'/'
>>>
>>> request_ctx.pop()
>>>` 
```

```py
`# with a context manager
$ python

>>> from app import app
>>> from flask import request
>>>
>>> with app.test_request_context('/'):
...     request.method
...     request.path
...
'GET'
'/'
>>>` 
```

> 当您想要使用请求数据而没有完整请求的开销时，通常在测试期间使用`test_request_context`。

## 测试示例

应用程序和请求上下文最常遇到的问题是当您的应用程序处于测试状态时:

```py
`import pytest
from flask import current_app

from app import app

@pytest.fixture
def client():
    with app.test_client() as client:
        assert current_app.config["ENV"] == "production"  # Error!
        yield client

def test_index_page(client):
   response = client.get('/')

   assert response.status_code == 200
   assert b'Welcome!' in response.data` 
```

运行时，测试将在夹具中失败:

```py
`$ pytest
________________________ ERROR at setup of test_index_page _____________________

@pytest.fixture
def client():
    with app.test_client() as client:
>       assert current_app.config["ENV"] == "production"
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
    def _find_app():
        top = _app_ctx_stack.top
        if top is None:
>           raise RuntimeError(_app_ctx_err_msg)
E           RuntimeError: Working outside of application context.
E
E           This typically means that you attempted to use functionality that needed
E           to interface with the current application object in some way. To solve
E           this, set up an application context with app.app_context().  See the
E           documentation for more information.
================================= 1 error in 0.13s =================================` 
```

要解决这个问题，请在访问`current_app`之前创建一个应用程序上下文*:*

```py
`import pytest
from flask import current_app

from app import app

@pytest.fixture
def client():
    with app.test_client() as client:
        with app.app_context():  # New!!
            assert current_app.config["ENV"] == "production"
        yield client

def test_index_page(client):
   response = client.get('/')

   assert response.status_code == 200
   assert b'Welcome!' in response.data` 
```

## 摘要

总而言之，在视图函数、CLI 命令和测试函数中使用以下对象:

| 目标 | 语境 | 常见错误 | 解决办法 |
| --- | --- | --- | --- |
| `current_app` | 应用程序上下文 | 在应用程序上下文之外工作 | `with app.app_context():` |
| `g` | 应用程序上下文 | 在应用程序上下文之外工作 | `with app.test_request_context('/'):` |
| `request` | 请求上下文 | 在请求上下文之外工作 | `with app.test_request_context('/'):` |
| `session` | 请求上下文 | 在请求上下文之外工作 | `with app.test_request_context('/'):` |

测试期间应使用以下方法:

| 烧瓶法 | 描述 |
| --- | --- |
| `test_client` | Flask 应用程序的测试客户端 |
| `test_request_context` | 用于测试的推送请求上下文 |

## 结论

这篇博文只是触及了应用程序和请求上下文的表面。请务必阅读本系列的第二部分以了解更多:[深入了解 Flask 的应用和请求上下文](/blog/flask-contexts-advanced/)

查看以下课程，了解如何构建、测试和部署 Flask 应用程序: