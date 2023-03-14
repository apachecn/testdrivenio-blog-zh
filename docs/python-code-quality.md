# Python 代码质量

> 原文：<https://testdriven.io/blog/python-code-quality/>

*到底什么是代码质量？我们如何衡量它？我们如何提高代码质量，清理我们的 Python 代码？*

[代码质量](https://en.wikipedia.org/wiki/Software_quality)一般指你的代码功能性和可维护性如何。在以下情况下，代码被视为高质量:

1.  这符合它的目的
2.  它的行为是可以测试的
3.  它遵循一贯的风格
4.  可以理解
5.  它不包含安全漏洞
6.  这是有据可查的
7.  很容易维护

因为我们已经在 Python 中的[测试](/blog/testing-python/)和 Python 中的[现代测试驱动开发](/blog/modern-tdd/)文章中解决了前两点，所以本文的重点是第三至第七点。

本文探讨了如何使用 linters、代码格式化器和安全漏洞扫描器来提高 Python 代码的质量。

> [完整 Python](/guides/complete-python/) 指南:
> 
> 1.  [现代 Python 环境——依赖性和工作空间管理](/blog/python-environments/)
> 2.  [Python 中的测试](/blog/testing-python/)
> 3.  [Python 中的现代测试驱动开发](/blog/modern-tdd/)
> 4.  [Python 代码质量](/blog/python-code-quality/)(本文！)
> 5.  [Python 类型检查](/blog/python-type-checking/)
> 6.  [记录 Python 代码和项目](/blog/documenting-python/)
> 7.  [Python 项目工作流程](/blog/python-project-workflow/)

## 棉短绒

[Linters](https://en.wikipedia.org/wiki/Lint_(software)) 通过源代码分析标记编程错误、bug、风格错误和可疑结构。林挺工具易于设置，提供合理的默认值，并通过消除对风格有不同意见的开发人员之间的摩擦来改善整体开发体验。

> 虽然林挺是一种常见的实践，但它仍然被许多开发人员所不喜欢，因为开发人员往往非常固执己见。

让我们看一个简单的例子。

版本一:

```py
`numbers = []

while True:
    answer = input('Enter a number: ')
    if answer != 'quit':
        numbers.append(answer)
    else:
        break

print('Numbers: %s' % numbers)` 
```

版本二:

```py
`numbers = []

while (answer := input("Enter a number: ")) != "quit":
    numbers.append(answer)

print(f"Numbers: {numbers}")` 
```

版本三:

```py
`numbers = []

while True:
    answer = input("Enter a number: ")
    if answer == "quit":
        break
    numbers.append(answer)

print(f"Numbers: {numbers}")` 
```

哪个更好？

就功能而言，它们是相同的。

你喜欢哪一个？你的项目合作者更喜欢哪一个？

作为一名软件开发人员，你很可能在团队中工作。而且，在团队环境中，所有开发人员遵循相同的编码标准是非常重要的。不然看别人的代码就难多了。代码审查的重点应该是更高层次的问题，而不是普通的语法格式问题。

例如，如果你决定用感叹号结束每一句话，读者将很难推断出语气。如果你更进一步，忽略像大写和间距规则这样的通用标准，你的句子将很难阅读。阅读你的作品需要更多的脑力。你会失去读者和合作者。代码也类似。我们使用风格指南使我们的开发伙伴(包括我们自己)更容易推断意图并与我们合作。

作为 Python 开发人员，我们很幸运能够拥有 [PEP-8](https://pep8.org/) 风格指南，它提供了一组约定、指南和最佳实践，使我们的代码更容易阅读和维护。它主要关注命名约定、代码注释和布局问题(比如缩进和空白)。几个例子:

就林挺工具而言，虽然有很多这样的工具，但大部分都是在代码逻辑或强制代码标准中寻找错误:

1.  **代码逻辑**——这些检查编程错误，强制执行代码标准，搜索代码气味，并检查代码复杂性。 [Pyflakes](https://github.com/PyCQA/pyflakes) 和 [McCabe](https://github.com/PyCQA/mccabe) (复杂度检查器)是林挺代码逻辑最流行的工具。
2.  **代码风格**——这些只是执行代码标准(基于 PEP-8)。 [pycodestyle](https://github.com/pycqa/pycodestyle) 就属于这一类。

### 薄片 8

[Flake8](https://flake8.pycqa.org/) 是 Pyflakes、pycodestyle 和 McCabe 的包装器。

它可以像任何其他 PyPI 包一样安装:

假设您将以下代码保存到一个名为 *my_module.py* 的文件中:

```py
`from requests import *

def get_error_message(error_type):
    if error_type == 404:
        return 'red'
    elif error_type == 403:
        return 'orange'
    elif error_type == 401:
        return 'yellow'
    else:
        return 'blue'

def main():
    res = get('https://api.github.com/events')
    STATUS = res.status_code
    if res.ok:
        print(f'{STATUS}')
    else:
        print(get_error_message(STATUS))

if __name__ == '__main__':
    main()` 
```

要 lint 这个文件，您只需运行:

```py
`$ python -m flake8 my_module.py` 
```

这将产生以下输出:

```py
`my_module.py:1:1: F403 'from requests import *' used; unable to detect undefined names
my_module.py:3:1: E302 expected 2 blank lines, found 1
my_module.py:15:11: F405 'get' may be undefined, or defined from star imports: requests
my_module.py:25:1: E303 too many blank lines (4)` 
```

> 根据代码编辑器的配置，您可能还会看到一个`my_module.py:26:11: W292 no newline at end of file`错误。

对于每个违规，都会打印一行，其中包含以下数据:

1.  文件路径(相对于运行 Flake8 的目录)
2.  行数
3.  列号
4.  违反的[规则的 ID](https://www.flake8rules.com/)
5.  规则描述

以`F`开头的违例是来自 [Pyflakes](https://flake8.pycqa.org/en/latest/user/error-codes.html) 的错误，而以`E`开头的违例来自 [pycodestyle](https://pycodestyle.pycqa.org/en/latest/intro.html#error-codes) 。

纠正违规后，您应该:

```py
`from requests import get

def get_error_message(error_type):
    if error_type == 404:
        return 'red'
    elif error_type == 403:
        return 'orange'
    elif error_type == 401:
        return 'yellow'
    else:
        return 'blue'

def main():
    res = get('https://api.github.com/events')
    STATUS = res.status_code
    if res.ok:
        print(f'{STATUS}')
    else:
        print(get_error_message(STATUS))

if __name__ == '__main__':
    main()` 
```

除了 PyFlakes 和 pycodestyle，您还可以使用 Flake8 来检查[圈复杂度](https://en.wikipedia.org/wiki/Cyclomatic_complexity)。

例如，`get_error_message`函数的复杂度为 4，因为有四个可能的分支(或代码路径):

```py
`def get_error_message(error_type):
    if error_type == 404:
        return 'red'
    elif error_type == 403:
        return 'orange'
    elif error_type == 401:
        return 'yellow'
    else:
        return 'blue'` 
```

要强制最大复杂性为 3 或更低，请运行:

```py
`$ python -m flake8 --max-complexity 3 my_module.py` 
```

Flake8 应失败，出现以下情况:

```py
`my_module.py:4:1: C901 'get_error_message' is too complex (4)` 
```

像这样重构代码:

```py
`def get_error_message(error_type):
    colors = {
        404: 'red',
        403: 'orange',
        401: 'yellow',
    }
    return colors[error_type] if error_type in colors else 'blue'` 
```

薄片 8 现在应该通过:

```py
`$ python -m flake8 --max-complexity 3 my_module.py` 
```

你可以通过它强大的插件系统给 Flake8 添加额外的检查。例如，为了实施 PEP-8 命名约定，安装 [pep8-naming](https://github.com/PyCQA/pep8-naming) :

```py
`$ pip install pep8-naming` 
```

运行:

```py
`$ python -m flake8 my_module.py` 
```

您应该看到:

```py
`my_module.py:15:6: N806 variable 'STATUS' in function should be lowercase` 
```

修复:

```py
`def main():
    res = get('https://api.github.com/events')
    status = res.status_code
    if res.ok:
        print(f'{status}')
    else:
        print(get_error_message(status))` 
```

查看[牛逼的 Flake8 扩展](https://github.com/DmytroLitvinov/awesome-flake8-extensions)获得最受欢迎的扩展列表。

> Pylama 也是一种流行的林挺工具，像 Flake8 一样，它可以将几块棉绒粘合在一起。

## 代码格式化程序

虽然 linters 只是检查代码中的问题，但代码格式化程序实际上是基于一组标准来重新格式化代码的。

让你的代码保持正确的格式是一项必要而枯燥的工作，应该由计算机来完成。

为什么有必要？

遵循一致性风格指南的格式良好的代码更容易阅读，这使得找到 bug 和新开发人员更容易。它还减少了合并冲突。

> 可读性很重要。
> 
> –Python 的[禅](https://en.wikipedia.org/wiki/Zen_of_Python)

同样，因为这是一项枯燥的工作，开发人员常常固执己见(制表符对空格，单引号对双引号，等等)。)，使用代码格式化工具根据一组标准自动重新格式化您的代码。

### 伊索特

[isort](https://pycqa.github.io/isort/) 用于自动将代码中的导入分成以下几组:

1.  标准程序库
2.  第三方的
3.  当地的

然后，成组的导入按字母顺序逐个排列。

```py
`# standard library
import datetime
import os

# third-party
import requests
from flask import Flask
from flask.cli import AppGroup

# local
from your_module import some_method` 
```

安装:

> 如果你更喜欢 Flake8 插件，请查看 [flake8-isort](https://github.com/gforcada/flake8-isort) 。[薄片 8-进口订单](https://github.com/PyCQA/flake8-import-order)也很受欢迎。

对当前目录和子目录中的文件运行它:

要对单个文件运行它:

```py
`$ python -m isort my_module.py` 
```

之前:

```py
`import os
import datetime
from your_module import some_method
from flask.cli import AppGroup
import requests
from flask import Flask` 
```

之后:

```py
`import datetime
import os

import requests
from flask import Flask
from flask.cli import AppGroup

from your_module import some_method` 
```

要检查您的导入是否正确排序，并且不做任何更改，请使用`--check-only`标志:

```py
`$ python -m isort my_module.py --check-only

ERROR: my_module.py Imports are incorrectly sorted and/or formatted.` 
```

要查看更改，而不应用更改，请使用`--diff`标志:

```py
`$ python -m isort my_module.py --diff

--- my_module.py:before      2022-02-28 22:04:45.977272
+++ my_module.py:after       2022-02-28 22:04:48.254686
@@ -1,6 +1,7 @@
+import datetime
 import os
-import datetime
+
+import requests
+from flask import Flask
+from flask.cli import AppGroup
 from your_module import some_method
-from flask.cli import AppGroup
-import requests
-from flask import Flask` 
```

使用 isort 和 Black 时，您应该使用`--profile black` [选项](https://pycqa.github.io/isort/docs/configuration/black_compatibility.html),以避免代码风格冲突:

```py
`$ python -m isort --profile black .` 
```

### 黑色

[Black](https://github.com/psf/black) 是一个 Python 代码格式化程序，用于根据 Black 的[代码风格指南](https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html)重新格式化你的代码，它非常接近 PEP-8。

> 更喜欢 Flake8 插件？检查一下 [flake8-black](https://github.com/peterjc/flake8-black) 。

要递归编辑当前目录中的文件:

它也可以针对单个文件运行:

```py
`$ python -m black my_module.py` 
```

之前:

```py
`import pytest

@pytest.fixture(scope="module")
def authenticated_client(app):
    client = app.test_client()
    client.post("/login", data=dict(email="[[email protected]](/cdn-cgi/l/email-protection)", password="notreal"), follow_redirects=True)
    return client` 
```

之后:

```py
`import pytest

@pytest.fixture(scope="module")
def authenticated_client(app):
    client = app.test_client()
    client.post(
        "/login",
        data=dict(email="[[email protected]](/cdn-cgi/l/email-protection)", password="notreal"),
        follow_redirects=True,
    )
    return client` 
```

如果您只想检查您的代码是否遵循 Black code 风格标准，您可以使用`--check`标志:

```
`$ python -m black my_module.py --check

would reformat my_module.py
Oh no! 💥 💔 💥
1 file would be reformatted.` 
```py

与此同时,`--diff`标志显示了当前代码和重新格式化后的代码之间的差异:

```
`$ python -m black my_module.py --diff

--- my_module.py        2022-02-28 22:04:45.977272 +0000
+++ my_module.py        2022-02-28 22:05:15.124565 +0000
@@ -1,7 +1,12 @@
 import pytest
+

 @pytest.fixture(scope="module")
 def authenticated_client(app):
     client = app.test_client()
-    client.post("/login", data=dict(email="[[email protected]](/cdn-cgi/l/email-protection)", password="notreal"), follow_redirects=True)
-    return client
\ No newline at end of file
+    client.post(
+        "/login",
+        data=dict(email="[[email protected]](/cdn-cgi/l/email-protection)", password="notreal"),
+        follow_redirects=True,
+    )
+    return client
would reformat my_module.py

All done! ✨ 🍰 ✨
1 file would be reformatted.` 
```py

> [YAPF](https://github.com/google/yapf) 和 [autopep8](https://github.com/hhatto/autopep8) 是类似于 Black 的代码格式化程序，也值得一看。

## 安全漏洞扫描器

安全漏洞可以说是代码质量最重要的方面，然而它们经常被忽视。您的代码的安全性取决于它最薄弱的环节。幸运的是，有许多工具可以帮助检测我们代码中可能的漏洞。让我们来看看其中的两个。

### 强盗

[Bandit](https://github.com/PyCQA/bandit) 是一款旨在发现 Python 代码中常见安全[问题](https://bandit.readthedocs.io/en/latest/plugins/index.html#complete-test-plugin-listing)的工具，比如硬编码的密码字符串、反序列化不可信代码、在 except 块中使用`pass`等等。

> 更喜欢 Flake8 插件？检查一下 [flake8-bandit](https://github.com/tylerwince/flake8-bandit) 。

像这样运行它:

代码:

```
`evaluate = 'print("Hi!")'
eval(evaluate)

evaluate = 'open("secret_file.txt").read()'
eval(evaluate)` 
```py

您应该会看到以下警告:

```
`>> Issue: [B307:blacklist] Use of possibly insecure function - consider using safer
    ast.literal_eval.
   Severity: Medium   Confidence: High
   Location: my_module.py:2
   More Info:
    https://bandit.readthedocs.io/en/latest/blacklists/blacklist_calls.html#b307-eval
1   evaluate = 'print("Hi!")'
2   eval(evaluate)
3

--------------------------------------------------
>> Issue: [B307:blacklist] Use of possibly insecure function - consider using safer
    ast.literal_eval.
   Severity: Medium   Confidence: High
   Location: my_module.py:6
   More Info:
    https://bandit.readthedocs.io/en/latest/blacklists/blacklist_calls.html#b307-eval
5   evaluate = 'open("secret_file.txt").read()'
6   eval(evaluate)

--------------------------------------------------` 
```py

### 安全

安全是另一个让你的代码免于安全问题的工具。

它用于对照[安全数据库](https://github.com/pyupio/safety-db)检查您已安装的依赖项中已知的安全漏洞，安全数据库是 Python 包中已知安全漏洞的数据库。

激活虚拟环境后，您可以像这样运行它:

安装烧瓶 v0.12.2 时的样品输出:

```
`+==============================================================================+
|                                                                              |
|                               /$$$$$$            /$$                         |
|                              /$$__  $$          | $$                         |
|           /$$$$$$$  /$$$$$$ | $$  \__//$$$$$$  /$$$$$$   /$$   /$$           |
|          /$$_____/ |____  $$| $$$$   /$$__  $$|_  $$_/  | $$  | $$           |
|         |  $$$$$$   /$$$$$$$| $$_/  | $$$$$$$$  | $$    | $$  | $$           |
|          \____  $$ /$$__  $$| $$    | $$_____/  | $$ /$$| $$  | $$           |
|          /$$$$$$$/|  $$$$$$$| $$    |  $$$$$$$  |  $$$$/|  $$$$$$$           |
|         |_______/  \_______/|__/     \_______/   \___/   \____  $$           |
|                                                          /$$  | $$           |
|                                                         |  $$$$$$/           |
|  by pyup.io                                              \______/            |
|                                                                              |
+==============================================================================+
| REPORT                                                                       |
| checked 37 packages, using default DB                                        |
+============================+===========+==========================+==========+
| package                    | installed | affected                 | ID       |
+============================+===========+==========================+==========+
| flask                      | 0.12.2    | <0.12.3                  | 36388    |
| flask                      | 0.12.2    | <1.0                     | 38654    |
+==============================================================================+` 
```py

现在你已经知道了工具，下一个问题是:什么时候应该使用它们？

通常，这些工具会运行:

1.  编码时(在 ide 或代码编辑器中)
2.  提交时(使用预提交挂钩)
3.  当代码签入源代码管理时(通过 CI 管道)

### 在您的 ide 或代码编辑器中

最好尽早并经常检查可能对质量有负面影响的问题。因此，强烈建议在开发过程中对代码进行 lint 和格式化。许多流行的 ide 都内置了 linters 和 formatters。你可以为你的代码编辑器找到一个插件，用于上面提到的大多数工具。此类插件会实时警告您代码风格违规和潜在的编程错误。

资源:

1.  [林挺](https://code.visualstudio.com/docs/python/linting)和[在 Visual Studio 代码中格式化](https://code.visualstudio.com/docs/python/editing#_formatting) Python
2.  [黑色编辑器集成](https://black.readthedocs.io/en/stable/integrations/editors.html)
3.  [崇高文字包查找器](https://packagecontrol.io/)
4.  [Atom 包](https://atom.io/packages)

### 提交前挂钩

由于您在编码时不可避免地会遗漏一些警告，所以在提交时用预提交 git 挂钩检查质量问题是一个好的实践。在 lint 之前，可以先格式化代码。这样，您可以避免在 CI 管道中提交无法通过代码质量检查的代码。

推荐使用[预提交](https://pre-commit.com/)框架来管理 git 挂钩。

安装完成后，添加一个名为*的预提交配置文件。将 pre-commit-config.yaml* 提交到项目中。要运行 Flake8，请添加以下配置:

```
`repos: -  repo:  https://gitlab.com/PyCQA/flake8 rev:  4.0.1 hooks: -  id:  flake8` 
```py

最后，要设置 git 挂钩脚本，运行:

```
`(venv)$ pre-commit install` 
```py

现在，每次运行`git commit` Flake8 都会在实际提交之前运行。如果有任何问题，提交将被中止。

### CI 管道

尽管您可能在代码编辑器和预提交钩子中使用代码质量工具，但您不能总是指望您的队友和其他合作者也这样做。因此，您应该在 CI 管道中运行代码质量检查。此时，您应该运行 linters 和安全漏洞检测器，并确保代码遵循特定的代码风格。您可以在测试的同时运行这样的检查。

## 真实项目

让我们创建一个简单的项目来看看所有这些是如何工作的。

首先，创建一个新文件夹:

```
`$ mkdir flask_example
$ cd flask_example` 
```py

接下来，用[诗歌](https://python-poetry.org)初始化你的项目:

```
`$ poetry init

Package name [flask_example]:
Version [0.1.0]:
Description []:
Author [Your name <[[email protected]](/cdn-cgi/l/email-protection)>, n to skip]:
License []:
Compatible Python versions [^3.10]:

Would you like to define your main dependencies interactively? (yes/no) [yes] no
Would you like to define your development dependencies interactively? (yes/no) [yes] no
Do you confirm generation? (yes/no) [yes]` 
```py

之后，添加 Flask、pytest、Flake8、Black、isort、Bandit 和 Safety:

```
`$ poetry add flask
$ poetry add --dev pytest flake8 black isort safety bandit` 
```py

创建一个文件来保存名为 *test_app.py* 的测试:

```
`from app import app
import pytest

@pytest.fixture
def client():
    app.config['TESTING'] = True

    with app.test_client() as client:
        yield client

def test_home(client):
    response = client.get('/')

    assert response.status_code == 200` 
```py

接下来，为 Flask 应用程序添加一个名为 *app.py* 的文件:

```
`from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return 'OK'

if __name__ == '__main__':
    app.run()` 
```py

现在，我们准备添加预提交配置。

首先，初始化一个新的 git 存储库:

接下来，安装预提交并设置 git 挂钩脚本:

```
`$ poetry add --dev pre-commit
$ poetry run pre-commit install` 
```py

为名为*的配置创建一个文件。预提交配置. yaml* :

```
`repos: -  repo:  https://gitlab.com/PyCQA/flake8 rev:  4.0.1 hooks: -  id:  flake8` 
```py

在提交之前，运行 isort 和 Black:

```
`$ poetry run isort . --profile black
$ poetry run black .` 
```py

提交您的更改以触发预提交挂钩:

```
`$ git add .
$ git commit -m 'Initial commit'` 
```py

最后，让我们通过 [GitHub Actions](https://github.com/features/actions) 配置一个 CI 管道。

创建以下文件和文件夹:

```
`.github
└── workflows
    └── main.yaml` 
```py

*。github/workflows/main.yaml* :

```
`name:  CI on:  [push] jobs: test: strategy: fail-fast:  false matrix: python-version:  [3.10.2] poetry-version:  [1.1.13] os:  [ubuntu-latest] runs-on:  ${{ matrix.os }} steps: -  uses:  actions/[[email protected]](/cdn-cgi/l/email-protection) -  uses:  actions/[[email protected]](/cdn-cgi/l/email-protection) with: python-version:  ${{ matrix.python-version }} -  name:  Run image uses:  abatilo/[[email protected]](/cdn-cgi/l/email-protection) with: poetry-version:  ${{ matrix.poetry-version }} -  name:  Install dependencies run:  poetry install -  name:  Run tests run:  poetry run pytest code-quality: strategy: fail-fast:  false matrix: python-version:  [3.10.2] poetry-version:  [1.1.13] os:  [ubuntu-latest] runs-on:  ${{ matrix.os }} steps: -  uses:  actions/[[email protected]](/cdn-cgi/l/email-protection) -  uses:  actions/[[email protected]](/cdn-cgi/l/email-protection) with: python-version:  ${{ matrix.python-version }} -  name:  Run image uses:  abatilo/[[email protected]](/cdn-cgi/l/email-protection) with: poetry-version:  ${{ matrix.poetry-version }} -  name:  Install dependencies run:  poetry install -  name:  Run black run:  poetry run black . --check -  name:  Run isort run:  poetry run isort . --check-only --profile black -  name:  Run flake8 run:  poetry run flake8 . -  name:  Run bandit run:  poetry run bandit . -  name:  Run saftey run:  poetry run safety check` 
```py

这种配置:

*   每次推送时运行- `on: [push]`
*   运行在最新版本的 Ubuntu - `ubuntu-latest`
*   使用 Python 3.10.2 - `python-version: [3.10.2]`，`python-version: ${{ matrix.python-version }}`
*   1.1.13 -使用诗歌版本`poetry-version: [1.1.13]`，`poetry-version: ${{ matrix.poetry-version }}`

定义了两个作业:`test`和`code-quality`。顾名思义，测试在`test`作业中运行，而我们的代码质量检查在`code-quality`作业中运行。

将配置项配置添加到 git 并提交:

```
`$ git add .github/workflows/main.yaml
$ git commit -m 'Add CI config'` 
```py

在 [GitHub](https://github.com/new) 上创建一个新的资源库，并将您的项目推送到新创建的 remote。

例如:

```
`$ git remote add origin [[email protected]](/cdn-cgi/l/email-protection):jangia/flask_example.git
$ git branch -M main
$ git push -u origin main` 
```

您应该在 GitHub 存储库的 *Actions* 选项卡上看到您的工作流正在运行。

## 结论

代码质量是软件开发中最具争议的话题之一。尤其是代码风格，在开发人员中是一个敏感的问题，因为我们花费了大量的开发时间来阅读代码。当代码具有符合 PEP-8 标准的一致风格时，阅读和推断意图要容易得多。由于这是一个枯燥、平凡的过程，它应该由计算机通过代码格式化程序如 Black 和 isort 来处理。同样，Flake8、Bandit 和 Safety 有助于确保您的代码安全无误。

* * *

> [完整 Python](/guides/complete-python/) 指南:
> 
> 1.  [现代 Python 环境——依赖性和工作空间管理](/blog/python-environments/)
> 2.  [Python 中的测试](/blog/testing-python/)
> 3.  [Python 中的现代测试驱动开发](/blog/modern-tdd/)
> 4.  [Python 代码质量](/blog/python-code-quality/)(本文！)
> 5.  [Python 类型检查](/blog/python-type-checking/)
> 6.  [记录 Python 代码和项目](/blog/documenting-python/)
> 7.  [Python 项目工作流程](/blog/python-project-workflow/)