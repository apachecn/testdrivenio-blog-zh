# 使用 Celery 和 Django 数据库事务

> 原文：<https://testdriven.io/blog/celery-database-transactions/>

在本文中，我们将研究如何在数据库提交事务之前阻止依赖于 Django 数据库事务的 Celery 任务执行。这是一个相当普遍的问题。

> 姜戈+芹菜系列:
> 
> 1.  与 Django 和 Celery 的异步任务
> 2.  [在 Django 用芹菜和码头工人处理定期任务](/blog/django-celery-periodic-tasks/)
> 3.  [自动重试失败的芹菜任务](/blog/retrying-failed-celery-tasks/)
> 4.  [处理芹菜和数据库事务](/blog/celery-database-transactions/)(本文！)

## 目标

阅读后，您应该能够:

1.  描述什么是数据库事务以及如何在 Django 中使用它
2.  解释为什么芹菜工会出现`DoesNotExist`错误，以及如何解决
3.  防止任务在数据库提交事务之前执行

## 什么是数据库事务？

数据库事务是作为一个单元提交(应用于数据库)或回滚(从数据库中撤消)的工作单元。

大多数数据库使用以下模式:

1.  开始交易。
2.  执行一组数据操作和/或查询。
3.  如果没有错误发生，则提交事务。
4.  如果出现错误，则回滚事务。

如您所见，事务是让您的数据远离混乱的一种非常有用的方式。

## 如何在 Django 中使用数据库事务

