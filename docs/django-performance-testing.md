# Django 中的自动化性能测试

> 原文：<https://testdriven.io/blog/django-performance-testing/>

低效的数据库查询是 Django 最常见的性能缺陷之一。尤其是 N+1 查询会在早期对应用程序的性能产生负面影响。当您使用针对每个记录的单独查询从关联表中选择记录，而不是在单个查询中获取所有记录时，就会出现这种情况。不幸的是，这种低效很容易被 Django ORM 引入。也就是说，它们是可以通过自动化测试快速发现和预防的。

这篇文章着眼于如何:

1.  测试一个请求执行的查询数量以及查询的持续时间
2.  使用 [nplusone](https://github.com/jmcarp/nplusone) 包防止 N+1 次查询

## N+1 个查询

> 我们将在这篇文章中使用的示例应用程序可以在 [GitHub](https://github.com/testdrivenio/django-performance-testing) 上找到。

比方说，您正在使用一个 Django 应用程序，它有以下模型:

```
`# courses/models.py

from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)

    def __str__(self):
        return self.name

class Course(models.Model):
    title = models.CharField(max_length=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)

    def __str__(self):
        return self.title` 
```

现在，如果您的任务是创建一个新视图，用于返回所有课程的 JSON 响应，包括标题和作者姓名，您可以编写以下代码:

```
`# courses/views.py

from django.http import JsonResponse

from courses.models import Course

def all_courses(request):
    queryset = Course.objects.all()

    courses = []

    for course in queryset:
        courses.append(
            {"title": course.title, "author": course.author.name}
        )

    return JsonResponse(courses, safe=False)` 
```

这段代码可以工作，但是效率非常低，因为它会进行太多的数据库查询:

*   1 获取所有课程的查询
*   在每次迭代中获取分支的 n 次查询

在解决这个问题之前，让我们看看进行了多少次查询，并测量执行时间。

## 度量中间件

您会注意到该项目包含了定制的中间件，它计算并记录每个请求的执行时间:

```
`# core/middleware.py

import logging
import time

from django.db import connection, reset_queries

def metric_middleware(get_response):
    def middleware(request):
        reset_queries()

        # Get beginning stats
        start_queries = len(connection.queries)
        start_time = time.perf_counter()

        # Process the request
        response = get_response(request)

        # Get ending stats
        end_time = time.perf_counter()
        end_queries = len(connection.queries)

        # Calculate stats
        total_time = end_time - start_time
        total_queries = end_queries - start_queries

        # Log the results
        logger = logging.getLogger("debug")
        logger.debug(f"Request: {request.method}  {request.path}")
        logger.debug(f"Number of Queries: {total_queries}")
        logger.debug(f"Total time: {(total_time):.2f}s")

        return response

    return middleware` 
```

运行数据库种子命令，向数据库添加 10 名作者和 100 门课程:

```
`$ python manage.py seed_db` 
```

Django 开发服务器启动并运行后，在浏览器中导航到[http://localhost:8000/courses/](http://localhost:8000/courses/)。您应该会看到 JSON 响应。回到您的终端，记下指标:

```
`Request: GET /courses/
Number of Queries: 101
Total time: 0.10s` 
```

这是一个很大的疑问！这是非常低效的。每增加一个作者和课程都需要一个额外的数据库查询，所以随着数据库的增长，性能会继续下降。幸运的是，解决这个问题非常简单:您可以添加一个`select_related`方法来创建一个 SQL join，它将在初始数据库查询中包含作者。

```
`queryset = Course.objects.select_related("author").all()` 
```

在进行任何代码更改之前，让我们先从一些测试开始。

## 性能测试

从下面的测试开始，它使用 django _ assert _ num _ queriespy test fixture 来确保当数据库中存在一个或多个作者和课程记录时，数据库只被命中一次:

```
`import json

import pytest
from faker import Faker
from django.test import override_settings

from courses.models import Course, Author

@pytest.mark.django_db
def test_number_of_sql_queries_all_courses(client, django_assert_num_queries):
    fake = Faker()

    author_name = fake.name()
    author = Author(name=author_name)
    author.save()
    course_title = fake.sentence(nb_words=4)
    course = Course(title=course_title, author=author)
    course.save()

    with django_assert_num_queries(1):
        res = client.get("/courses/")
        data = json.loads(res.content)

        assert res.status_code == 200
        assert len(data) == 1

        author_name = fake.name()
        author = Author(name=author_name)
        author.save()
        course_title = fake.sentence(nb_words=4)
        course = Course(title=course_title, author=author)
        course.save()

        res = client.get("/courses/")
        data = json.loads(res.content)

        assert res.status_code == 200
        assert len(data) == 2` 
```

> 不使用 pytest？使用 [assertNumQueries](https://docs.djangoproject.com/en/3.2/topics/testing/tools/#django.test.TransactionTestCase.assertNumQueries) 测试方法代替`django_assert_num_queries`。

此外，我们还可以使用 [nplusone](https://github.com/jmcarp/nplusone) 来防止引入未来的 N+1 个查询。在安装完包并将其添加到设置文件之后，您可以使用`@override_settings`装饰器将它添加到您的测试中:

```
`...
@pytest.mark.django_db
@override_settings(NPLUSONE_RAISE=True)
def test_number_of_sql_queries_all_courses(client, django_assert_num_queries):
    ...` 
```

或者，如果您想在整个测试套件中自动启用 nplusone，那么将以下内容添加到您的测试根 *conftest.py* 文件中:

```
`from django.conf import settings

def pytest_configure(config):
    settings.NPLUSONE_RAISE = True` 
```

回到示例应用程序，然后运行测试。您应该会看到以下错误:

```
`nplusone.core.exceptions.NPlusOneError: Potential n+1 query detected on `Course.author`` 
```

现在，进行推荐的更改——添加`select_related`方法——然后再次运行测试。他们现在应该通过了。

## 结论

这篇文章介绍了如何使用 [nplusone](https://github.com/jmcarp/nplusone) 包在代码库中自动阻止 N+1 个查询，并使用`django_assert_num_queries` pytest fixture 测试执行的查询数量。

随着应用程序的增长和用户的增加，这应该有助于防止性能瓶颈。如果您将它添加到现有的代码库中，您可能需要花费一些时间来修复中断的查询，以便它们具有恒定的数据库命中次数。如果在修复和优化之后仍然遇到性能问题，您可能需要添加额外的缓存层，对数据库的某些部分进行反规范化，和/或配置数据库索引。