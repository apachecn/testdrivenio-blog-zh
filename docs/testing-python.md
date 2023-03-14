# 用 Python 测试

> 原文：<https://testdriven.io/blog/testing-python/>

自动化测试一直是软件开发中的热门话题，但在持续集成和微服务时代，它被谈论得更多。有许多工具可以帮助您在 Python 项目中编写、运行和评估测试。让我们来看看其中的几个。

> 本文是[完整 Python](/guides/complete-python/) 指南的一部分:
> 
> 1.  [现代 Python 环境——依赖性和工作空间管理](/blog/python-environments/)
> 2.  [Python 中的测试](/blog/testing-python/)(本文！)
> 3.  [Python 中的现代测试驱动开发](/blog/modern-tdd/)
> 4.  [Python 代码质量](/blog/python-code-quality/)
> 5.  [Python 类型检查](/blog/python-type-checking/)
> 6.  [记录 Python 代码和项目](/blog/documenting-python/)
> 7.  [Python 项目工作流程](/blog/python-project-workflow/)

## pytest

虽然 Python 标准库附带了一个名为“nittest”的单元测试框架，但是 [pytest](https://docs.pytest.org/) 是测试 Python 代码的首选测试框架。

pytest 让它变得简单(而且有趣！)来编写、组织和运行测试。与 Python 标准库中的 unittest 相比，pytest:

1.  需要更少的样板代码，因此您的测试套件将更具可读性。
2.  支持简单的`assert`语句，与 unittest 中的`assertSomething`方法——如`assertEquals`、`assertTrue`和`assertContains`——相比，它可读性更好，也更容易记住。
3.  更新更频繁，因为它不是 Python 标准库的一部分。
4.  通过夹具系统简化测试状态的设置和拆除。
5.  使用功能方法。

另外，使用 pytest，您可以在所有 Python 项目中保持一致的风格。假设您的堆栈中有两个 web 应用程序——一个用 Django 构建，另一个用 Flask 构建。如果没有 pytest，您很可能会利用 Django 测试框架以及 Flask 扩展，如 Flask-Testing。所以，你的测试套件会有不同的风格。另一方面，使用 pytest，两个测试套件将具有一致的风格，使得从一个跳到另一个更加容易。

pytest 也有一个大型的、由社区维护的插件生态系统。

一些例子:

*   pytest-django -提供了一套专门用于测试 django 应用程序的工具
*   pytest-xdist -用于并行运行测试
*   添加代码覆盖支持
*   [pytest-instafail](https://github.com/pytest-dev/pytest-instafail) -立即显示故障和错误，而不是等到运行结束

> 要查看完整的插件列表，请查看文档中的[插件列表](https://docs.pytest.org/en/latest/reference/plugin_list.html)。

## 嘲弄的

自动化测试应该是快速的、隔离的/独立的、确定的/可重复的。因此，如果您需要测试向第三方 API 发出外部 HTTP 请求的代码，您应该真正模拟该请求。为什么？如果你不知道，那么具体的测试将会是-

1.  速度慢，因为它通过网络发出 HTTP 请求
2.  取决于第三方服务和网络本身的速度
3.  不确定性，因为根据 API 的响应，测试可能会产生不同的结果

> 模仿其他长时间运行的操作也是一个好主意，比如数据库查询和异步任务，因为自动化测试通常会在每次提交到源代码控制时频繁运行。

模仿是指在运行时用模仿对象替换真实对象的行为。因此，当被模仿的方法被调用时，我们只是返回一个预期的响应，而不是通过网络发送一个真正的 HTTP 请求。

例如:

```py
`import requests

def get_my_ip():
    response = requests.get(
        'http://ipinfo.io/json'
    )
    return response.json()['ip']

def test_get_my_ip(monkeypatch):
    my_ip = '123.123.123.123'

    class MockResponse:

        def __init__(self, json_body):
            self.json_body = json_body

        def json(self):
            return self.json_body

    monkeypatch.setattr(
        requests,
        'get',
        lambda *args, **kwargs: MockResponse({'ip': my_ip})
    )

    assert get_my_ip() == my_ip` 
```

这里发生了什么事？

我们使用 pytest 的 [monkeypatch](https://docs.pytest.org/en/stable/monkeypatch.html) fixture 将所有从`requests`模块对`get`方法的调用替换为总是返回`MockedResponse`实例的`lambda`回调。

> 我们使用一个对象，因为`requests`返回一个[响应](https://requests.readthedocs.io/en/latest/api/#requests.Response)对象。

我们可以用来自`unittest.mock`模块的 [create_autospec](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.create_autospec) 方法来简化测试。该方法创建一个模拟对象，该对象具有与作为参数传递的对象相同的属性和方法:

```py
`from unittest import mock

import requests
from requests import Response

def get_my_ip():
    response = requests.get(
        'http://ipinfo.io/json'
    )
    return response.json()['ip']

def test_get_my_ip(monkeypatch):
    my_ip = '123.123.123.123'
    response = mock.create_autospec(Response)
    response.json.return_value = {'ip': my_ip}

    monkeypatch.setattr(
        requests,
        'get',
        lambda *args, **kwargs: response
    )

    assert get_my_ip() == my_ip` 
```

尽管 pytest 推荐使用 monkeypatch 方法进行模仿，但是标准库中的 [pytest-mock](https://github.com/pytest-dev/pytest-mock/) 扩展和普通的 [unittest.mock](https://docs.python.org/3/library/unittest.mock.html) 库也是不错的方法。

## 代码覆盖率

测试的另一个重要方面是代码覆盖率。这是一个指标，它告诉你在测试运行期间执行的行数与你的代码库中所有行的总数之间的比率。为此，我们可以使用 pytest-cov 插件，它集成了 pytest 和 T2 的覆盖率。

安装后，要运行覆盖报告测试，添加`--cov`选项，如下所示:

```py
`$ python -m pytest --cov=.` 
```

它将产生如下输出:

```py
`================================== test session starts ==================================
platform linux -- Python 3.7.9, pytest-5.4.3, py-1.9.0, pluggy-0.13.1
rootdir: /home/johndoe/sample-project
plugins: cov-2.10.1
collected 6 items

tests/test_sample_project.py ....                                             [ 66%]
tests/test_sample_project_mock.py .                                           [ 83%]
tests/test_sample_project_mock_1.py .                                         [100%]

----------- coverage: platform linux, python 3.7.9-final-0 -----------
Name                                  Stmts   Miss  Cover
---------------------------------------------------------
sample_project/__init__.py                1      1     0%
tests/__init__.py                         0      0   100%
tests/test_sample_project.py              5      0   100%
tests/test_sample_project_mock.py        13      0   100%
tests/test_sample_project_mock_1.py      12      0   100%
---------------------------------------------------------
TOTAL                                    31      1    97%

==================================  6 passed in 0.13s ==================================` 
```

对于项目路径中的每个文件，您将获得:

*   Stmts -代码行数
*   miss——测试没有执行的行数
*   Cover -文件的覆盖率

在底部，有一行是整个项目的总数。

请记住，尽管鼓励实现高覆盖率，但这并不意味着你的测试是好的测试，测试你代码的每一条快乐和异常路径。例如，使用像`assert sum(3, 2) == 5`这样的断言的测试可以达到很高的覆盖率，但是你的代码实际上仍然没有经过测试，因为异常路径没有被覆盖。

## 突变测试

变异测试有助于确保您的测试实际上覆盖了代码的全部行为。换句话说，它分析测试套件的有效性或健壮性。在突变测试过程中，一个工具会遍历源代码的每一行，做出一些小的改变(称为突变)来破坏你的代码。在每一次变异之后，该工具都会运行您的单元测试，并检查您的测试是否失败。如果您的测试仍然通过，那么您的代码没有通过变异测试。

例如，假设您有以下代码:

```py
`if x > y:
    z = 50
else:
    z = 100` 
```

变异工具可能会将运算符从`>`改为`>=`，如下所示:

```py
`if x >= y:
    z = 50
else:
    z = 100` 
```

mutmut 是 Python 的一个变异测试库。让我们看看它的运行情况。

假设您有下面的`Loan`类:

```py
`# loan.py

from dataclasses import dataclass
from enum import Enum

class LoanStatus(str, Enum):
    PENDING = "PENDING"
    ACCEPTED = "ACCEPTED"
    REJECTED = "REJECTED"

@dataclass
class Loan:
    amount: float
    status: LoanStatus = LoanStatus.PENDING

    def reject(self):
        self.status = LoanStatus.REJECTED

    def rejected(self):
        return self.status == LoanStatus.REJECTED` 
```

现在，假设您想要自动拒绝超过 250，000 的贷款请求:

```py
`# reject_loan.py

def reject_loan(loan):
    if loan.amount > 250_000:
        loan.reject()

    return loan` 
```

然后，您编写了以下测试:

```py
`# test_reject_loan.py

from loan import Loan
from reject_loan import reject_loan

def test_reject_loan():
    loan = Loan(amount=100_000)

    assert not reject_loan(loan).rejected()` 
```

当你用 mutmut 进行突变测试时，你会看到你有两个存活的突变体:

```
`$ mutmut run --paths-to-mutate reject_loan.py --tests-dir=.

- Mutation testing starting -

These are the steps:
1. A full test suite run will be made to make sure we
   can run the tests successfully and we know how long
   it takes (to detect infinite loops for example)
2. Mutants will be generated and checked

Results are stored in .mutmut-cache.
Print found mutants with `mutmut results`.

Legend for output:
🎉 Killed mutants.   The goal is for everything to end up in this bucket.
⏰ Timeout.          Test suite took 10 times as long as the baseline so were killed.
🤔 Suspicious.       Tests took a long time, but not long enough to be fatal.
🙁 Survived.         This means your tests needs to be expanded.
🔇 Skipped.          Skipped.

1. Running tests without mutations
⠏ Running...Done

2. Checking mutants
⠸ 2/2  🎉 0  ⏰ 0  🤔 0  🙁 2  🔇 0` 
```py

你可以通过 ID 查看幸存的变种人:

```
`$ mutmut show 1

--- reject_loan.py
+++ reject_loan.py
@@ -1,7 +1,7 @@
 # reject_loan.py

 def reject_loan(loan):
-    if loan.amount > 250_000:
+    if loan.amount >= 250_000:
         loan.reject()

     return loan` 
```py

```
`$ mutmut show 2

--- reject_loan.py
+++ reject_loan.py
@@ -1,7 +1,7 @@
 # reject_loan.py

 def reject_loan(loan):
-    if loan.amount > 250_000:
+    if loan.amount > 250001:
         loan.reject()

     return loan` 
```py

改进您的测试:

```
`from loan import Loan
from reject_loan import reject_loan

def test_reject_loan():
    loan = Loan(amount=100_000)
    assert not reject_loan(loan).rejected()

    loan = Loan(amount=250_001)
    assert reject_loan(loan).rejected()

    loan = Loan(amount=250_000)
    assert not reject_loan(loan).rejected()` 
```py

如果你再次进行突变测试，你会发现没有突变存活下来:

```
`$ mutmut run --paths-to-mutate reject_loan.py --tests-dir=.

- Mutation testing starting -

These are the steps:
1. A full test suite run will be made to make sure we
   can run the tests successfully and we know how long
   it takes (to detect infinite loops for example)
2. Mutants will be generated and checked

Results are stored in .mutmut-cache.
Print found mutants with `mutmut results`.

Legend for output:
🎉 Killed mutants.   The goal is for everything to end up in this bucket.
⏰ Timeout.          Test suite took 10 times as long as the baseline so were killed.
🤔 Suspicious.       Tests took a long time, but not long enough to be fatal.
🙁 Survived.         This means your tests needs to be expanded.
🔇 Skipped.          Skipped.

1. Running tests without mutations
⠏ Running...Done

2. Checking mutants
⠙ 2/2  🎉 2  ⏰ 0  🤔 0  🙁 0  🔇 0` 
```py

现在您的测试更加健壮了。在 *reject_loan.py* 中的任何无意的改变都会导致测试失败。

> Python 的变异测试工具不像其他一些工具那样成熟。比如[突变体](https://github.com/mbj/mutant)就是 Ruby 的一个成熟的突变测试工具。要了解更多关于突变测试的知识，请在[推特](https://twitter.com/_m_b_j_)上关注突变作者。

与任何其他方法一样，突变测试也有一个权衡。虽然它提高了测试套件捕捉 bug 的能力，但它是以速度为代价的，因为您必须运行整个测试套件数百次。它也迫使你真正地测试一切。这有助于发现异常路径，但是您将有更多的测试用例需要维护。

## 假设

[假设](https://hypothesis.readthedocs.io/en/latest/)是在 Python 中进行[基于属性的测试](https://hypothesis.works/articles/what-is-property-based-testing/)的库。基于属性的测试不需要为每一个你想要测试的参数编写不同的测试用例，它会生成大量的随机测试数据，这些数据依赖于之前的测试运行。这有助于增加测试套件的健壮性，同时减少测试冗余。简而言之，您的测试代码将会更干净、更简洁，并且总体上更高效，同时仍然覆盖了广泛的测试数据。

例如，假设您必须为以下函数编写测试:

```
`def increment(num: int) -> int:
    return num + 1` 
```py

您可以编写以下测试:

```
`import pytest

@pytest.mark.parametrize(
    'number, result',
    [
        (-2, -1),
        (0, 1),
        (3, 4),
        (101234, 101235),
    ]
)
def test_increment(number, result):
    assert increment(number) == result` 
```py

这种方法没有错。您的代码已经过测试，代码覆盖率很高(准确地说是 100%)。也就是说，基于可能的输入范围，您的代码测试得有多好？有相当多的整数可以测试，但是测试中只使用了其中的四个。在某些情况下，这就足够了。在其他情况下，四种情况是不够的——即非确定性机器学习代码。那些非常小或非常大的数字呢？或者假设您的函数接受一个整数列表而不是单个整数——如果这个列表是空的，或者它包含一个元素、数百个元素或数千个元素呢？在某些情况下，我们根本无法提供(更不用说甚至想出)所有可能的情况。这就是基于属性的测试发挥作用的地方。

> 机器学习算法是基于属性的测试的一个很好的用例，因为很难为复杂的数据集产生(和维护)测试实例。

像假说这样的框架提供了生成随机测试数据的方法(假说称之为[策略](https://hypothesis.readthedocs.io/en/latest/data.html?#core-strategies))。假设还存储以前测试运行的结果，并使用它们来创建新的案例。

> 策略是基于输入数据的形状生成伪随机数据的算法。它是伪随机的，因为生成的数据是基于以前测试的数据。

通过假设使用基于属性的测试的相同测试如下所示:

```
`from hypothesis import given
import hypothesis.strategies as st

@given(st.integers())
def test_add_one(num):
    assert increment(num) == num - 1` 
```py

`st.integers()`是一种假设策略，它生成用于测试的随机整数，而`@given`装饰器用于参数化测试函数。因此，当调用测试函数时，从策略中生成的整数将被传递到测试中。

```
`$ python -m pytest test_hypothesis.py --hypothesis-show-statistics

================================== test session starts ===================================
platform darwin -- Python 3.8.5, pytest-6.1.1, py-1.9.0, pluggy-0.13.1
rootdir: /home/johndoe/sample-project
plugins: hypothesis-5.37.3
collected 1 item

test_hypothesis.py .                                                               [100%]
================================= Hypothesis Statistics ==================================

test_hypothesis.py::test_add_one:

  - during generate phase (0.06 seconds):
    - Typical runtimes: < 1ms, ~ 50% in data generation
    - 100 passing examples, 0 failing examples, 0 invalid examples

  - Stopped because settings.max_examples=100

=================================== 1 passed in 0.08s ====================================` 
```py

## 类型检查

测试是代码，它们应该被如此对待。就像您的业务代码一样，您需要维护和重构它们。你甚至可能需要时不时地处理一些 bug。正因为如此，保持你的测试简短、简单、直截了当是一个好习惯。您还应该注意不要过度测试您的代码。

运行时(或动态)类型的检查器，如 [Typeguard](https://typeguard.readthedocs.io/) 和 [pydantic](https://pydantic-docs.helpmanual.io/) ，可以帮助最小化测试的数量。让我们来看一个 pydantic 的例子。

例如，假设我们有一个只有一个属性的`User`，一个电子邮件地址:

```
`class User:

    def __init__(self, email: str):
        self.email = email

user = User(email='[[email protected]](/cdn-cgi/l/email-protection)')` 
```py

我们希望确保所提供的电子邮件确实是有效的电子邮件地址。因此，为了验证它，我们必须在某个地方添加一些助手代码。除了编写测试，我们还必须花时间为此编写正则表达式。pydantic 可以帮助解决这个问题。我们可以用它来定义我们的`User`模型:

```
`from pydantic import BaseModel, EmailStr

class User(BaseModel):
    email: EmailStr

user = User(email='[[email protected]](/cdn-cgi/l/email-protection)')` 
```py

现在，在创建每个新的`User`实例之前，pydantic 将验证 email 参数。当它不是一个有效的电子邮件-即`User(email='something')` -一个[验证错误](https://pydantic-docs.helpmanual.io/usage/models/#error-handling)将被提出。这消除了编写我们自己的验证器的需要。我们也不需要测试它，因为 pydantic [的维护者为我们处理了](https://github.com/samuelcolvin/pydantic/blob/ab671a36708a14017e2ccc62f72c9f7628628737/tests/test_types.py)。

我们可以减少对任何用户提供的数据进行测试的次数。相反，我们只需要测试是否正确处理了`ValidationError`。

让我们看看 Flask 应用程序中的一个简单例子:

```
`import uuid

from flask import Flask, jsonify
from pydantic import ValidationError, BaseModel, EmailStr, Field

app = Flask(__name__)

@app.errorhandler(ValidationError)
def handle_validation_exception(error):
    response = jsonify(error.errors())
    response.status_code = 400
    return response

class Blog(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    author: EmailStr
    title: str
    content: str` 
```py

测试:

```
`import json

def test_create_blog_bad_request(client):
    """
 GIVEN request data with invalid values or missing attributes
 WHEN endpoint /create-blog/ is called
 THEN it should return status 400 and JSON body
 """
    response = client.post(
        '/create-blog/',
        data=json.dumps(
            {
            'author': 'John Doe',
            'title': None,
            'content': 'Some extra awesome content'
        }
        ),
        content_type='application/json',
    )

    assert response.status_code == 400
    assert response.json is not None` 
```

## 结论

测试常常让人觉得是一项令人畏惧的任务。总有这样的时候，但是希望这篇文章提供了一些工具，您可以使用它们来使测试变得更容易。将您的测试工作集中在减少古怪的测试上。你的测试也应该是快速的，隔离的/独立的，确定的/可重复的。最后，对您的测试套件有信心将帮助您更频繁地部署到生产环境中，更重要的是，有助于您晚上睡觉。

测试愉快！

> [完整 Python](/guides/complete-python/) 指南:
> 
> 1.  [现代 Python 环境——依赖性和工作空间管理](/blog/python-environments/)
> 2.  [Python 中的测试](/blog/testing-python/)(本文！)
> 3.  [Python 中的现代测试驱动开发](/blog/modern-tdd/)
> 4.  [Python 代码质量](/blog/python-code-quality/)
> 5.  [Python 类型检查](/blog/python-type-checking/)
> 6.  [记录 Python 代码和项目](/blog/documenting-python/)
> 7.  [Python 项目工作流程](/blog/python-project-workflow/)