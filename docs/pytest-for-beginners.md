# 初学 Pytest

> 原文：<https://testdriven.io/blog/pytest-for-beginners/>

自动化测试是开发过程中必不可少的一部分。

尽管一开始编写测试看起来像是延长了开发过程，但从长远来看，它为您节省了大量时间。

编写良好的测试通过确保您的代码如您所愿，降低了生产环境中出现问题的可能性。测试还能帮助你覆盖边缘案例，使重构更容易。

在本文中，我们将了解如何使用 [pytest](https://docs.pytest.org/) ，这样您就能够自己使用它来改进您的开发过程，并遵循更高级的 pytest 教程。

## 目标

在本文结束时，您将能够:

1.  解释 pytest 是什么以及如何使用它
2.  用 pytest 自己编写一个测试
3.  跟随使用 pytest 的更复杂的教程
4.  准备测试所需的数据和/或文件
5.  将测试参数化
6.  测试所需的模拟功能

## 为什么选择 pytest

虽然经常被忽视，但测试是如此重要，以至于 Python 自带了一个名为 [unittest](https://docs.python.org/3/library/unittest.html) 的内置测试框架。不过，在 unittest 中编写测试可能会很复杂，所以近年来， [pytest](https://docs.pytest.org/) 框架已经成为标准。

pytest 的一些显著优势是:

1.  需要更少的样板代码，使您的测试套件更具可读性
2.  使用普通的 assert 语句，而不是 unittest 的 assertSomething 方法(例如，`assertEquals`、`assertTrue`)
3.  夹具系统简化了测试状态的设置和拆除
4.  功能方法
5.  大型社区维护的插件生态系统

## 入门指南

因为这是指南而不是教程，所以我们准备了一个简单的 FastAPI 应用程序，您可以在阅读本文时参考。可以从 [GitHub](https://github.com/testdrivenio/pytest_for_beginners_test_project) 中克隆。

在*基本*分支上，我们的 API 有 4 个端点(在 *main.py* 中定义),它们使用 *calculations.py* 中的函数返回对两个整数执行某种基本算术运算(`+` / `-` / `*` / `/`)的结果。在 *advanced_topics* 分支上，增加了两个功能:

1.  `CalculationsStoreJSON`(在 *store_calculations.py* 内部)类——允许你在 JSON 文件中存储和检索计算结果。
2.  `get_number_fact`(inside*number _ facts . py*)——调用远程 API 来检索关于某个数字的事实。

> 理解本文不需要 FastAPI 知识。

在本文的第一部分，我们将使用*基础知识*分支。

创建并激活虚拟环境，并安装要求:

```
`$ python3.10 -m venv venv
$ source venv/bin/activate
(venv)$ pip install -r requirements.txt` 
```

## 组织和命名

为了组织您的测试，您可以使用三种可能性，所有这些都在示例项目中使用:

| 组织于 | 例子 |
| --- | --- |
| Python 包(文件夹包含一个 *__init__。py* 文件) | "测试 _ 计算" |
| 组件 | *测试 _ 交换 _ 操作. py* |
| 班级 | `TestCalculationEndpoints` |

> 说到组织测试的最佳实践，每个程序员都有自己的偏好。
> 
> 本文的目的不是展示最佳实践，而是向您展示所有的可能性。

如果您遵守以下[约定](https://docs.pytest.org/en/7.1.x/explanation/goodpractices.html#conventions-for-python-test-discovery)，pytest 将自己发现测试:

*   您将您的测试添加到一个以`test_`开始或者以`_test.py`结束的文件中(例如`test_foo.py`或者`foo_test.py`
*   您可以给测试函数加上前缀`test_`(例如`def test_foo()`)
*   如果您正在使用类，您可以将您的测试作为方法添加到以`Test`(例如，`class TestFoo`)为前缀的类中

> 不遵循命名约定的测试将不会被发现，所以要小心命名。
> 
> 值得注意的是，命名约定[可以在命令行或](https://docs.pytest.org/en/7.1.x/reference/reference.html#configuration-options)[配置文件](https://docs.pytest.org/en/7.1.x/customize.html#configuration-file-formats)中更改。

## 测试解剖学

让我们看看`test_return_sum`(在*test _ calculation _ endpoints . py*文件中)测试函数是什么样子的:

```
`# tests/test_endpoints/test_calculation_endpoints.py

def test_return_sum(self):
   # Arrange
   test_data = {
      "first_val": 10,
      "second_val": 8
   }
   client = TestClient(app)

   # Act
   response = client.post("/sum/", json=test_data)

   # Assert
   assert response.status_code == 200
   assert response.json() == 18` 
```

根据 pytest [文档](https://docs.pytest.org/en/7.1.x/fixture.html#what-fixtures-are)，每个测试功能由四个步骤组成:

1.  *安排*——你为考试准备一切的地方(`test_data = {"first_val": 10, "second_val": 8}`)
2.  *动作*——启动你想要测试的行为的奇异的、改变状态的动作(`client.post("/sum/", json=test_data)`)
3.  *断言*——将*动作的结果*与期望的结果(`assert response.json() == 18`)进行比较
4.  *清理*——清理特定于测试的数据(通常在测试更复杂特性的测试中，你可以在[我们的提示](https://testdriven.io/tips/6114106d-9e03-4289-a2cb-c7f4d37d5051/)中看到一个例子)

## 运行测试

pytest 为您提供了很多控制，让您可以决定要运行哪些测试:

1.  所有的测试
2.  特定包
3.  特定模块
4.  特定类别
5.  特效试验
6.  对应于特定关键字的测试

让我们看看这是如何工作的...

> 如果您正在使用我们的示例应用程序，那么如果您安装了需求，那么已经安装了`pytest`。
> 
> 对于您自己的项目，`pytest`可以作为任何其他带有 pip 的包安装:
> 
> ```
> `(venv)$ pip install pytest` 
> ```

### 运行所有的测试

运行`pytest`命令将简单地运行 pytest 可以找到的所有测试:

```
`(venv)$ python -m pytest

=============================== test session starts ===============================
platform darwin -- Python 3.10.4, pytest-7.1.2, pluggy-1.0.0
rootdir: /Users/michael/repos/testdriven/pytest_for_beginners_test_project
plugins: anyio-3.6.1
collected 8 items

tests/test_calculations/test_anticommutative_operations.py ..               [ 25%]
tests/test_calculations/test_commutative_operations.py ..                   [ 50%]
tests/test_endpoints/test_calculation_endpoints.py ....                     [100%]

================================ 8 passed in 5.19s ================================` 
```

pytest 将通知您找到了多少个测试，以及这些测试是在哪个模块中找到的。在我们的示例应用程序中，pytest 找到了 8 个测试，它们都通过了。

在消息的底部，您可以看到有多少测试通过/失败。

#### 不正确的命名模式

正如已经讨论过的，不遵守正确命名约定的测试将不会被发现。错误命名的测试不会产生任何错误，所以您需要注意这一点。

例如，如果您将`TestCalculationEndpoints`类重命名为`CalculationEndpointsTest`，那么其中的所有测试都不会运行:

```
`=============================== test session starts ===============================
platform darwin -- Python 3.10.4, pytest-7.1.2, pluggy-1.0.0
rootdir: /Users/michael/repos/testdriven/pytest_for_beginners_test_project
plugins: anyio-3.6.1
collected 4 items

tests/test_calculations/test_anticommutative_operations.py ..               [ 50%]
tests/test_calculations/test_commutative_operations.py ..                   [100%]

================================ 4 passed in 0.15s ================================` 
```

在继续之前，将名称改回`TestCalculationEndpoints`。

#### 测试失败

你的测试不会总是第一次就通过。

破坏`test_calculate_sum`中`assert`语句的预测输出，看看失败测试的输出是什么样的:

```
`# tests/test_calculations/test_commutative_operations.py

def test_calculate_sum():

    calculation = calculate_sum(5, 3)

    assert calculation == 7 # whops, a mistake` 
```

运行测试。您应该会看到类似如下的内容:

```
`=============================== test session starts ===============================
platform darwin -- Python 3.10.4, pytest-7.1.2, pluggy-1.0.0
rootdir: /Users/michael/repos/testdriven/pytest_for_beginners_test_project
plugins: anyio-3.6.1
collected 8 items

tests/test_calculations/test_anticommutative_operations.py ..               [ 25%]
tests/test_calculations/test_commutative_operations.py F.                   [ 50%]
tests/test_endpoints/test_calculation_endpoints.py ....                     [100%]

==================================== FAILURES =====================================
_______________________________ test_calculate_sum ________________________________

    def test_calculate_sum():

        calculation = calculate_sum(5, 3)

>       assert calculation == 7
E       assert 8 == 7

tests/test_calculations/test_commutative_operations.py:8: AssertionError
============================= short test summary info =============================
FAILED tests/test_calculations/test_commutative_operations.py::test_calculate_sum
=========================== 1 failed, 7 passed in 0.26s ===========================` 
```

在消息的底部，您可以看到一个*简短测试摘要信息*部分。这将告诉您哪个测试失败了，失败在哪里。在这种情况下，实际输出- `8` -与预期输出- `7`不匹配。

如果你向上滚动一点，失败的测试会详细显示出来，所以更容易指出哪里出错了(对更复杂的测试有帮助)。

在继续之前修复这个测试。

### 在特定的包或模块中运行测试

要运行一个特定的包或模块，您只需要在 pytest 命令中添加一个特定测试集的完整相对路径。

**对于一个包:**

```
`(venv)$ python -m pytest tests/test_calculations` 
```

该命令将运行“tests/test_calculations”包中的所有测试。

**对于模块:**

```
`(venv)$ python -m pytest tests/test_calculations/test_commutative_operations.py` 
```

该命令将运行*测试/测试 _ 计算/测试 _ 交换 _ 操作. py* 模块中的所有测试。

两者的输出将与前一个相似，除了执行的测试数量会更少。

### 在特定类中运行测试

要在 pytest 中访问一个特定的类，您需要写一个到它的模块的相对路径，然后在`::`之后添加这个类:

```
`(venv)$ python -m pytest tests/test_endpoints/test_calculation_endpoints.py::TestCalculationEndpoints` 
```

这个命令将执行`TestCalculationEndpoints`类中的所有测试。

### 运行特定测试

您可以像访问类一样访问特定的测试，在相对路径后加上两个冒号，后跟测试名称:

```
`(venv)$ python -m pytest tests/test_calculations/test_commutative_operations.py::test_calculate_sum` 
```

如果您希望运行的函数在一个类中，则需要以如下形式运行一个测试:

`relative_path_to_module::TestClass::test_method`

例如:

```
`(venv)$ python -m pytest tests/test_endpoints/test_calculation_endpoints.py::TestCalculationEndpoints::test_return_sum` 
```

### 按关键字运行测试

现在，假设您只想运行与组织相关的测试。因为我们在处理除法的测试名称中包含了单词“divided ”,所以您可以像这样运行这些测试:

```
`(venv)$ python -m pytest -k "dividend"` 
```

因此，8 个测试中的 2 个将运行:

```
`=============================== test session starts ===============================
platform darwin -- Python 3.10.4, pytest-7.1.2, pluggy-1.0.0
rootdir: /Users/michael/repos/testdriven/pytest_for_beginners_test_project
plugins: anyio-3.6.1
collected 8 items / 6 deselected / 2 selected

tests/test_calculations/test_anticommutative_operations.py .                [ 50%]
tests/test_endpoints/test_calculation_endpoints.py .                        [100%]

========================= 2 passed, 6 deselected in 0.18s =========================` 
```

> 这些并不是选择特定测试子集的唯一方法。更多信息请参考[官方文档](https://docs.pytest.org/en/7.1.x/how-to/usage.html)。

## 值得记住的 pytest 标志

pytest 包括许多[标志](https://docs.pytest.org/en/7.1.x/reference/reference.html#command-line-flags)；您可以使用`pytest --help`命令将它们全部列出。

其中最有用的有:

1.  `pytest -v`增加一级详细度，`pytest -vv`增加两级详细度。例如，当使用参数化(用不同的输入/输出多次运行相同的测试)时，只运行`pytest`会通知您有多少测试版本通过，有多少失败，同时添加`-v`也会输出使用了哪些参数。如果你添加`-vv`，你会看到每个测试版本的输入参数。您可以在 [pytest 文档](https://docs.pytest.org/en/7.1.x/how-to/output.html#verbosity)上看到更详细的示例。
2.  `pytest -lf`仅重新运行上次运行失败的测试。如果没有失败，所有的测试都将运行。
3.  添加`-x`标志会导致 pytest 在第一次出错或测试失败时立即退出。

## 参数化

我们讨论了基础知识，现在开始讨论更高级的话题。

> 如果你正在跟进回购，将分支从*基础*切换到*高级 _ 主题* ( `git checkout advanced_topics`)。

有时候，一个简单的输入就足够了，但是也有很多时候你想要测试多个输入——例如，电子邮件、密码等等。

你可以通过`@pytest.mark.parametrize`装饰器用[参数化](https://docs.pytest.org/en/7.1.x/how-to/parametrize.html)来添加多个输入和它们各自的输出。

例如，对于反交换运算，传递的数字的顺序很重要。明智的做法是涵盖更多情况，以确保该函数在所有情况下都能正常工作:

```
`# tests/test_calculations/test_anticommutative_operations.py

import pytest

from calculations import calculate_difference

@pytest.mark.parametrize(
    "first_value, second_value, expected_output",
    [
        (10, 8, 2),
        (8, 10, -2),
        (-10, -8, -2),
        (-8, -10, 2),
    ]
)
def test_calculate_difference(first_value, second_value, expected_output):

    calculation = calculate_difference(first_value, second_value)

    assert calculation == expected_output` 
```

`@pytest.mark.parametrize`具有严格的结构形式:

1.  您向装饰者传递两个参数:
    1.  参数名称以逗号分隔的字符串
    2.  参数值的列表，其位置对应于参数名的位置
2.  您将参数名传递给测试函数(它们不依赖于位置)

如果您运行该测试，它将运行 4 次，每次使用不同的输入和输出:

```
`(venv)$ python -m pytest -v  tests/test_calculations/test_anticommutative_operations.py::test_calculate_difference

=============================== test session starts ===============================
platform darwin -- Python 3.10.4, pytest-7.1.2, pluggy-1.0.0
rootdir: /Users/michael/repos/testdriven/pytest_for_beginners_test_project
plugins: anyio-3.6.1
collected 4 items

tests/test_calculations/test_anticommutative_operations.py::test_calculate_difference[10-8-2] PASSED [ 25%]
tests/test_calculations/test_anticommutative_operations.py::test_calculate_difference[8-10--2] PASSED [ 50%]
tests/test_calculations/test_anticommutative_operations.py::test_calculate_difference[-10--8--2] PASSED [ 75%]
tests/test_calculations/test_anticommutative_operations.py::test_calculate_difference[-8--10-2] PASSED [100%]

================================ 4 passed in 0.01s ================================` 
```

## 固定装置

当*排列*步骤在多个测试中完全相同时，或者如果它太复杂以至于损害了测试的可读性时，将*排列*(以及随后的*清理*)步骤移动到一个单独的[夹具](https://docs.pytest.org/en/7.1.x/fixture.html)功能是一个好主意。

### 创造

用`@pytest.fixture`装饰器将一个函数标记为 fixture。

旧版本的`TestCalculationEndpoints`在每个方法中都有一个创建`TestClient`的步骤。

例如:

```
`# tests/test_endpoints/test_calculation_endpoints.py

def test_return_sum(self):
    test_data = {
        "first_val": 10,
        "second_val": 8
    }
    client = TestClient(app)

    response = client.post("/sum/", json=test_data)

    assert response.status_code == 200
    assert response.json() == 18` 
```

在 *advanced_topics* 分支中，您会看到该方法现在看起来更加清晰:

```
`# tests/test_endpoints/test_calculation_endpoints.py

def test_return_sum(self, test_app):
    test_data = {
        "first_val": 10,
        "second_val": 8
    }

    response = test_app.post("/sum/", json=test_data)

    assert response.status_code == 200
    assert response.json() == 18` 
```

后两个保持原样，所以你可以比较它们(不要在现实生活中这样做；毫无意义)。

`test_return_sum`现在使用一个名为`test_app`的夹具，你可以在 *conftest.py* 文件中看到:

```
`# tests/conftest.py

import pytest
from starlette.testclient import TestClient

from main import app

@pytest.fixture(scope="module")
def test_app():
    client = TestClient(app)

    return client` 
```

这是怎么回事？

1.  `@pytest.fixture()`装饰器将函数`test_app`标记为 fixture。当 pytest 读取该模块时，它会将该函数添加到 fixtures 列表中。测试函数可以使用列表中的任何 fixture。
2.  这个 fixture 是一个返回`TestClient`的简单函数，因此可以执行测试 API 调用。
3.  将测试函数参数与一系列夹具进行比较。如果参数的值与 fixture 的名称匹配，fixture 将被解析，其返回值将作为参数写入测试函数。
4.  test 函数使用 fixture 的结果进行测试，使用它的方式与使用任何其他变量值的方式相同。

> 另一个需要注意的重要事情是，函数不是传递给 fixture 本身，而是传递给 fixture 值。

### 范围

Fixtures 是在测试第一次请求时创建的，但是它们是基于它们的作用域而被销毁的。夹具被破坏后，如果另一个试验需要，需要再次调用它；因此，您需要注意耗时的固定程序(例如，API 调用)的范围。

有五个可能的[范围](https://docs.pytest.org/en/7.1.x/how-to/fixtures.html#fixture-scopes)，从最窄到最宽:

| 范围 | 描述 |
| --- | --- |
| 功能(默认) | 测试结束时，夹具被破坏。 |
| 班级 | 在**级**最后一次测试的拆卸过程中，夹具被破坏。 |
| 组件 | 在拆卸**模块**中的最后一次测试时，夹具被破坏。 |
| 包裹 | 在拆卸**包装**中的最后一次测试时，夹具被破坏。 |
| 会议 | 夹具在测试**阶段**结束时被销毁。 |

要更改上例中的范围，只需设置`scope`参数:

```
`# tests/conftest.py

import pytest
from starlette.testclient import TestClient

from main import app

@pytest.fixture(scope="function") # scope changed
def test_app():
    client = TestClient(app)

    return client` 
```

> 定义尽可能小的范围有多重要取决于 fixture 的耗时程度。创建一个`TestClient`不是很耗时，所以改变范围不会缩短测试运行。但是，例如，使用调用外部 API 的 fixture 运行 10 个测试可能非常耗时，所以最好使用`module`作用域。

## 临时文件

当您的生产代码必须处理文件时，您的测试也是如此。

为了避免多个测试文件之间的干扰，甚至是应用程序的其余部分和额外的清理过程，最好使用一个唯一的临时目录。

在示例应用程序中，我们存储了在 JSON 文件上执行的所有操作，以供将来分析。现在，因为您肯定不希望在测试运行期间更改生产文件，所以您需要创建一个单独的临时 JSON 文件。

要测试的代码可以在 *store_calculations.py* 中找到:

```
`# store_calculations.py

import json

class CalculationsStoreJSON:
    def __init__(self, json_file_path):
        self.json_file_path = json_file_path
        with open(self.json_file_path / "calculations.json", "w") as file:
            json.dump([], file)

    def add(self, calculation):
        with open(self.json_file_path/"calculations.json", "r+") as file:
            calculations = json.load(file)
            calculations.append(calculation)
            file.seek(0)
            json.dump(calculations, file)

    def list_operation_usages(self, operation):
        with open(self.json_file_path / "calculations.json", "r") as file:
            calculations = json.load(file)

        return [calculation for calculation in calculations if calculation['operation'] == operation]` 
```

注意，在初始化`CalculationsStoreJSON`时，您必须提供一个`json_file_path`，JSON 文件将存储在这里。这可以是磁盘上的任何有效路径；对于产品代码和测试，您可以用同样的方式传递路径。

幸运的是，pytest 提供了许多[内置夹具](https://docs.pytest.org/en/7.1.x/reference/fixtures.html#built-in-fixtures)，其中一个我们可以在这里使用，叫做 [tmppath](https://docs.pytest.org/en/7.1.x/how-to/tmp_path.html) :

```
`# tests/test_advanced/test_calculations_storage.py

from store_calculations import CalculationsStoreJSON

def test_correct_calculations_listed_from_json(tmp_path):
    store = CalculationsStoreJSON(tmp_path)
    calculation_with_multiplication = {"value_1": 2, "value_2": 4, "operation": "multiplication"}

    store.add(calculation_with_multiplication)

    assert store.list_operation_usages("multiplication") == [{"value_1": 2, "value_2": 4, "operation": "multiplication"}]` 
```

该测试检查在使用`CalculationsStoreJSON.add()`方法将计算保存到 JSON 文件时，我们是否可以使用`UserStoreJSON.list_operation_usages()`检索某些操作的列表。

我们将`tmp_path` fixture 传递给这个测试，它返回一个 path ( `pathlib.Path`)对象，该对象指向基目录中的一个临时目录。

当使用`tmp_path`时，pytest 创建一个:

1.  基本临时目录
2.  对每个测试函数调用来说唯一的临时目录(在基目录中)

> 值得注意的是，为了帮助调试，pytest 会在每个测试会话期间创建一个新的基本临时目录，而旧的基本目录会在 3 个会话之后被删除。

## 猴子补丁

使用 [monkeypatching](https://en.wikipedia.org/wiki/Monkey_patch) ，您可以在运行时动态修改一段代码的行为，而无需实际更改源代码。

虽然它不一定仅限于测试，但在 pytest 中，它用于修改被测单元内部代码部分的行为。它通常用于替换昂贵的函数调用，比如对 API 的 HTTP 调用，使用一些预定义的快速且易于控制的虚拟行为。

例如，不是调用真正的 API 来获得响应，而是返回一些在测试中使用的硬编码响应。

让我们深入了解一下。在我们的应用程序中，有一个函数返回从公共 API 中检索到的某个数字的事实:

```
`# number_facts.py

import requests

def get_number_fact(number):
    url = f"http://numbersapi.com/{number}?json"
    response = requests.get(url)
    json_resp = response.json()

    if json_resp["found"]:
        return json_resp["text"]

    return "No fact about this number."` 
```

您不想在测试期间调用 API，因为:

*   它很慢
*   这很容易出错(API 可能会关闭，您的互联网连接可能很差，...)

在这种情况下，您想要模拟响应，所以它返回我们感兴趣的部分*，而不需要*实际发出 HTTP 请求:

```
`# tests/test_advanced/test_number_facts.py

import requests

from number_facts import get_number_fact

class MockedResponse:

    def __init__(self, json_body):
        self.json_body = json_body

    def json(self):
        return self.json_body

def mock_get(*args, **kwargs):
    return MockedResponse({
        "text": "7 is the number of days in a week.",
        "found": "true",
    })

def test_get_number_fact(monkeypatch):
    monkeypatch.setattr(requests, 'get', mock_get)

    number = 7
    fact = '7 is the number of days in a week.'

    assert get_number_fact(number) == fact` 
```

这里发生了很多事情:

1.  测试函数中使用了 pytest 的内置 [monkeypatch](https://docs.pytest.org/en/7.1.x/how-to/monkeypatch.html) 夹具。
2.  使用`monkeypatch.setattr`，我们用自己的函数`mock_get`覆盖了`requests`包的`get`函数。在这个测试的执行过程中，应用程序代码中对`requests.get`的所有调用现在都将实际调用`mock_get`。
3.  `mock_get`函数返回一个`MockedResponse`实例，用我们在`mock_get`函数(`{'"text": "7 is the number of days in a week.", "found": "true",}`)中分配的值替换`json_body`。
4.  每次测试被调用时，不是像产品代码(`get_number_fact`)那样执行`requests.get("http://numbersapi.com/7?json")`，而是返回一个带有硬编码事实的`MockedResponse`。

这样，您仍然可以验证函数的行为(从 API 响应中获得一个数字的事实)，而无需真正调用 API。

## 结论

pytest 在过去几年中成为标准有许多原因，最值得注意的是:

1.  它简化了测试的编写。
2.  由于其全面的输出，可以很容易地查明哪些测试失败以及失败的原因。
3.  它为重复或复杂的测试准备、为测试目的创建文件以及测试隔离提供了解决方案。

pytest 提供了比我们在本文中讨论的更多的东西。

他们的文档包括有用的[操作指南](https://docs.pytest.org/en/7.1.x/contents.html#how-to-guides)，深入涵盖了我们在这里浏览的大部分内容。他们还提供了一些[的例子](https://docs.pytest.org/en/7.1.x/example/index.html#)。

pytest 还附带了一个广泛的[插件列表](https://docs.pytest.org/en/7.0.x/reference/plugin_list.html)，您可以用它来扩展 pytest 的功能。

这里有一些你可能会觉得有用的:

*   pytest-cov 增加了对检查代码覆盖率的支持。
*   pytest-django 增加了一套测试 django 应用程序的有用工具。
*   pytest-xdist 允许您并行运行测试，从而缩短测试运行所需的时间。
*   [pytest-random](https://pypi.org/project/pytest-randomly/)以随机的顺序运行测试，防止它们意外地相互依赖。
*   pytest-asincio 让测试异步程序变得更加容易。
*   pytest-mock 提供了一个 mocker fixture，它是标准 unittest 模拟包和附加实用程序的包装器。

本文应该有助于您理解 pytest 库是如何工作的，以及使用它可以完成什么。然而，理解 pytest 如何工作和**测试**如何工作是不一样的。学习编写有意义的测试需要实践和理解你期望你的代码做什么。