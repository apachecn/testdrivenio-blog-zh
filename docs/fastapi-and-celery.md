# FastAPI 和 Celery 的异步任务

> 原文：<https://testdriven.io/blog/fastapi-and-celery/>

如果长时间运行的流程是应用程序工作流的一部分，而不是阻塞响应，您应该在后台处理它，在正常的请求/响应流之外。

也许您的 web 应用程序要求用户在注册时提交一个缩略图(可能需要重新调整大小)并确认他们的电子邮件。如果您的应用程序处理了图像并直接在请求处理程序中发送了确认电子邮件，那么最终用户将不得不在页面加载或更新之前不必要地等待他们完成处理。相反，您会希望将这些进程传递给任务队列，并让一个单独的工作进程来处理它，这样您就可以立即将响应发送回客户端。最终用户可以在处理过程中在客户端做其他事情。您的应用程序还可以自由地响应来自其他用户和客户端的请求。

为了实现这一点，我们将带您完成设置和配置 [Celery](https://docs.celeryq.dev/en/stable/) 和 Redis 的过程，以便在 FastAPI 应用程序中处理长时间运行的流程。我们还将使用 Docker 和 Docker Compose 将所有内容联系在一起。最后，我们将看看如何用单元测试和集成测试来测试 Celery 任务。

## 目标

本教程结束时，您将能够:

1.  将芹菜集成到 FastAPI 应用程序中，并创建任务。
2.  集装箱 FastAPI、Celery 和 Redis with Docker。
3.  使用单独的工作进程在后台运行进程。
4.  将芹菜日志保存到文件中。
5.  设置 [Flower](https://flower.readthedocs.io/en/latest/) 来监控和管理芹菜作业和工人。
6.  用单元测试和集成测试来测试芹菜任务。

## 后台任务

同样，为了改善用户体验，长时间运行的流程应该在正常的 HTTP 请求/响应流程之外，在后台进程中运行。

示例:

1.  运行机器学习模型
2.  发送确认电子邮件
3.  刮擦和爬行
4.  分析数据
5.  处理图像
6.  生成报告

当你构建一个应用程序时，试着区分应该在请求/响应生命周期中运行的任务(比如 CRUD 操作)和应该在后台运行的任务。

值得注意的是，你可以利用 FastAPI 的 [BackgroundTasks](https://fastapi.tiangolo.com/tutorial/background-tasks/) 类在后台运行任务，该类直接来自 [Starlette](https://github.com/encode/starlette/blob/0.14.2/starlette/background.py) 。

例如:

```
`from fastapi import BackgroundTasks

def send_email(email, message):
    pass

@app.get("/")
async def ping(background_tasks: BackgroundTasks):
    background_tasks.add_task(send_email, "[[email protected]](/cdn-cgi/l/email-protection)", "Hi!")
    return {"message": "pong!"}` 
```

那么，什么时候应该用芹菜代替`BackgroundTasks`？

1.  **CPU 密集型任务** : Celery 应该用于执行繁重后台计算的任务，因为`BackgroundTasks`运行在服务于应用程序请求的同一个事件循环中。
2.  **任务队列:**如果你需要一个任务队列来管理任务和工作者，那么你应该使用芹菜。通常，您会希望检索一个作业的状态，然后根据该状态执行一些操作——例如，发送一封错误电子邮件，启动一个不同的后台任务，或者重试该任务。芹菜为你管理这一切。

## 工作流程

我们的目标是开发一个与 Celery 协同工作的 FastAPI 应用程序，以处理正常请求/响应周期之外的长时间运行的流程。

1.  最终用户通过向服务器端发送 POST 请求开始一项新任务。
2.  在路由处理程序中，一个任务被添加到队列中，任务 ID 被发送回客户端。
3.  使用 AJAX，当任务本身在后台运行时，客户机继续轮询服务器以检查任务的状态。

![fastapi and celery queue user flow](img/e8244e81354e37e9e72770349e5fdbe8.png)

## 项目设置

从 [fastapi-celery](https://github.com/testdrivenio/fastapi-celery) repo 中克隆出基础项目，然后将 [v1](https://github.com/testdrivenio/fastapi-celery/tree/v1) 标签签出到主分支:

```
`$ git clone https://github.com/testdrivenio/fastapi-celery --branch v1 --single-branch
$ cd fastapi-celery
$ git checkout v1 -b master` 
```

由于我们总共需要管理三个进程(FastAPI、Redis、Celery worker)，我们将使用 Docker 来简化我们的工作流，方法是将它们连接起来，以便它们都可以通过一个命令从一个终端窗口运行。

从项目根目录，创建映像并启动 Docker 容器:

```
`$ docker-compose up -d --build` 
```

构建完成后，导航到 [http://localhost:8004](http://localhost:8004) :

![fastapi project](img/3358350675aa80256128e0220314d789.png)

确保测试也通过:

```
`$ docker-compose exec web python -m pytest

================================== test session starts ===================================
platform linux -- Python 3.9.5, pytest-6.2.4, py-1.10.0, pluggy-0.13.1
rootdir: /usr/src/app
collected 1 item

tests/test_tasks.py .                                                               [100%]

=================================== 1 passed in 0.06s ====================================` 
```

在继续之前，快速浏览一下项目结构:

```
`├── .gitignore
├── LICENSE
├── README.md
├── docker-compose.yml
└── project
    ├── Dockerfile
    ├── main.py
    ├── requirements.txt
    ├── static
    │   ├── main.css
    │   └── main.js
    ├── templates
    │   ├── _base.html
    │   ├── footer.html
    │   └── home.html
    └── tests
        ├── __init__.py
        ├── conftest.py
        └── test_tasks.py` 
```

## 触发任务

在*project/templates/home . html*中设置了一个`onclick`事件处理程序来监听按钮点击:

```
`<div class="btn-group" role="group" aria-label="Basic example">
  <button type="button" class="btn btn-primary" onclick="handleClick(1)">Short</a>
  <button type="button" class="btn btn-primary" onclick="handleClick(2)">Medium</a>
  <button type="button" class="btn btn-primary" onclick="handleClick(3)">Long</a>
</div>` 
```

`onclick`调用 *project/static/main.js* 中的`handleClick`，它向服务器发送一个 AJAX POST 请求，并带有适当的任务类型:`1`、`2`或`3`。

```
`function  handleClick(type)  { fetch('/tasks',  { method:  'POST', headers:  { 'Content-Type':  'application/json' }, body:  JSON.stringify({  type:  type  }), }) .then(response  =>  response.json()) .then(res  =>  getStatus(res.data.task_id)); }` 
```

在服务器端，已经配置了一个路由来处理 *project/main.py* 中的请求:

```
`@app.post("/tasks", status_code=201)
def run_task(payload = Body(...)):
    task_type = payload["type"]
    return JSONResponse(task_type)` 
```

现在有趣的部分来了——给芹菜布线！

## 芹菜装置

首先将 Celery 和 Redis 添加到 *requirements.txt* 文件中:

```
`aiofiles==0.6.0
celery==4.4.7
fastapi==0.64.0
Jinja2==2.11.3
pytest==6.2.4
redis==3.5.3
requests==2.25.1
uvicorn==0.13.4` 
```

> 本教程使用芹菜 v [4.4.7](https://pypi.org/project/celery/4.4.7/) 因为花[不支持芹菜 5](https://github.com/mher/flower/issues/1029) 。

Celery 使用消息[代理](https://docs.celeryq.dev/en/stable/getting-started/backends-and-brokers/index.html) - [RabbitMQ](https://www.rabbitmq.com/) 、 [Redis](https://redis.io/) 或 [AWS 简单队列服务(SQS)](https://aws.amazon.com/sqs/) -来促进 Celery worker 和 web 应用程序之间的通信。消息被添加到代理中，然后由工作人员进行处理。一旦完成，结果被添加到后端。

Redis 将被用作代理和后端。将 Redis 和芹菜[工人](https://docs.celeryq.dev/en/stable/userguide/workers.html)添加到 *docker-compose.yml* 文件中，如下所示:

```
`version:  '3.8' services: web: build:  ./project ports: -  8004:8000 command:  uvicorn main:app --host 0.0.0.0 --reload volumes: -  ./project:/usr/src/app environment: -  CELERY_BROKER_URL=redis://redis:6379/0 -  CELERY_RESULT_BACKEND=redis://redis:6379/0 depends_on: -  redis worker: build:  ./project command:  celery worker --app=worker.celery --loglevel=info volumes: -  ./project:/usr/src/app environment: -  CELERY_BROKER_URL=redis://redis:6379/0 -  CELERY_RESULT_BACKEND=redis://redis:6379/0 depends_on: -  web -  redis redis: image:  redis:6-alpine` 
```

请注意`celery worker --app=worker.celery --loglevel=info`:

1.  `celery worker`是用来开动芹菜[的工人](https://docs.celeryq.dev/en/stable/userguide/workers.html#starting-the-worker)
2.  `--app=worker.celery`运行芹菜[应用程序](https://docs.celeryq.dev/en/stable/userguide/application.html)(我们将很快对其进行定义)
3.  `--loglevel=info`将[记录级别](https://docs.celeryq.dev/en/v4.4.7/reference/celery.bin.worker.html#cmdoption-celery-worker-l)设置为信息

接下来，在“项目”中创建一个名为 *worker.py* 的新文件:

```
`import os
import time

from celery import Celery

celery = Celery(__name__)
celery.conf.broker_url = os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379")
celery.conf.result_backend = os.environ.get("CELERY_RESULT_BACKEND", "redis://localhost:6379")

@celery.task(name="create_task")
def create_task(task_type):
    time.sleep(int(task_type) * 10)
    return True` 
```

这里，我们创建了一个新的 Celery 实例，并使用[任务](https://docs.celeryq.dev/en/stable/reference/celery.html#celery.Celery.task)装饰器，我们定义了一个名为`create_task`的新 Celery 任务函数。

> 请记住，任务本身将由芹菜工人执行。

## 触发任务

更新路由处理程序以启动任务，并使用任务 ID 进行响应:

```
`@app.post("/tasks", status_code=201)
def run_task(payload = Body(...)):
    task_type = payload["type"]
    task = create_task.delay(int(task_type))
    return JSONResponse({"task_id": task.id})` 
```

不要忘记导入任务:

```
`from worker import create_task` 
```

构建映像并旋转新容器:

```
`$ docker-compose up -d --build` 
```

要触发新任务，请运行:

```
`$ curl http://localhost:8004/tasks -H "Content-Type: application/json" --data '{"type": 0}'` 
```

您应该会看到类似这样的内容:

```
`{
  "task_id": "14049663-6257-4a1f-81e5-563c714e90af"
}` 
```

## 任务状态

回到客户端的`handleClick`功能:

```
`function  handleClick(type)  { fetch('/tasks',  { method:  'POST', headers:  { 'Content-Type':  'application/json' }, body:  JSON.stringify({  type:  type  }), }) .then(response  =>  response.json()) .then(res  =>  getStatus(res.data.task_id)); }` 
```

当响应从最初的 AJAX 请求返回时，我们继续每秒调用带有任务 ID 的`getStatus()`:

```
`function  getStatus(taskID)  { fetch(`/tasks/${taskID}`,  { method:  'GET', headers:  { 'Content-Type':  'application/json' }, }) .then(response  =>  response.json()) .then(res  =>  { const  html  =  `
 <tr>
 <td>${taskID}</td>
 <td>${res.data.task_status}</td>
 <td>${res.data.task_result}</td>
 </tr>`; document.getElementById('tasks').prepend(html); const  newRow  =  document.getElementById('table').insertRow(); newRow.innerHTML  =  html; const  taskStatus  =  res.data.task_status; if  (taskStatus  ===  'finished'  ||  taskStatus  ===  'failed')  return  false; setTimeout(function()  { getStatus(res.data.task_id); },  1000); }) .catch(err  =>  console.log(err)); }` 
```

如果响应成功，一个新行被添加到 DOM 上的表中。

更新`get_status`路线处理器以返回状态:

```
`@app.get("/tasks/{task_id}")
def get_status(task_id):
    task_result = AsyncResult(task_id)
    result = {
        "task_id": task_id,
        "task_status": task_result.status,
        "task_result": task_result.result
    }
    return JSONResponse(result)` 
```

导入[异步结果](https://docs.celeryq.dev/en/stable/reference/celery.result.html):

```
`from celery.result import AsyncResult` 
```

更新容器:

```
`$ docker-compose up -d --build` 
```

触发新任务:

```
`$ curl http://localhost:8004/tasks -H "Content-Type: application/json" --data '{"type": 1}'` 
```

然后，从响应中获取`task_id`并调用更新的端点来查看状态:

```
`$ curl http://localhost:8004/tasks/f3ae36f1-58b8-4c2b-bf5b-739c80e9d7ff

{
  "task_id": "455234e0-f0ea-4a39-bbe9-e3947e248503",
  "task_result": true,
  "task_status": "SUCCESS"
}` 
```

也在浏览器中测试一下:

![fastapi, celery, docker](img/6742ab16c2cdfd3f0d9c5ae1141840d5.png)

## 芹菜原木

更新 *docker-compose.yml* 中的`worker`服务，以便将芹菜日志转储到一个日志文件:

```
`worker: build:  ./project command:  celery worker --app=worker.celery --loglevel=info --logfile=logs/celery.log volumes: -  ./project:/usr/src/app environment: -  CELERY_BROKER_URL=redis://redis:6379/0 -  CELERY_RESULT_BACKEND=redis://redis:6379/0 depends_on: -  web -  redis` 
```

向“项目”添加一个名为“日志”的新目录。然后，将名为 *celery.log* 的新文件添加到新创建的目录中。

更新:

```
`$ docker-compose up -d --build` 
```

由于我们设置了一个卷，您应该看到日志文件在本地被填满:

```
`[2021-05-08 15:32:24,407: INFO/MainProcess] Connected to redis://redis:6379/0
[2021-05-08 15:32:24,415: INFO/MainProcess] mingle: searching for neighbors
[2021-05-08 15:32:25,434: INFO/MainProcess] mingle: all alone
[2021-05-08 15:32:25,448: INFO/MainProcess] [[email protected]](/cdn-cgi/l/email-protection) ready.
[2021-05-08 15:32:29,834: INFO/MainProcess]
    Received task: create_task[013df48c-4548-4a2b-9b22-7267da215361]
[2021-05-08 15:32:39,825: INFO/ForkPoolWorker-7]
    Task create_task[013df48c-4548-4a2b-9b22-7267da215361]
    succeeded in 10.02114040000015s: True` 
```

## 花卉仪表板

[Flower](https://flower.readthedocs.io/en/latest/) 是一个轻量级的、实时的、基于网络的芹菜监控工具。您可以监控当前正在运行的任务，增加或减少工作池，查看图表和一些统计数据，等等。

添加到 *requirements.txt* :

```
`aiofiles==0.6.0
celery==4.4.7
fastapi==0.64.0
flower==0.9.7
Jinja2==2.11.3
pytest==6.2.4
redis==3.5.3
requests==2.25.1
uvicorn==0.13.4` 
```

然后，向 *docker-compose.yml* 添加一个新服务:

```
`dashboard: build:  ./project command:  flower --app=worker.celery --port=5555 --broker=redis://redis:6379/0 ports: -  5556:5555 environment: -  CELERY_BROKER_URL=redis://redis:6379/0 -  CELERY_RESULT_BACKEND=redis://redis:6379/0 depends_on: -  web -  redis -  worker` 
```

测试一下:

```
`$ docker-compose up -d --build` 
```

导航到 [http://localhost:5556](http://localhost:5556) 查看仪表板。您应该看到一名员工准备就绪:

![flower dashboard](img/07ab11c9f8690305169988750faf0d13.png)

开始几项任务来全面测试仪表板:

![flower dashboard](img/78214ca99bcbbd5d203dbc1a66871a45.png)

试着增加几个工人，看看会有什么影响:

```
`$ docker-compose up -d --build --scale worker=3` 
```

## 试验

让我们从最基本的测试开始:

```
`def test_task():
    assert create_task.run(1)
    assert create_task.run(2)
    assert create_task.run(3)` 
```

将上述测试用例添加到*project/tests/test _ tasks . py*中，然后添加以下导入:

```
`from worker import create_task` 
```

单独运行测试:

```
`$ docker-compose exec web python -m pytest -k "test_task and not test_home"` 
```

运行应该需要大约一分钟:

```
`================================== test session starts ===================================
platform linux -- Python 3.9.5, pytest-6.2.4, py-1.10.0, pluggy-0.13.1
rootdir: /usr/src/app
plugins: celery-4.4.7
collected 2 items / 1 deselected / 1 selected

tests/test_tasks.py .                                                               [100%]

====================== 1 passed, 1 deselected in 60.05s (0:01:00)  ========================` 
```

值得注意的是，在上面的断言中，我们使用了`.run`方法(而不是`.delay`)来直接运行任务，而没有芹菜工人。

想要模仿`.run`方法来加快速度吗？

```
`@patch("worker.create_task.run")
def test_mock_task(mock_run):
    assert create_task.run(1)
    create_task.run.assert_called_once_with(1)

    assert create_task.run(2)
    assert create_task.run.call_count == 2

    assert create_task.run(3)
    assert create_task.run.call_count == 3` 
```

导入:

```
`from unittest.mock import patch, call` 
```

测试:

```
`$ docker-compose exec web python -m pytest -k "test_mock_task"

================================== test session starts ===================================
platform linux -- Python 3.9.5, pytest-6.2.4, py-1.10.0, pluggy-0.13.1
rootdir: /usr/src/app
plugins: celery-4.4.7
collected 3 items / 2 deselected / 1 selected

tests/test_tasks.py .                                                               [100%]

============================ 1 passed, 2 deselected in 0.13s =============================` 
```

快多了！

全面整合测试怎么样？

```
`def test_task_status(test_app):
    response = test_app.post(
        "/tasks",
        data=json.dumps({"type": 1})
    )
    content = response.json()
    task_id = content["task_id"]
    assert task_id

    response = test_app.get(f"tasks/{task_id}")
    content = response.json()
    assert content == {"task_id": task_id, "task_status": "PENDING", "task_result": None}
    assert response.status_code == 200

    while content["task_status"] == "PENDING":
        response = test_app.get(f"tasks/{task_id}")
        content = response.json()
    assert content == {"task_id": task_id, "task_status": "SUCCESS", "task_result": True}` 
```

请记住，这个测试使用开发中使用的相同的代理和后端。您可能想要实例化一个新的 Celery 应用程序来进行测试。

添加导入:

确保测试通过。

## 结论

这是关于如何配置 Celery 在 FastAPI 应用程序中运行长时间运行的任务的基本指南。您应该让队列处理任何可能阻塞或减慢面向用户的代码的进程。

Celery 还可以用于执行可重复的任务，分解复杂的资源密集型任务，以便将计算工作量分布到多个机器上，从而减少(1)完成时间和(2)处理客户端请求的机器上的负载。

从[回购](https://github.com/testdrivenio/fastapi-celery)中抓取代码。