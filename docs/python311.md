# Python 3.11:新特性

> 原文：<https://testdriven.io/blog/python311/>

Python 3.11 发布于[2021 年](https://www.python.org/downloads/release/python-3110/)10 月 24 日。本文着眼于 Python 语言中最有趣的新增内容，这些内容将有助于使您的 Python 代码更加整洁:

1.  更快的 CPython
2.  改进的类型提示
3.  更好的错误消息
4.  异常注释
5.  TOML 库

## 安装 Python 3.11

如果您有 Docker，您可以快速构建一个 Python 3.11 shell 来使用本文中的示例，如下所示:

```py
`$ docker run -it --rm python:3.11` 
```

不用 Docker？我们建议用 [pyenv](https://github.com/pyenv/pyenv) 安装 Python 3.11:

从[Modern Python Environments-dependency and workspace management](https://testdriven.io/blog/python-environments/#installing-python)文章中，您可以了解更多关于使用 pyenv 管理 Python 的信息。

## 更快的 CPython

Python 3.11 比以往任何时候都快！正如[发布说明](https://docs.python.org/3.11/whatsnew/3.11.html)中所说，Python 3.11 比 Python 3.10 快 10 - 60%。平均来说，要快 25%。Python 的核心开发人员在许多方面都做了很好的工作，改善了执行时间。

### 更快启动

启动方面，不是*读取`__pycache__` - >解组- >堆分配的代码对象- >求值*，Python 3.11 使用了新的进程。Python 解释器启动所必需的核心模块现在被冻结在解释器中——它们的代码是静态分配的。新的流程是*静态分配代码对象- >评估*。后者快 10 - 15%。

### 更快的运行时间

每次在 Python 中调用函数时，都会创建一个框架。它保存关于函数执行的信息。核心开发人员简化了他们的创建过程、内部信息和内存分配。另一个改进发生在内联函数调用上。从现在开始，大多数 Python 函数调用都不消耗 C 堆栈空间。

也许在这方面最重要的改进是 [PEP 659:专门化自适应解释器](https://peps.python.org/pep-0659/)。这个 PEP 是 fast Python 的关键元素之一。它的主要思想是 Python 代码有类型很少改变的区域。在这些区域中，当 Python 发现某些操作总是只针对特定类型的数据时，它可以通过使用更专门化的类型来优化操作。例如，如果只使用整数，它可以对整数使用乘法，而不是使用一般的乘法。另一个例子是直接调用底层 C 实现的常见内置函数，如`len`和`str`，而不是通过内部调用约定。

您可以通过检查生成的字节码来观察这一点:

```py
`import dis
from random import random

def dollars_to_pounds(dollars):
    return 0.87 * dollars

dis.dis(dollars_to_pounds, adaptive=True)
#   5           0 RESUME                   0
#
#   6           2 LOAD_CONST               1 (0.87)
#               4 LOAD_FAST                0 (dollars)
#               6 BINARY_OP                5 (*)
#              10 RETURN_VALUE

for _ in range(8):
    dollars_to_pounds(random() * 100)

dis.dis(dollars_to_pounds, adaptive=True)
#   5           0 RESUME_QUICK             0
#
#   6           2 LOAD_CONST__LOAD_FAST     1 (0.87)
#               4 LOAD_FAST                0 (dollars)
#               6 BINARY_OP_MULTIPLY_FLOAT     5 (*)  # <--  CHANGED!
#              10 RETURN_VALUE` 
```

正如您所看到的，在使用 float 调用了 8 次`dollars_to_pounds`之后，bytcode 得到了优化。它没有使用`BINARY_OP`，而是使用了专门的`BINARY_OP_MULTIPLY_FLOAT`，在乘以浮点数时速度更快。如果你用一个旧的 Python 版本运行同样的东西，不会有任何不同。

更多信息，请参考[官方文档](https://docs.python.org/3/whatsnew/3.11.html#whatsnew311-faster-cpython)以及相关的 [PEP 659 -专业自适应解释器](https://peps.python.org/pep-0659/)。

## 改进的类型提示

与前几个版本一样，在类型提示方面有所改进。

### 自身类型

Python 终于支持了一个`Self`类型。因此，现在您可以轻松地键入类方法和 dunder 方法:

```py
`from typing import Self
from dataclasses import dataclass

@dataclass
class Car:
    manufacture: str
    model: str

    @classmethod
    def from_dict(cls, car_data: dict[str, str]) -> Self:
        return cls(manufacture=car_data["manufacture"], model=car_data["model"])

print(Car.from_dict({"manufacture": "Alfa Romeo", "model": "Stelvio"}))
# Car(manufacture='Alfa Romeo', model='Stelvio')` 
```

更多信息:

1.  [正式文件](https://docs.python.org/3.11/whatsnew/3.11.html#whatsnew311-pep673)
2.  [PEP-673](https://peps.python.org/pep-0673/)

### TypedDict 不需要

另一个改进是用于类型化词典的`NotRequired`类型:

```py
`from typing import TypedDict, NotRequired

class Car(TypedDict):
    manufacture: str
    model: NotRequired[str]

car1: Car = {"manufacture": "Alfa Romeo", "model": "Stelvio"}  # OK
car2: Car = {"manufacture": "Alfa Romeo"}  # model (model is not required)
car3: Car = {"model": "Stelvio"}  # ERROR (missing required field manufacture)` 
```

更多信息:

1.  [正式文件](https://docs.python.org/3.11/whatsnew/3.11.html#whatsnew311-pep655)
2.  [PEP-655](https://peps.python.org/pep-0655/)

### 文字字符串类型

还有一个新的`LiteralString`类型，允许文字字符串和从其他文字字符串创建的字符串。这可用于在执行 SQL 和 shell 命令时进行类型检查，以增加防止注入攻击的另一层安全性。

[举例](https://docs.python.org/3/whatsnew/3.11.html#pep-675-arbitrary-literal-string-type):

```py
`def run_query(sql: LiteralString) -> ...
    ...

def caller(
    arbitrary_string: str,
    query_string: LiteralString,
    table_name: LiteralString,
) -> None:
    run_query("SELECT * FROM students")       # ok
    run_query(query_string)                   # ok
    run_query("SELECT * FROM " + table_name)  # ok
    run_query(arbitrary_string)               # type checker error
    run_query(                                # type checker error
        f"SELECT * FROM students WHERE name = {arbitrary_string}"
    )` 
```

更多信息:

1.  [正式文件](https://docs.python.org/3/whatsnew/3.11.html#whatsnew311-pep675)
2.  [PEP-675](https://peps.python.org/pep-0675/)

--

还有一些与类型提示相关的其他改进。你可以在[官方发行说明](https://docs.python.org/3/whatsnew/3.11.html#new-features-related-to-type-hints)中读到它们。

## 更好的错误消息

令人兴奋的新特性之一是更具描述性的回溯。Python 3.10 引入了更好、更具描述性的错误，带来了一些改进。Python 3.11 更进了一步，改进了对确切错误位置的描述。让我们看一个例子。

有语法错误的代码:

```py
`def average_grade(grades):
    return sum(grades) / len(grades)

average_grade([])` 
```

Python 3.10 中的错误:

```py
 `return sum(grades) / len(grades)
ZeroDivisionError: division by zero` 
```

Python 3.11 中的错误:

```py
 `return sum(grades) / len(grades)
           ~~~~~~~~~~~~^~~~~~~~~~~~~
ZeroDivisionError: division by zero` 
```

来点更复杂的怎么样？

Python 3.10:

```py
`def send_email_to_contact(contact, subject, content):
    print(f"Sending email to {contact['emails']['address']} with subject={subject.title()} and {content=}")

contact = {
    "first_name": "Lightning",
    "last_name": "McQueen",
    "emails": [
        {
            "address": "[[email protected]](/cdn-cgi/l/email-protection)",
            "display_name": "Lightning McQueen"
        }
    ]
}

send_email_to_contact(contact, "Hello", "Hello there! Long time no see.")
#    print(f"Sending email to {contact['emails']['address']} with subject={subject.title()} and {content=}")
# TypeError: list indices must be integers or slices, not str

send_email_to_contact({}, "Hello", "Hello there! Long time no see.")
#    print(f"Sending email to {contact['emails']['address']} with subject={subject.title()} and {content=}")
# KeyError: 'emails'

send_email_to_contact({"emails": {"address": "[[email protected]](/cdn-cgi/l/email-protection)"}}, None, "Hello there! Long time no see.")
#     print(f"Sending email to {contact['emails']['address']} with subject={subject.title()} and {content=}")
# AttributeError: 'NoneType' object has no attribute 'title'` 
```

Python 3.11:

```py
`def send_email_to_contact(contact, subject, content):
    print(f"Sending email to {contact['emails']['address']} with subject={subject.title()} and {content=}")

contact = {
    "first_name": "Lightning",
    "last_name": "McQueen",
    "emails": [
        {
            "address": "[[email protected]](/cdn-cgi/l/email-protection)",
            "display_name": "Lightning McQueen"
        }
    ]
}

send_email_to_contact(contact, "Hello", "Hello there! Long time no see.")
#     print(f"Sending email to {contact['emails']['address']} with {subject=} and {content=}")
#                               ~~~~~~~~~~~~~~~~~^^^^^^^^^^^
# TypeError: list indices must be integers or slices, not str

send_email_to_contact({}, "Hello", "Hello there! Long time no see.")
#     print(f"Sending email to {contact['emails']['address']} with subject={subject.title()} and {content=}")
#                               ~~~~~~~^^^^^^^^^^
# KeyError: 'emails'

send_email_to_contact({"emails": {"address": "[[email protected]](/cdn-cgi/l/email-protection)"}}, None, "Hello there! Long time no see.")
#     print(f"Sending email to {contact['emails']['address']} with subject={subject.title()} and {content=}")
#                                                                           ^^^^^^^^^^^^^
# AttributeError: 'NoneType' object has no attribute 'title'` 
```

正如您所看到的，通过新的回溯可以更容易地找到错误所在。异常行中的确切问题点被很好地标记出来。在旧版本中，您只能看到异常本身和引发异常的行。

更多信息:

1.  [正式文件](https://docs.python.org/3/whatsnew/3.11.html#whatsnew311-pep657)
2.  [PEP-657](https://peps.python.org/pep-0657/)

## 异常注释

`add_note`被添加到`BaseExceptions`。这允许您在创建异常后向其添加额外的上下文。例如:

```py
`try:
    raise ValueError()
except ValueError as exc:
    exc.add_note("When this happened my dog was barking and my kids were sleeping.")
    raise

#     raise ValueError()
# ValueError
# When this happened my dog was barking and my kids were sleeping.` 
```

更多信息:

1.  [正式文件](https://docs.python.org/3/whatsnew/3.11.html#whatsnew311-pep678)
2.  [PEP-678](https://peps.python.org/pep-0678/)

## TOML 库

Python 现在有了一个用于解析 [TOML](https://toml.io/) 文件的库，名为 [tomllib](https://docs.python.org/3.11/library/tomllib.html) 。它的用法和内置的`json`库非常相似。

例如，假设您有下面的 *pyproject.toml* 文件:

```py
`[tool.poetry] name  =  "example" version  =  "0.1.0" description  =  "" authors  =  [] [tool.poetry.dependencies] python  =  "^3.11" [tool.poetry.dev-dependencies] [build-system] requires  =  ["poetry-core>=1.0.0"] build-backend  =  "poetry.core.masonry.api"` 
```

您可以像这样加载文件:

```py
`import pprint
import tomllib

with open("pyproject.toml", "rb") as f:
    data = tomllib.load(f)

pp = pprint.PrettyPrinter(depth=4)
pp.pprint(data)
"""
{'build-system': {'build-backend': 'poetry.core.masonry.api',
 'requires': ['poetry-core>=1.0.0']},
 'tool': {'poetry': {'authors': [],
 'dependencies': {'python': '^3.11'},
 'description': '',
 'dev-dependencies': {},
 'name': 'example',
 'version': '0.1.0'}}}
"""` 
```

更多信息:

1.  [正式文件](https://docs.python.org/3/library/tomllib.html#module-tomllib)
2.  [PEP-680](https://peps.python.org/pep-0680/)

## 结论

如您所见，Python 3.11 带来了许多令人兴奋的改进和特性。另一方面，一些遗留模块被弃用，另一些则被完全删除。因为这篇文章只涵盖了最有趣的新特性和变化，所以请务必查看所有变化的官方发布说明。

快乐的蟒蛇！