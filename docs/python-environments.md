# 现代 Python 环境——依赖性和工作空间管理

> 原文：<https://testdriven.io/blog/python-environments/>

一旦您经历了为单个“hello world”风格的应用程序设置 Python 环境的痛苦，您将需要经历一个更加困难的过程，弄清楚如何为多个 Python 项目管理多个环境。一些项目可能是新的，而另一些则是十年前的陈旧代码。幸运的是，有许多工具可以帮助简化依赖性和工作空间管理。

在本文中，我们将回顾用于依赖性和工作空间管理的可用工具，以便解决以下问题:

1.  在同一台机器上安装和切换不同版本的 Python
2.  管理依赖关系和虚拟环境
3.  复制环境

> [完整 Python](/guides/complete-python/) 指南:
> 
> 1.  [现代 Python 环境——依赖性和工作空间管理](/blog/python-environments/)(本文！)
> 2.  [Python 中的测试](/blog/testing-python/)
> 3.  [Python 中的现代测试驱动开发](/blog/modern-tdd/)
> 4.  [Python 代码质量](/blog/python-code-quality/)
> 5.  [Python 类型检查](/blog/python-type-checking/)
> 6.  [记录 Python 代码和项目](/blog/documenting-python/)
> 7.  [Python 项目工作流程](/blog/python-project-workflow/)

## 安装 Python

