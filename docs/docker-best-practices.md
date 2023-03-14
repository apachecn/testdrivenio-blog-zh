# 面向 Python 开发者的 Docker 最佳实践

> 原文：<https://testdriven.io/blog/docker-best-practices/>

本文着眼于编写 Docker 文件和使用 Docker 时需要遵循的一些最佳实践。虽然列出的大多数实践适用于所有开发人员，不管是哪种语言，但是有一些只适用于开发基于 Python 的应用程序的人员。

--

**Dockerfiles** :

1.  [使用多阶段构建](/blog/docker-best-practices/#use-multi-stage-builds)
2.  [对 Dockerfile 命令进行适当排序](/blog/docker-best-practices/#order-dockerfile-commands-appropriately)
3.  [使用小型 Docker 基本图像](/blog/docker-best-practices/#use-small-docker-base-images)
4.  [最小化层数](/blog/docker-best-practices/#minimize-the-number-of-layers)
5.  [使用非特权容器](/blog/docker-best-practices/#use-unprivileged-containers)
6.  [更喜欢复制而不是添加](/blog/docker-best-practices/#prefer-copy-over-add)
7.  [将 Python 包缓存到 Docker 主机](/blog/docker-best-practices/#cache-python-packages-to-the-docker-host)
8.  [每个容器仅运行一个流程](/blog/docker-best-practices/#run-only-one-process-per-container)
9.  [优先使用数组而不是字符串语法](/blog/docker-best-practices/#prefer-array-over-string-syntax)
10.  [了解入口点和 CMD 的区别](/blog/docker-best-practices/#understand-the-difference-between-entrypoint-and-cmd)
11.  [包含健康检查指令](/blog/docker-best-practices/#include-a-healthcheck-instruction)

**图像**:

1.  [版本 Docker 图片](/blog/docker-best-practices/#version-docker-images)
2.  [不要在图像中存储秘密](/blog/docker-best-practices/#dont-store-secrets-in-images)
3.  [使用一个. dockerignore 文件](/blog/docker-best-practices/#use-a-dockerignore-file)
4.  [扫描您的 docker 文件和图像](/blog/docker-best-practices/#lint-and-scan-your-dockerfiles-and-images)
5.  [签署并验证图像](/blog/docker-best-practices/#sign-and-verify-images)

**奖励提示**

1.  [使用 Python 虚拟环境](/blog/docker-best-practices/#using-python-virtual-environments)
2.  [设置内存和 CPU 限制](/blog/docker-best-practices/#set-memory-and-cpu-limits)
3.  [记录到标准输出或标准错误](/blog/docker-best-practices/#log-to-stdout-or-stderr)
4.  [为 Gunicorn 心跳使用共享内存挂载](/blog/docker-best-practices/#use-a-shared-memory-mount-for-gunicorn-heartbeat)

## 码头文件

### 使用多阶段构建

*利用多阶段构建创建更精简、更安全的 Docker 映像。*

多阶段 Docker 构建允许你将你的 Docker 文件分成几个阶段。例如，您可以有一个用于编译和构建应用程序的阶段，然后可以将该阶段复制到后续阶段。因为只有最后一个阶段用于创建映像，所以与构建应用程序相关的依赖项和工具都被丢弃了，留下了一个精简的模块化生产就绪映像。

Web 开发示例:

```
`# temp stage
FROM  python:3.9-slim  as  builder

WORKDIR  /app

ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

RUN  apt-get update && \
    apt-get install -y --no-install-recommends gcc

COPY  requirements.txt .
RUN  pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# final stage
FROM  python:3.9-slim

WORKDIR  /app

COPY  --from=builder /app/wheels /wheels
COPY  --from=builder /app/requirements.txt .

RUN  pip install --no-cache /wheels/*` 
```

在这个例子中， [GCC](https://gcc.gnu.org/) 编译器是安装某些 Python 包所必需的，所以我们添加了一个临时的构建时阶段来处理构建阶段。因为最终的运行时映像不包含 GCC，所以它更轻便、更安全。

尺寸比较:

```
`REPOSITORY                 TAG                    IMAGE ID       CREATED          SIZE
docker-single              latest                 8d6b6a4d7fb6   16 seconds ago   259MB
docker-multi               latest                 813c2fa9b114   3 minutes ago    156MB` 
```

数据科学示例:

```
`# temp stage
FROM  python:3.9  as  builder

RUN  pip wheel --no-cache-dir --no-deps --wheel-dir /wheels jupyter pandas

# final stage
FROM  python:3.9-slim

WORKDIR  /notebooks

COPY  --from=builder /wheels /wheels
RUN  pip install --no-cache /wheels/*` 
```

尺寸比较:

```
`REPOSITORY                  TAG                   IMAGE ID       CREATED         SIZE
ds-multi                    latest                b4195deac742   2 minutes ago   357MB
ds-single                   latest                7c23c43aeda6   6 minutes ago   969MB` 
```

总之，多阶段构建可以减小生产映像的大小，帮助您节省时间和金钱。此外，这将简化您的生产容器。此外，由于更小的尺寸和简单性，潜在的攻击面也更小。

### 正确排序 Dockerfile 命令

密切注意 Dockerfile 命令的顺序，以利用层缓存。

Docker 将每个步骤(或层)缓存在一个特定的 docker 文件中，以加速后续的构建。当一个步骤发生变化时，不仅该特定步骤的缓存会失效，所有后续步骤的缓存也会失效。

示例:

```
`FROM  python:3.9-slim

WORKDIR  /app

COPY  sample.py .

COPY  requirements.txt .

RUN  pip install -r /requirements.txt` 
```

在这个 over 文件中，我们在安装需求之前复制了应用程序代码*。现在，每次我们改变 *sample.py* ，构建将重新安装软件包。这是非常低效的，尤其是当使用 Docker 容器作为开发环境时。因此，将频繁更改的文件放在 docker 文件的末尾是至关重要的。*

> 您还可以通过使用*来帮助防止不必要的缓存失效。dockerignore* 文件来排除不必要的文件被添加到 Docker 构建上下文和最终映像中。稍后会有更多相关内容。

因此，在上面的 docker 文件中，您应该将`COPY sample.py .`命令移到底部:

```
`FROM  python:3.9-slim

WORKDIR  /app

COPY  requirements.txt .

RUN  pip install -r /requirements.txt

COPY  sample.py .` 
```

注意事项:

1.  始终将可能发生变化的层放在 Dockerfile 文件中尽可能低的位置。
2.  组合`RUN apt-get update`和`RUN apt-get install`命令。(这也有助于减小图像尺寸。我们很快会谈到这一点。)
3.  如果您想关闭特定 Docker 版本的缓存，添加`--no-cache=True`标志。

### 使用小型 Docker 基本图像

较小的 Docker 映像更加模块化和安全。

对于较小的图像，构建、推和拉图像会更快。它们也更安全，因为它们只包含运行应用程序所需的必要库和系统依赖项。

您应该使用哪个 Docker 基础映像？

不幸的是，这要看情况。

下面是 Python 的各种 Docker 基本图像的大小比较:

```
`REPOSITORY   TAG                 IMAGE ID       CREATED      SIZE
python       3.9.6-alpine3.14    f773016f760e   3 days ago   45.1MB
python       3.9.6-slim          907fc13ca8e7   3 days ago   115MB
python       3.9.6-slim-buster   907fc13ca8e7   3 days ago   115MB
python       3.9.6               cba42c28d9b8   3 days ago   886MB
python       3.9.6-buster        cba42c28d9b8   3 days ago   886MB` 
```

虽然基于 [Alpine Linux](https://www.alpinelinux.org/) 的 Alpine 版本是最小的，但是如果你找不到可以使用它的编译过的二进制文件，它通常会导致编译时间的增加。结果，您可能不得不自己构建二进制文件，这会增加映像大小(取决于所需的系统级依赖项)和构建时间(由于必须从源代码编译)。

> 请参考[Python 应用的最佳 Docker 基础映像](https://pythonspeed.com/articles/base-image-python-docker-images/)和[使用 Alpine 会使 Python Docker 编译速度慢 50 倍](https://pythonspeed.com/articles/alpine-docker-python/)，了解为什么最好避免使用基于 Alpine 的基础映像。

归根结底，这都是为了平衡。当你有疑问时，从一个`*-slim`风格开始，尤其是在开发模式下，当你构建你的应用程序时。当您添加新的 Python 包时，您希望避免不断更新 Dockerfile 文件来安装必要的系统级依赖项。当您强化您的应用程序和 docker 文件以用于生产时，您可能希望探索使用 Alpine 作为多阶段构建的最终映像。

> 此外，不要忘记定期更新您的基本映像，以提高安全性和性能。当一个新版本的基础映像发布时——即`3.9.6-slim`->-`3.9.7-slim`——你应该拉新的映像并更新你的运行容器以获得所有最新的安全补丁。

### 尽量减少层数

这是一个好主意，尽量结合使用`RUN`、`COPY`和`ADD`命令，因为它们可以创建图层。每一层都增加了图像的大小，因为它们被缓存。因此，随着层数的增加，尺寸也增加。

您可以使用`docker history`命令对此进行测试:

```
`$ docker images
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
dockerfile   latest    180f98132d02   51 seconds ago   259MB

$ docker history 180f98132d02

IMAGE          CREATED              CREATED BY                                      SIZE      COMMENT
180f98132d02   58 seconds ago       COPY . . # buildkit                             6.71kB    buildkit.dockerfile.v0
<missing>      58 seconds ago       RUN /bin/sh -c pip install -r requirements.t…   35.5MB    buildkit.dockerfile.v0
<missing>      About a minute ago   COPY requirements.txt . # buildkit              58B       buildkit.dockerfile.v0
<missing>      About a minute ago   WORKDIR /app
...` 
```

注意尺寸。只有`RUN`、`COPY`和`ADD`命令可以增加图像的大小。您可以通过尽可能组合命令来减小图像大小。例如:

```
`RUN  apt-get update
RUN  apt-get install -y netcat` 
```

可以组合成一个单独的`RUN`命令:

```
`RUN  apt-get update && apt-get install -y netcat` 
```

因此，创建单个层而不是两个层，这减小了最终图像的大小。

虽然减少层数是个好主意，但更重要的是，减少层数本身不是目标，而是减少图像大小和构建时间的副作用。换句话说，更多地关注前面的三个实践——多阶段构建、Dockerfile 命令的顺序和使用小型基础映像——而不是试图优化每一个命令。

注意事项:

1.  `RUN`、`COPY`和`ADD`各自创建层。
2.  每一层都包含与前一层的不同之处。
3.  图层增加了最终图像的大小。

小贴士:

1.  组合相关命令。
2.  在创建它们的同一个运行`step`中删除不必要的文件。
3.  尽量减少运行`apt-get upgrade`的次数，因为它会将所有包升级到最新版本。
4.  对于多阶段构建，不要太担心过度优化临时阶段中的命令。

最后，为了提高可读性，最好按字母数字顺序对多行参数进行排序:

```
`RUN  apt-get update && apt-get install -y \
    git \
    gcc \
    matplotlib \
    pillow  \
    && rm -rf /var/lib/apt/lists/*` 
```

### 使用无特权的容器

默认情况下，Docker 在容器内部以 root 用户身份运行容器进程。但是，这是一种不好的做法，因为在容器中作为根用户运行的进程在 Docker 主机中也是作为根用户运行的。因此，如果攻击者获得了对您的容器的访问权限，他们就可以访问所有的根权限，并可以对 Docker 主机执行多种攻击，比如-

1.  将敏感信息从主机的文件系统复制到容器
2.  执行远程命令

为了防止这种情况，请确保使用非 root 用户运行容器进程:

```
`RUN  addgroup --system app && adduser --system --group app

USER  app` 
```

您可以更进一步，删除 shell 访问，并确保没有主目录:

```
`RUN  addgroup --gid 1001 --system app && \
    adduser --no-create-home --shell /bin/false --disabled-password --uid 1001 --system --group app

USER  app` 
```

验证:

```
`$ docker run -i sample id

uid=1001(app) gid=1001(app) groups=1001(app)` 
```

这里，容器中的应用程序在非 root 用户下运行。但是，请记住，Docker 守护进程和容器本身仍然以 root 权限运行。请务必查看[以非根用户身份运行 Docker 守护程序](https://docs.docker.com/engine/security/rootless/)，以获得以非根用户身份运行守护程序和容器的帮助。

### 首选复制而非添加

*使用`COPY`，除非你确定你需要`ADD`附带的额外功能。*

*`COPY`和`ADD`有什么区别？*

这两个命令都允许您将文件从特定位置复制到 Docker 映像中:

```
`ADD  <src> <dest>
COPY  <src> <dest>` 
```

虽然它们看起来服务于相同的目的，但`ADD`还有一些额外的功能:

*   `COPY`用于将本地文件或目录从 Docker 主机复制到镜像。
*   `ADD`可用于下载外部文件。还有，如果你用的是压缩文件(tar，gzip，bzip2 等。)作为`<src>`参数，`ADD`会自动将内容解压到给定的位置。

```
`# copy local files on the host to the destination
COPY  /source/path  /destination/path
ADD  /source/path  /destination/path

# download external file and copy to the destination
ADD  http://external.file/url  /destination/path

# copy and extract local compresses files
ADD  source.file.tar.gz /destination/path` 
```

### 将 Python 包缓存到 Docker 主机

当需求文件发生变化时，需要重新构建映像来安装新的包。前面的步骤将被缓存，正如在[中提到的最小化层数](/blog/docker-best-practices/#minimize-the-number-of-layers)。在重建映像时下载所有软件包会导致大量网络活动，并花费大量时间。每次重新构建花费相同的时间来跨构建下载公共包。

您可以通过将 pip 缓存目录映射到主机上的一个目录来避免这种情况。因此，对于每次重新构建，缓存的版本会持续存在，并可以提高构建速度。

将一个卷作为`-v $HOME/.cache/pip-docker/:/root/.cache/pip`添加到 docker 运行中，或者作为 Docker 编写文件中的一个映射。

> 以上目录仅供参考。确保映射缓存目录，而不是站点包(构建包所在的位置)。

将缓存从 docker 映像移动到主机可以节省最终映像中的空间。

如果您正在利用 [Docker BuildKit](https://docs.docker.com/develop/develop-images/build_enhancements/) ，使用 BuildKit 缓存挂载来管理缓存:

```
`# syntax = docker/dockerfile:1.2

...

COPY  requirements.txt .

RUN  --mount=type=cache,target=/root/.cache/pip \
        pip install -r requirements.txt

...` 
```

### 每个容器仅运行一个流程

*为什么建议每个容器只运行一个流程？*

让我们假设您的应用程序堆栈由两个 web 服务器和一个数据库组成。虽然您可以轻松地从一个容器中运行所有这三个服务，但是您应该在一个单独的容器中运行每个服务，以便于重用和扩展每个服务。

1.  **扩展** -每个服务都在一个单独的容器中，你可以根据需要水平扩展你的一个 web 服务器来处理更多的流量。
2.  **可重用性**——也许您有另一个需要容器化数据库的服务。您可以简单地重用同一个数据库容器，而不会带来两个不必要的服务。
3.  **日志**——耦合容器使得日志更加复杂。我们将在本文后面更详细地讨论这个问题。
4.  **可移植性和可预测性** -当可用的表面积减少时，制作安全补丁或调试问题就容易多了。

### 优先使用数组而不是字符串语法

您可以在 docker 文件中以数组(exec)或字符串(shell)格式编写`CMD`和`ENTRYPOINT`命令:

```
`# array (exec)
CMD  ["gunicorn",  "-w",  "4",  "-k",  "uvicorn.workers.UvicornWorker",  "main:app"]

# string (shell)
CMD  "gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app"` 
```

两者都是正确的，并实现几乎相同的事情；但是，您应该尽可能使用 exec 格式。来自 [Docker 文档](https://docs.docker.com/compose/faq/#why-do-my-services-take-10-seconds-to-recreate-or-stop):

1.  确保在 docker 文件中使用 exec 格式的`CMD`和`ENTRYPOINT`。
2.  例如，使用`["program", "arg1", "arg2"]`而不是`"program arg1 arg2"`。使用字符串形式会导致 Docker 使用 bash 运行您的进程，而 bash 不能正确处理信号。Compose 总是使用 JSON 格式，所以如果您覆盖了 Compose 文件中的命令或入口点，也不用担心。

因此，由于大多数 shell 不处理发送给子进程的信号，如果使用 shell 格式，`CTRL-C`(它生成一个`SIGTERM`)可能不会停止子进程。

示例:

```
`FROM  ubuntu:18.04

# BAD: shell format
ENTRYPOINT  top -d

# GOOD: exec format
ENTRYPOINT  ["top",  "-d"]` 
```

这两个都试试。注意，使用 shell 格式时，`CTRL-C`不会终止进程。相反，你会看到`^C^C^C^C^C^C^C^C^C^C^C`。

另一个警告是，shell 格式携带 shell 的 PID，而不是进程本身。

```
`# array format
[[email protected]](/cdn-cgi/l/email-protection):/app# ps ax
  PID TTY      STAT   TIME COMMAND
    1 ?        Ss     0:00 python manage.py runserver 0.0.0.0:8000
    7 ?        Sl     0:02 /usr/local/bin/python manage.py runserver 0.0.0.0:8000
   25 pts/0    Ss     0:00 bash
  356 pts/0    R+     0:00 ps ax

# string format
[[email protected]](/cdn-cgi/l/email-protection):/app# ps ax
  PID TTY      STAT   TIME COMMAND
    1 ?        Ss     0:00 /bin/sh -c python manage.py runserver 0.0.0.0:8000
    8 ?        S      0:00 python manage.py runserver 0.0.0.0:8000
    9 ?        Sl     0:01 /usr/local/bin/python manage.py runserver 0.0.0.0:8000
   13 pts/0    Ss     0:00 bash
  342 pts/0    R+     0:00 ps ax` 
```

### 理解入口点和 CMD 的区别

我应该使用 ENTRYPOINT 还是 CMD 来运行容器进程？

在容器中运行命令有两种方式:

```
`CMD  ["gunicorn",  "config.wsgi",  "-b",  "0.0.0.0:8000"]

# and

ENTRYPOINT  ["gunicorn",  "config.wsgi",  "-b",  "0.0.0.0:8000"]` 
```

两者本质上做同样的事情:用 Gunicorn 服务器在`config.wsgi`启动应用程序，并将其绑定到`0.0.0.0:8000`。

`CMD`很容易被覆盖。如果您运行`docker run <image_name> uvicorn config.asgi`，上面的 CMD 将被新的参数替换，例如`uvicorn config.asgi`。然而要覆盖`ENTRYPOINT`命令，必须指定`--entrypoint`选项:

```
`docker run --entrypoint uvicorn config.asgi <image_name>` 
```

在这里，很明显我们覆盖了入口点。因此，建议使用`ENTRYPOINT`而不是`CMD`来防止意外覆盖命令。

它们也可以一起使用。

例如:

```
`ENTRYPOINT  ["gunicorn",  "config.wsgi",  "-w"]
CMD  ["4"]` 
```

像这样一起使用时，运行来启动容器的命令是:

```
`gunicorn config.wsgi -w 4` 
```

如上所述，`CMD`很容易被覆盖。因此，`CMD`可以用来将参数传递给`ENTRYPOINT`命令。工人的数量可以很容易地这样改变:

```
`docker run <image_name> 6` 
```

这将启动六个 Gunicorn 工人的集装箱，而不是四个。

### 包括健康检查指令

*使用一个`HEALTHCHECK`来确定在容器中运行的进程是否不仅启动并运行，而且还“健康”。*

Docker 公开了一个 API，用于检查容器中运行的进程的状态，这提供了比进程是否“正在运行”更多的信息，因为“正在运行”包括“它已启动并正在工作”、“仍在启动”，甚至“陷入了某种无限循环错误状态”。您可以通过 [HEALTHCHECK](https://docs.docker.com/engine/reference/builder/#healthcheck) 指令与该 API 进行交互。

例如，如果您正在提供一个 web 应用程序，那么您可以使用下面的方法来确定`/`端点是否启动并可以处理服务请求:

```
`HEALTHCHECK  CMD  curl --fail http://localhost:8000 || exit 1` 
```

如果运行`docker ps`，可以看到`HEALTHCHECK`的状态。

健康的例子:

```
`CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                            PORTS                                       NAMES
09c2eb4970d4   healthcheck   "python manage.py ru…"   10 seconds ago   Up 8 seconds (health: starting)   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp   xenodochial_clarke` 
```

不健康的例子:

```
`CONTAINER ID   IMAGE         COMMAND                  CREATED              STATUS                          PORTS                                       NAMES
09c2eb4970d4   healthcheck   "python manage.py ru…"   About a minute ago   Up About a minute (unhealthy)   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp   xenodochial_clarke` 
```

您可以更进一步，设置一个仅用于健康检查的自定义端点，然后配置`HEALTHCHECK`来测试返回的数据。例如，如果端点返回一个 JSON 响应`{"ping": "pong"}`，您可以指示`HEALTHCHECK`验证响应体。

以下是使用`docker inspect`查看健康检查状态的方式:

```
`❯ docker inspect --format "{{json .State.Health }}" ab94f2ac7889
{
  "Status": "healthy",
  "FailingStreak": 0,
  "Log": [
    {
      "Start": "2021-09-28T15:22:57.5764644Z",
      "End": "2021-09-28T15:22:57.7825527Z",
      "ExitCode": 0,
      "Output": "..."` 
```

> 这里，输出被裁剪，因为它包含整个 HTML 输出。

您还可以将运行状况检查添加到 Docker 撰写文件中:

```
`version:  "3.8" services: web: build:  . ports: -  '8000:8000' healthcheck: test:  curl --fail http://localhost:8000 || exit 1 interval:  10s timeout:  10s start_period:  10s retries:  3` 
```

选项:

*   `test`:要测试的命令。
*   `interval`:测试的时间间隔，即每`x`个时间单位测试一次。
*   `timeout`:等待响应的最长时间。
*   `start_period`:何时开始健康检查。当在容器准备好之前执行附加任务时，如运行迁移，可以使用它。
*   `retries`:将测试指定为`failed`之前的最大重试次数。

> 如果您使用的是 Docker Swarm 之外的编排工具，例如 Kubernetes 或 AWS ECS，那么该工具很可能有自己的内部系统来处理健康检查。添加`HEALTHCHECK`指令前，请参考特定工具的文档。

## 形象

### 版本 Docker 图像

尽可能避免使用`latest`标签。

如果你依赖于`latest`标签(它不是真正的“标签”，因为当一个图像没有被显式标记时，它被默认应用)，你不能根据图像标签来判断你的代码运行的是哪个版本。这使得回滚变得很困难，并且很容易被覆盖(无论是意外的还是恶意的)。标签，就像你的基础设施和部署，应该是不可变的 T2。

无论您如何处理您的内部映像，您都不应该对基本映像使用`latest`标记，因为您可能会无意中部署一个对生产环境有重大更改的新版本。

对于内部映像，使用描述性标记可以更容易地判断代码运行的版本，处理回滚，并避免命名冲突。

例如，您可以使用以下描述符来组成标签:

1.  时间戳
2.  Docker 图像 id
3.  Git 提交哈希
4.  语义版本

> 要了解更多选项，请查看“正确版本化 Docker 图像”堆栈溢出问题的答案。

例如:

```
`docker build -t web-prod-a072c4e5d94b5a769225f621f08af3d4bf820a07-0.1.4 .` 
```

这里，我们使用以下内容来构成标记:

1.  项目名称:`web`
2.  环境名称:`prod`
3.  Git 提交哈希:`a072c4e5d94b5a769225f621f08af3d4bf820a07`
4.  语义版本:`0.1.4`

选择一个标记方案并与之保持一致是很重要的。由于提交散列使得将图像标签快速绑定到代码变得容易，所以强烈建议将它们包含在您的标签方案中。

### 不要在图像中存储秘密

秘密是敏感的信息片段，例如密码、数据库凭证、SSH 密钥、令牌和 TLS 证书等等。这些不应该在没有加密的情况下放进你的图像中，因为获得图像访问权的未授权用户只能检查图层来提取秘密。

不要以纯文本形式将秘密添加到 Docker 文件中，尤其是如果您将图像推送到像 [Docker Hub](https://hub.docker.com/) 这样的公共注册表中:

```
`FROM  python:3.9-slim

ENV  DATABASE_PASSWORD "SuperSecretSauce"` 
```

相反，应该通过以下途径注射:

1.  环境变量(运行时)
2.  构建时参数(在构建时)
3.  像 Docker Swarm(通过 Docker secrets)或 Kubernetes(通过 Kubernetes secrets)这样的编排工具

此外，您还可以通过将常用机密文件和文件夹添加到您的*中来帮助防止泄密。dockerignore* 文件:

最后，明确哪些文件将被复制到映像，而不是递归地复制所有文件:

```
`# BAD
COPY  . .

# GOOD
copy  ./app.py .` 
```

明确也有助于限制缓存破坏。

#### 环境变量

您可以通过环境变量传递秘密，但是它们将在所有子进程、链接容器和日志中可见，也可以通过`docker inspect`传递。更新它们也很困难。

```
`$ docker run --detach --env "DATABASE_PASSWORD=SuperSecretSauce" python:3.9-slim

d92cf5cf870eb0fdbf03c666e7fcf18f9664314b79ad58bc7618ea3445e39239

$ docker inspect --format='{{range .Config.Env}}{{println .}}{{end}}' d92cf5cf870eb0fdbf03c666e7fcf18f9664314b79ad58bc7618ea3445e39239

DATABASE_PASSWORD=SuperSecretSauce
PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
LANG=C.UTF-8
GPG_KEY=E3FF2839C048B25C084DEBE9B26995E310250568
PYTHON_VERSION=3.9.7
PYTHON_PIP_VERSION=21.2.4
PYTHON_SETUPTOOLS_VERSION=57.5.0
PYTHON_GET_PIP_URL=https://github.com/pypa/get-pip/raw/c20b0cfd643cd4a19246ccf204e2997af70f6b21/public/get-pip.py
PYTHON_GET_PIP_SHA256=fa6f3fb93cce234cd4e8dd2beb54a51ab9c247653b52855a48dd44e6b21ff28b` 
```

这是最直接的秘密管理方法。虽然它不是最安全的，但它会让诚实的人保持诚实，因为它提供了一层薄薄的保护，有助于保持秘密不被好奇的眼睛发现。

使用共享卷传递秘密是一个更好的解决方案，但它们应该通过 [Vault](https://www.vaultproject.io/) 或 [AWS 密钥管理服务](https://aws.amazon.com/kms/) (KMS)进行加密，因为它们保存在光盘上。

#### 构建时参数

你可以在构建时使用[构建时参数](https://docs.docker.com/engine/reference/commandline/build/#set-build-time-variables---build-arg)传递秘密，但是那些通过`docker history`访问映像的人可以看到它们。

示例:

```
`FROM  python:3.9-slim

ARG  DATABASE_PASSWORD` 
```

构建:

```
`$ docker build --build-arg "DATABASE_PASSWORD=SuperSecretSauce" .` 
```

如果您只需要在构建过程中临时使用密码(例如，用于克隆私有回购或下载私有包的 SSH 密钥)，则应该使用多阶段构建，因为构建器历史记录在临时阶段会被忽略:

```
`# temp stage
FROM  python:3.9-slim  as  builder

# secret
ARG  SSH_PRIVATE_KEY

# install git
RUN  apt-get update && \
    apt-get install -y --no-install-recommends git

# use ssh key to clone repo
RUN  mkdir -p /root/.ssh/ && \
    echo "${PRIVATE_SSH_KEY}" > /root/.ssh/id_rsa
RUN  touch /root/.ssh/known_hosts &&
    ssh-keyscan bitbucket.org >> /root/.ssh/known_hosts
RUN  git clone [[email protected]](/cdn-cgi/l/email-protection):testdrivenio/not-real.git

# final stage
FROM  python:3.9-slim

WORKDIR  /app

# copy the repository from the temp image
COPY  --from=builder /your-repo /app/your-repo

# use the repo for something!` 
```

多阶段构建仅保留最终图像的历史记录。请记住，您可以将此功能用于您的应用程序所需的永久机密，如数据库凭证。

您还可以使用 Docker build 中新的`--secret`选项将秘密传递给 Docker 映像，这些秘密不会存储在映像中。

```
`# "docker_is_awesome" > secrets.txt

FROM  alpine

# shows secret from default secret location:
RUN  --mount=type=secret,id=mysecret cat /run/secrets/mysecret` 
```

这将从`secrets.txt`文件中挂载秘密。

建立形象:

```
`docker build --no-cache --progress=plain --secret id=mysecret,src=secrets.txt .

# output
...
#4 [1/2] FROM docker.io/library/alpine
#4 sha256:665ba8b2cdc0cb0200e2a42a6b3c0f8f684089f4cd1b81494fbb9805879120f7
#4 CACHED

#5 [2/2] RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret
#5 sha256:75601a522ebe80ada66dedd9dd86772ca932d30d7e1b11bba94c04aa55c237de
#5 0.635 docker_is_awesome#5 DONE 0.7s

#6 exporting to image` 
```

最后，查看历史记录，看看秘密是否泄露:

```
`❯ docker history 49574a19241c
IMAGE          CREATED         CREATED BY                                      SIZE      COMMENT
49574a19241c   5 minutes ago   CMD ["/bin/sh"]                                 0B        buildkit.dockerfile.v0
<missing>      5 minutes ago   RUN /bin/sh -c cat /run/secrets/mysecret # b…   0B        buildkit.dockerfile.v0
<missing>      4 weeks ago     /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>      4 weeks ago     /bin/sh -c #(nop) ADD file:aad4290d27580cc1a…   5.6MB` 
```

> 有关构建时秘密的更多信息，请查看[不要泄露 Docker 映像的构建秘密](https://pythonspeed.com/articles/docker-build-secrets/)。

#### 码头工人的秘密

如果你正在使用 [Docker Swarm](https://docs.docker.com/engine/swarm/) ，你可以用 [Docker secrets](https://docs.docker.com/engine/reference/commandline/secret/) 来管理秘密。

例如，初始化 Docker 群模式:

创建 docker 机密:

```
`$ echo "supersecretpassword" | docker secret create postgres_password -
qdqmbpizeef0lfhyttxqfbty0

$ docker secret ls
ID                          NAME                DRIVER    CREATED         UPDATED
qdqmbpizeef0lfhyttxqfbty0   postgres_password             4 seconds ago   4 seconds ago` 
```

当一个容器被授予访问上述秘密的权限时，它将在`/run/secrets/postgres_password`挂载。这个文件将以明文形式包含秘密的实际值。

使用不同的业务流程工具？

1.  AWS EKS - [通过 Kubernetes 使用 AWS Secrets Manager secrets】](https://docs.aws.amazon.com/eks/latest/userguide/manage-secrets.html)
2.  数字海洋 Kubernetes - [保护数字海洋 Kubernetes 集群的推荐步骤](https://www.digitalocean.com/community/tutorials/recommended-steps-to-secure-a-digitalocean-kubernetes-cluster)
3.  Google Kubernetes 引擎- [与其他产品一起使用 Secret Manager】](https://cloud.google.com/secret-manager/docs/using-other-products#google-kubernetes-engine)
4.  Nomad - [保险库集成和检索动态机密](https://learn.hashicorp.com/tutorials/nomad/vault-postgres?in=nomad/integrate-vault)

### 使用. dockerignore 文件

我们已经提到使用一个 [*。dockerignore* 文件](https://docs.docker.com/engine/reference/builder/#dockerignore-file)已经好几次了。该文件用于指定您不希望添加到发送到 Docker 守护进程的初始构建上下文中的文件和文件夹，Docker 守护进程随后将构建您的映像。换句话说，您可以使用它来定义您需要的构建上下文。

构建 Docker 映像时，在评估`COPY`或`ADD`命令之前，整个 Docker 上下文——即项目的根——被发送到 Docker 守护进程*。这可能非常昂贵，尤其是如果您在项目中有许多依赖项、大型数据文件或构建工件的话。另外，Docker CLI 和守护程序可能不在同一台机器上。因此，如果守护进程在远程机器上执行，您应该更加注意构建上下文的大小。*

你应该给*添加什么？dockerignore* 文件？

1.  临时文件和文件夹
2.  构建日志
3.  地方机密
4.  本地开发文件，如 *docker-compose.yml*
5.  版本控制文件夹，如“.git“，”。hg”，和”。svn "

示例:

```
`**/.git
**/.gitignore
**/.vscode
**/coverage
**/.env
**/.aws
**/.ssh
Dockerfile
README.md
docker-compose.yml
**/.DS_Store
**/venv
**/env` 
```

总而言之，一个结构合理的*。dockerignore* 可以帮助:

1.  减小 Docker 图像的大小
2.  加快构建过程
3.  防止不必要的缓存失效
4.  防止泄密

### Lint 和扫描您的 docker 文件和图像

林挺是检查您的源代码的程序和风格错误以及可能导致潜在缺陷的不良实践的过程。就像编程语言一样，静态文件也可以被链接。特别是对于 docker 文件，linters 有助于确保它们的可维护性，避免不推荐使用的语法，并遵循最佳实践。林挺:你的形象应该成为你 CI 渠道的一个标准部分。

Hadolint 是最受欢迎的 Dockerfile linter:

```
`$ hadolint Dockerfile

Dockerfile:1 DL3006 warning: Always tag the version of an image explicitly
Dockerfile:7 DL3042 warning: Avoid the use of cache directory with pip. Use `pip install --no-cache-dir <package>`
Dockerfile:9 DL3059 info: Multiple consecutive `RUN` instructions. Consider consolidation.
Dockerfile:17 DL3025 warning: Use arguments JSON notation for CMD and ENTRYPOINT arguments` 
```

你可以在 https://hadolint.github.io/hadolint/的[网站上看到它的运行。还有一个](https://hadolint.github.io/hadolint/) [VS 代码扩展](https://marketplace.visualstudio.com/items?itemName=exiasr.hadolint)。

您可以将林挺与扫描图像和容器的漏洞相结合。

一些选项:

1.  [Snyk](https://docs.docker.com/engine/scan/) 是 Docker 本地漏洞扫描的独家提供商。您可以使用`docker scan` CLI 命令扫描图像。
2.  Trivy 可以用来扫描容器映像、文件系统、git 库和其他配置文件。
3.  Clair 是一个开源项目，用于静态分析应用程序容器中的漏洞。
4.  Anchore 是一个开源项目，为集装箱图像的检查、分析和认证提供集中服务。

总之，lint 和扫描您的 docker 文件和图像，找出任何偏离最佳实践的潜在问题。

### 签名并验证图像

*您如何知道用于运行生产代码的图像没有被篡改？*

篡改可以通过[中间人](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) (MITM)攻击通过网络进行，也可以来自被完全破坏的注册表。

[Docker 内容信任](https://docs.docker.com/engine/security/trust/) (DCT)支持来自远程注册中心的 Docker 图像的签名和验证。

要验证图像的完整性和真实性，请设置以下环境变量:

现在，如果您尝试提取未签名的图像，您将收到以下错误:

```
`Error: remote trust data does not exist for docker.io/namespace/unsigned-image:
notary.docker.io does not have trust data for docker.io/namespace/unsigned-image` 
```

您可以从[使用 Docker 内容信任签名图像](https://docs.docker.com/engine/security/trust/#signing-images-with-docker-content-trust)文档中了解签名图像。

从 Docker Hub 下载图像时，请确保使用[官方图像](https://docs.docker.com/docker-hub/official_images/)或来自可信来源的验证图像。较大的团队应该考虑使用他们自己的内部[私有容器注册中心](https://docs.docker.com/registry/deploying/)。

## 额外提示

### 使用 Python 虚拟环境

是否应该在容器中使用虚拟环境？

在大多数情况下，只要坚持每个容器只运行一个进程，虚拟环境就是不必要的。由于容器本身提供了隔离，包可以在系统范围内安装。也就是说，您可能希望在多阶段构建中使用虚拟环境，而不是构建 wheel 文件。

带轮子的例子:

```
`# temp stage
FROM  python:3.9-slim  as  builder

WORKDIR  /app

ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

RUN  apt-get update && \
    apt-get install -y --no-install-recommends gcc

COPY  requirements.txt .
RUN  pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# final stage
FROM  python:3.9-slim

WORKDIR  /app

COPY  --from=builder /app/wheels /wheels
COPY  --from=builder /app/requirements.txt .

RUN  pip install --no-cache /wheels/*` 
```

virtualenv 示例:

```
`# temp stage
FROM  python:3.9-slim  as  builder

WORKDIR  /app

ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

RUN  apt-get update && \
    apt-get install -y --no-install-recommends gcc

RUN  python -m venv /opt/venv
ENV  PATH="/opt/venv/bin:$PATH"

COPY  requirements.txt .
RUN  pip install -r requirements.txt

# final stage
FROM  python:3.9-slim

COPY  --from=builder /opt/venv /opt/venv

WORKDIR  /app

ENV  PATH="/opt/venv/bin:$PATH"` 
```

### 设置内存和 CPU 限制

限制 Docker 容器的内存使用是一个好主意，尤其是当您在一台机器上运行多个容器时。这可以防止任何容器使用所有可用的内存，从而降低其余容器的性能。

限制内存使用的最简单方法是在 Docker cli 中使用`--memory`和`--cpu`选项:

```
`$ docker run --cpus=2 -m 512m nginx` 
```

上面的命令将容器的使用限制为 2 个 CPU 和 512 兆的主内存。

您可以在 Docker 合成文件中做同样的事情，如下所示:

```
`version:  "3.9" services: redis: image:  redis:alpine deploy: resources: limits: cpus:  2 memory:  512M reservations: cpus:  1 memory:  256M` 
```

记下`reservations`字段。它用于设置一个软限制，当主机内存或 CPU 资源不足时，该限制优先。

其他资源:

1.  [内存、CPU 和 GPU 的运行时选项](https://docs.docker.com/config/containers/resource_constraints/)
2.  [Docker 编写资源约束](https://docs.docker.com/compose/compose-file/compose-file-v3/#resources)

### 记录到 stdout 或 stderr

在 Docker 容器中运行的应用程序应该将日志消息写入标准输出(stdout)和标准错误(stderr ),而不是文件。

然后，您可以配置 Docker 守护进程将您的日志消息发送到一个集中的日志记录解决方案(如 [CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) 或 [Papertrail](https://www.papertrail.com/) )。

更多信息，请查看来自[的](https://12factor.net/)[将日志视为事件流](https://12factor.net/logs)、来自[的十二因素应用](https://docs.docker.com/config/containers/logging/configure/)以及来自 Docker 文档的配置日志驱动程序。

### 为 Gunicorn Heartbeat 使用共享内存挂载

Gunicorn 使用基于文件的心跳系统来确保所有分叉的工作进程都是活动的。

在大多数情况下，心跳文件位于“/tmp”中，通常通过 [tmpfs](https://en.wikipedia.org/wiki/Tmpfs) 存储在内存中。因为 Docker 默认情况下不利用 tmpfs，所以文件将存储在磁盘支持的文件系统中。这会导致[问题](https://docs.gunicorn.org/en/20.1.0/faq.html#how-do-i-avoid-gunicorn-excessively-blocking-in-os-fchmod)，比如随机冻结，因为心跳系统使用`os.fchmod`，如果目录实际上在磁盘支持的文件系统上，这可能会阻塞一个工作进程。

幸运的是，有一个简单的修复方法:通过`--worker-tmp-dir`标志将 heartbeat 目录更改为内存映射目录。

```
`gunicorn --worker-tmp-dir /dev/shm config.wsgi -b 0.0.0.0:8000` 
```

## 结论

本文研究了几个最佳实践，使您的 docker 文件和图像更干净、更精简、更安全。

其他资源:

1.  [Docker 开发最佳实践](https://docs.docker.com/develop/dev-best-practices/)
2.  [编写 docker 文件的最佳实践](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

--

**Dockerfiles** :

1.  [使用多阶段构建](/blog/docker-best-practices/#use-multi-stage-builds)
2.  [对 Dockerfile 命令进行适当排序](/blog/docker-best-practices/#order-dockerfile-commands-appropriately)
3.  [使用小型 Docker 基本图像](/blog/docker-best-practices/#use-small-docker-base-images)
4.  [最小化层数](/blog/docker-best-practices/#minimize-the-number-of-layers)
5.  [使用非特权容器](/blog/docker-best-practices/#use-unprivileged-containers)
6.  [更喜欢复制而不是添加](/blog/docker-best-practices/#prefer-copy-over-add)
7.  [将 Python 包缓存到 Docker 主机](/blog/docker-best-practices/#cache-python-packages-to-the-docker-host)
8.  [每个容器仅运行一个流程](/blog/docker-best-practices/#run-only-one-process-per-container)
9.  [优先使用数组而不是字符串语法](/blog/docker-best-practices/#prefer-array-over-string-syntax)
10.  [了解入口点和 CMD 的区别](/blog/docker-best-practices/#understand-the-difference-between-entrypoint-and-cmd)
11.  [包含健康检查指令](/blog/docker-best-practices/#include-a-healthcheck-instruction)

**图像**:

1.  [版本 Docker 图片](/blog/docker-best-practices/#version-docker-images)
2.  [不要在图像中存储秘密](/blog/docker-best-practices/#dont-store-secrets-in-images)
3.  [使用一个. dockerignore 文件](/blog/docker-best-practices/#use-a-dockerignore-file)
4.  [扫描您的 docker 文件和图像](/blog/docker-best-practices/#lint-and-scan-your-dockerfiles-and-images)
5.  [签署并验证图像](/blog/docker-best-practices/#sign-and-verify-images)

**奖励提示**

1.  [使用 Python 虚拟环境](/blog/docker-best-practices/#using-python-virtual-environments)
2.  [设置内存和 CPU 限制](/blog/docker-best-practices/#set-memory-and-cpu-limits)
3.  [记录到标准输出或标准错误](/blog/docker-best-practices/#log-to-stdout-or-stderr)
4.  [为 Gunicorn 心跳使用共享内存挂载](/blog/docker-best-practices/#use-a-shared-memory-mount-for-gunicorn-heartbeat)