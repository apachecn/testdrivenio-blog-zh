# Python 3.10:新特性

> 原文：<https://testdriven.io/blog/python310/>

Python 3.10 发布于[2021 年](https://www.python.org/downloads/release/python-3100/)10 月 4 日。本文着眼于 Python 语言中最有趣的新增内容，这些内容将有助于使您的 Python 代码更加整洁:

1.  结构模式匹配
2.  带括号的上下文管理器
3.  更清晰的错误消息
4.  改进的类型注释
5.  拉链的严格论证

## 安装 Python 3.10

如果您有 Docker，您可以快速构建一个 Python 3.10 shell 来使用本文中的示例，如下所示:

```
`$ docker run -it --rm python:3.10` 
```

不用 Docker？我们建议用 [pyenv](https://github.com/pyenv/pyenv) 安装 Python 3.10:

从[Modern Python Environments-dependency and workspace management](https://testdriven.io/blog/python-environments/#installing-python)文章中，您可以了解更多关于使用 pyenv 管理 Python 的信息。

## 结构模式匹配

在所有的新特性中，结构模式匹配最受关注，也是最有争议的。它在 Python 语言中引入了一个`match/case`语句，看起来非常像其他编程语言中的`switch/case`语句。使用结构模式匹配，您可以根据一个或多个模式测试对象，以确定比较对象的结构是否与给定的模式之一匹配。

快速示例:

```
`code = 404

match code:
    case 200:
        print("OK")
    case 404:
        print("Not found")
    case 500:
        print("Server error")
    case _:
        print("Code not found")` 
```

[图案](https://www.python.org/dev/peps/pep-0634/#id2)可以是:

1.  文字
2.  捕获
3.  通配符
4.  价值观念
5.  组
6.  顺序
7.  绘图
8.  班

上面的例子使用了[文字模式](https://www.python.org/dev/peps/pep-0634/#literal-patterns)。

想去掉这个例子中的幻数吗？像这样利用[值模式](https://www.python.org/dev/peps/pep-0634/#value-patterns):

```
`from http import HTTPStatus

code = 404

match code:
    case HTTPStatus.OK:
        print("OK")
    case HTTPStatus.NOT_FOUND:
        print("Not found")
    case HTTPStatus.INTERNAL_SERVER_ERROR:
        print("Server error")
    case _:
        print("Code not found")

# => "Not found"` 
```

您也可以使用[或模式](https://www.python.org/dev/peps/pep-0634/#or-patterns)组合多个模式:

```
`from http import HTTPStatus

code = 400

match code:
    case HTTPStatus.OK:
        print("OK")
    case HTTPStatus.NOT_FOUND | HTTPStatus.BAD_REQUEST:
        print("You messed up")
    case HTTPStatus.INTERNAL_SERVER_ERROR | HTTPStatus.BAD_GATEWAY:
        print("Our bad")
    case _:
        print("Code not found")

# => "You messed up"` 
```

结构化模式匹配与其他语言中的`switch/case`语法的不同之处在于，您可以解包复杂的数据类型，并基于结果数据执行操作:

```
`point = (0, 10)

match point:
    case (0, 0):
        print("Origin")
    case (0, y):
        print(f"Y={y}")
    case (x, 0):
        print(f"X={x}")
    case (x, y):
        print(f"X={x}, Y={y}")
    case _:
        raise ValueError("Not a point")

# => Y=10` 
```

这里，值(`10`)从主题(`(0, 10)`)绑定到案例内部的变量。这是[捕获模式](https://www.python.org/dev/peps/pep-0634/#capture-patterns)。

你可以用[类模式](https://www.python.org/dev/peps/pep-0634/#class-patterns)实现几乎同样的事情:

```
`from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

point = Point(10, 10)

match point:
    case Point(x=0, y=0):
        print("Origin")
    case Point(x=0, y=y):
        print(f"On Y axis with Y={y}")
    case Point(x=x, y=0):
        print(f"On X axis with X={x}")
    case Point(x=x, y=y):
        print(f"Somewhere in a X, Y plane with X={x}, Y={y}")
    case _:
        print("Not a point")

# => Somewhere in a X, Y plane with X=10, Y=10` 
```

您可以使用[保护符](https://www.python.org/dev/peps/pep-0634/#id1)来添加 if 子句，如下所示:

```
`from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

point = Point(7, 0)

match point:
    case Point(x, y) if x == y:
        print(f"The point is located on the diagonal Y=X at {x}.")
    case Point(x, y):
        print(f"Point is not on the diagonal.")

# => Point is not on the diagonal.` 
```

因此，当 guard 为`False`时，将评估下一个案例。

> 值得注意的是，`match`和`case`是[软关键字](https://www.python.org/dev/peps/pep-0622/#backwards-compatibility)，所以你仍然可以在你现有的代码中使用它们作为变量名。

更多信息，请参考[正式文件](https://docs.python.org/3.10/whatsnew/3.10.html#pep-634-structural-pattern-matching)以及相关 pep:

1.  [PEP 622 -提案](https://www.python.org/dev/peps/pep-0622/)
2.  [PEP 634 -规格](https://www.python.org/dev/peps/pep-0634/)
3.  [PEP 635 -动机和基本原理](https://www.python.org/dev/peps/pep-0635/)
4.  [人教版 636 -教程](https://www.python.org/dev/peps/pep-0636/)

## 带括号的上下文管理器

使用上下文管理器时，Python 现在支持跨多行的延续:

```
`with (
    CtxManager1() as ctx1,
    CtxManager2() as ctx2
):` 
```

在以前的版本中，你必须把所有东西都放在同一行或者嵌套`with`语句。

Python < 3.10:

```
`import unittest
from unittest import mock

from my_module import my_function

class Test(unittest.TestCase):
    def test(self):
        with mock.patch("my_module.secrets.token_urlsafe") as a, mock.patch("my_module.string.capwords") as b, mock.patch("my_module.collections.defaultdict") as c:
            my_function()
            a.assert_called()
            b.assert_called()
            c.assert_called()

    def test_same(self):
        with mock.patch("my_module.secrets.token_urlsafe") as a:
            with mock.patch("my_module.string.capwords") as b:
                with mock.patch("my_module.collections.defaultdict") as c:
                    my_function()
                    a.assert_called()
                    b.assert_called()
                    c.assert_called()` 
```

Python >= 3.10:

```
`class Test(unittest.TestCase):
    def test(self):
        with (
            mock.patch("my_module.secrets.token_urlsafe") as a,
            mock.patch("my_module.string.capwords") as b,
            mock.patch("my_module.collections.defaultdictl") as c,
        ):
            my_function()
            a.assert_called()
            b.assert_called()
            c.assert_called()` 
```

值得注意的是，在 Python 3.9 中，Python 切换到基于 [PEG](/blog/python39/#new-parser) 的解析器支持了这个特性。在可预见的未来，我们应该会在每个新的 Python 版本中看到像这样的新特性和变化，这*应该* (1)导致更优雅的语法和(2)让所有人感到不安。

更多信息:

1.  [发行说明](https://docs.python.org/3.10/whatsnew/3.10.html#parenthesized-context-managers)
2.  [PEP-617](https://www.python.org/dev/peps/pep-0617/)
3.  [原始 bug](https://bugs.python.org/issue12782)

## 更清晰的错误消息

Python 3.10 改进了错误消息，提供了关于错误和错误实际发生位置的更精确的信息。

例如，在 Python 3.10 之前，如果您缺少了一个右括号`}`

```
`import datetime

expected = {'Jan', 'Mike', 'Marry',

today = datetime.datetime.today()` 
```

-您将看到以下错误消息:

```
 `File "example.py", line 5
    today = datetime.datetime.today()
          ^
SyntaxError: invalid syntax` 
```

使用 Python 3.10，您将看到:

```
 `File "example.py", line 3
    expected = {'Jan', 'Mike', 'Marry',
               ^
SyntaxError: '{' was never closed` 
```

正如您所看到的，使用新的错误消息更容易发现实际问题。这对初学者特别有帮助，使其更容易识别错误的真正原因

1.  缺少关键字
2.  不正确或拼写错误的关键字或变量名
3.  缺少冒号
4.  不正确的缩进
5.  缺少右括号或大括号

更多信息请参考[官方文件](https://docs.python.org/3.10/whatsnew/3.10.html#pep-634-structural-pattern-matching)。

> 看起来 Python 3.11 也将发布对错误消息的另一个[改进。](https://docs.python.org/3.11/whatsnew/3.11.html#enhanced-error-locations-in-tracebacks)

## 改进的类型注释

Python 3.10 提供了许多与类型注释相关的改进:

1.  PEP 604:允许将联合类型写成 X | Y
2.  PEP 613:显式类型别名
3.  PEP 647:用户定义的类型保护
4.  PEP 612:参数规范变量

### 联合运算符

首先，您还可以使用一个新的类型联合操作符`|`，这样您就可以表示 X 或 Y 类型，而不必从`typing`模块导入`Union`:

```
`# before
from typing import Union

def sum_xy(x: Union[int, float], y: Union[int, float]) -> Union[int, float]:
    return x + y

# after
def sum_xy(x: int | float, y: int | float) -> int | float:
    return x + y` 
```

更多信息:

1.  [正式文件](https://docs.python.org/3.10/whatsnew/3.10.html#pep-604-new-type-union-operator)
2.  [PEP-604](https://www.python.org/dev/peps/pep-0604/)

### 键入别名

另一个好处是显式定义类型别名的能力。静态类型检查器和其他开发人员有时会在区分变量赋值和类型别名方面遇到问题。

例如:

```
`StrCache = "Cache[str]"    # a type alias
LOG_PREFIX = "LOG[DEBUG]"  # a module constant` 
```

在 Python 3.10 中，可以使用`TypeAlias`显式定义类型别名:

```
`StrCache: TypeAlias = "Cache[str]"  # a type alias
LOG_PREFIX = "LOG[DEBUG]"           # a module constant` 
```

这将为类型检查者和阅读您代码的其他开发人员理清思路。

更多信息:

1.  [正式文件](https://docs.python.org/3.10/whatsnew/3.10.html#pep-613-typealias)
2.  [PEP-613](https://www.python.org/dev/peps/pep-0613/)

### 防护类型

类型保护有助于类型收缩，这是将类型从不太精确的类型(基于其定义)移动到更精确的类型(在程序的代码流中)的过程。

以下面两种风格的`is_employee`为例:

```
`# without type guards
def is_employee(user: User) -> bool:
    return isinstance(user, Employee)

# with type guards
from typing import TypeGuard

def is_employee(user: User) -> TypeGuard[Employee]:
    return isinstance(user, Employee)` 
```

因此，使用第二种风格的`is_employee`，当它返回`True`时，类型检查器将能够把`user`从`User`缩小到`Employee`。

更多信息:

1.  [正式文件](https://docs.python.org/3.10/whatsnew/3.10.html#pep-647-user-defined-type-guards)
2.  [PEP-647](https://www.python.org/dev/peps/pep-0647/)

### 参数规格变量

为了支持更高阶函数(例如 decorators)的真正注释，Python 3.10 增加了`typing.ParamSpec`和`typing.Concatenate`。

例如(从 [PEP-612](https://www.python.org/dev/peps/pep-0612/) ):

```
`from typing import Awaitable, Callable, TypeVar

R = TypeVar("R")

def add_logging(f: Callable[..., R]) -> Callable[..., Awaitable[R]]:
    async def inner(*args: object, **kwargs: object) -> R:
        await log_to_database()
        return f(*args, **kwargs)
    return inner

@add_logging
def takes_int_str(x: int, y: str) -> int:
    return x + 7

await takes_int_str(1, "A")
await takes_int_str("B", 2) # fails at runtime` 
```

在这个例子中，当使用不正确的参数调用修饰函数时，您的代码将在运行时失败。

现在，您可以通过使用`typing.ParamSpec`来强制参数类型:

```
`from typing import Awaitable, Callable, ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def add_logging(f: Callable[P, R]) -> Callable[P, Awaitable[R]]:
    async def inner(*args: P.args, **kwargs: P.kwargs) -> R:
        await log_to_database()
        return f(*args, **kwargs)
    return inner

@add_logging
def takes_int_str(x: int, y: str) -> int:
    return x + 7

await takes_int_str(1, "A") # Accepted
await takes_int_str("B", 2) # Correctly rejected by the type checker` 
```

类型检查器，比如 [mypy](https://mypy.readthedocs.io/) ，会在分析代码时捕捉错误。

更多信息:

1.  [正式文件](https://docs.python.org/3.10/whatsnew/3.10.html#pep-612-parameter-specification-variables)
2.  [PEP-612](https://www.python.org/dev/peps/pep-0612/)

### 注释的延期评估

不是在函数定义时评估注释，注释的延迟评估建议将它们作为字符串保存在内置的 __annotations__ 字典中。在运行时利用注释的工具将需要通过`typing.get_type_hints()`显式地评估注释，而不是依赖已经被评估的注释。

这原本是 Python 3.10 的一部分，但是由于 Python 的作用域规则，像 [pydantic](https://pydantic-docs.helpmanual.io/) 这样的工具很难利用`typing.get_type_hints()`来获得类型。所以， [Python 指导委员会](https://mail.python.org/archives/list/python-dev@python.org/thread/CLVXXPQ2T2LQ5MP2Y53VVQFCXYWQJHKZ/)决定推迟改变。我们可能会在 Python 3.11 中看到。

更多信息:

1.  [PEP 563，PEP 649，以及 Python 类型注释的未来](https://realpython.com/python-news-april-2021/#pep-563-pep-649-and-the-future-of-python-type-annotations)
2.  [重要提示:PEP 563、PEP 649 和 pydantic 的未来](https://github.com/samuelcolvin/pydantic/issues/2678)
3.  [FastAPI 和 Pydantic 的未来一片光明](https://dev.to/tiangolo/the-future-of-fastapi-and-pydantic-is-bright-3pbm)
4.  [PEP-563](https://www.python.org/dev/peps/pep-0563/)

## 拉链的严格论证

当你试图[压缩](https://docs.python.org/3/library/functions.html#zip)两个长度不同的可重复项时会发生什么？

```
`names = ["Jan", "Mike", "Marry", "Daisy"]
grades = ["B+", "A", "A+"]

for name, grade in zip(names, grades):
    print(name, grade)

"""
Jan B+
Mike A
Marry A+
"""` 
```

当到达较短的 iterable 末尾时，迭代停止。

Python 3.10 引入了一个新的`strict`关键字参数，以确保在运行时所有的 iterables 都具有相同的长度:

```
`names = ["Jan", "Mike", "Marry", "Daisy"]
grades = ["B+", "A", "A+"]

for name, grade in zip(names, grades, strict=True):
    print(name, grade)

# ValueError: zip() argument 2 is shorter than argument 1` 
```

参考 [PEP-618](https://www.python.org/dev/peps/pep-0618/) 了解更多信息。

## 其他更新和优化

[distutils](https://docs.python.org/3/library/distutils.html) 包是[弃用的](https://docs.python.org/3.10/whatsnew/3.10.html#distutils)，将在 Python 3.12 中被完全移除。其功能存在于[设置工具](https://github.com/pypa/setuptools)和[包装](https://github.com/pypa/packaging)中。

最后，Python 3.10 引入了几个导致性能提高的[优化](https://docs.python.org/3.10/whatsnew/3.10.html#optimizations)。对于小对象，构造函数`str()`、`bytes()`和`bytearray()`要快 30%到 40% [。](https://bugs.python.org/issue41334) [runpy](https://docs.python.org/3/library/runpy.html) 模块现在导入更少的模块，所以`python -m module-name`平均快 1.4 倍。

## 结论

如您所见，Python 3.10 带来了许多新特性。一些旧的，很少使用的功能已经贬值或完全删除。本文只是简单介绍了该语言的新特性和变化。请务必查看所有更改的官方发行说明:[Python 3.10 中的新特性](https://docs.python.org/3.10/whatsnew/3.10.html)。

快乐的蟒蛇！