> 你可以在 [Github](https://github.com/testdrivenio/django-celery-project) 上找到这篇文章的源代码。如果您在阅读本文时决定使用这段代码，请记住用户名必须是惟一的。你可以使用一个随机的用户名生成器进行测试，比如 [Faker](https://github.com/joke2k/faker) 。

让我们先来看看 Django 的观点:

```py
`def test_view(request):
    user = User.objects.create_user('john', '[[email protected]](/cdn-cgi/l/email-protection)', 'johnpassword')
    logger.info(f'create user {user.pk}')
    raise Exception('test')` 
```

当你访问这个视图时会发生什么？

### 默认行为

Django 的默认行为是[自动提交](https://docs.djangoproject.com/en/3.2/topics/db/transactions/#autocommit):每个查询都直接提交给数据库，除非事务是活动的。换句话说，通过自动提交，每个查询都会启动一个事务，并提交或回滚该事务。如果您有一个包含三个查询的视图，那么每个查询将一个接一个地运行。如果一个失败了，另外两个就会被提交。

因此，在上面的视图中，在提交事务后*引发异常，创建用户`john`。*

### 显式控制

如果您希望对数据库事务有更多的控制，您可以用 [transaction.atomic](https://docs.djangoproject.com/en/3.2/topics/db/transactions/#controlling-transactions-explicitly) 覆盖默认行为。在这种模式下，在调用视图函数之前，Django 启动一个事务。如果响应没有问题，Django 就提交事务。另一方面，如果视图产生异常，Django 回滚事务。如果有三个查询，其中一个失败了，那么所有查询都不会提交。

因此，让我们使用`transaction.atomic`重写视图:

```py
`def transaction_test(request):
    with transaction.atomic():
        user = User.objects.create_user('john1', '[[email protected]](/cdn-cgi/l/email-protection)', 'johnpassword')
        logger.info(f'create user {user.pk}')
        raise Exception('force transaction to rollback')` 
```

现在`user create`操作会在异常出现时回滚，所以最终不会创建用户。

是一个非常有用的工具，可以让你的数据有条理，尤其是当你需要在模型中操作数据的时候。

它也可以像这样用作装饰器:

```py
`@transaction.atomic
def transaction_test2(request):
    user = User.objects.create_user('john1', '[[email protected]](/cdn-cgi/l/email-protection)hebeatles.com', 'johnpassword')
    logger.info(f'create user {user.pk}')
    raise Exception('force transaction to rollback')` 
```

因此，如果视图中出现了一些错误，而我们没有发现它，那么事务将会回滚。

如果您想将`transaction.atomic`用于所有视图功能，您可以在 Django 设置文件中将`ATOMIC_REQUESTS`设置为`True`:

```py
`ATOMIC_REQUESTS=True

# or

DATABASES["default"]["ATOMIC_REQUESTS"] = True` 
```

然后，您可以覆盖该行为，以便视图以自动提交模式运行:

```py
`@transaction.non_atomic_requests` 
```

## 不存在异常

如果您对 Django 如何管理数据库事务没有很好的理解，当您在 Celery worker 中遇到与数据库相关的随机错误时，可能会很困惑。

让我们看一个例子:

```py
`@transaction.atomic
def transaction_celery(request):
    username = random_username()
    user = User.objects.create_user(username, '[[email protected]](/cdn-cgi/l/email-protection)', 'johnpassword')
    logger.info(f'create user {user.pk}')
    task_send_welcome_email.delay(user.pk)

    time.sleep(1)
    return HttpResponse('test')` 
```

任务代码如下所示:

```py
`@shared_task()
def task_send_welcome_email(user_pk):
    user = User.objects.get(pk=user_pk)
    logger.info(f'send email to {user.email}  {user.pk}')` 
```

1.  因为视图使用了`transaction.atomic`装饰器，所以所有的数据库操作只有在视图中没有出现错误时才会被提交，包括芹菜任务。
2.  这个任务相当简单:我们创建一个用户，然后将主键传递给发送欢迎电子邮件的任务。
3.  `time.sleep(1)`用于引入竞争条件。

运行时，您会看到以下错误:

```py
`django.contrib.auth.models.User.DoesNotExist: User matching query does not exist.` 
```

为什么？

1.  任务入队后，我们暂停 1 秒钟。
2.  由于任务立即执行，`user = User.objects.get(pk=user_pk)`失败，因为 Django 中的事务尚未提交，用户不在数据库中。

## 解决办法

有三种方法可以解决这个问题:

1.  禁用数据库事务，这样 Django 就可以使用`autocommit`特性。为此，您可以简单地移除`transaction.atomic`装饰器。然而，不推荐这样做，因为原子数据库事务是一个强大的工具。

2.  强制芹菜任务在一段时间后运行。

    例如，要暂停 10 秒钟:

    ```py
    task_send_welcome_email.apply_async(args=[user.pk], countdown=10) 
    ```

3.  Django 有一个名为`transaction.on_commit`的回调函数，在事务成功提交后执行。要使用它，请按如下方式更新视图:

    ```py
    @transaction.atomic
    def transaction_celery2(request):
        username = random_username()
        user = User.objects.create_user(username, '[[email protected]](/cdn-cgi/l/email-protection)', 'johnpassword')
        logger.info(f'create user {user.pk}')
        # the task does not get called until after the transaction is committed
        transaction.on_commit(lambda: task_send_welcome_email.delay(user.pk))

        time.sleep(1)
        return HttpResponse('test') 
    ```

    现在，直到数据库事务提交之后才调用该任务。所以，当 Celery worker 找到用户时，它可以被找到，因为 worker 中的代码总是在 Django 数据库事务成功提交后的运行*。*

    **这是推荐的方案**。

> 值得注意的是，您可能不希望事务立即提交，尤其是在大规模环境中运行时。如果数据库或实例处于高利用率状态，强制提交只会增加现有的使用率。在这种情况下，您可能希望使用第二种解决方案，并等待足够长的时间(也许 20 秒)，以确保在任务执行之前对数据库进行了更改。

## 测试

Django 的`TestCase`将每个测试包装在一个事务中，然后在每个测试后回滚。因为没有事务被提交，`on_commit()`也不会运行。因此，如果您需要测试在`on_commit`回调中触发的代码，您可以在测试代码中使用 [TransactionTestCase](https://docs.djangoproject.com/en/3.2/topics/testing/tools/#transactiontestcase) 或[test case . captureoncommitcallbacks()](https://docs.djangoproject.com/en/3.2/topics/testing/tools/#django.test.TestCase.captureOnCommitCallbacks)。

## 芹菜任务中的数据库事务

如果您的 Celery 任务需要更新数据库记录，那么在 Celery 任务中使用数据库事务是有意义的。

一个简单的方法是`with transaction.atomic()`:

```py
`@shared_task()
def task_transaction_test():
    with transaction.atomic():
        from .views import random_username
        username = random_username()
        user = User.objects.create_user(username, '[[email protected]](/cdn-cgi/l/email-protection)', 'johnpassword')
        user.save()
        logger.info(f'send email to {user.pk}')
        raise Exception('test')` 
```

更好的方法是编写一个支持`transaction`的自定义`decorator`:

```py
`class custom_celery_task:
    """
 This is a decorator we can use to add custom logic to our Celery task
 such as retry or database transaction
 """
    def __init__(self, *args, **kwargs):
        self.task_args = args
        self.task_kwargs = kwargs

    def __call__(self, func):
        @functools.wraps(func)
        def wrapper_func(*args, **kwargs):
            try:
                with transaction.atomic():
                    return func(*args, **kwargs)
            except Exception as e:
                # task_func.request.retries
                raise task_func.retry(exc=e, countdown=5)

        task_func = shared_task(*self.task_args, **self.task_kwargs)(wrapper_func)
        return task_func

...

@custom_celery_task(max_retries=5)
def task_transaction_test():
    # do something` 
```

## 结论

本文研究了如何让 Celery 很好地与 Django 数据库事务一起工作。

本文的源代码可以在 [GitHub](https://github.com/testdrivenio/django-celery-project) 上找到。

感谢您的阅读。如果您有任何问题，请随时[联系我](/authors/yin/)。

> 姜戈+芹菜系列:
> 
> 1.  与 Django 和 Celery 的异步任务
> 2.  [在 Django 用芹菜和码头工人处理定期任务](/blog/django-celery-periodic-tasks/)
> 3.  [自动重试失败的芹菜任务](/blog/retrying-failed-celery-tasks/)
> 4.  [处理芹菜和数据库事务](/blog/celery-database-transactions/)(本文！)