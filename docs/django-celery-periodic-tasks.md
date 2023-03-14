# 用芹菜和码头工人处理 Django 的定期任务

> 原文：<https://testdriven.io/blog/django-celery-periodic-tasks/>

当您构建和扩展 Django 应用程序时，您不可避免地需要定期在后台自动运行某些任务。

一些例子:

*   生成定期报告
*   清除缓存
*   发送批量电子邮件通知
*   运行夜间维护作业

这是构建和扩展不属于 Django 核心的 web 应用程序所需的少数功能之一。幸运的是，芹菜提供了一个强大的解决方案，它相当容易实现，叫做[芹菜击败](https://docs.celeryq.dev/en/latest/userguide/periodic-tasks.html)。

在接下来的文章中，我们将向您展示如何使用 Docker 设置 Django、Celery 和 Redis，以便使用 Celery Beat 定期运行自定义的 Django 管理命令。

> 姜戈+芹菜系列:
> 
> 1.  与 Django 和 Celery 的异步任务
> 2.  [在 Django 用芹菜和 Docker 处理周期性任务](/blog/django-celery-periodic-tasks/)(本文！)
> 3.  [自动重试失败的芹菜任务](/blog/retrying-failed-celery-tasks/)
> 4.  [处理芹菜和数据库事务](/blog/celery-database-transactions/)

## 目标

学完本教程后，您应该能够:

1.  集装箱化姜戈、Celery，并与码头工重归于好
2.  将芹菜集成到 Django 应用程序中并创建任务
3.  编写一个定制的 Django 管理命令
4.  安排一个定制的 Django 管理命令通过 Celery Beat 定期运行

## 项目设置

从 [django-celery-beat](https://github.com/testdrivenio/django-celery-beat) repo 中克隆出基地项目，然后检查[基地](https://github.com/testdrivenio/django-celery-beat/tree/base)分支:

```
`$ git clone https://github.com/testdrivenio/django-celery-beat --branch base --single-branch
$ cd django-celery-beat` 
```

由于我们总共需要管理四个进程(Django、Redis、worker 和 scheduler)，我们将使用 Docker 来简化我们的工作流，方法是将它们连接起来，以便它们都可以通过一个命令从一个终端窗口运行。

从项目根目录，创建映像并启动 Docker 容器:

```
`$ docker-compose up -d --build` 
```

接下来，应用迁移:

```
`$ docker-compose exec web python manage.py migrate` 
```

一旦构建完成，导航到 [http://localhost:1337](http://localhost:1337) 以确保应用程序按预期工作。您应该会看到以下文本:

在继续之前，快速浏览一下项目结构:

```
`├── .gitignore
├── docker-compose.yml
└── project
    ├── Dockerfile
    ├── core
    │   ├── __init__.py
    │   ├── asgi.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── entrypoint.sh
    ├── manage.py
    ├── orders
    │   ├── __init__.py
    │   ├── admin.py
    │   ├── apps.py
    │   ├── migrations
    │   │   ├── 0001_initial.py
    │   │   └── __init__.py
    │   ├── models.py
    │   ├── tests.py
    │   ├── urls.py
    │   └── views.py
    ├── products.json
    ├── requirements.txt
    └── templates
        └── orders
            └── order_list.html` 
```

> 想学习如何构建这个项目吗？查看 Postgres、Gunicorn 和 Nginx 的博客文章。

## Celery and Redis(合唱团)

现在，我们需要为芹菜、芹菜泥和 Redis 添加容器。

我们首先将依赖项添加到 *requirements.txt* 文件中:

```
`Django==3.2.4
celery==5.1.2
redis==3.5.3` 
```

接下来，将以下内容添加到 *docker-compose.yml* 文件的末尾:

```
`redis: image:  redis:alpine celery: build:  ./project command:  celery -A core worker -l info volumes: -  ./project/:/usr/src/app/ environment: -  DEBUG=1 -  SECRET_KEY=dbaa1_i7%*[[email protected]](/cdn-cgi/l/email-protection)(-a_r([[email protected]](/cdn-cgi/l/email-protection)%m -  DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1] depends_on: -  redis celery-beat: build:  ./project command:  celery -A core beat -l info volumes: -  ./project/:/usr/src/app/ environment: -  DEBUG=1 -  SECRET_KEY=dbaa1_i7%*[[email protected]](/cdn-cgi/l/email-protection)(-a_r([[email protected]](/cdn-cgi/l/email-protection)%m -  DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1] depends_on: -  redis` 
```

我们还需要更新 web 服务的`depends_on`部分:

```
`web: build:  ./project command:  python manage.py runserver 0.0.0.0:8000 volumes: -  ./project/:/usr/src/app/ ports: -  1337:8000 environment: -  DEBUG=1 -  SECRET_KEY=dbaa1_i7%*[[email protected]](/cdn-cgi/l/email-protection)(-a_r([[email protected]](/cdn-cgi/l/email-protection)%m -  DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1] depends_on: -  redis  # NEW` 
```

完整的 *docker-compose.yml* 文件现在应该是这样的:

```
`version:  '3.8' services: web: build:  ./project command:  python manage.py runserver 0.0.0.0:8000 volumes: -  ./project/:/usr/src/app/ ports: -  1337:8000 environment: -  DEBUG=1 -  SECRET_KEY=dbaa1_i7%*[[email protected]](/cdn-cgi/l/email-protection)(-a_r([[email protected]](/cdn-cgi/l/email-protection)%m -  DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1] depends_on: -  redis redis: image:  redis:alpine celery: build:  ./project command:  celery -A core worker -l info volumes: -  ./project/:/usr/src/app/ environment: -  DEBUG=1 -  SECRET_KEY=dbaa1_i7%*[[email protected]](/cdn-cgi/l/email-protection)(-a_r([[email protected]](/cdn-cgi/l/email-protection)%m -  DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1] depends_on: -  redis celery-beat: build:  ./project command:  celery -A core beat -l info volumes: -  ./project/:/usr/src/app/ environment: -  DEBUG=1 -  SECRET_KEY=dbaa1_i7%*[[email protected]](/cdn-cgi/l/email-protection)(-a_r([[email protected]](/cdn-cgi/l/email-protection)%m -  DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1] depends_on: -  redis` 
```

在构建新容器之前，我们需要在 Django 应用程序中配置 Celery。

## 芹菜配置

### 设置

在“core”目录中，创建一个 *celery.py* 文件，并添加以下代码:

```
`import os

from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "core.settings")

app = Celery("core")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()` 
```

这里发生了什么事？

1.  首先，我们为环境变量`DJANGO_SETTINGS_MODULE`设置一个默认值，以便芹菜知道如何找到 Django 项目。
2.  接下来，我们创建了一个名为`core`的新 Celery 实例，并将值赋给一个名为`app`的变量。
3.  然后我们从`django.conf`的 settings 对象中加载 celery 配置值。我们使用了`namespace="CELERY"`来防止与其他 Django 场景的冲突。换句话说，芹菜的所有配置设置都必须以`CELERY_`为前缀。
4.  最后，`app.autodiscover_tasks()`告诉 Celery 从`settings.INSTALLED_APPS`中定义的应用程序中寻找 Celery 任务。

将以下代码添加到 *core/__init__。py* :

```
`from .celery import app as celery_app

__all__ = ("celery_app",)` 
```

最后，用以下 Celery 设置更新 *core/settings.py* 文件，以便它可以连接到 Redis:

```
`CELERY_BROKER_URL = "redis://redis:6379"
CELERY_RESULT_BACKEND = "redis://redis:6379"` 
```

构建新容器以确保一切正常工作:

```
`$ docker-compose up -d --build` 
```

查看每个服务的日志，看它们是否准备好，没有错误:

```
`$ docker-compose logs 'web'
$ docker-compose logs 'celery'
$ docker-compose logs 'celery-beat'
$ docker-compose logs 'redis'` 
```

如果一切顺利，我们现在有四个容器，每个都有不同的服务。

现在我们准备创建一个示例任务，看看它是否能正常工作。

### 创建任务

创建一个名为 *core/tasks.py* 的新文件，并为一个刚刚登录到控制台的示例任务添加以下代码:

```
`from celery import shared_task
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

@shared_task
def sample_task():
    logger.info("The sample task just ran.")` 
```

### 计划任务

在您的 *settings.py* 文件的末尾，添加以下代码来调度`sample_task`每分钟运行一次，使用芹菜节拍:

```
`CELERY_BEAT_SCHEDULE = {
    "sample_task": {
        "task": "core.tasks.sample_task",
        "schedule": crontab(minute="*/1"),
    },
}` 
```

这里，我们使用 [CELERY_BEAT_SCHEDULE](https://docs.celeryq.dev/en/latest/userguide/configuration.html#std:setting-beat_schedule) 设置定义了一个周期性任务。我们给这个任务命名为`sample_task`，然后声明了两个设置:

1.  `task`声明要运行的任务。
2.  `schedule`设置任务运行的时间间隔。这可以是整数、时间增量或 crontab。我们为我们的任务使用了一个 crontab 模式，告诉它每分钟运行一次。你可以在这里找到更多关于芹菜的时间安排的信息。

确保添加导入:

```
`from celery.schedules import crontab

import core.tasks` 
```

重新启动容器以获取新设置:

```
`$ docker-compose up -d --build` 
```

完成后，看看容器中的芹菜段:

```
`$ docker-compose logs -f 'celery'` 
```

您应该会看到类似如下的内容:

```
`celery_1  |  -------------- [queues]
celery_1  |                 .> celery           exchange=celery(direct) key=celery
celery_1  |
celery_1  |
celery_1  | [tasks]
celery_1  |   . core.tasks.sample_task` 
```

我们可以看到，芹菜拿起了我们的样本任务，`core.tasks.sample_task`。

每分钟您都应该在日志中看到一行，以“sample task 刚刚运行”结束：

```
`celery_1  | [2021-07-01 03:06:00,003: INFO/MainProcess]
              Task core.tasks.sample_task[b8041b6c-bf9b-47ce-ab00-c37c1e837bc7] received
celery_1  | [2021-07-01 03:06:00,004: INFO/ForkPoolWorker-8]
              core.tasks.sample_task[b8041b6c-bf9b-47ce-ab00-c37c1e837bc7]:
              The sample task just ran.` 
```

## 自定义 Django 管理命令

Django 提供了[个](https://docs.djangoproject.com/en/3.2/ref/django-admin/#available-commands)的内置`django-admin`命令，比如:

*   `migrate`
*   `startproject`
*   `startapp`
*   `dumpdata`
*   `makemigrations`

除了内置命令，Django 还让我们可以选择创建自己的[定制命令](https://docs.djangoproject.com/en/3.2/howto/custom-management-commands/):

> 自定义管理命令对于运行独立脚本或从 UNIX crontab 或 Windows 计划任务控制面板定期执行的脚本特别有用。

因此，我们将首先配置一个新命令，然后使用 Celery Beat 自动运行它。

首先创建一个名为*orders/management/commands/my _ custom _ command . py*的新文件。然后，添加运行它所需的最少代码:

```
`from django.core.management.base import BaseCommand, CommandError

class Command(BaseCommand):
    help = "A description of the command"

    def handle(self, *args, **options):
        pass` 
```

`BaseCommand`有一些[方法](https://docs.djangoproject.com/en/3.2/howto/custom-management-commands/#methods)可以被覆盖，但是唯一需要的方法是`handle`。`handle`是自定义命令的入口点。换句话说，当我们运行命令时，这个方法被调用。

为了测试，我们通常只添加一个快速打印语句。然而，根据 Django 文档，建议使用`stdout.write`:

> 当您使用管理命令并希望提供控制台输出时，您应该写入 self.stdout 和 self.stderr，而不是直接打印到 stdout 和 stderr。通过使用这些代理，测试您的定制命令变得更加容易。还要注意，您不需要用换行符结束消息，它会自动添加，除非您指定了结束参数。

所以，添加一个`self.stdout.write`命令:

```
`from django.core.management.base import BaseCommand, CommandError

class Command(BaseCommand):
    help = "A description of the command"

    def handle(self, *args, **options):
        self.stdout.write("My sample command just ran.") # NEW` 
```

要进行测试，从命令行运行:

```
`$ docker-compose exec web python manage.py my_custom_command` 
```

您应该看到:

```
`My sample command just ran.` 
```

有了那个，让我们把一切都绑在一起！

## 使用芹菜节拍安排自定义命令

既然我们已经启动了容器，测试了我们可以安排一个任务定期运行，并编写了一个定制的 Django Admin 示例命令，那么是时候配置 Celery Beat 定期运行这个定制命令了。

### 设置

在这个项目中，我们有一个非常基本的应用程序，叫做订单。包含`Product`和`Order`两个型号。让我们创建一个自定义命令，发送当天已确认订单的电子邮件报告。

首先，我们将通过这个项目中包含的 fixture 向数据库添加一些产品和订单:

```
`$ docker-compose exec web python manage.py loaddata products.json` 
```

接下来，通过 Django 管理界面添加一些样本订单。为此，首先创建一个超级用户:

```
`$ docker-compose exec web python manage.py createsuperuser` 
```

出现提示时，填写用户名、电子邮件和密码。然后在网络浏览器中导航至[http://127 . 0 . 0 . 1:1337/admin](http://127.0.0.1:1337/admin)。使用您刚刚创建的超级用户登录并创建几个订单。确保至少有一个人有今天的`confirmed_date`。

让我们为电子邮件报告创建一个新的自定义命令。

创建一个名为*orders/management/commands/email _ report . py*的文件:

```
`from datetime import timedelta, time, datetime

from django.core.mail import mail_admins
from django.core.management import BaseCommand
from django.utils import timezone
from django.utils.timezone import make_aware

from orders.models import Order

today = timezone.now()
tomorrow = today + timedelta(1)
today_start = make_aware(datetime.combine(today, time()))
today_end = make_aware(datetime.combine(tomorrow, time()))

class Command(BaseCommand):
    help = "Send Today's Orders Report to Admins"

    def handle(self, *args, **options):
        orders = Order.objects.filter(confirmed_date__range=(today_start, today_end))

        if orders:
            message = ""

            for order in orders:
                message += f"{order}  \n"

            subject = (
                f"Order Report for {today_start.strftime('%Y-%m-%d')} "
                f"to {today_end.strftime('%Y-%m-%d')}"
            )

            mail_admins(subject=subject, message=message, html_message=None)

            self.stdout.write("E-mail Report was sent.")
        else:
            self.stdout.write("No orders confirmed today.")` 
```

在代码中，我们在数据库中查询今天的`confirmed_date`订单，将订单组合成一条消息作为电子邮件正文，并使用 Django 的内置`mail_admins`命令将电子邮件发送给管理员。

添加一个虚拟的管理员电子邮件，并将`EMAIL_BACKEND`设置为使用[控制台后端](https://docs.djangoproject.com/en/3.2/topics/email/#console-backend)，这样电子邮件就被发送到 stdout，在设置文件中:

现在应该可以从终端运行我们的新命令了。

```
`$ docker-compose exec web python manage.py email_report` 
```

输出应该如下所示:

```
`Content-Type: text/plain; charset="utf-8"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
Subject: [Django] Order Report for 2021-07-01 to 2021-07-02
From: [[email protected]](/cdn-cgi/l/email-protection)
To: [[email protected]](/cdn-cgi/l/email-protection)
Date: Thu, 01 Jul 2021 12:15:50 -0000
Message-ID: <162514175053[[email protected]](/cdn-cgi/l/email-protection)>

Order: 3947963f-1860-44d1-9b9a-4648fed04581 - product: Coffee
Order: ff449e6e-3dfd-48a8-9d5c-79a145d08253 - product: Rice

-------------------------------------------------------------------------------
E-mail Report was sent.` 
```

### 芹菜搅打

我们现在需要创建一个定期任务来每天运行这个命令。

向 *core/tasks.py* 添加新任务:

```
`from celery import shared_task
from celery.utils.log import get_task_logger
from django.core.management import call_command # NEW

logger = get_task_logger(__name__)

@shared_task
def sample_task():
    logger.info("The sample task just ran.")

# NEW
@shared_task
def send_email_report():
    call_command("email_report", )` 
```

因此，首先我们添加了一个`call_command`导入，用于以编程方式调用 django-admin 命令。在新任务中，我们使用带有自定义命令名称的`call_command`作为参数。

要调度该任务，请打开 *core/settings.py* 文件，并更新`CELERY_BEAT_SCHEDULE`设置以包含新任务:

```
`CELERY_BEAT_SCHEDULE = {
    "sample_task": {
        "task": "core.tasks.sample_task",
        "schedule": crontab(minute="*/1"),
    },
    "send_email_report": {
        "task": "core.tasks.send_email_report",
        "schedule": crontab(hour="*/1"),
    },
}` 
```

这里我们向`CELERY_BEAT_SCHEDULE`添加了一个名为`send_email_report`的新条目。正如我们对上一个任务所做的那样，我们声明了应该运行哪个任务——例如，`core.tasks.send_email_report`——并使用 crontab 模式来设置循环。

重新启动容器以确保新设置生效:

```
`$ docker-compose up -d --build` 
```

打开与`celery`服务相关的日志:

```
`$ docker-compose logs -f 'celery'` 
```

您应该会看到列出的`send_email_report`:

```
`celery_1  |  -------------- [queues]
celery_1  |                 .> celery           exchange=celery(direct) key=celery
celery_1  |
celery_1  |
celery_1  | [tasks]
celery_1  |   . core.tasks.sample_task
celery_1  |   . core.tasks.send_email_report` 
```

大约一分钟后，您应该看到电子邮件报告已发送:

```
`celery_1  | [2021-07-01 12:22:00,071: WARNING/ForkPoolWorker-8] Content-Type: text/plain; charset="utf-8"
celery_1  | MIME-Version: 1.0
celery_1  | Content-Transfer-Encoding: 7bit
celery_1  | Subject: [Django] Order Report for 2021-07-01 to 2021-07-02
celery_1  | From: [[email protected]](/cdn-cgi/l/email-protection)
celery_1  | To: [[email protected]](/cdn-cgi/l/email-protection)
celery_1  | Date: Thu, 01 Jul 2021 12:22:00 -0000
celery_1  | Message-ID: <162514212006[[email protected]](/cdn-cgi/l/email-protection)>
celery_1  |
celery_1  | Order: 3947963f-1860-44d1-9b9a-4648fed04581 - product: Coffee
celery_1  | Order: ff449e6e-3dfd-48a8-9d5c-79a145d08253 - product: Rice
celery_1  |
celery_1  |
celery_1  | [2021-07-01 12:22:00,071: WARNING/ForkPoolWorker-8] -------------------------------------------------------------------------------
celery_1  | [2021-07-01 12:22:00,071: WARNING/ForkPoolWorker-8]
celery_1  |
celery_1  | [2021-07-01 12:22:00,071: WARNING/ForkPoolWorker-8] E-mail Report was sent.` 
```

## 结论

在本文中，我们指导您为芹菜、芹菜段和 Redis 设置 Docker 容器。然后，我们展示了如何创建一个定制的 Django 管理命令和一个使用 Celery Beat 自动运行该命令的周期性任务。

想要更多吗？

1.  设置 [Flower](/blog/django-and-celery/#flower-dashboard) 来监控和管理芹菜作业和工人
2.  [用单元测试和集成测试来测试芹菜任务](/blog/django-and-celery/#tests)

从[回购](https://github.com/testdrivenio/django-celery-beat)中抓取代码。

> 姜戈+芹菜系列:
> 
> 1.  与 Django 和 Celery 的异步任务
> 2.  [在 Django 用芹菜和 Docker 处理周期性任务](/blog/django-celery-periodic-tasks/)(本文！)
> 3.  [自动重试失败的芹菜任务](/blog/retrying-failed-celery-tasks/)
> 4.  [处理芹菜和数据库事务](/blog/celery-database-transactions/)