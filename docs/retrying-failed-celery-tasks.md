# 自动重试失败的芹菜任务

> 原文：<https://testdriven.io/blog/retrying-failed-celery-tasks/>

在本文中，我们将看看如何自动重试失败的芹菜任务。

> 姜戈+芹菜系列:
> 
> 1.  与 Django 和 Celery 的异步任务
> 2.  [在 Django 用芹菜和码头工人处理定期任务](/blog/django-celery-periodic-tasks/)
> 3.  [自动重试失败的芹菜任务](/blog/retrying-failed-celery-tasks/)(本文！)
> 4.  [处理芹菜和数据库事务](/blog/celery-database-transactions/)

## 目标

阅读后，您应该能够:

1.  使用`retry`方法和装饰器参数重试失败的芹菜任务
2.  重试失败的任务时使用指数补偿
3.  使用基于类的任务来重用重试参数

## 芹菜任务

> 你可以在 [GitHub](https://github.com/testdrivenio/django-celery-project) 上找到这篇文章的源代码。

假设我们有这样一个芹菜任务:

```py
`@shared_task
def task_process_notification():
    if not random.choice([0, 1]):
        # mimic random error
        raise Exception()

    requests.post('https://httpbin.org/delay/5')` 
```

在现实世界中，这可能会调用内部或外部的第三方服务。不考虑服务，假设它是非常不可靠的，尤其是在高峰期。我们如何处理失败？

> 值得注意的是，许多芹菜初学者对为什么有些文章使用`app.task`而有些文章使用`shared_task`感到困惑。嗯，`shared_task`让您定义 Celery 任务，而不必导入 Celery 实例，这样可以使您的任务代码更加可重用。

## 解决方案 1:使用 Try/Except 块

我们可以使用 try/except 块来捕捉异常并引发`retry`:

```py
`@shared_task(bind=True)
def task_process_notification(self):
    try:
        if not random.choice([0, 1]):
            # mimic random error
            raise Exception()

        requests.post('https://httpbin.org/delay/5')
    except Exception as e:
        logger.error('exception raised, it would be retry after 5 seconds')
        raise self.retry(exc=e, countdown=5)` 
```

注意事项:

1.  由于我们将`bind`设置为`True`，这是一个[绑定的](https://docs.celeryq.dev/en/latest/userguide/tasks.html#bound-tasks)任务，因此任务的第一个参数将始终是当前任务实例(`self`)。正因为如此，我们可以调用`self.retry`来重试失败的任务。
2.  请记住`raise`由`self.retry`方法返回的异常以使其工作。
3.  通过将`countdown`参数设置为 5，任务将在延迟 5 秒后重试。

让我们在 Python shell 中运行下面的代码:

```py
`>>> from polls.tasks import task_process_notification
>>> task_process_notification.delay()` 
```

您应该会在 Celery worker 终端输出中看到如下输出:

```py
`Task polls.tasks.task_process_notification[06e1f985-90d4-4453-9870-fab57c5885c4] retry: Retry in 5s: Exception()
Task polls.tasks.task_process_notification[06e1f985-90d4-4453-9870-fab57c5885c4] retry: Retry in 5s: Exception()
Task polls.tasks.task_process_notification[06e1f985-90d4-4453-9870-fab57c5885c4] succeeded in 3.3638455480104312s: None` 
```

可以看到，芹菜任务失败了两次，第三次成功了。

## 解决方案 2:任务重试装饰器

Celery [4.0](https://docs.celeryq.dev/en/latest/history/whatsnew-4.0.html#task-auto-retry-decorator) 增加了对重试的内置支持，因此您可以让异常冒泡并在装饰器中指定如何处理它:

```py
`@shared_task(bind=True, autoretry_for=(Exception,), retry_kwargs={'max_retries': 7, 'countdown': 5})
def task_process_notification(self):
    if not random.choice([0, 1]):
        # mimic random error
        raise Exception()

    requests.post('https://httpbin.org/delay/5')` 
```

注意事项:

1.  `autoretry_for`获取您想要重试的异常类型的列表/元组。
2.  `retry_kwargs`获取附加[选项](https://docs.celeryq.dev/en/latest/userguide/tasks.html#list-of-options)的字典，用于指定如何执行自动重试。在上面的例子中，任务将在 5 秒钟的延迟后重试(通过`countdown`)，并且最多允许 7 次重试尝试(通过`max_retries`)。芹菜将在 7 次尝试失败后停止重试，并引发异常。

## 指数后退

如果您的芹菜任务需要向第三方服务发送请求，那么使用[指数回退](https://en.wikipedia.org/wiki/Exponential_backoff)来避免服务不堪重负是个好主意。

Celery 默认支持这一点:

```py
`@shared_task(bind=True, autoretry_for=(Exception,), retry_backoff=True, retry_kwargs={'max_retries': 5})
def task_process_notification(self):
    if not random.choice([0, 1]):
        # mimic random error
        raise Exception()

    requests.post('https://httpbin.org/delay/5')` 
```

在此示例中，第一次重试应该在 1 秒后运行，第二次在 2 秒后运行，第三次在 4 秒后运行，第四次在 8 秒后运行，依此类推:

```py
`[02:09:59,014: INFO/ForkPoolWorker-8] Task polls.tasks.task_process_notification[fbe041b6-e6c1-453d-9cc9-cb99236df6ff] retry: Retry in 1s: Exception()
[02:10:00,210: INFO/ForkPoolWorker-2] Task polls.tasks.task_process_notification[fbe041b6-e6c1-453d-9cc9-cb99236df6ff] retry: Retry in 2s: Exception()
[02:10:02,291: INFO/ForkPoolWorker-4] Task polls.tasks.task_process_notification[fbe041b6-e6c1-453d-9cc9-cb99236df6ff] retry: Retry in 4s: Exception()` 
```

您也可以将`retry_backoff`设置为一个数字，用作延迟因子:

```py
`@shared_task(bind=True, autoretry_for=(Exception,), retry_backoff=5, retry_kwargs={'max_retries': 5})
def task_process_notification(self):
    if not random.choice([0, 1]):
        # mimic random error
        raise Exception()

    requests.post('https://httpbin.org/delay/5')` 
```

示例:

```py
`[02:21:45,887: INFO/ForkPoolWorker-8] Task polls.tasks.task_process_notification[6a0b2682-74f5-410b-af1e-352069238f3d] retry: Retry in 5s: Exception()
[02:21:55,170: INFO/ForkPoolWorker-2] Task polls.tasks.task_process_notification[6a0b2682-74f5-410b-af1e-352069238f3d] retry: Retry in 10s: Exception()
[02:22:15,706: INFO/ForkPoolWorker-4] Task polls.tasks.task_process_notification[6a0b2682-74f5-410b-af1e-352069238f3d] retry: Retry in 20s: Exception()
[02:22:55,450: INFO/ForkPoolWorker-6] Task polls.tasks.task_process_notification[6a0b2682-74f5-410b-af1e-352069238f3d] retry: Retry in 40s: Exception()` 
```

默认情况下，指数补偿还会引入[随机抖动](https://docs.celeryq.dev/en/stable/userguide/tasks.html#Task.retry_jitter)，以避免所有任务同时运行。

## 随机性

当您为 Celery 任务(需要向另一个服务发送请求)构建自定义重试策略时，您应该在延迟计算中添加一些随机性，以防止所有任务同时执行导致[蜂拥](https://en.wikipedia.org/wiki/Thundering_herd_problem)。

芹菜也给你盖上了`retry_jitter`:

```py
`@shared_task(bind=True, autoretry_for=(Exception,), retry_backoff=5, retry_jitter=True, retry_kwargs={'max_retries': 5})
def task_process_notification(self):
    if not random.choice([0, 1]):
        # mimic random error
        raise Exception()

    requests.post('https://httpbin.org/delay/5')` 
```

该选项默认设置为`True`，这有助于防止当您使用 Celery 的内置`retry_backoff`时出现雷群问题。

## 任务基类

如果您发现自己在 Celery 任务装饰器中编写了相同的重试参数，您可以(从 Celery [4.4](https://docs.celeryq.dev/en/v4.4.4/whatsnew-4.4.html#task-class-definitions-can-now-have-retry-attributes) 开始)在一个基类中定义重试参数，然后您可以将它用作 Celery 任务中的基类:

```py
`class BaseTaskWithRetry(celery.Task):
    autoretry_for = (Exception, KeyError)
    retry_kwargs = {'max_retries': 5}
    retry_backoff = True

@shared_task(bind=True, base=BaseTaskWithRetry)
def task_process_notification(self):
    raise Exception()` 
```

因此，如果您在 Python shell 中运行该任务，您将看到以下内容:

```py
`[03:12:29,002: INFO/ForkPoolWorker-8] Task polls.tasks.task_process_notification[3231ef9b-00c7-4ab1-bf0b-2fdea6fa8348] retry: Retry in 1s: Exception()
[03:12:30,445: INFO/ForkPoolWorker-8] Task polls.tasks.task_process_notification[3231ef9b-00c7-4ab1-bf0b-2fdea6fa8348] retry: Retry in 2s: Exception()
[03:12:33,080: INFO/ForkPoolWorker-8] Task polls.tasks.task_process_notification[3231ef9b-00c7-4ab1-bf0b-2fdea6fa8348] retry: Retry in 3s: Exception()` 
```

## 结论

在这篇芹菜文章中，我们研究了如何自动重试失败的芹菜任务。

同样，本文的源代码可以在 [GitHub](https://github.com/testdrivenio/django-celery-project) 上找到。

感谢您的阅读。如果您有任何问题，请随时[联系我](/authors/yin/)。

> 姜戈+芹菜系列:
> 
> 1.  与 Django 和 Celery 的异步任务
> 2.  [在 Django 用芹菜和码头工人处理定期任务](/blog/django-celery-periodic-tasks/)
> 3.  [自动重试失败的芹菜任务](/blog/retrying-failed-celery-tasks/)(本文！)
> 4.  [处理芹菜和数据库事务](/blog/celery-database-transactions/)