虽然您可以从官方的二进制文件或通过系统的包管理器下载并安装 Python，但是您应该避开这些方法，除非您足够幸运地在当前和未来的项目中使用相同版本的 Python。由于情况可能并非如此，我们建议用 [pyenv](https://github.com/pyenv/pyenv) 安装 Python。

pyenv 是一个工具，它简化了在同一台机器上不同版本 Python 之间的安装和切换。它保持 Python 的系统版本不变，这是一些操作系统正常运行所必需的，同时仍然可以根据特定项目的需求轻松切换 Python 版本。

> 不幸的是，pyenv 不能在 Linux 的 [Windows 子系统之外的 Windows 上工作。如果你是这种情况，请查看](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux) [pyenv-win](https://github.com/pyenv-win/pyenv-win) 。

一旦[安装了](https://github.com/pyenv/pyenv#installation)，你就可以轻松地安装 Python 的特定版本，如下所示:

```py
`$ pyenv install 3.8.5
$ pyenv install 3.8.6
$ pyenv install 3.9.0
$ pyenv install 3.10.2

$ pyenv versions
* system
  3.8.5
  3.8.6
  3.9.0
  3.10.2` 
```

然后，您可以像这样设置您的全局 Python 版本:

```py
`$ pyenv global 3.8.6

$ pyenv versions
  system
  3.8.5
* 3.8.6 (set by /Users/michael/.pyenv/version)
  3.9.0
  3.10.2

$ python -V
Python 3.8.6` 
```

> 请记住，这不会在系统级修改或影响 Python。

以类似的方式，您可以为当前文件夹设置 Python 解释器:

```py
`$ pyenv local 3.10.2

$ pyenv versions
  system
  3.8.5
  3.8.6
  3.9.0
* 3.10.2 (set by /Users/michael/repos/testdriven/python-environments/.python-version)

$ python -V
Python 3.10.2` 
```

现在，每次在该文件夹中运行 Python 时，都将使用版本 3.10.2。

> 如果您发现您不能在不同版本的 Python 之间切换，您可能需要重新配置您的 shell 配置文件。详见[安装指南](https://github.com/pyenv/pyenv#installation)。

## 管理依赖关系

在本节中，我们将了解几个用于管理依赖关系和虚拟环境的工具。

### venv + pip

[venv](https://docs.python.org/3/library/venv.html) 和[pip](https://pypi.org/project/pip/)(**p**package**I**installer for**p**ython)，它们预装在 Python 的大多数版本中，分别是最流行的管理虚拟环境和包的工具。它们使用起来相当简单。

> 虚拟环境可以防止依赖性版本冲突。您可以在不同的虚拟环境中安装同一依赖项的不同版本。

您可以在当前文件夹中创建一个名为 *my_venv* 的新虚拟环境，如下所示:

创建好环境后，您仍然需要通过在虚拟环境中获取 *activate* 脚本来激活它:

```py
`$ source my_venv/bin/activate
(my_venv)$` 
```

> 停用运行`deactivate`。然后，为了重新激活，在根项目目录中运行`source my_venv/bin/activate`。

在虚拟环境激活时运行`which python`将返回虚拟环境内 Python 解释器的路径:

```py
`(my_venv)$ which python

/Users/michael/repos/testdriven/python-environments/my_venv/bin/python` 
```

您可以通过在激活虚拟环境的情况下运行`pip install <package-name>`来安装项目的本地包:

```py
`(my_venv)$ python -m pip install requests` 
```

pip 从 [PyPI](https://pypi.org/) (Python 包索引)下载包，然后在虚拟环境中提供给 Python 解释器。

为了环境的可再现性，您通常希望在一个 *requirements.txt* 文件中保存一个项目所需包的列表。您可以手动创建文件并添加它们，或者使用 [pip 冻结](https://pip.pypa.io/en/latest/cli/pip_freeze/)命令来生成它:

```py
`(my_venv)$ python -m pip freeze > requirements.txt

(my_venv)$ cat requirements.txt
certifi==2021.10.8
charset-normalizer==2.0.12
idna==3.3
requests==2.27.1
urllib3==1.26.8` 
```

> 想要只获取顶级依赖项(例如，`requests==2.27.1`)？检查一下 [pip-chill](https://pip-chill.readthedocs.io/en/latest/) 。

虽然 venv 和 pip 都很容易使用，但与更现代的工具如[poem](https://python-poetry.org)和 [Pipenv](https://github.com/pypa/pipenv) 相比，它们还是非常原始的。venv 和 pip 对它使用的 Python 版本一无所知。您必须手动管理所有依赖关系和虚拟环境。你必须自己创建和管理 *requirements.txt* 文件。更重要的是，你必须手工分离开发(pytest，black，isort，...)和生产(Flask，Django，FastAPI，..)依赖关系，使用一个 *requirements-dev.txt* 文件。

*需求-开发文本*:

```py
`# prod
-r requirements.txt

# dev
black==22.1.0
coverage==6.3.2
flake8==4.0.1
ipython==8.0.1
isort==5.10.1
pytest-django==4.5.2
pytest-cov==3.0.0
pytest-xdist==2.5.0
pytest-mock==3.7.0` 
```

*需求. txt* :

```py
`Django==4.0.2
django-allauth==0.49.0
django-crispy-forms==1.14.0
django-rq==2.5.1
django-rq-email-backend==0.1.3
gunicorn==20.1.0
psycopg2-binary==2.9.3
redis==4.1.4
requests==2.27.1
rq==1.10.1
whitenoise==6.0.0` 
```

诗歌和 Pipenv 结合了 venv 和 pip 的功能。它们还使分离开发和生产依赖变得容易，并通过锁文件实现确定性构建。他们与 pyenv 合作得很好。

> 锁定文件锁定(或锁定)整个依赖关系树中的所有依赖关系版本。

### 诗意

诗歌可以说是 Python 中功能最丰富的依赖管理工具。它附带了一个用于创建和管理 Python 项目的强大的 CLI。一旦[安装了](https://python-poetry.org/docs/#installation)，脚手架上一个新的项目开始运行:

```py
`$ poetry new sample-project
$ cd sample-project` 
```

这将创建以下文件和文件夹:

```py
`sample-project
├── README.rst
├── pyproject.toml
├── sample_project
│   └── __init__.py
└── tests
    ├── __init__.py
    └── test_sample_project.py` 
```

依赖项在 *pyproject.toml* 文件中进行管理:

```py
`[tool.poetry] name  =  "sample-project" version  =  "0.1.0" description  =  "" authors  =  ["John Doe <[[email protected]](/cdn-cgi/l/email-protection)>"] [tool.poetry.dependencies] python  =  "^3.10" [tool.poetry.dev-dependencies] pytest  =  "^5.2" [build-system] requires  =  ["poetry-core>=1.0.0"] build-backend  =  "poetry.core.masonry.api"` 
```

> 更多关于 pyproject.toml 的信息，新的 Python 包配置文件将“每个项目都视为一个包”，查看[py project . toml 到底是什么？](https://snarky.ca/what-the-heck-is-pyproject-toml/)文章。

要添加新的依赖项，只需运行:

```py
`$ poetry add [--dev] <package name>` 
```

> `--dev`标志表示该依赖关系仅用于开发模式。默认情况下，不会安装开发依赖项。

例如:

这将从 PyPI 下载并安装 Flask 到 poem 管理的虚拟环境中，将它和所有子依赖项一起添加到*poem . lock*文件中，并自动将它(一个顶级依赖项)添加到 *pyproject.toml* :

```py
`[tool.poetry.dependencies] python  =  "^3.10" Flask  =  "^2.0.3"` 
```

注意[版本约束](https://python-poetry.org/docs/basic-usage#version-constraints) : `"^2.0.3"`。

要在虚拟环境中运行一个命令，在命令前加上前缀[poem run](https://python-poetry.org/docs/cli/#run)。例如，使用`pytest`运行测试:

```py
`$ poetry run python -m pytest` 
```

`poetry run <command>`将在虚拟环境内运行命令。不过，它不会激活虚拟环境。要激活诗歌的虚拟环境，你需要运行`poetry shell`。要停用它，你可以简单地运行`exit`命令。因此，您可以在进行项目之前激活您的虚拟环境，并在完成后停用，或者您可以在整个开发过程中使用`poetry run <command>`。

最后，诗歌与 pyenv 配合得很好。查看官方文档中的[管理环境](https://python-poetry.org/docs/managing-environments/)，了解更多相关信息。

### Pipenv

Pipenv 试图解决与诗歌相同的问题:

1.  管理依赖关系和虚拟环境
2.  复制环境

一旦[安装了](https://github.com/pypa/pipenv#installation)，要用 Pipenv 创建一个新项目，运行:

```py
`$ mkdir sample-project
$ cd sample-project
$ pipenv --python 3.10.2` 
```

这将创建一个新的虚拟环境，并将一个 *Pipfile* 添加到项目中:

```py
`[[source]] name  =  "pypi" url  =  "https://pypi.org/simple" verify_ssl  =  true [dev-packages] [packages] [requires] python_version  =  "3.10"` 
```

> 一个 *Pipfile* 的工作方式很像诗歌世界中的 *pyproject.toml* 文件。

您可以像这样安装一个新的依赖项:

```py
`$ pipenv install [--dev] <package name>` 
```

> `--dev`标志表示该依赖关系仅用于开发模式。默认情况下，不会安装开发依赖项。

例如:

与诗歌一样，Pipenv 在虚拟环境中下载并安装 Flask，将所有子依赖项固定在 *Pipfile.lock* 文件中，并将顶级依赖项添加到 *Pipfile* 中。

要在 Pipenv 管理的虚拟环境中运行脚本，您需要使用 [pipenv run](https://github.com/pypa/pipenv#-usage) 命令来运行它。例如，要用`pytest`运行测试，运行:

```py
`$ pipenv run python -m pytest` 
```

像诗歌一样，`pipenv run <command>`将从虚拟环境内部运行命令。要激活 Pipenv 的虚拟环境，您需要运行`pipenv shell`。要停用它，你可以运行`exit`。

Pipenv 与 pyenv 配合也很好。例如，当您想从尚未安装的 Python 版本创建一个虚拟环境时，它会询问您是否想先用 pyenv 安装它:

```py
`$ pipenv --python 3.7.5

Warning: Python 3.7.5 was not found on your system…
Would you like us to install CPython 3.7.5 with Pyenv? [Y/n]: Y` 
```

### 推荐

我应该使用哪个？

1.  venv 和 pip
2.  诗意
3.  Pipenv

建议从 venv 和 pip 开始。他们是最容易共事的。熟悉他们，自己想清楚他们擅长什么，欠缺什么。

诗歌还是 Pipenv？

因为他们都解决相同的问题，这归结为个人偏好。

注意事项:

1.  用诗歌发布到 PyPI 要容易得多，所以如果你正在创建一个 Python 包，就用诗歌吧。
2.  这两个项目在依赖关系解析方面都相当慢，所以如果你正在使用 Docker，你可能想避开它们。
3.  从开源开发的角度来看，诗歌的速度更快，可以说对用户的反馈更敏感。

除了上述工具之外，还可以查看以下内容，以帮助您在同一台计算机上安装不同版本的 Python 并在不同版本之间进行切换，管理依赖关系和虚拟环境，以及再现环境:

1.  Docker 是一个构建、部署和管理容器化应用的平台。它非常适合创造可复制的环境。
2.  在数据科学和机器学习社区中非常受欢迎的 Conda ，可以帮助管理依赖性和虚拟环境，以及复制环境。
3.  当你只需要简化虚拟环境之间的切换并在一个地方管理它们时， [virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/) 和 [pyenv 插件 pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv) 值得一看。
4.  [pip-tools](https://github.com/jazzband/pip-tools) 简化了依赖性管理和环境再现性。它经常与 venv 结合。

|  | Python 版本 | 依赖性管理 | 虚拟环境 | 环境再现性 |
| --- | --- | --- | --- | --- |
| pyenv | ✅ | ❌ | ❌ | ❌ |
| venv + pip | ❌ | ✅ | ✅ | ❌ |
| venv+pip-工具 | ❌ | ✅ | ✅ | ✅ |
| 诗意 | ❌ | ✅ | ✅ | ✅ |
| Pipenv | ❌ | ✅ | ✅ | ✅ |
| 码头工人 | ❌ | ❌ | ❌ | ✅ |
| 康达 | ✅ | ✅ | ✅ | ❌ |

## 管理项目

让我们来看看如何使用 pyenv 和 poems 管理一个 Flask 项目。

首先，创建一个名为“flask_example”的新目录，并在其中移动:

```py
`$ mkdir flask_example
$ cd flask_example` 
```

其次，用 pyenv 为项目设置 Python 版本:

接下来，用诗歌初始化一个新的 Python 项目:

```py
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
```

添加烧瓶:

最后但同样重要的是，添加 [pytest](https://pytest.org/) 作为开发依赖项:

```py
`$ poetry add --dev pytest` 
```

现在我们已经建立了一个基本的环境，我们可以为单个端点编写一个测试。

添加一个名为 *test_app.py* 的文件:

```py
`import pytest

from app import app

@pytest.fixture
def client():
    app.config['TESTING'] = True

    with app.test_client() as client:
        yield client

def test_health_check(client):
    response = client.get('/health-check/')

    assert response.status_code == 200` 
```

之后，将一个基本的 Flask 应用程序添加到一个名为 *app.py* 的新文件中:

```py
`from flask import Flask

app = Flask(__name__)

@app.route('/health-check/')
def health_check():
    return 'OK'

if __name__ == '__main__':
    app.run()` 
```

现在，要运行测试，运行:

```py
`$ poetry run python -m pytest` 
```

您可以像这样运行开发服务器:

```py
`$ poetry run python -m flask run` 
```

> 命令在 poems 的虚拟环境中运行一个命令。

## 结论

本文研究了解决以下与依赖关系和工作空间管理相关的问题的最流行的工具:

1.  在同一台机器上安装和切换不同版本的 Python
2.  管理依赖关系和虚拟环境
3.  复制环境

您在工作流程中使用的具体工具并不重要，重要的是您能够解决这些问题。挑选一些工具，让你在 Python 中轻松开发。实验。它们的存在是为了让你的日常开发工作流程变得更简单，这样你就可以尽可能地高效工作。尝试所有这些方法，并使用适合您的开发风格的方法。不做评判。

快乐编码。

> [完整 Python](/guides/complete-python/) 指南:
> 
> 1.  [现代 Python 环境——依赖性和工作空间管理](/blog/python-environments/)(本文！)
> 2.  [Python 中的测试](/blog/testing-python/)
> 3.  [Python 中的现代测试驱动开发](/blog/modern-tdd/)
> 4.  [Python 代码质量](/blog/python-code-quality/)
> 5.  [Python 类型检查](/blog/python-type-checking/)
> 6.  [记录 Python 代码和项目](/blog/documenting-python/)
> 7.  [Python 项目工作流程](/blog/python-project-workflow/)