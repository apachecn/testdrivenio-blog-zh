# 将 Django 应用程序部署到 Fly.io

> 原文：<https://testdriven.io/blog/django-fly/>

在本教程中，我们将看看如何部署一个 [Django](https://www.djangoproject.com/) 应用程序到 [Fly.io](https://fly.io) 。

## 目标

学完本教程后，您应该能够:

1.  解释什么是 Fly.io，它是如何工作的。
2.  将 Django 应用程序部署到 Fly.io。
3.  在 Fly.io 上运行一个 [PostgreSQL](https://www.postgresql.org/) 实例。
4.  通过 [Fly Volumes](https://fly.io/docs/reference/volumes/) 设置持久存储。
5.  将域名链接到您的 web 应用程序。
6.  用[获得一个 SSL 证书，让我们加密](https://letsencrypt.org/)并在 HTTPS 上提供您的应用程序。

## Fly.io 是什么？

Fly.io 是一个流行的平台即服务(PaaS)平台，为 web 应用程序提供托管服务。与许多其他 PaaS 托管提供商不同，他们不是转售 [AWS](https://aws.amazon.com/) 或 [GCP](https://cloud.google.com/) 服务，而是在运行[于世界各地](https://fly.io/docs/reference/regions/)的物理专用服务器上托管你的应用。正因为如此，他们能够提供比其他 PaaS 更便宜的主机服务，比如 Heroku。他们的主要关注点是尽可能靠近他们的客户部署应用程序(在撰写本文时，你可以在 24 个地区中挑选)。Fly.io 支持三种构建器:Dockerfile、 [Buildpacks](https://buildpacks.io/) ，或者预构建的 Docker 映像。

它们提供了强大的[缩放和自动缩放功能](https://fly.io/docs/reference/scaling/)。

与其他 PaaS 提供商相比，Fly.io 采用不同的方法来管理您的资源。它没有花哨的管理仪表板；相反，所有的工作都是通过他们名为 [flyctl](https://fly.io/docs/hands-on/install-flyctl/) 的 CLI 来完成的。

他们的免费计划包括:

*   多达 3 个共享 cpu-1x 256 MB 虚拟机
*   3GB 永久卷存储(总计)
*   160GB 出站数据传输

这应该足够运行一些小应用程序来测试他们的平台了。

### 为什么要 Fly.io？

*   小型项目的免费计划
*   巨大的[地区支持](https://fly.io/docs/reference/regions/)
*   出色的文档和完整的 API 文档
*   轻松实现水平和垂直缩放
*   相对便宜

## 项目设置

在本教程中，我们将部署一个简单的图像托管应用程序，名为 [django-images](https://github.com/duplxey/django-images) 。

> 在学习教程的过程中，通过部署您自己的 Django 应用程序来检查您的理解。

首先，从 GitHub 上的[库](https://github.com/duplxey/django-images)中获取代码:

创建新的虚拟环境并激活它:

```
`$ python3 -m venv venv && source venv/bin/activate` 
```

安装需求并迁移数据库:

```
`(venv)$ pip install -r requirements.txt
(venv)$ python manage.py migrate` 
```

运行服务器:

```
`(venv)$ python manage.py runserver` 
```

打开您最喜欢的网络浏览器，导航到 [http://localhost:8000](http://localhost:8000) 。使用右边的表格上传图像，确保一切正常。上传图像后，您应该会看到它显示在表格中:

![django-images Application Preview](img/841a1885ad9f230b72a4000cd391d99a.png)

## 安装 Flyctl

要使用 Fly 平台，你首先需要安装 [Flyctl](https://fly.io/docs/flyctl/) ，这是一个命令行界面，允许你做从创建帐户到将应用程序部署到 Fly 的所有事情。

要在 Linux 上安装它，请运行:

```
`$ curl -L https://fly.io/install.sh | sh` 
```

> 对于其他操作系统，请看一下[安装指南](https://fly.io/docs/hands-on/install-flyctl/)。

安装完成后，将`flyctl`添加到`PATH`:

```
`$ export FLYCTL_INSTALL="/home/$USER/.fly"
$ export PATH="$FLYCTL_INSTALL/bin:$PATH"` 
```

接下来，使用您的 Fly.io 帐户进行身份验证:

```
`$ fly auth login

# In case you don't have an account yet:
# fly auth signup` 
```

该命令将打开您的默认 web 浏览器，并要求您登录。登录后，单击“继续”登录 Fly CLI。

为确保一切正常，请尝试列出应用程序:

```
`$ fly apps list

NAME         OWNER           STATUS          PLATFORM        LATEST DEPLOY` 
```

您应该会看到一个空表，因为您还没有任何应用程序。

## 配置 Django 项目

在教程的这一部分，我们将准备并对接 Django 应用程序，以便部署到 Fly.io。

### 环境变量

我们不应该在源代码中存储秘密，所以让我们利用环境变量。最简单的方法是使用名为 [python-dotenv](https://saurabh-kumar.com/python-dotenv/) 的第三方 Python 包。首先将其添加到 *requirements.txt* :

> 随意使用不同的包来处理环境变量，如 [django-environ](https://github.com/joke2k/django-environ) 或 [python-decouple](https://github.com/henriquebastos/python-decouple/) 。

然后，在 *core/settings.py* 的顶部导入并初始化 python-dotenv，如下所示:

```
`# core/settings.py

from pathlib import Path

from dotenv import load_dotenv

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

load_dotenv(BASE_DIR / '.env')` 
```

接下来，从环境中加载`SECRET_KEY`、`DEBUG`、`ALLOWED_HOSTS`和`CSRF_TRUSTED_ORIGINS`:

```
`# core/settings.py

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = os.getenv('SECRET_KEY')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = os.getenv('DEBUG', '0').lower() in ['true', 't', '1']

ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS').split(' ')
CSRF_TRUSTED_ORIGINS = os.getenv('CSRF_TRUSTED_ORIGINS').split(' ')` 
```

不要忘记在文件顶部导入`os`:

### 数据库ˌ资料库

要使用 Postgres 代替 SQLite，我们首先需要安装数据库适配器。

将下面一行添加到 *requirements.txt* 中:

当我们在本教程的后面创建 Postgres 实例时，一个受[十二因素应用](https://12factor.net/)启发的名为`DATABASE_URL`的环境变量将被设置并以如下格式传递给我们的 web 应用:

```
`postgres://USER:PASSWORD@HOST:PORT/NAME` 
```

为了在 Django 中使用它，我们可以使用一个名为 [dj-database-url](https://pypi.org/project/dj-database-url/) 的包。这个包将 URL 转换成 Django 数据库参数。

像这样添加到 *requirements.txt* 中:

接下来，导航到 *core/settings.py* ，将`DATABASES`更改如下:

```
`# core/settings.py

DATABASES = {
    'default': dj_database_url.parse(os.environ.get('DATABASE_URL'), conn_max_age=600),
}` 
```

不要忘记重要的一点:

### 格尼科恩

接下来，让我们安装 [Gunicorn](https://gunicorn.org/) ，这是一个生产级的 WSGI 服务器，将用于生产，而不是 Django 的开发服务器。

添加到 *requirements.txt* :

### Dockerfile

如简介中所述，有三种方式将应用部署到 Fly.io:

1.  Dockerfile
2.  [构建包](https://buildpacks.io/)
3.  预建的 Docker 图像

在本教程中，我们将使用第一种方法，因为它是最灵活的，并且给了我们对 web 应用程序最大的控制权。它也很棒，因为它允许我们在未来轻松地切换到另一个托管服务(支持 Docker)。

首先，在项目根目录下创建一个名为 *Dockerfile* 的新文件，内容如下:

```
`# pull official base image
FROM  python:3.9.6-alpine

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

# create the app directory - and switch to it
RUN  mkdir -p /app
WORKDIR  /app

# install dependencies
COPY  requirements.txt /tmp/requirements.txt
RUN  set -ex && \
    pip install --upgrade pip && \
    pip install -r /tmp/requirements.txt && \
    rm -rf /root/.cache/

# copy project
COPY  . /app/

# expose port 8000
EXPOSE  8000

CMD  ["gunicorn",  "--bind",  ":8000",  "--workers",  "2",  "core.wsgi:application"]` 
```

> 如果您正在部署自己的 Django 应用程序，请确保相应地更改`CMD`。

接下来，创建一个*。dockerignore* :

```
`*.pyc *.pyo *.mo *.db *.css.map *.egg-info *.sql.gz .cache .project .idea .pydevproject .DS_Store .git/ .sass-cache .vagrant/ __pycache__ dist docs env logs Dockerfile` 
```

> 这是一个通用的*。Django 的 dockerignore* 模板。如果您想从图像中排除任何其他内容，请确保对其进行更改。

## 部署应用程序

在本节教程中，我们将启动一个 Postgres 实例，并将我们的 Django 应用程序部署到 Fly.io。

### 发动

要创建和配置新的应用程序，请运行:

```
`$ fly launch

Creating app in /dev/django-flyio
Scanning source code
Detected a Dockerfile app
? Choose an app name (leave blank to generate one): django-images
automatically selected personal organization: Nik Tomazic
? Choose a region for deployment: Frankfurt, Germany (fra)
Created app django-images in organization personal
Wrote config file fly.toml
? Would you like to set up a Postgresql database now? Yes
? Select configuration: Development - Single node, 1x shared CPU, 256MB RAM, 1GB disk
Creating postgres cluster in organization personal
Creating app...
Setting secrets on app django-images-db...
Provisioning 1 of 1 machines with image flyio/postgres:14.4
Waiting for machine to start...
Machine 21781350b24d89 is created
==> Monitoring health checks
  Waiting for 21781350b24d89 to become healthy (started, 3/3)
Postgres cluster django-images-db created
Postgres cluster django-images-db is now attached to django-images
? Would you like to set up an Upstash Redis database now? No
? Would you like to deploy now? No
Your app is ready! Deploy with `flyctl deploy`` 
```

注意事项:

1.  选择一个应用名称:**自定义名称或留空以生成一个随机名称**
2.  选择部署区域:**离你最近的区域**
3.  是否要现在设置 Postgresql 数据库:**是**
4.  数据库配置:**开发**
5.  您想现在设置一个 Upstash Redis 数据库吗:**否**
6.  是否要立即部署:**否**

该命令将在 Fly.io 上创建一个应用程序，启动一个 Postgres 实例，并在项目根目录中创建一个名为 *fly.toml* 的应用程序配置文件。

确保应用程序已成功创建:

```
`$ fly apps list

NAME                            OWNER           STATUS          PLATFORM        LATEST DEPLOY
django-images                   personal        pending
django-images-db                personal        deployed        machines
fly-builder-damp-wave-89        personal        deployed        machines` 
```

你会注意到三个应用程序。第一个是实际的 web 应用程序，然后是 Postgres 实例，最后是一个 Fly builder。Fly builders 用于构建您的 Docker 映像，将它们推送到容器注册表，并部署您的应用程序。

检查你的应用程序的状态:

```
`$ fly status

App
  Name     = django-images
  Owner    = personal
  Version  = 0
  Status   = pending
  Hostname = django-images.fly.dev
  Platform =

App has not been deployed yet.` 
```

主机名告诉您 web 应用程序可以访问哪个地址。记下它，因为我们需要在教程的后面将其添加到`ALLOWED_HOSTS`和`CSRF_TRUSTED_ORIGINS`中。

### 应用程序配置

让我们稍微修改一下[应用配置](https://fly.io/docs/reference/configuration/)文件，使其能够很好地与 Django 配合使用。

首先，将端口`8080`更改为 Django 的首选端口`8000`:

```
`# fly.toml app  =  "django-images" kill_signal  =  "SIGINT" kill_timeout  =  5 processes  =  [] [env] PORT  =  "8000"  # new [experimental] allowed_public_ports  =  [] auto_rollback  =  true [[services]] http_checks  =  [] internal_port  =  8000  # changed processes  =  ["app"] protocol  =  "tcp" script_checks  =  [] [services.concurrency] hard_limit  =  25 soft_limit  =  20 type  =  "connections"` 
```

接下来，为了确保数据库得到迁移，添加一个[部署](https://fly.io/docs/reference/configuration/#the-deploy-section)部分，并在新创建的部分中定义一个`release_command`:

```
`# fly.toml [deploy] release_command  =  "python manage.py migrate --noinput"` 
```

在这个版本部署之前,`release_command`运行在一个临时的 VM 中——使用成功构建的版本。这对于运行数据库迁移等一次性命令非常有用。

> 如果将来需要运行多个命令，可以在项目文件中创建一个 bash 脚本，然后像这样执行它:
> 
> ```
> `# fly.toml [deploy] release_command  =  "sh /path/to/your/script"` 
> ```

### 秘密

设置我们在 Django 的 *settings.py* 中使用的秘密:

```
`$ fly secrets set DEBUG="1"
$ fly secrets set ALLOWED_HOSTS="localhost 127.0.0.1 [::1] <your_app_hostname>"
$ fly secrets set CSRF_TRUSTED_ORIGINS="https://<your_app_hostname>"
$ fly secrets set SECRET_KEY="[[email protected]](/cdn-cgi/l/email-protection)"` 
```

确保将`<your_app_hostname>`替换为您实际的应用程序主机名。例如:

```
`$ fly secrets set ALLOWED_HOSTS="localhost 127.0.0.1 [::1] django-images.fly.dev"
$ fly secrets set CSRF_TRUSTED_ORIGINS="https://django-images.fly.dev"` 
```

为了使调试更容易，我们临时启用了调试模式。不要担心，因为我们将在教程的后面更改它。

确保密码设置成功:

```
`$ fly secrets list

NAME                    DIGEST                  CREATED AT
ALLOWED_HOSTS           06d92bcb15cf7eb1        30s ago
CSRF_TRUSTED_ORIGINS    06d92bcb15cf7eb1        21s ago
DATABASE_URL            e63c286f83782cf3        5m31s ago
DEBUG                   3baf154b33091aa0        45s ago
SECRET_KEY              62ac51c770a436f9        10s ago` 
```

> 部署 Fly 应用程序后，每个秘密修改都会触发重新部署。如果您需要一次设置多个密码，并且不希望您的应用程序多次重新部署，您可以在一个命令中连接这些密码，如下所示:
> 
> ```
> `$ fly secrets set NAME1="VALUE1" NAME2="VALUE2"` 
> ```

### 部署

要将应用程序部署到 Fly 平台，请运行:

```
`$ fly deploy

==> Verifying app config
--> Verified app config
==> Building image
Remote builder fly-builder-damp-wave-89 ready
==> Creating build context
--> Creating build context done
==> Building image with Docker
--> docker host: 20.10.12 linux x86_64
[+] Building 24.3s (11/11) FINISHED                                                             0.0s
--> Building image done
==> Pushing image to fly
The push refers to repository [registry.fly.io/django-images]
bee487f02b7f: Pushed
deployment-01GH4AGWQEZ607T7F93RB9G4NB: digest: sha256:85309cd5c7fe58f3a59b13d50576d8568525012bc6e665ba7b5cc1df3da16a9e size: 2619
--> Pushing image done
image: registry.fly.io/django-images:deployment-01GH4AGWQEZ607T7F93RB9G4NB
image size: 152 MB
==> Creating release
--> release v2 created
==> Monitoring deployment

 1 desired, 1 placed, 1 healthy, 0 unhealthy [health checks: 1 total, 1 passing]
--> v1 deployed successfully` 
```

该命令将使用 Fly builder 构建 Docker 映像，将其推送到容器注册表，并使用它来部署您的应用程序。在部署您的应用程序之前，`release_command`将在一个临时虚拟机上运行。

第一次部署你的应用大约需要五分钟，所以你可以在等待的时候喝杯咖啡。

部署应用程序后，检查其状态:

```
`$ fly status

App
  Name     = django-images
  Owner    = personal
  Version  = 0
  Status   = running
  Hostname = django-images.fly.dev
  Platform = nomad

Deployment Status
  ID          = 563c84b4-bf10-874c-e0e9-9820cbdd6725
  Version     = v1
  Status      = successful
  Description = Deployment completed successfully
  Instances   = 1 desired, 1 placed, 1 healthy, 0 unhealthy

Instances
ID              PROCESS VERSION REGION  DESIRED STATUS  HEALTH CHECKS           RESTARTS        CREATED
c009e8b0        app     1       fra     run     running 1 total, 1 passing      0               1m8s ago` 
```

检查日志:

```
`$ fly logs

[info]Starting init (commit: 81d5330)...
[info]Mounting /dev/vdc at /app/data w/ uid: 0, gid: 0 and chmod 0755
[info]Preparing to run: `gunicorn --bind :8000 --workers 2 core.wsgi:application` as root
[info]2022/11/05 17:09:36 listening on [fdaa:0:c65c:a7b:86:2:bd43:2]:22 (DNS: [fdaa::3]:53)
[info][2022-11-05 17:09:36 +0000] [529] [INFO] Starting gunicorn 20.1.0
[info][2022-11-05 17:09:36 +0000] [529] [INFO] Listening at: http://0.0.0.0:8000 (529)
[info][2022-11-05 17:09:36 +0000] [529] [INFO] Using worker: sync
[info][2022-11-05 17:09:36 +0000] [534] [INFO] Booting worker with pid: 534
[info][2022-11-05 17:09:36 +0000] [535] [INFO] Booting worker with pid: 535` 
```

一切看起来都很棒。让我们在浏览器中打开应用程序，确保它能够正常工作:

通过上传图像进行测试。

## 持久存储

Fly.io(以及许多其他类似的服务，如 Heroku)提供了一个短暂的文件系统。这意味着您的数据不是持久的，可能会在应用程序关闭或重新部署时消失。如果你的应用程序需要保留文件，这是非常糟糕的。

为了解决这个问题，Fly.io 为 Fly 应用程序提供了[卷](https://fly.io/docs/reference/volumes/)、持久存储。这听起来很棒，但它不是生产的最佳解决方案，因为卷被绑定到一个区域和一个服务器-这限制了您的应用程序的可伸缩性和跨不同区域的分布。

此外，Django 本身并不是为生产中的静态/媒体文件服务的。

我强烈建议你使用 AWS S3 或类似的服务，而不是 Volumes。如果您的应用程序不需要处理媒体文件，您仍然可以使用卷和[白化](http://whitenoise.evans.io/en/stable/)来处理静态文件。

> 要了解如何使用 Django 设置 AWS S3，请看一下在亚马逊 S3 上存储 Django 静态和媒体文件的[。](/blog/storing-django-static-and-media-files-on-amazon-s3/)

为了本教程的简单性和完整性，我们仍将使用飞卷。

首先，在与您的应用程序相同的区域创建一个宗卷:

```
`$ fly volumes create <volume_name> --region <region> --size <in_gigabytes>

# For example:
# fly volumes create django_images_data --region fra --size 1

        ID: vol_53q80vdd16xvgzy6
      Name: django_images_data
       App: django-images
    Region: fra
      Zone: d7f9
   Size GB: 1
 Encrypted: true
Created at: 04 Nov 22 13:20 UTC` 
```

在我的例子中，我在法兰克福地区创建了一个 1 GB 的卷。

接下来，进入 *core/settings.py* ，修改`STATIC_ROOT`和`MEDIA_ROOT`如下:

```
`# core/settings.py

STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'data/staticfiles'

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'data/mediafiles'` 
```

我们必须将静态和媒体文件放在一个子文件夹中，因为 Fly volume 只允许我们挂载一个目录。

要将目录挂载到 Fly 卷，请转到您的 *fly.toml* 并添加以下内容:

```
`# fly.toml [mounts] source="django_images_data" destination="/app/data"` 
```

1.  `source`是您的 Fly 卷的名称
2.  `destination`是您想要挂载的目录的绝对路径

重新部署你的应用程序:

最后，让我们收集静态文件。

SSH 进入 Fly 服务器，导航到“app”目录，运行`collectstatic`:

```
`$ fly ssh console
# cd /app
# python manage.py collectstatic --noinput` 
```

为了确保文件收集成功，请查看一下 */app/data* 文件夹:

```
`# ls /app/data

lost+found   staticfiles` 
```

太好了！您可以通过`exit`退出 SSH。

通过检查管理面板，确保已经成功收集了静态文件:

```
`http://<your_app_hostname>/admin` 
```

## Django 管理访问

要访问 Django 管理面板，我们需要创建一个超级用户。我们有两个选择:

1.  使用`fly ssh`执行命令。
2.  创建一个 Django 命令来创建一个超级用户，并将其添加到 *fly.toml* 中。

由于这是一次性任务，我们将使用第一种方法。

首先，使用 Fly CLI SSH 进入服务器:

```
`$ fly ssh console

Connecting to fdaa:0:e25c:a7b:8a:4:b5c5:2... complete` 
```

然后，导航到我们的应用程序文件所在的 */app* ，运行`createsuperuser`命令:

```
`# cd /app
# python manage.py createsuperuser` 
```

按照提示操作，然后运行`exit`关闭 SSH 连接。

要确保已成功创建超级用户，请导航至管理控制面板并登录:

```
`http://<your_app_hostname>/admin` 
```

或者使用:

然后导航至*/管理*。

## 添加域

教程的这一部分要求您有一个域名。

> 需要一个便宜的域名来练习？几个域名注册商有特殊优惠。“xyz”域。或者，您可以在 [Freenom](https://www.freenom.com/en/index.html?lang=en) 创建一个免费域名。

若要将域添加到您的应用程序，您首先需要获取一个证书:

```
`$ fly certs add <your_full_domain_name>

# For example:
# fly certs add fly.testdriven.io` 
```

接下来，进入你的域名注册服务商 DNS 设置，添加一个新的“CNAME 记录”指向你的应用程序的主机名，如下所示:

```
`+----------+--------------+----------------------------+-----------+ | Type     | Host         | Value                      | TTL       |
+----------+--------------+----------------------------+-----------+ | A Record | <some host> | <your_app_hostname> | Automatic |
+----------+--------------+----------------------------+-----------+` 
```

示例:

```
`+----------+--------------+----------------------------+-----------+ | Type     | Host         | Value                      | TTL       |
+----------+--------------+----------------------------+-----------+ | A Record | fly          | django-images.fly.dev      | Automatic |
+----------+--------------+----------------------------+-----------+` 
```

> 如果您不想使用子域，您可以遵循相同的步骤，但只需将 DNS 主机更改为`@`，并相应地配置 Fly.io 证书。

检查域是否已成功添加:

```
`$ fly certs list

Host Name                 Added                Status
fly.testdriven.io         5 minutes ago        Awaiting configuration` 
```

检查证书是否已颁发:

```
`$ fly certs check <your_full_domain_name>

# For example:
# fly certs check fly.testdriven.io

The certificate for fly.testdriven.io has not been issued yet.
Your certificate for fly.testdriven.io is being issued. Status is Awaiting certificates.` 
```

如果证书尚未颁发，请等待大约十分钟，然后重试:

```
`$ fly certs check fly.testdriven.io

The certificate for fly.testdriven.io has been issued.
Hostname                  = fly.testdriven.io
DNS Provider              = enom
Certificate Authority     = Let's Encrypt
Issued                    = rsa,ecdsa
Added to App              = 8 minutes ago
Source                    = fly` 
```

最后，将新的域添加到`ALLOWED_HOSTS`和`CSRF_TRUSTED_ORIGINS`:

```
`$ fly secrets set ALLOWED_HOSTS="localhost 127.0.0.1 [::1] <your_app_hostname> <your_full_domain_name>"
$ fly secrets set CSRF_TRUSTED_ORIGINS="https://<your_app_hostname> https://<your_full_domain_name>"

# For example:
# fly secrets set ALLOWED_HOSTS="localhost 127.0.0.1 [::1] django-images.fly.dev fly.testdriven.io"
# fly secrets set CSRF_TRUSTED_ORIGINS="https://django-images.fly.dev https://fly.testdriven.io"` 
```

当你修改你的密码时，你的应用将重新启动。重新启动后，您的 web 应用程序应该可以在新添加的域中访问。

## 结论

在本教程中，我们已经成功地将 Django 应用程序部署到 Fly.io。我们已经处理了 PostgreSQL 数据库、通过 [Fly Volumes](https://fly.io/docs/reference/volumes/) 的持久存储，并添加了域名。现在，您应该对 Fly 平台的工作原理有了一个大致的了解，并且能够部署自己的应用程序。

### 下一步是什么？

1.  与 [AWS S3](https://aws.amazon.com/s3/) 或类似的服务交换 [Fly Volumes](https://fly.io/docs/reference/volumes/) ，以更好、更安全的方式提供静态/媒体文件。
2.  设置`DEBUG=0`禁用调试模式。请记住，当启用调试模式时，应用程序仅提供静态/媒体文件。有关如何在生产中处理静态和媒体文件的更多信息，请参考[在 Django](/blog/django-static-files/) 中处理静态和媒体文件。
3.  看看[缩放和自动缩放](https://fly.io/docs/reference/scaling/)和[区域支持](https://fly.io/docs/reference/regions/)。