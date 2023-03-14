# 在 DigitalOcean Droplet 上将 Django 应用程序部署到 Dokku

> 原文：<https://testdriven.io/blog/django-dokku/>

在本教程中，我们将了解如何安全地将 [Django](https://www.djangoproject.com/) 应用程序部署到 [DigitalOcean droplet](https://www.digitalocean.com/products/droplets) 上的 [Dokku](https://dokku.com/) 。

## 目标

学完本教程后，您应该能够:

1.  解释什么是 Dokku 以及它是如何工作的
2.  创建一个数字海洋水滴，并在上面安装 Dokku
3.  通过 git 将 Django 应用程序部署到 Dokku
4.  创建一个由 Dokku 管理的 Postgres 数据库
5.  配置 Dokku 以保持存储
6.  配置 Nginx 来服务静态和媒体文件
7.  通过“让我们加密并在 HTTPS 上为您的应用提供服务”获得 TLS 证书

## 什么是 Dokku？

Dokku 是一个开源的平台即服务(PaaS ),允许你构建和管理应用从构建到扩展的生命周期。它基本上是一个迷你的 Heroku，你可以在你的 Linux 机器上自托管。因为它使用 Heroku 的[构建包](https://dokku.com/docs/deployment/builders/herokuish-buildpacks/)，它允许你部署任何可以在 Heroku 上部署的东西。Dokku 由 [Docker](https://www.docker.com/) 提供支持，并与 git 很好地集成。

Dokku 极其轻便。其唯一的[系统要求](https://dokku.com/docs/getting-started/installation/#system-requirements)是:

1.  Ubuntu 18.04/20.04/22.04 或 Debian 10+操作系统
2.  1 GB 系统内存(如果您没有，可以使用[这种变通方法](https://dokku.com/docs/getting-started/advanced-installation/#vms-with-less-than-1-gb-of-memory)

由于其系统要求较低，你可以在 DigitalOcean droplet 上托管它，每月只需 6 美元。

### 为什么是多库？

1.  完全免费和开源
2.  允许您轻松部署应用程序
3.  丰富的命令行界面
4.  支持多种[社区插件](https://dokku.com/docs/community/plugins/)
5.  轻量级选手

> Dokku 提供了一个名为 Dokku PRO 的高级计划，它进一步简化了事情，并带有一个用户友好的界面。你可以在他们的[官网](https://pro.dokku.com/)了解更多。

## 项目设置

在本教程中，我们将部署一个简单的图像托管应用程序，名为 [django-images](https://github.com/duplxey/django-images) 。

> 在学习教程的过程中，通过部署您自己的 Django 应用程序来检查您的理解。

首先，从 GitHub 上的[库](https://github.com/duplxey/django-images)中获取代码:

创建新的虚拟环境并激活它:

```py
`$ python3 -m venv venv && source venv/bin/activate` 
```

安装需求并迁移数据库:

```py
`(venv)$ pip install -r requirements.txt
(venv)$ python manage.py migrate` 
```

运行服务器:

```py
`(venv)$ python manage.py runserver` 
```

打开您最喜欢的网络浏览器，导航到 [http://localhost:8000](http://localhost:8000) 。使用右边的表格上传图像，确保一切正常。上传图像后，您应该会看到它显示在表格中:

![django-images Application Preview](img/da0e8d50358615ec0bbada6f51b06a08.png)

## 数字海洋

让我们在数字海洋上创建一个水滴。

首先，你需要[注册](https://m.do.co/c/d8f211a4b4c2)一个[数字海洋](https://www.digitalocean.com/)账户(如果你还没有的话)，然后[生成](https://www.digitalocean.com/docs/apis-clis/api/create-personal-access-token/)一个访问令牌，这样你就可以访问[数字海洋 API](https://developers.digitalocean.com/documentation/v2/) 。

将生成的令牌添加到您的环境:

```py
`$ export DIGITAL_OCEAN_ACCESS_TOKEN=<your_digital_ocean_token>` 
```

### 创建一个液滴

让我们创建一个 2 GB 的 droplet，因为我们的 Django 应用程序需要一些内存，而 Dokku 需要 1 GB 的系统内存才能工作。

创建一个水滴:

```py
`$ curl -X POST -H 'Content-Type: application/json' \
     -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' -d \
    '{"name":"django-dokku","region":"fra1","size":"s-1vcpu-2gb","image":"ubuntu-20-04-x64"}' \
    "https://api.digitalocean.com/v2/droplets"` 
```

> 该命令在`frankfurt-1`区域创建一个 2 GB 内存的 droplet。请随意选择更好的系统规格并更改地区。如果你不知道数字海洋的标识符，你可以使用[这个网站](https://slugs.do-api.dev/)。

DigitalOcean 旋转一滴大约需要 3-5 分钟。

检查其状态:

```py
`$ curl \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    "https://api.digitalocean.com/v2/droplets?name=django-dokku"` 
```

如果您安装了 [jq](https://stedolan.github.io/jq/) ，您可以像这样解析 JSON 响应:

```py
`$ curl \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    "https://api.digitalocean.com/v2/droplets?name=django-dokku" \
    | jq '.droplets[0].status'` 
```

一旦 droplet 可用，您将收到一封电子邮件，其中包含 droplet 的 IP 地址和密码。使用该信息 SSH 到服务器。

当你第一次 SSH 进入 droplet 时，出于安全原因，你将被迫更改密码。您可以选择一个非常强的密码，因为我们将在下一步启用无密码 SSH 登录。

要生成 SSH 密钥，请运行:

将密钥保存到 */root/。ssh/id_rsa* 并且不设置密码短语。这将分别生成一个公钥和私钥- *id_rsa* 和 *id_rsa.pub* 。要设置无密码 SSH 登录，请将公钥复制到 *authorized_keys* 文件中，并设置适当的权限:

```py
`$ cat ~/.ssh/id_rsa.pub
$ vi ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/id_rsa` 
```

复制私钥的内容:

退出 SSH 会话，然后将密钥设置为本地计算机上的环境变量:

```py
`export PRIVATE_KEY='-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA04up8hoqzS1+APIB0RhjXyObwHQnOzhAk5Bd7mhkSbPkyhP1
...
iWlX9HNavcydATJc1f0DpzF0u4zY8PY24RVoW8vk+bJANPp1o2IAkeajCaF3w9nf
q/SyqAWVmvwYuIhDiHDaV2A==
-----END RSA PRIVATE KEY-----'` 
```

将密钥添加到 ssh 代理中:

```py
`$ ssh-add - <<< "${PRIVATE_KEY}"` 
```

要进行测试，请运行:

## 安装 Dokku

> 警告:Dokku 引导脚本应该在新安装的 Linux 上运行。

要安装最新的稳定 Dokku 版本，首先 SSH 回到 droplet，然后导航到[安装最新的稳定版本](https://dokku.com/docs/getting-started/installation/#installing-the-latest-stable-version)并复制 bootstrap 命令。在撰写本文时，它看起来是这样的:

```py
`wget https://raw.githubusercontent.com/dokku/dokku/v0.28.1/bootstrap.sh
sudo DOKKU_TAG=v0.28.1 bash bootstrap.sh` 
```

这个脚本将安装 Docker、Dokku 和所有其他依赖项。安装将需要大约 5 到 10 分钟，所以在此期间您可以随意去喝杯咖啡。

安装完成后，检查 Dokku 版本:

```py
`$ dokku version

dokku version 0.28.1` 
```

Dokku 的部署是通过 git 处理的。每次将代码推送到远程 repo 时，您都需要通过输入 dokku 用户的密码或使用 SSH 密钥来验证自己。

为了避免基于密码的认证，将您的 SSH 密钥添加到 Dokku:

```py
`$ cat ~/.ssh/authorized_keys | dokku ssh-keys:add admin` 
```

> 如果你收到一条消息说`Duplicate ssh public key specified`，这意味着 Dokku 已经添加了你的 SSH 密钥。请随意忽略此警告。

## 创建 Dokku 应用程序

要创建 Dokku 应用程序，请运行:

```py
`$ dokku apps:create django-dokku

-----> Creating django-dokku...` 
```

检查应用程序是否已成功创建:

```py
`$ dokku apps:list

=====> My Apps
django-dokku` 
```

> 如果您在使用 Dokku 时遇到困难或忘记了某个命令，您可以在该命令后添加`:help`来查看该命令的功能及其属性。
> 
> 比如:`dokku apps:help`。

接下来，将您的 droplet 的 IP 地址链接到您的 Dokku 应用程序:

```py
`$ dokku domains:add django-dokku <your_droplet_ip>` 
```

## Dokku 数据库设置

为了让 Django 工作，我们还需要一个数据库。理论上我们可以使用默认的 [SQLite](https://www.sqlite.org/index.html) 数据库，但是让我们把它换成一个生产就绪的数据库: [Postgres](https://www.postgresql.org/) 。

默认情况下，Dokku 不会在新创建的应用程序上提供数据存储，例如 MySQL、Postgres。要用 Dokku 创建数据库服务，你首先需要安装一个社区插件——例如 [dokku-postgres](https://github.com/dokku/dokku-postgres) 、[Dokku-MySQL](https://github.com/dokku/dokku-mysql)——然后把它链接到应用程序。

从安装 dokku-postgres 开始:

```py
`$ sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git postgres` 
```

接下来创建一个名为`mydb`的 Postgres 服务，运行:

```py
`$ dokku postgres:create mydb

       Waiting for container to be ready
       Creating container database
       Securing connection to database
=====> Postgres container created: mydb
=====> mydb postgres service information
       Config dir:          /var/lib/dokku/services/postgres/mydb/data
       Config options:
       Data dir:            /var/lib/dokku/services/postgres/mydb/data
       Dsn:                 postgres://postgres:[[email protected]](/cdn-cgi/l/email-protection):5432/mydb
       Exposed ports:       -
       Id:                  7f8274383725b8dca9e64f0d7e3835ae4e6a16a3f5bc9bf015ff9bd32b6c5895
       Internal ip:         172.17.0.3
       Links:               -
       Service root:        /var/lib/dokku/services/postgres/mydb
       Status:              running
       Version:             postgres:14.4` 
```

Dokku 将推出一个安装了 Postgres 的新 Docker 容器。

检查码头集装箱:

```py
`$ docker ps

CONTAINER ID   IMAGE           COMMAND                  CREATED        STATUS        PORTS      NAMES
7f8274383725   postgres:14.4   "docker-entrypoint.s…"   1 hour ago     Up 1 hour     5432/tcp   dokku.postgres.mydb` 
```

最后，将 Postgres 服务链接到 Dokku 应用程序:

```py
`$ dokku postgres:link mydb django-dokku

-----> Setting config vars
       DATABASE_URL:  postgres://postgres:[[email protected]](/cdn-cgi/l/email-protection):5432/mydb
-----> Restarting app django-dokku
 !     App image (dokku/django-dokku:latest) not found` 
```

这将设置一个名为`DATABASE_URL`的新环境变量。这个变量是一个受[十二因素应用](https://www.12factor.net/backing-services)启发的 URL，稍后将允许我们连接到数据库。

```py
`postgres://postgres:88e242667bf9579a47c4cf5895524b8c@dokku-postgres-mydb:5432/mydb syntax: protocol://username:password@host:port/dbname` 
```

太好了！我们的数据库现在运行在 Docker 容器中，并链接到 Dokku 应用程序。

> 要了解更多关于 dokku-postgres 的信息，请参考资源库的[自述文件](https://github.com/dokku/dokku-postgres)。

## 配置 Django 项目

在教程的这一部分，我们将准备 Django 应用程序，以便与 Dokku 一起部署。

### 环境变量

我们不应该在源代码中存储秘密，所以让我们利用环境变量。最简单的方法是使用名为 [python-dotenv](https://saurabh-kumar.com/python-dotenv/) 的第三方 Python 包。首先将其添加到 *requirements.txt* :

> 随意使用不同的包来处理环境变量，如 [django-environ](https://github.com/joke2k/django-environ) 或 [python-decouple](https://github.com/henriquebastos/python-decouple/) 。

然后，在 *core/settings.py* 的顶部导入并初始化 python-dotenv，如下所示:

```py
`# core/settings.py

from pathlib import Path

from dotenv import load_dotenv  # new

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

load_dotenv(BASE_DIR / '.env')  # new` 
```

接下来，从环境中加载`SECRET_KEY`、`DEBUG`和`ALLOWED_HOSTS`:

```py
`# core/settings.py

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = os.getenv('SECRET_KEY')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = os.getenv('DEBUG', '0').lower() in ['true', 't', '1']

ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS').split(' ')` 
```

不要忘记重要的一点:

### 数据库ˌ资料库

要使用 Postgres 代替 SQLite，我们首先需要安装数据库适配器。

将下面一行添加到 *requirements.txt* 中:

还记得作为环境变量添加的`DATABASE_URL`吗？为了在 Django 中使用它，我们可以使用一个名为 [dj-database-url](https://pypi.org/project/dj-database-url/) 的包。这个包将 URL 转换成 Django 数据库参数。

像这样添加到 *requirements.txt* 中:

接下来，转到 *core/settings.py* ，把`DATABASES`改成这样:

```py
`# core/settings.py

DATABASES = {
    'default': dj_database_url.parse(os.environ.get('DATABASE_URL'), conn_max_age=600),
}` 
```

不要忘记在文件顶部导入`dj-database-url`:

### 格尼科恩

接下来，让我们安装 [Gunicorn](https://gunicorn.org/) ，这是一个生产级的 WSGI 服务器，将用于生产，而不是 Django 的开发服务器。

添加到 *requirements.txt* :

### 轮廓

像 Heroku 一样，Dokku 也使用一个 [Procfile](https://dokku.com/docs/deployment/builders/dockerfiles/#procfiles-and-multiple-processes) 来指定应用程序在启动时执行的命令。

在项目根目录中创建一个 *Procfile* 并填充它:

```py
`web:  gunicorn  core.wsgi:application release:  django-admin  migrate  --no-input  &&  django-admin  collectstatic  --no-input` 
```

所有使用 Heroku 构建包的语言和框架都声明了一个`web`进程类型，它启动应用服务器。发布步骤在发布阶段迁移数据库并收集静态文件。

### runtime.txt

要指定 Dokku 应该为您的应用程序使用的 Python 版本，请在项目根目录下创建一个名为 *runtime.txt* 的新文件。然后像这样填充它:

> 要查看所有受支持的 Python 运行时列表，请查看 Heroku Python 支持。

我们的应用程序现在可以部署了。让我们添加所有文件并将它们提交给 git:

```py
`$ git add -A
$ git commit -m "prepared the app for Dokku deployment"` 
```

## 部署应用程序

在部署应用程序之前，回到 droplet 上，我们需要将 Django 设置中使用的环境变量添加到 Dokku 中:

```py
`$ dokku config:set --no-restart django-dokku SECRET_KEY=h8710y7yaaaqopvbxtyxebdmtcewi_vuf1ah4gaxyjj4goij0u
$ dokku config:set --no-restart django-dokku DEBUG=1
$ dokku config:set --no-restart django-dokku ALLOWED_HOSTS=*` 
```

为了使调试简单一点，我们将暂时启用调试模式并将`ALLOWED_HOSTS`设置为`*`。这是一个糟糕的安全实践，因为它允许你的 Django 站点在任何主机/域上服务，使你容易受到 [HTTP 主机头攻击](https://docs.djangoproject.com/en/4.1/topics/security/#host-headers-virtual-hosting)。

别担心，我们稍后会更改这两个设置。

> 我们添加了`--no-restart`标志，以防止 Dokku 在添加每个新的环境变量后重新启动 django-dokku 应用程序。

此外，添加`DJANGO_SETTINGS_MODULE`，这是使用`django-admin`命令所必需的:

```py
`$ dokku config:set --no-restart django-dokku DJANGO_SETTINGS_MODULE=core.settings` 
```

检查环境变量，确保所有内容都已正确添加:

```py
`$ dokku config:show django-dokku

=====> django-dokku env vars
ALLOWED_HOSTS:           *
DATABASE_URL:            postgres://postgres:[[email protected]](/cdn-cgi/l/email-protection):5432/mydb
DEBUG:                   1
DJANGO_SETTINGS_MODULE:  core.settings
DOKKU_PROXY_PORT:        80
DOKKU_PROXY_PORT_MAP:    http:80:5000
SECRET_KEY:              h8710y7yaaaqopvbxtyxebdmtcewi_vuf1ah4gaxyjj4goij0u` 
```

就是这样。

现在，让我们回到本地开发环境。

向我们的 git 存储库添加一个新的 remote，并将代码推送到 droplet:

```py
`$ git remote add dokku [[email protected]](/cdn-cgi/l/email-protection)<your_droplet_ip_address>:django-dokku
$ git push dokku master` 
```

> 当您创建一个新的 Dokku 应用程序时，也应该创建一个 git 存储库。
> 
> 如果您得到一个“<app_name>似乎不是一个 git 存储库”的错误，SSH 进入您的 droplet 并运行:`dokku git:initialize django-dokku`。</app_name>

您应该会在终端中看到一个很长的输出。本质上，Dokku 是:

1.  清洁环境
2.  设置环境变量
3.  安装 *requirements.txt* 中的所有需求
4.  收集静态文件和迁移数据库
5.  创建和配置 Nginx 实例
6.  舒婷拆下旧容器，旋出一个新的

Dokku 完成部署后，您可以 SSH 到您的 droplet 并检查应用程序的状态:

最后，在浏览器中找到你的 droplet 的 IP 地址，测试应用程序，看看它是否工作。

```py
`http://<your_droplet_ip_address>/` 
```

不错！您的应用程序现已部署到 Dokku。每次你想更新你部署的应用程序时，提交你的更改并推送到`dokku`原点。

## Dokku 存储设置

Dokku 应用程序在容器中运行。因此，如果一个容器被破坏，你的文件和其他数据也会被破坏。为了在容器的生命周期之外保存数据，我们需要通过 Dokku 的[持久存储](https://dokku.com/docs/advanced-usage/persistent-storage/)插件来配置存储。

### 持久存储

为了持久存储，我们首先需要创建一个存储目录。多库推荐使用`/var/lib/dokku/data/storage/<app_name>`。

所以，回到 droplet，运行:

```py
`$ mkdir /var/lib/dokku/data/storage/django-dokku/
$ chown -R dokku:dokku /var/lib/dokku/data/storage/django-dokku/` 
```

接下来，让我们挂载目录。该命令有两个参数:

1.  应用程序名称
2.  一个`host-path:container-path`或`docker-volume:container-path`组合

```py
`$ dokku storage:mount django-dokku /var/lib/dokku/data/storage/django-dokku/staticfiles:/app/staticfiles
$ dokku storage:mount django-dokku /var/lib/dokku/data/storage/django-dokku/mediafiles:/app/mediafiles` 
```

重新部署 Dokku 应用程序以确保收集到文件:

```py
`$ dokku ps:restart django-dokku` 
```

列出目录:

```py
`$ ls /var/lib/dokku/data/storage/django-dokku

mediafiles  staticfiles` 
```

太好了，我们的静态和媒体文件已经收集好了。

目前我们有`DEBUG=1`并通过 Django 提供我们的静态和媒体文件:

```py
`# core/urls.py

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
    urlpatterns += static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)` 
```

让我们禁用调试模式，并学习如何通过 Nginx 服务它们。

```py
`$ dokku config:set django-dokku DEBUG=0` 
```

您的服务器将重新启动。如果您访问 web 应用程序，您将无法再看到任何图像。

要将 Nginx 配置为在生产中提供静态和媒体文件，请在 Dokku 应用程序文件中创建一个名为“nginx.conf.d”的新目录。然后在这个目录中创建一个名为 *static.conf* 的新文件。

```py
`$ mkdir -p /home/dokku/django-dokku/nginx.conf.d
$ vi /home/dokku/django-dokku/nginx.conf.d/static.conf` 
```

将以下内容放入 *static.conf* :

```py
`location  /static/  { alias  /var/lib/dokku/data/storage/django-dokku/staticfiles/; } location  /media/  { alias  /var/lib/dokku/data/storage/django-dokku/mediafiles/; }` 
```

目录结构:

```py
`└── nginx.conf.d
    └── static.conf` 
```

这创建了两条路径，一条用于存储静态文件，另一条用于媒体文件。

让 dokku 用户成为配置的所有者，并重新启动应用程序:

```py
`$ chown -R dokku:dokku /home/dokku/django-dokku/nginx.conf.d
$ dokku ps:restart django-dokku` 
```

应用程序重启后，在浏览器中访问它，检查媒体文件是否已恢复。

## Django 管理员访问权限

要访问 Django 管理面板，我们需要创建一个超级用户。我们有两个选择:

1.  使用`dokku run`执行命令。
2.  创建一个 Django 命令来创建一个超级用户，并将其添加到 *Procfile* 中。

由于这是一次性任务，我们将使用第一种方法:

```py
`$ dokku run django-dokku python manage.py createsuperuser` 
```

按照提示操作，就完成了。

要确保已成功创建超级用户，请导航到管理控制面板并登录。

```py
`http://<your_droplet_ip_address>/admin` 
```

## 添加域

教程的这一部分要求您有一个域名。

> 需要一个便宜的域名来练习？几个域名注册商有特殊优惠。“xyz”域。或者，您可以在 [Freenom](https://www.freenom.com/en/index.html?lang=en) 创建一个免费域名。

在 Dokku 中，有两种类型的域:

1.  *全局域*允许您使用通配符访问特定应用。例如，如果您的域是 testdriven.io，而您的应用程序名为 myapp，您将能够通过 myapp.testdriven.io 访问您的应用程序。
2.  *应用领域*直接指向特定应用。因此，如果您的域是 testdriven.io，那么您将能够在 testdriven.io 访问您的应用程序。

> 要了解更多关于 Dokku 域如何工作的信息，请参考[官方文档](https://dokku.com/docs/configuration/domains/)。

由于我们只有一个应用程序，我们将利用一个应用程序域。检查当前应用程序域设置:

```py
`$ dokku domains:report django-dokku

=====> django-dokku domains information
       Domains app enabled:           true
       Domains app vhosts:            165.227.135.236
       Domains global enabled:        true
       Domains global vhosts:         django-dokku` 
```

若要添加域，请前往您的域的注册商> DNS 设置，并创建一个指向您的 droplet 的 IP 地址的新“A 记录”,如下所示:

```py
`+----------+--------------+----------------------------+-----------+ | Type     | Host         | Value                      | TTL       |
+----------+--------------+----------------------------+-----------+ | A Record | <some host> | <your_droplet_ip_address> | Automatic |
+----------+--------------+----------------------------+-----------+` 
```

示例:

```py
`+----------+--------------+----------------------------+-----------+ | Type     | Host         | Value                      | TTL       |
+----------+--------------+----------------------------+-----------+ | A Record | django-dokku | 159.89.24.5                | Automatic |
+----------+--------------+----------------------------+-----------+` 
```

> 如果您不想使用子域，您可以遵循相同的步骤，但只需将 DNS 主机更改为`@`并相应地配置 Dokku。

最后，将域添加到 Dokku:

```py
`$  dokku  domains:set  django-dokku  django-dokku.testdriven.io ----->  Set  django-dokku.testdriven.io  for  django-dokku ----->  Configuring  django-dokku.testdriven.io...(using  built-in  template) ----->  Creating  http  nginx.conf Reloading  nginx` 
```

> 确保用您的域或子域替换 django-dokku.testdriven.io。

Dokku 将配置所有的东西(包括 Nginx 配置)来与域一起工作。

再次检查域报告，您将看到一个新的应用虚拟主机:

```py
`$ dokku domains:report django-dokku

=====> django-dokku domains information
       Domains app enabled:           true
       Domains app vhosts:            django-dokku.testdriven.io
       Domains global enabled:        true
       Domains global vhosts:         django-dokku` 
```

尝试通过域访问您的 web 应用程序，看看它是否工作。

### 让我们加密的 HTTPS

在这最后一部分，我们将使用[加密](https://letsencrypt.org/)来获得一个 TLS 证书，然后通过 HTTPS 服务 web 应用程序。

首先，安装另一个名为 [dokku-letsencrypt](https://github.com/dokku/dokku-letsencrypt) 的 Dokku 社区插件:

```py
`$ sudo dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git` 
```

在获得证书之前，您需要添加一个带有您的电子邮件地址的 env 变量:

```py
`$ dokku config:set --no-restart django-dokku DOKKU_LETSENCRYPT_EMAIL=<your_email>

-----> Setting config vars
       DOKKU_LETSENCRYPT_EMAIL:  youremail` 
```

此电子邮件用于签发证书并通知您证书到期。

接下来，运行插件命令来启用 HTTPS:

```py
`$ sudo dokku letsencrypt:enable django-dokku` 
```

这个命令将检索证书，安装它，自动配置 Nginx，创建从 HTTP 到 HTTPS 的重定向，并设置 [HSTS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security) 报头。

尝试通过 HTTP 访问您的网站，您应该会被重定向到 HTTPS。

![Dokku HTTPS served](img/22a2a575b22b97e249d7ca5ae620e5de.png)

要设置自动证书续订，请运行:

```py
`$ dokku letsencrypt:cron-job --add` 
```

这将每天运行检查，并更新所有应该更新的证书。

最后，配置 Django 的`ALLOWED_HOSTS`只允许通过 HTTPS 访问网站:

```py
`$ dokku config:set django-dokku "ALLOWED_HOSTS=localhost 127.0.0.1 [::1] <your_domain>"` 
```

确保用您的实际域名替换`<your_domain>`。

等待服务器重启——就这样！

## 结论

在本教程中，我们已经成功地在 DigitalOcean droplet 上将 Django 应用程序部署到 Dokku。现在，您应该对 Dokku 的工作原理有了相当的了解，并且能够部署您自己的 Django 应用程序了。

从 django-dokku-digitalocean 回购中获取最终代码。

### 下一步是什么？

1.  您应该通过[启用和配置防火墙](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-20-04)来使您的 droplet 更加安全。
2.  将数据库放在容器中是不好的做法。考虑改用数字海洋管理的数据库或类似的服务。
3.  对于媒体文件，考虑切换到 AWS S3，因为它更安全、更便宜。