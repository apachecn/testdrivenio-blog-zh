# 使用 Bazel 进行可重复的构建

> 原文：<https://testdriven.io/blog/bazel-builds/>

如果您使用相同的源代码和相同的提交在两台不同的机器上运行两个构建，您希望得到相同的结果吗？

嗯，在大多数情况下你不会！

在本文中，我们将找出大多数构建过程中不确定性的来源，并看看如何使用 [Bazel](https://bazel.build/) 来创建可重复的、密封的构建。然后我们将创建一个可重复的 Flask 应用程序，它可以用 Bazel 构建，这样 Python 解释器和所有依赖项都是密封的。

## 可重现的构建

根据[可再生构建](https://reproducible-builds.org/docs/definition/)项目，“如果给定相同的源代码、构建环境和构建指令，任何一方都可以重新创建所有指定工件的逐位相同副本，那么构建就是可再生的”。这意味着为了实现可重现的构建，您必须移除所有不确定性的来源。虽然这可能很困难，但有几个好处:

1.  由于在大型构建图中缓存了中间构建工件，它可以大大加快构建时间。
2.  您可以可靠地确定工件的二进制起源，比如它是从什么来源构建的。
3.  可复制的代码[更加安全，并减少了攻击面](https://dl.acm.org/doi/10.1145/2664243.2664288)。

## 气密性

不确定性最常见的原因之一是对构建的输入。我指的是*除源代码之外的所有东西*:编译器、构建工具、第三方库以及任何其他可能影响构建的输入。为了使您的构建是密封的，所有的引用必须是明确的，无论是完全解析的版本号还是散列。密封信息应该作为源代码的一部分签入。

密封建筑可以采摘樱桃。假设您想要修复运行在生产环境中的旧版本中的一个 bug。如果你有一个封闭的构建过程，你可以检查旧版本，修复错误，然后重新构建代码。由于 hermeticity，所有的构建工具都在源代码库中进行了版本化，所以两个月前构建的项目不会使用今天版本的编译器，因为它可能与两个月前的源代码不兼容。

## 内部随机性

内部随机性是您在实现可重现构建之前必须解决的一个问题，这是一个很难解决的问题。

时间戳是内部随机性的一个常见来源。它们通常用于跟踪构建完成的时间。摆脱他们。对于可重现的构建，时间戳是不相关的，因为您已经在使用源代码控制跟踪您的构建环境。

对于不初始化值的语言，您需要显式地这样做，以避免由于从内存中捕获随机字节而导致构建中的随机性。

没有简单的方法可以绕过它——您必须检查您的代码！

> [GCC](https://gcc.gnu.org/) 在某些情况下使用随机数，所以你可能需要使用[-随机数-种子选项](https://gcc.gnu.org/onlinedocs/gcc/Developer-Options.html)来产生可重复的相同的目标文件。
> 
> 关于内部随机性原因的更多信息，请查看来自[可重复构建](https://reproducible-builds.org/docs/definition/)项目的[文档](https://reproducible-builds.org/docs/)部分。

## 使用 Bazel 进行可重复的构建

这一切听起来可能有点让人不知所措，但实际上并没有听起来那么复杂。Bazel 让这一过程变得更加简单。

我们现在来看一个使用 Bazel 编译和分发 Flask 应用程序的例子。

### 装置

Bazel 是创建可重复的密封构件的最佳解决方案之一。它支持许多语言，如 Python、Java、C、C++、Go 等等。从[安装](https://docs.bazel.build/versions/3.7.0/install.html) Bazel 开始。

为了构建我们的 Flask 应用程序，我们需要指导 Bazel 密封地使用 python 3.8.3。这意味着我们不能依赖安装在主机上的 Python 版本——我们必须从头编译它！

### 工作空间

在创建一个保存项目的文件夹后，开始设置一个[工作区](https://docs.bazel.build/glossary.html#workspace)，它保存项目的源代码和 Bazel 的构建输出。

创建一个名为*工作空间*的文件:

```
`workspace(name = "my_flask_app")

load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

_configure_python_based_on_os = """
if [[ "$OSTYPE" == "darwin"* ]]; then
 ./configure --prefix=$(pwd)/bazel_install --with-openssl=$(brew --prefix openssl)
else
 ./configure --prefix=$(pwd)/bazel_install
fi
"""

# Fetch Python and build it from scratch
http_archive(
    name = "python_interpreter",
    build_file_content = """
exports_files(["python_bin"])
filegroup(
 name = "files",
 srcs = glob(["bazel_install/**"], exclude = ["**/* *"]),
 visibility = ["//visibility:public"],
)
""",
    patch_cmds = [
        "mkdir $(pwd)/bazel_install",
        _configure_python_based_on_os,
        "make",
        "make install",
        "ln -s bazel_install/bin/python3 python_bin",
    ],
    sha256 = "dfab5ec723c218082fe3d5d7ae17ecbdebffa9a1aea4d64aa3a2ecdd2e795864",
    strip_prefix = "Python-3.8.3",
    urls = ["https://www.python.org/ftp/python/3.8.3/Python-3.8.3.tar.xz"],
)

# Fetch official Python rules for Bazel
http_archive(
    name = "rules_python",
    sha256 = "b6d46438523a3ec0f3cead544190ee13223a52f6a6765a29eae7b7cc24cc83a0",
    url = "https://github.com/bazelbuild/rules_python/releases/download/0.1.0/rules_python-0.1.0.tar.gz",
)

load("@rules_python//python:repositories.bzl", "py_repositories")

py_repositories()` 
```

我们首先使用 [http_archive](https://docs.bazel.build/versions/master/repo/http.html) 规则获取 Python 源代码档案，然后从头开始构建它。

这样，我们就可以确保对 Python 二进制文件和版本的控制。请记住:您不希望使用安装在主机上的 Python 版本，否则您的构建将无法重现。这里的密封性由`urls`字段和`sha256`字段来保证，前者告诉 Bazel 在哪里可以找到依赖关系，后者是它的唯一标识符。每个构建都将使用相同的明确的 Python 版本。

接下来，我们获取正式的 Python Bazel 规则。这里，`sha256`被用作标识符。稍后我们将使用这些规则来创建构建并测试目标。在此之前，我们必须定义我们的[工具链](https://docs.bazel.build/glossary.html#toolchain)。

接下来，我们将配置一个[构建文件](https://docs.bazel.build/versions/master/glossary.html#build-file)。

创建一个名为 *BUILD* 的文件:

```
`load("@rules_python//python:defs.bzl", "py_runtime", "py_runtime_pair")

py_runtime(
    name = "python3_runtime",
    files = ["@python_interpreter//:files"],
    interpreter = "@python_interpreter//:python_bin",
    python_version = "PY3",
    visibility = ["//visibility:public"],
)

py_runtime_pair(
    name = "py_runtime_pair",
    py2_runtime = None,
    py3_runtime = ":python3_runtime",
)

toolchain(
    name = "py_3_toolchain",
    toolchain = ":py_runtime_pair",
    toolchain_type = "@bazel_tools//tools/python:toolchain_type",
)` 
```

该配置将从我们在工作区中定义的 Python 解释器创建一个 Python 运行时，然后在工具链中使用。

最后，为了注册工具链，将下面一行添加到*工作空间*文件的末尾:

```
`# The Python toolchain must be registered ALWAYS at the end of the file
register_toolchains("//:py_3_toolchain")` 
```

不错！现在，您已经建立了一个密封的 Bazel 构建环境。不要只相信我的话，让我们写一个测试。

### 试验

为了用 Python 编写测试，我们需要 [pytest](https://pytest.org/) ，所以让我们添加一个 *requirements.txt* 文件:

```
`attrs==20.3.0 --hash=sha256:31b2eced602aa8423c2aea9c76a724617ed67cf9513173fd3a4f03e3a929c7e6
more-itertools==8.2.0 --hash=sha256:5dd8bcf33e5f9513ffa06d5ad33d78f31e1931ac9a18f33d37e77a180d393a7c
packaging==20.3 --hash=sha256:82f77b9bee21c1bafbf35a84905d604d5d1223801d639cf3ed140bd651c08752
pluggy==0.13.1 --hash=sha256:966c145cd83c96502c3c3868f50408687b38434af77734af1e9ca461a4081d2d
py==1.8.1 --hash=sha256:c20fdd83a5dbc0af9efd622bee9a5564e278f6380fffcacc43ba6f43db2813b0
pyparsing==2.0.2 --hash=sha256:17e43d6b17588ed5968735575b3983a952133ec4082596d214d7090b56d48a06
pytest==5.4.1 --hash=sha256:0e5b30f5cb04e887b91b1ee519fa3d89049595f428c1db76e73bd7f17b09b172
six==1.15.0 --hash=sha256:8b74bedcbbbaca38ff6d7491d76f2b06b3592611af620f8426e82dddb04a5ced
wcwidth==0.1.9 --hash=sha256:cafe2186b3c009a04067022ce1dcd79cb38d8d65ee4f4791b8888d6599d1bbe1` 
```

除了 pytest，我们还添加了所有的子依赖项。我们还添加了版本和 sha256 散列作为 hermeticity 的标识符。

现在，我们可以通过添加用于处理依赖关系的`pip_install`规则来再次修改工作区。在`register_toolchain`之前添加以下内容:

```
`# Third party libraries
load("@rules_python//python:pip.bzl", "pip_install")

pip_install(
    name = "py_deps",
    python_interpreter_target = "@python_interpreter//:python_bin",
    requirements = "//:requirements.txt",
)` 
```

您现在应该已经:

```
`workspace(name = "my_flask_app")

load("@bazel_tools//tools/build_defs/repo:git.bzl", "git_repository")
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

_configure_python_based_on_os = """
if [[ "$OSTYPE" == "darwin"* ]]; then
 ./configure --prefix=$(pwd)/bazel_install --with-openssl=$(brew --prefix openssl)
else
 ./configure --prefix=$(pwd)/bazel_install
fi
"""

# Fetch Python and build it from scratch
http_archive(
    name = "python_interpreter",
    build_file_content = """
exports_files(["python_bin"])
filegroup(
 name = "files",
 srcs = glob(["bazel_install/**"], exclude = ["**/* *"]),
 visibility = ["//visibility:public"],
)
""",
    patch_cmds = [
        "mkdir $(pwd)/bazel_install",
        _configure_python_based_on_os,
        "make",
        "make install",
        "ln -s bazel_install/bin/python3 python_bin",
    ],
    sha256 = "dfab5ec723c218082fe3d5d7ae17ecbdebffa9a1aea4d64aa3a2ecdd2e795864",
    strip_prefix = "Python-3.8.3",
    urls = ["https://www.python.org/ftp/python/3.8.3/Python-3.8.3.tar.xz"],
)

# Fetch official Python rules for Bazel
http_archive(
    name = "rules_python",
    sha256 = "b6d46438523a3ec0f3cead544190ee13223a52f6a6765a29eae7b7cc24cc83a0",
    url = "https://github.com/bazelbuild/rules_python/releases/download/0.1.0/rules_python-0.1.0.tar.gz",
)

load("@rules_python//python:repositories.bzl", "py_repositories")

py_repositories()

# Third party libraries
load("@rules_python//python:pip.bzl", "pip_install")

pip_install(
    name = "py_deps",
    python_interpreter_target = "@python_interpreter//:python_bin",
    requirements = "//:requirements.txt",
)

# The Python toolchain must be registered ALWAYS at the end of the file
register_toolchains("//:py_3_toolchain")` 
```

我们现在准备好编写测试了。

创建一个名为“test”的新文件夹，并添加一个名为 *compiler_version_test.py* 的新测试文件:

```
`import os
import platform
import sys

import pytest

class TestPythonVersion:
    def test_version(self):
        assert(os.path.abspath(os.path.join(os.getcwd(),"..", "python_interpreter", "python_bin")) in sys.executable)
        assert(platform.python_version() == "3.8.3")

if __name__ == "__main__":
    import pytest
    raise SystemExit(pytest.main([__file__]))` 
```

这将测试 Python 可执行文件是否存在以及版本是否正确。

为了将它包含在构建过程中，将一个*构建*文件添加到“test”文件夹中:

```
`load("@rules_python//python:defs.bzl", "py_test")
load("@py_deps//:requirements.bzl", "requirement")

py_test(
    name = "compiler_version_test",
    srcs = ["compiler_version_test.py"],
    deps = [
        requirement("attrs"),
        requirement("more-itertools"),
        requirement("packaging"),
        requirement("pluggy"),
        requirement("py"),
        requirement("pytest"),
        requirement("wcwidth"),
    ],
)` 
```

这里我们定义了一个名为`compiler_version_test`的 [py_test](https://docs.bazel.build/versions/master/be/python.html#py_test) 规则、源文件和所需的依赖项。一切都很清楚。

此时，您应该会看到这样的内容:

```
`├── BUILD
├── WORKSPACE
├── requirements.txt
└── test
    ├── BUILD
    └── compiler_version_test.py` 
```

这样，我们就可以运行我们的第一个“baz elized”Python 测试了！

从项目根目录运行:

```
`$ bazel test //test:compiler_version_test` 
```

输出:

```
`Starting local Bazel server and connecting to it...
INFO: Analyzed target //test:compiler_version_test (31 packages loaded, 8550 targets configured).
INFO: Found 1 test target...
Target //test:compiler_version_test up-to-date:
  bazel-bin/test/compiler_version_test
INFO: Elapsed time: 172.459s, Critical Path: 3.10s
INFO: 2 processes: 2 darwin-sandbox.
INFO: Build completed successfully, 2 total actions
//test:compiler_version_test                                             PASSED in 0.6s

Executed 1 out of 1 test: 1 test passes.
INFO: Build completed successfully, 2 total actions` 
```

至此，您已经有了一个密封配置的工作 Python 环境。

### 瓶

我们现在准备开发 Flask 应用程序。

创建一个“src”文件夹。然后，向其中添加一个名为 *flask_app.py* 的文件:

```
`import platform
import subprocess
import sys

from flask import Flask

def cmd(args):
    process = subprocess.Popen(args, stdout=subprocess.PIPE)
    out, _ = process.communicate()
    return out.decode('ascii').strip()

app = Flask(__name__)

@app.route('/')
def python_versions():
    bazel_python_path = f'Python executable used by Bazel is: {sys.executable} <br/><br/>'
    bazel_python_version = f'Python version used by Bazel is: {platform.python_version()} <br/><br/>'
    host_python_path = f'Python executable on the HOST machine is: {cmd(["which", "python3"])} <br/><br/>'
    host_python_version = f'Python version on the HOST machine is: {cmd(["python3", "-c", "import platform; print(platform.python_version())"])}'
    python_string = (
        bazel_python_path
        + bazel_python_version
        + host_python_path
        + host_python_version
    )
    return python_string

if __name__ == '__main__':
    app.run()` 
```

这是一个简单的 Flask 应用程序，将显示主机的二进制路径和 Python 版本以及 Bazel 使用的版本。

要构建它，我们需要向“src”添加一个*构建*文件:

```
`load("@rules_python//python:defs.bzl", "py_binary")
load("@py_deps//:requirements.bzl", "requirement")

py_binary(
    name = "flask_app",
    srcs = ["flask_app.py"],
    python_version = "PY3",
    deps = [
        requirement("flask"),
        requirement("jinja2"),
        requirement("werkzeug"),
        requirement("itsdangerous"),
        requirement("click"),
    ],
)` 
```

我们还需要用以下内容扩展 *requirements.txt* 文件:

```
`click==5.1 --hash=sha256:0c22a2cd5a1d741e993834df99133de07eff6cc1bf06f137da2c5f3bab9073a6
flask==1.1.2 --hash=sha256:8a4fdd8936eba2512e9c85df320a37e694c93945b33ef33c89946a340a238557
itsdangerous==0.24 --hash=sha256:cbb3fcf8d3e33df861709ecaf89d9e6629cff0a217bc2848f1b41cd30d360519
Jinja2==2.10.0 --hash=sha256:74c935a1b8bb9a3947c50a54766a969d4846290e1e788ea44c1392163723c3bd
MarkupSafe==0.23 --hash=sha256:a4ec1aff59b95a14b45eb2e23761a0179e98319da5a7eb76b56ea8cdc7b871c3
Werkzeug==0.15.5 --hash=sha256:87ae4e5b5366da2347eb3116c0e6c681a0e939a33b2805e2c0cbd282664932c4` 
```

完整文件:

```
`attrs==20.3.0 --hash=sha256:31b2eced602aa8423c2aea9c76a724617ed67cf9513173fd3a4f03e3a929c7e6
click==5.1 --hash=sha256:0c22a2cd5a1d741e993834df99133de07eff6cc1bf06f137da2c5f3bab9073a6
flask==1.1.2 --hash=sha256:8a4fdd8936eba2512e9c85df320a37e694c93945b33ef33c89946a340a238557
itsdangerous==0.24 --hash=sha256:cbb3fcf8d3e33df861709ecaf89d9e6629cff0a217bc2848f1b41cd30d360519
Jinja2==2.10.0 --hash=sha256:74c935a1b8bb9a3947c50a54766a969d4846290e1e788ea44c1392163723c3bd
MarkupSafe==0.23 --hash=sha256:a4ec1aff59b95a14b45eb2e23761a0179e98319da5a7eb76b56ea8cdc7b871c3
more-itertools==8.2.0 --hash=sha256:5dd8bcf33e5f9513ffa06d5ad33d78f31e1931ac9a18f33d37e77a180d393a7c
packaging==20.3 --hash=sha256:82f77b9bee21c1bafbf35a84905d604d5d1223801d639cf3ed140bd651c08752
pluggy==0.13.1 --hash=sha256:966c145cd83c96502c3c3868f50408687b38434af77734af1e9ca461a4081d2d
py==1.8.1 --hash=sha256:c20fdd83a5dbc0af9efd622bee9a5564e278f6380fffcacc43ba6f43db2813b0
pyparsing==2.0.2 --hash=sha256:17e43d6b17588ed5968735575b3983a952133ec4082596d214d7090b56d48a06
pytest==5.4.1 --hash=sha256:0e5b30f5cb04e887b91b1ee519fa3d89049595f428c1db76e73bd7f17b09b172
six==1.15.0 --hash=sha256:8b74bedcbbbaca38ff6d7491d76f2b06b3592611af620f8426e82dddb04a5ced
wcwidth==0.1.9 --hash=sha256:cafe2186b3c009a04067022ce1dcd79cb38d8d65ee4f4791b8888d6599d1bbe1
Werkzeug==0.15.5 --hash=sha256:87ae4e5b5366da2347eb3116c0e6c681a0e939a33b2805e2c0cbd282664932c4` 
```

然后，要运行该应用程序，请运行:

```
`$ bazel run //src:flask_app` 
```

您应该看到:

```
`INFO: Analyzed target //src:flask_app (10 packages loaded, 184 targets configured).
INFO: Found 1 target...
Target //src:flask_app up-to-date:
  bazel-bin/src/flask_app
INFO: Elapsed time: 7.430s, Critical Path: 1.12s
INFO: 4 processes: 4 internal.
INFO: Build completed successfully, 4 total actions
INFO: Build completed successfully, 4 total actions
 * Serving Flask app "flask_app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)` 
```

现在，应用程序正在本地主机上运行。打开浏览器并导航至 [http://127.0.0.1:5000/](http://127.0.0.1:5000/) 。您应该会看到类似如下的内容:

```
`Python  executable  used  by  Bazel  is:  /private/var/tmp/_bazel_michael/0c5c16dff39796b913e37a926dff4861/execroot/my_flask_app/bazel-out/darwin-fastbuild/bin/src/flask_app.runfiles/python_interpreter/python_bin Python  version  used  by  Bazel  is:  3.8.3 Python  executable  in  the  HOST  machine  is:  /Users/michael/.pyenv/versions/3.9.0/bin/python3 Python  version  in  the  HOST  machine  is:  3.9.0` 
```

正如我们所料，Bazel 使用的是我们从头编译的 Python 版本 3.8.3，而不是我主机上的 Python 3.9.0。

### 再现性测试

最后，我们确定构建是可重复的吗？

要进行测试，请运行两次构建，并通过比较 md5 哈希来检查输出二进制文件是否有任何差异:

```
`$ md5sum $(bazel info bazel-bin)/src/flask_app

2075a7ec4e8eb7ced16f0d9b3d8c5619  /private/var/tmp/_bazel_michael/0c5c16dff39796b913e37a926dff4861/execroot/my_flask_app/bazel-out/darwin-fastbuild/bin/src/flask_app

$ bazel clean --expunge_async
# or 'bazel clean --expunge' on non-linux platforms

INFO: Starting clean.

$ bazel build //...

Starting local Bazel server and connecting to it...
INFO: Analyzed 4 targets (38 packages loaded, 8729 targets configured).
INFO: Found 4 targets...
INFO: Elapsed time: 183.923s, Critical Path: 1.65s
INFO: 7 processes: 7 internal.
INFO: Build completed successfully, 7 total actions

$ md5sum $(bazel info bazel-bin)/src/flask_app

2075a7ec4e8eb7ced16f0d9b3d8c5619  /private/var/tmp/_bazel_michael/0c5c16dff39796b913e37a926dff4861/execroot/my_flask_app/bazel-out/darwin-fastbuild/bin/src/flask_app` 
```

这里，我们计算了刚刚构建的二进制文件的散列，清理了所有构建工件和依赖项，并再次运行构建。新的二进制文件和旧的一模一样！

--

所以你的体型是密封的，对吗？

好吧，其实并不是完全可复制的，我们来看看为什么。

跳回*工作区*文件。在这个文件中，我们试图在 Bazel 内部构建 Python，以实现完全的可再现性。然而，使用`http_archive`的`patch_cmds`意味着 Python 是使用运行构建的主机的编译器构建的。Python 解释器被固定在一个精确的版本上，它将依赖于机器的 GCC 和系统库，而这些库并没有以任何方式被固定或控制。换句话说，构建是不完全可复制的*。*

不过，这是有解决办法的！

示例:

1.  您可以使用固定的 GCC 版本从 Docker 容器运行`bazel build`,然后在您的项目中签入 Docker 信息。这是 CI 系统中的一种常见方法。
2.  您可以使用预编译的二进制可执行文件，将其签入源代码控制，并将其固定在构建版本上，而不是从头开始编译 Python。
3.  您可以使用类似于 [Nix](https://github.com/tweag/rules_nixpkgs) 的工具，它允许将外部依赖项(如系统库)密封地导入 Bazel。

## 结论

总结最大的收获:

1.  不要想当然地认为你的构建是可重复的
2.  密封性使樱桃采摘成为可能
3.  构建的输入必须用源代码进行版本控制
4.  内部随机性可能是偷偷摸摸的，但必须消除
5.  多亏了 Bazel，您有了一个密封的 Python 工作环境
6.  您已经看到了如何以可重现的方式编译 Flask 应用程序

既然您已经熟悉了可复制的、密封的构建的含义，那么您的使您的构建可复制的旅程就开始了。

测试你目前正在做的项目的二进制文件的 md5，让我知道结果。