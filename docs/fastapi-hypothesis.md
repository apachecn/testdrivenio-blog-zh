# 使用假设和图式测试 FastAPI

> 原文：<https://testdriven.io/blog/fastapi-hypothesis/>

测试是开发软件的必要部分。具有高测试覆盖率的软件项目从来都不是完美的，但是它是软件质量的一个很好的初始指示器。为了鼓励测试的编写，测试应该有趣且易于编写。它们也应该像代码库中的其他代码一样被小心对待。因此，在添加新测试时，您需要考虑维护测试套件的成本。在可读性和可维护性之间取得平衡，同时确保测试覆盖广泛的场景并不容易

在本文中，我们将看看基于属性的测试是如何帮助解决这个问题的。我们首先来看看什么是基于属性的测试，以及为什么你可能想要使用它。然后，我们将展示如何使用[假设](https://hypothesis.works/)和[模式](https://schemathesis.readthedocs.io/)对 FastAPI 应用基于属性的测试。

## 基于属性的测试

什么是基于属性的测试？

[基于属性的测试](https://hypothesis.works/articles/what-is-property-based-testing/)基于给定函数或程序的属性。这些测试有助于确保被测试的函数或程序符合其属性。

### 利益

为什么要使用基于属性的测试？

1.  **作用域**:基于属性的测试不需要为每个要测试的参数编写不同的测试用例，而是允许您为单个测试中的每个参数测试一系列参数。这有助于提高测试套件的稳健性，同时减少测试冗余。简而言之，您的测试代码将更干净、更 DRY，并且总体上更高效，同时更有效，因为您将能够更容易地测试所有这些边缘情况。
2.  **Reproducibility** :测试代理保存测试用例以及它们的结果，在失败的情况下，这些结果可以用于重现和重放测试。

让我们看一个简单的例子来说明这一点:

```
`def factorial(num: int) -> int:
    if num < 0:
        raise ValueError("Number must be >= 0")

    total = 1
    for _ in range(1, num + 1):
        total *= _
    return total

# test

import pytest

def test_factorial_less_than_0():
    with pytest.raises(ValueError):
        assert factorial(-1) == 1

def test_factorial():
    assert factorial(0) == 1
    assert factorial(1) == 1
    assert factorial(3) == 6
    assert factorial(7) == 5040
    assert factorial(12) == 479001600
    assert factorial(44) == 2658271574788448768043625811014615890319638528000000000` 
```

这有什么不好？

1.  测试用例写起来很无聊
2.  很难找到随机的、无偏见的测试例子
3.  测试套件的大小会迅速膨胀，因此很难阅读和维护
4.  还是那句话，没意思！
5.  很难充实边缘案例

人类不应该在这上面浪费时间。这是最适合计算机完成的任务。

## 基于属性的假设测试

[假设](https://hypothesis.readthedocs.io/en/latest/)是在 Python 中进行基于属性的测试的工具。假设使得编写测试和找到所有边缘案例变得容易。

> 它的工作原理是生成与您的规格相匹配的任意数据，并检查您的保证在这种情况下仍然有效。如果它找到了一个没有的例子，它就把这个例子缩小，简化它，直到找到一个更小的例子，这个例子仍然会引起问题。然后它保存这个例子，以便以后一旦发现你的代码有问题，以后不会忘记。

这里最重要的部分是所有失败的测试都将被尝试，即使错误已经被修复！

### 快速启动

假设集成到您正常的 pytest 或 unittest 工作流中。

从安装库开始:

接下来，您需要定义一个[策略](https://hypothesis.readthedocs.io/en/latest/data.html?#core-strategies)，这是一个生成随机数据的方法。

示例:

| 战略 | 它会产生什么 |
| --- | --- |
| 二进制的 | 字节字符串 |
| 文本 | 用线串 |
| 整数 | 整数 |
| 漂浮物 | 漂浮物 |
| 分数 | 分数实例 |

策略应该组合在一起，以生成复杂的测试输入数据。因此，与其编写和维护自己的数据生成器，不如让 Hypothesis 为您管理所有这些。

让我们将上面的`test_factorial`重构为一个基于属性的测试:

```
`from hypothesis import given
from hypothesis.strategies import integers

@given(integers(min_value=1, max_value=30))
def test_factorial(num: int):
    result = factorial(num) / factorial(num - 1)
    assert num == result` 
```

> 这个测试现在断言一个数的阶乘除以该数的阶乘减一就是原始数。

在这里，我们将[整数](https://hypothesis.readthedocs.io/en/latest/data.html#hypothesis.strategies.integers)策略传递给了`@given`装饰器，这是假设的入口点。这个装饰器本质上把测试函数变成了一个参数化的函数，这样当它被调用时，从策略生成的数据将被传递到测试中。

如果已经发现故障，假设使用[收缩](https://hypothesis.readthedocs.io/en/latest/data.html#shrinking)来找到最小的故障情况。

示例:

```
`from hypothesis import Verbosity, given, settings
from hypothesis import strategies as st

@settings(verbosity=Verbosity.verbose)
@given(st.integers())
def test_shrinking(num: int):
    assert num >= -2` 
```

测试输出:

```
`...

Trying example: test_shrinking(
    num=-4475302896957925906,
)
Traceback (most recent call last):
  File "shrinking.py", line 8, in test_shrinking
    assert num >= -2
AssertionError

Trying example: test_shrinking(
    num=2872,
)
Trying example: test_shrinking(
    num=-93,
)
Traceback (most recent call last):
  File "shrinking.py", line 8, in test_shrinking
    assert num >= -2
AssertionError

Trying example: test_shrinking(
    num=14443,
)
Trying example: test_shrinking(
    num=56,
)
Trying example: test_shrinking(
    num=-13873,
)
Traceback (most recent call last):
  File "shrinking.py", line 8, in test_shrinking
    assert num >= -2
AssertionError

Trying example: test_shrinking(
    num=23519,
)
Trying example: test_shrinking(
    num=-91,
)
Traceback (most recent call last):
  File "shrinking.py", line 8, in test_shrinking
    assert num >= -2
AssertionError

Trying example: test_shrinking(
    num=-93,
)
Traceback (most recent call last):
  File "shrinking.py", line 8, in test_shrinking
    assert num >= -2
AssertionError

Trying example: test_shrinking(
    num=0,
)
Trying example: test_shrinking(
    num=0,
)
Trying example: test_shrinking(
    num=-29,
)
Traceback (most recent call last):
  File "shrinking.py", line 8, in test_shrinking
    assert num >= -2
AssertionError

Trying example: test_shrinking(
    num=-13,
)
Traceback (most recent call last):
  File "shrinking.py", line 8, in test_shrinking
    assert num >= -2
AssertionError

Trying example: test_shrinking(
    num=-5,
)
Traceback (most recent call last):
  File "shrinking.py", line 8, in test_shrinking
    assert num >= -2
AssertionError

Trying example: test_shrinking(
    num=-1,
)
Trying example: test_shrinking(
    num=-2,
)
Trying example: test_shrinking(
    num=-4,
)
Traceback (most recent call last):
  File "shrinking.py", line 8, in test_shrinking
    assert num >= -2
AssertionError

Trying example: test_shrinking(
    num=-3,
)
Traceback (most recent call last):
  File "shrinking.py", line 8, in test_shrinking
    assert num >= -2
AssertionError

Trying example: test_shrinking(
    num=3,
)
Trying example: test_shrinking(
    num=-3,
)
Traceback (most recent call last):
  File "shrinking.py", line 8, in test_shrinking
    assert num >= -2
AssertionError

Trying example: test_shrinking(
    num=0,
)
Trying example: test_shrinking(
    num=-1,
)
Trying example: test_shrinking(
    num=-2,
)
Trying example: test_shrinking(
    num=3,
)
Falsifying example: test_shrinking(
    num=-3,
)` 
```

这里，我们针对整数池测试了表达式`num >= -2`。假设从第一个失败案例`num = -4475302896957925906`开始，这是一个相当大的数字。然后收缩`num`的值，直到假设发现值`num = -3`是最小的失败案例。

## 对 FastAPI 使用假设

假设已被证明是一个简单而强大的测试工具。让我们看看如何在 FastAPI 中使用它。

```
`# server.py

import uvicorn
from fastapi import FastAPI

app = FastAPI()

@app.get("/api/{s}")
def homepage(s: int):
    return {"message": s * s}

if __name__ == "__main__":
    uvicorn.run(app)` 
```

因此，`/api/{s}`路由接受一个名为`s`的 URL 参数，它应该是一个整数。

```
`# test_server.py

from hypothesis import given, strategies as st
from fastapi.testclient import TestClient

from server import app

client = TestClient(app)

@given(st.integers())
def test_home(s):
    res = client.get(f"/api/{s}")

    assert res.status_code == 200
    assert res.json() == {"message": s * s}` 
```

就像之前我们使用`integers`策略生成随机整数，正的和负的，用于测试。

## 图式

[概要](https://schemathesis.readthedocs.io/en/stable/)是一个基于 [OpenAPI](https://www.openapis.org/) 和 [GraphQL](https://spec.graphql.org/June2018/) 规范的现代 API 测试工具。它使用幕后假设将基于属性的测试应用于 API 模式。换句话说，给定一个模式，Schemathesis 可以自动为您生成测试用例。由于 FastAPI 是基于 OpenAPI 标准的，Schemathesis 与它配合得很好。

如果您运行上面的 *server.py* 文件并导航到[http://localhost:8000/openapi . JSON](http://localhost:8000/openapi.json)，您应该会看到由 FastAPI 生成的 open API 规范。它定义了所有的端点及其输入类型。使用这个规范，Schemathesis 可以用来生成测试数据。

### 使用 CLI 快速入门

安装:

```
`$ pip install schemathesis` 
```

一旦安装完毕，运行测试最简单的方法就是通过 [schemathesis](https://schemathesis.readthedocs.io/en/stable/cli.html) 命令。当 Uvicorn 在一个终端窗口中运行时，打开一个新窗口并运行:

```
`$ schemathesis run http://localhost:8000/openapi.json` 
```

您应该看到:

```
`========================= Schemathesis test session starts ========================
Schema location: http://localhost:8000/openapi.json
Base URL: http://localhost:8000/
Specification version: Open API 3.0.2
Workers: 1
Collected API operations: 1

GET /api/{s} .                                                               [100%]

===================================== SUMMARY =====================================

Performed checks:
    not_a_server_error                    100 / 100 passed          PASSED

================================ 1 passed in 0.61s ================================` 
```

请注意，这只针对`not_a_server_error`进行了检查。图式有五个内置的[检查](https://schemathesis.readthedocs.io/en/stable/cli.html#how-are-responses-checked):

1.  `not_a_server_error`:响应具有 5xx HTTP 状态
2.  `status_code_conformance`:API 模式中未定义响应状态
3.  `content_type_conformance`:API 模式中未定义响应内容类型
4.  `response_schema_conformance`:响应内容不符合为此特定响应定义的模式
5.  `response_headers_conformance`:响应头不包含所有已定义的头。

您可以使用`--checks all`选项执行所有内置检查:

```
`$ schemathesis run --checks all http://localhost:8000/openapi.json

========================= Schemathesis test session starts ========================
Schema location: http://localhost:8000/openapi.json
Base URL: http://localhost:8000/
Specification version: Open API 3.0.2
Workers: 1
Collected API operations: 1

GET /api/{s} .                                                               [100%]

===================================== SUMMARY =====================================

Performed checks:
    not_a_server_error                              100 / 100 passed          PASSED
    status_code_conformance                         100 / 100 passed          PASSED
    content_type_conformance                        100 / 100 passed          PASSED
    response_headers_conformance                    100 / 100 passed          PASSED
    response_schema_conformance                     100 / 100 passed          PASSED

================================ 1 passed in 0.87s ================================` 
```

### 附加选项

您可以测试一个[特定的端点](https://schemathesis.readthedocs.io/en/stable/cli.html#testing-specific-operations)或 HTTP 方法，而不是整个应用程序:

```
`$ schemathesis run --endpoint /api/. http://localhost:8000/openapi.json

$ schemathesis run --method GET http://localhost:8000/openapi.json` 
```

最大响应时间可用于帮助充实可能降低端点速度的边缘情况。时间以毫秒为单位。

```
`$ schemathesis run --max-response-time=50 HTTP://localhost:8000/openapi.json` 
```

您的一些端点需要授权吗？

```
`$ schemathesis run -H "Authorization: Bearer TOKEN" http://localhost:8000/openapi.json

$ schemathesis run -H "Authorization: ..." -H "X-API-Key: ..." HTTP://localhost:8000/openapi.json` 
```

您可以使用多个工作人员来加快测试速度:

```
`$ schemathesis run --workers 8 http://localhost:8000/openapi.json` 
```

通常，Schemathesis 为每个端点生成随机数据。[状态测试](https://schemathesis.readthedocs.io/en/stable/stateful.html)确保数据来自之前的测试/响应:

```
`$ schemathesis run --stateful=links http://localhost:8000/openapi.json` 
```

最后，重放测试很简单，因为每个测试用例都与一个种子值相关联。当一个测试用例失败时，它会提供种子，这样您就可以重现失败的用例:

```
`$ schemathesis run http://localhost:8000/openapi.json

============================ Schemathesis test session starts ============================
platform Darwin -- Python 3.10.6, schemathesis-3.17.2, hypothesis-6.54.4,
    hypothesis_jsonschema-0.22.0, jsonschema-4.15.0
rootdir: /hypothesis-examples
hypothesis profile 'default' ->
    database=DirectoryBasedExampleDatabase('/hypothesis-examples/.hypothesis/examples')
Schema location: http://localhost:8000/openapi.json
Base URL: http://localhost:8000/
Specification version: Open API 3.0.2
Workers: 1
collected endpoints: 1

GET /api/{s} F                                                                      [100%]

======================================== FAILURES ========================================
_____________________________________ GET: /api/{s} ______________________________________
1. Received a response with 5xx status code: 500

Path parameters : {'s': 0}

Run this Python code to reproduce this failure:
  requests.get('http://localhost:8000/api/0', headers={'User-Agent': 'schemathesis/2.6.0'})

Or add this option to your command line parameters:
    --hypothesis-seed=135947773389980684299156880789978283847
======================================== SUMMARY =========================================

Performed checks:
    not_a_server_error                    0 / 3 passed          FAILED

=================================== 1 passed in 0.10s ====================================` 
```

然后，要重现，运行:

```
`$ schemathesis run http://localhost:8000/openapi.json --hypothesis-seed=135947773389980684299156880789978283847` 
```

### Python 测试

您也可以在测试中使用模式:

```
`import schemathesis

schema = schemathesis.from_uri("http://localhost:8000/openapi.json")

@schema.parametrize()
def test_api(case):
    case.call_and_validate()` 
```

Schemathesis 还支持直接调用 ASGI(即 Uvicorn 和 Daphne)和 WSGI(即 Gunicorn 和 uWSGI)应用程序，而不是通过网络:

```
`import schemathesis
from schemathesis.specs.openapi.loaders import from_asgi

from server import app

schema = from_asgi("/openapi.json", app)

@schema.parametrize()
def test_api(case):
    response = case.call_asgi()
    case.validate_response(response)` 
```

## 结论

希望你能看到基于属性的测试有多强大。它可以使您的代码更加健壮，而不会降低样板测试代码的可读性。收缩以找到最简单的失败案例和重放等功能提高了工作效率。基于属性的测试将减少花费在编写手动测试上的时间，同时增加测试覆盖率。

像 Hypothesis 这样基于属性的测试工具应该是几乎所有 Python 工作流的一部分。Schemathesis 用于基于 OpenAPI 标准的自动化 API 测试，这在与 FastAPI 等以 API 为中心的框架结合使用时非常有用。