# 将 Django 应用程序部署到 Elastic Beanstalk

> 原文：<https://testdriven.io/blog/django-elastic-beanstalk/>

在本教程中，我们将逐步完成将生产就绪的 [Django](https://www.djangoproject.com/) 应用程序部署到 [AWS Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/) 的过程。

## 目标

本教程结束时，您将能够:

1.  解释什么是弹性豆茎
2.  初始化和配置弹性豆茎
3.  对运行在 Elastic Beanstalk 上的应用程序进行故障排除
4.  将弹性豆茎与 RDS 结合
5.  为静态和媒体文件支持配置 S3
6.  通过 AWS 证书管理器获取 SSL 证书
7.  使用 SSL 证书在 HTTPS 上提供您的应用程序

## 什么是弹性豆茎？

AWS Elastic Beanstalk (EB)是一个易于使用的服务，用于部署和扩展 web 应用程序。它连接多个 AWS 服务，例如计算实例( [EC2](https://aws.amazon.com/ec2/) )、数据库( [RDS](https://aws.amazon.com/rds/) )、负载平衡器([应用负载平衡器](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html))和文件存储系统( [S3](https://aws.amazon.com/s3/) )，等等。EB 允许您快速开发和部署 web 应用程序，而无需考虑底层基础设施。它[支持用 Go、Java、.NET、Node.js、PHP、Python 和 Ruby。如果您需要配置自己的软件栈或部署用 EB 目前不支持的语言(或版本)开发的应用程序，EB 也支持 Docker。](https://docs.aws.amazon.com/elasticbeanstalk/latest/platforms/platforms-supported.html)

典型的弹性豆茎设置:

![Elastic Beanstalk Architecture](img/87fe95004fb6ed7e3b5e308378b9c763.png)

AWS 弹性豆茎不另收费。您只需为应用程序消耗的资源付费。

> 要了解更多关于弹性豆茎的信息，请查看[什么是 AWS 弹性豆茎？](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/Welcome.html)来自[官方 AWS 弹性豆茎文档](https://docs.aws.amazon.com/elastic-beanstalk/index.html)。

### 弹性豆茎概念

在开始学习教程之前，让我们先来看看与 Elastic Beanstalk 相关的几个关键概念:

1.  一个 **[应用](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/concepts.html#concepts-application)** 是弹性 Beanstalk 组件的逻辑集合，包括环境、版本和环境配置。一个应用程序可以有多个[版本](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/concepts.html#concepts-version)。
2.  一个 **[环境](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/concepts.html#concepts-environment)** 是运行一个应用版本的 AWS 资源的集合。
3.  一个 **[平台](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/concepts.html#concepts-platform)** 是操作系统、编程语言运行时、web 服务器、应用服务器和弹性 Beanstalk 组件的组合。

这些术语将在整个教程中使用。

## 项目设置

在本教程中，我们将部署一个名为 [django-images](https://github.com/duplxey/django-images) 的简单图像托管应用程序。

> 按照教程进行操作时，通过部署您自己的应用程序来检查您的理解。

首先，从 GitHub 上的[库获取代码:](https://github.com/duplxey/django-images)

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

![django-images Application Preview](img/c6b8cc865e47f49d18337ed2ad575e3b.png)

## 弹性豆茎 CLI

> 在继续之前，请务必在[注册一个 AWS 帐户](https://portal.aws.amazon.com/billing/signup#/start)。通过创建一个账户，你可能也有资格加入 [AWS 免费等级](https://aws.amazon.com/free/)。

[Elastic Beanstalk 命令行界面](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3.html) (EB CLI)允许您执行各种操作来部署和管理您的 Elastic Beanstalk 应用程序和环境。

有两种安装 EB CLI 的方法:

1.  通过 [EB CLI 安装程序](https://github.com/aws/aws-elastic-beanstalk-cli-setup#2-quick-start)
2.  与 [pip (awsebcli)](https://pypi.org/project/awsebcli/)

> 建议使用安装程序(第一个选项)全局安装 EB CLI(任何特定虚拟环境之外),以避免可能的依赖冲突。更多详情请参考[本解释](https://github.com/aws/aws-elastic-beanstalk-cli-setup#51-for-the-experienced-python-developer-whats-the-advantage-of-this-mode-of-installation-instead-of-regular-pip-inside-a-virtualenv)。

安装 EB CLI 后，您可以通过运行以下命令来检查版本:

```
`$ eb --version

EB CLI 3.20.3 (Python 3.10.)` 
```

如果该命令不起作用，您可能需要将 EB CLI 添加到`$PATH`中。

> EB CLI 命令列表及其描述可在 [EB CLI 命令参考](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb3-cmd-commands.html)中找到。

## 初始化弹性豆茎

一旦我们运行了 EB CLI，我们就可以开始与 Elastic Beanstalk 交互了。让我们初始化一个新项目和一个 EB 环境。

### 初始化

在项目根目录(“django-images”)中，运行:

你会被提示一些问题。

#### 默认区域

您的弹性 Beanstalk 环境的 AWS 区域(和资源)。如果您不熟悉不同的 AWS 区域，请查看 [AWS 区域和可用区域](https://aws.amazon.com/about-aws/global-infrastructure/regions_az/)。一般来说，你应该选择离你的客户最近的地区。请记住，资源价格因地区而异。

#### 应用程序名称

这是您的弹性 Beanstalk 应用程序的名称。我建议按下回车键，使用默认设置:“django-images”。

#### 平台和平台分支

EB CLI 将检测到您正在使用 Python 环境。之后，它会给你不同的 Python 版本和 Amazon Linux 版本供你使用。选择“运行在 64 位亚马逊 Linux 2 上的 Python 3.8”。

#### 代码提交

CodeCommit 是一个安全的、高度可伸缩的、托管的源代码控制服务，托管私有的 Git 存储库。我们不会使用它，因为我们已经在使用 GitHub 进行源代码控制。所以说“不”。

#### 嘘

为了稍后连接到 EC2 实例，我们需要设置 SSH。出现提示时，说“是”。

#### 密钥对

为了连接到 EC2 实例，我们需要一个 RSA 密钥对。继续生成一个，它将被添加到您的“~/”中。ssh”文件夹。

回答完所有问题后，您会注意到项目根目录下有一个隐藏的目录，名为。elasticbeanstalk”。该目录应该包含一个 *config.yml* 文件，其中包含您刚才提供的所有数据。

```
`.elasticbeanstalk
└── config.yml` 
```

该文件应包含类似以下内容:

```
`branch-defaults: master: environment:  null group_suffix:  null global: application_name:  django-images branch:  null default_ec2_keyname:  aws-eb default_platform:  Python 3.8 running on 64bit Amazon Linux 2 default_region:  us-west-2 include_git_submodules:  true instance_profile:  null platform_name:  null platform_version:  null profile:  eb-cli repository:  null sc:  git workspace_type:  Application` 
```

### 创造

接下来，让我们创建弹性 Beanstalk 环境并部署应用程序:

同样，系统会提示您几个问题。

#### 环境名称

这表示 EB 环境的名称。我建议坚持使用默认值:“django-images-env”。

> 将`└-env`或`└-dev`后缀添加到您的环境中被认为是一种很好的做法，这样您就可以很容易地将 EB 应用程序与环境区分开来。

#### DNS CNAME 前缀

您的 web 应用程序将在`%cname%.%region%.elasticbeanstalk.com`可访问。同样，使用默认值。

#### 负载平衡

负载平衡器在您的环境实例之间分配流量。选择“应用程序”。

> 如果您想了解不同的负载平衡器类型，请查看适用于您的弹性 Beanstalk 环境的[负载平衡器。](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.managing.elb.html)

#### 现货车队请求

[Spot Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet.html) 请求允许您根据自己的标准按需启动实例。我们不会在本教程中使用它们，所以说“不”。

--

有了它，环境将会旋转起来:

1.  你的代码将被压缩并上传到一个新的 S3 桶。
2.  之后，将创建各种 AWS 资源，如负载平衡器、安全和自动伸缩组以及 EC2 实例。

还将部署一个新的应用程序。

这将需要大约三分钟，所以请随意拿一杯咖啡。

部署完成后，EB CLI 将修改*。elasticbeanstalk/config.yml* 。

您的项目结构现在应该如下所示:

```
`|-- .elasticbeanstalk
|   └-- config.yml
|-- .gitignore
|-- README.md
|-- core
|   |-- __init__.py
|   |-- asgi.py
|   |-- settings.py
|   |-- urls.py
|   └-- wsgi.py
|-- db.sqlite3
|-- images
|   |-- __init__.py
|   |-- admin.py
|   |-- apps.py
|   |-- forms.py
|   |-- migrations
|   |   |-- 0001_initial.py
|   |   └-- __init__.py
|   |-- models.py
|   |-- tables.py
|   |-- templates
|   |   └-- images
|   |       └-- index.html
|   |-- tests.py
|   |-- urls.py
|   └-- views.py
|-- manage.py
└-- requirements.txt` 
```

### 状态

部署应用后，您可以通过运行以下命令来检查其状态:

```
`$ eb status

Environment details for: django-images-env
  Application name: django-images
  Region: us-west-2
  Deployed Version: app-93ec-220218_095635133296
  Environment ID: e-z7dmesipvc
  Platform: arn:aws:elasticbeanstalk:us-west-2::platform/Python 3.8 running on 64bit Amazon Linux 2/3.3.10
  Tier: WebServer-Standard-1.0
  CNAME: django-images-env.us-west-2.elasticbeanstalk.com
  Updated: 2022-02-18 16:00:24.954000+00:00
  Status: Ready
  Health: Red` 
```

您可以看到我们环境的当前健康状况是`Red`，这意味着出现了问题。暂时不要担心这个问题，我们将在接下来的步骤中解决它。

您还可以看到，AWS 为我们分配了一个 CNAME，这是我们的 EB 环境的域名。我们可以通过打开浏览器并导航到 CNAME 来访问 web 应用程序。

### 打开

此命令将打开您的默认浏览器并导航到 CNAME 域。你会看到`502 Bad Gateway`，我们将在这里很快修复它

### 安慰

该命令将在您的默认浏览器中打开 Elastic Beanstalk 控制台:

![Elastic Beanstalk Console](img/13c04d9bed51ff69649d2f0b27cf9b10.png)

同样，您可以看到环境的健康状况是“严重的”，我们将在下一步中解决这个问题。

## 配置环境

在上一步中，我们尝试访问我们的应用程序，它返回了`502 Bad Gateway`。背后有几个原因:

1.  Python 需要`PYTHONPATH`来在我们的应用程序中找到模块。
2.  默认情况下，Elastic Beanstalk 试图从不存在的 *application.py* 启动 WSGI 应用程序。
3.  Django 需要`DJANGO_SETTINGS_MODULE`知道使用哪些设置。

> 默认情况下，Elastic Beanstalk 为 Python 应用程序提供了 [Gunicorn](https://gunicorn.org/) 。EB 在部署过程中自动安装 Gunicorn，因此我们不必将其添加到 *requirements.txt* 中。如果你想用别的东西替换 Gunicorn，看看用 Procfile 配置 WSGI 服务器的[。](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/python-configuration-procfile.html)

让我们修复这些错误。

在项目根目录下创建一个名为“”的新文件夹。ebextensions”。在新创建的文件夹中创建一个名为 *01_django.config* 的文件:

```
`# .ebextensions/01_django.config option_settings: aws:elasticbeanstalk:application:environment: DJANGO_SETTINGS_MODULE:  "core.settings" PYTHONPATH:  "/var/app/current:$PYTHONPATH" aws:elasticbeanstalk:container:python: WSGIPath:  "core.wsgi:application"` 
```

注意事项:

1.  我们将`PYTHONPATH`设置为 EC2 实例上的 Python 路径( [docs](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONPATH) )。
2.  我们将`DJANGO_SETTINGS_MODULE`指向我们的 Django 设置( [docs](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/command-options-specific.html#command-options-python) )。
3.  我们将`WSGIPath`更改为我们的 WSGI 应用程序( [docs](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/command-options-specific.html#command-options-python) )。

> EB 如何*。config* 文件管用吗？
> 
> 1.  你想要多少就有多少。
> 2.  它们按以下顺序加载:01_x、02_x、03_x 等。
> 3.  您不必记住这些设置；您可以通过运行`eb config`列出您的所有环境设置。
> 
> 如果您想了解更多关于高级环境定制的信息，请查看[带有配置文件的高级环境定制](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/ebextensions.html)。

此时，您的项目结构应该如下所示:

```
`|-- .ebextensions
|   └-- 01_django.config
|-- .elasticbeanstalk
|   └-- config.yml
|-- .gitignore
|-- README.md
|-- core
|   |-- __init__.py
|   |-- asgi.py
|   |-- settings.py
|   |-- urls.py
|   └-- wsgi.py
|-- db.sqlite3
|-- images
|   |-- __init__.py
|   |-- admin.py
|   |-- apps.py
|   |-- forms.py
|   |-- migrations
|   |   |-- 0001_initial.py
|   |   └-- __init__.py
|   |-- models.py
|   |-- tables.py
|   |-- templates
|   |   └-- images
|   |       └-- index.html
|   |-- tests.py
|   |-- urls.py
|   └-- views.py
|-- manage.py
└-- requirements.txt` 
```

在重新部署之前，我们必须做的另一件事是将我们的 CNAME 添加到 *core/settings.py* 中的`ALLOWED_HOSTS`:

```
`# core/settings.py

ALLOWED_HOSTS = [
    'xyz.elasticbeanstalk.com',  # make sure to replace it with your own EB CNAME
]` 
```

> 或者，对于测试，您可以只使用通配符:`ALLOWED_HOSTS = ['*']`。只是不要忘记在你完成测试后改变它！

将更改提交给 git 并部署:

```
`$ git add .
$ git commit -m "updates for eb"

$ eb deploy` 
```

> 您会注意到，如果您不提交，Elastic Beanstalk 不会检测到这些变化。这是因为 EB 与 git 集成，并且只检测提交的(更改的)文件。

部署完成后，运行`eb open`看看是否一切正常

哎哟。我们修复了以前的错误，但现在又出现了新的错误:

```
`NotSupportedError at /
deterministic=True requires SQLite 3.8.3 or higher` 
```

别担心。这只是 SQLite 的一个问题，无论如何都不应该在生产中使用。我们将在这里与 Postgres 交换它。

## 配置 RDS

Django 默认使用[的](https://docs.djangoproject.com/en/4.0/ref/databases/#sqlite-notes)数据库 [SQLite](https://www.sqlite.org/index.html) 。虽然这对于开发来说是完美的，但是对于生产来说，您通常会希望迁移到更健壮的数据库，比如 Postgres 或 MySQL。此外，由于版本依赖冲突，当前的 EB 平台不能很好地与 SQLite 一起工作。因为这两点，我们将用 SQlite 替换掉 [Postgres](https://www.postgresql.org/) 。

### 本地邮政汇票

首先，让 Postgres 在本地运行。您可以从 [PostgreSQL Downloads](https://www.postgresql.org/download/) 下载它，或者启动 Docker 容器:

```
`$ docker run --name django-images-postgres -p 5432:5432 \
    -e POSTGRES_USER=django-images -e POSTGRES_PASSWORD=complexpassword123 \
    -e POSTGRES_DB=django-images -d postgres` 
```

检查容器是否正在运行:

```
`$ docker ps -f name=django-images-postgres

CONTAINER ID   IMAGE      COMMAND                  CREATED              STATUS              PORTS                    NAMES
c05621dac852   postgres   "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:5432->5432/tcp   django-images-postgres` 
```

现在，让我们试着用 Django 应用程序连接它。在 *core/settings.py* 中，将`DATABASE`配置更改为以下内容:

```
`# core/settings.py

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'django-images',
        'USER': 'django-images',
        'PASSWORD': 'complexpassword123',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}` 
```

接下来，安装 Postgres 所需的 [psycopg2-binary](https://pypi.org/project/psycopg2-binary/) :

```
`(venv)$ pip install psycopg2-binary==2.9.3` 
```

添加到 *requirements.txt* :

```
`Django==4.0.2
Pillow==9.0.1
django-tables2==2.4.1
django-crispy-forms==1.14.0
psycopg2-binary==2.9.3` 
```

创建和应用迁移:

```
`(venv)$ python manage.py makemigrations
(venv)$ python manage.py migrate` 
```

运行服务器:

```
`(venv)$ python manage.py runserver` 
```

确保您仍然可以在 [http://localhost:8000](http://localhost:8000) 上传图像。

> 如果得到一个`DisallowedHost`错误，将`localhost`和`127.0.0.1`添加到 *core/settings.py* 内的`ALLOWED_HOSTS`中。

### AWS RDS Postgres

要为生产设置 Postgres，首先运行以下命令打开 AWS 控制台:

单击左侧栏上的“配置”，向下滚动到“数据库”，然后单击“编辑”。

使用以下设置创建一个数据库，然后单击“应用”:

*   引擎:postgres
*   引擎版本:12.9(自 db.t2.micro 以来的旧 Postgres 版本在 13.1+版本中不可用)
*   实例类:db.t2.micro
*   存储:5 GB(应该绰绰有余)
*   用户名:选择一个用户名
*   密码:选择一个强密码

> 如果你想留在 [AWS 免费层](https://aws.amazon.com/free/)内，确保你选择 db.t2.micro. RDS 价格会根据你选择的实例类呈指数增长。如果你不想和`micro`一起去，一定要复习 [AWS PostgreSQL 定价](https://aws.amazon.com/rds/postgresql/pricing/)。

![RDS settings](img/8b4fb88bc5665f2310e7e2999654b460.png)

环境更新完成后，EB 会自动将以下数据库凭证传递给我们的 Django 应用程序:

```
`RDS_DB_NAME
RDS_USERNAME
RDS_PASSWORD
RDS_HOSTNAME
RDS_PORT` 
```

我们现在可以使用 *core/settings.py* 中的这些变量来设置`DATABASE`:

```
`# core/settings.py

if 'RDS_DB_NAME' in os.environ:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql_psycopg2',
            'NAME': os.environ['RDS_DB_NAME'],
            'USER': os.environ['RDS_USERNAME'],
            'PASSWORD': os.environ['RDS_PASSWORD'],
            'HOST': os.environ['RDS_HOSTNAME'],
            'PORT': os.environ['RDS_PORT'],
        }
    }
else:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql_psycopg2',
            'NAME': 'django-images',
            'USER': 'django-images',
            'PASSWORD': 'complexpassword123',
            'HOST': 'localhost',
            'PORT': '5432',
        }
    }` 
```

不要忘记导入 *core/settings.py* 顶部的`os`包:

接下来，我们必须告诉 Elastic Beanstalk 在部署新的应用程序版本时运行`makemigrations`和`migrate`。我们可以通过编辑*来实现。EB extensions/01 _ django . config*文件。将以下内容添加到文件的底部:

```
`# .ebextensions/01_django.config container_commands: 01_makemigrations: command:  "source /var/app/venv/*/bin/activate && python3 manage.py makemigrations --noinput" leader_only:  true 02_migrate: command:  "source /var/app/venv/*/bin/activate && python3 manage.py migrate --noinput" leader_only:  true` 
```

现在，每当我们部署新的应用程序版本时，EB 环境都会执行上述命令。我们使用了`leader_only`，所以只有第一个 EC2 实例执行它们(以防我们的 EB 环境运行多个 EC2 实例)。

> 弹性 Beanstalk 配置支持两个不同的命令部分，[命令](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-ec2.html#linux-commands)和[容器 _ 命令](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-ec2.html#linux-container-commands)。它们之间的主要区别在于它们在部署过程中的运行时间:
> 
> 1.  `commands`在设置应用程序和 web 服务器以及提取应用程序版本文件之前运行。
> 2.  `container_commands`在应用程序和 web 服务器已设置且应用程序版本存档已提取之后，但在应用程序版本部署之前(在文件从暂存文件夹移动到其最终位置之前)运行。

让我们也添加一个命令来创建超级用户。我们可以使用 Django 直观的[定制命令框架](https://docs.djangoproject.com/en/4.0/howto/custom-management-commands/)来添加新命令。在“图像”应用程序中，创建以下文件和文件夹:

```
`└-- images
    └-- management
        |-- __init__.py
        └-- commands
            |-- __init__.py
            └-- createsu.py` 
```

*create u . py*:

```
`# images/management/commands/createsu.py

from django.contrib.auth.models import User
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = 'Creates a superuser.'

    def handle(self, *args, **options):
        if not User.objects.filter(username='admin').exists():
            User.objects.create_superuser(
                username='admin',
                password='complexpassword123'
            )
        print('Superuser has been created.')` 
```

接下来，将第三个容器命令添加到*。EB extensions/01 _ django . config*:

```
`# .ebextensions/01_django.config container_commands: 01_makemigrations: command:  "source /var/app/venv/*/bin/activate && python3 manage.py makemigrations --noinput" leader_only:  true 02_migrate: command:  "source /var/app/venv/*/bin/activate && python3 manage.py migrate --noinput" leader_only:  true # ------------------------------------- new ------------------------------------- 03_superuser: command:  "source /var/app/venv/*/bin/activate && python3 manage.py createsu" leader_only:  true # --------------------------------- end of new  ---------------------------------` 
```

> 创建`createsu`命令的另一种方法是 SSH 到一个 EC2 实例中，然后运行 Django 的默认`createsuperuser`命令。

将更改提交给 git 并部署:

```
`$ git add .
$ git commit -m "updates for eb"

$ eb deploy` 
```

等待部署完成。完成后，运行`eb open`在新的浏览器标签中打开你的应用。您的应用程序现在应该可以工作了。确保您可以上传图像。

## 文件存储的 S3

查看您的管理仪表板的部署版本。静态文件没有得到正确的服务。此外，我们不希望静态或媒体文件本地存储在 EC2 实例上，因为 EB 应用程序应该是无状态的，这使得将您的应用程序扩展到多个 EC2 实例更加容易。

虽然 AWS 提供了许多持久存储服务，但 T2 S3 T3 可以说是最受欢迎和最容易使用的。

要配置 S3，我们需要:

1.  创建 S3 存储桶
2.  为 S3 时段管理创建 IAM 组和用户
3.  设置弹性豆茎 S3 环境变量
4.  配置 Django 静态和媒体设置

### 创建 S3 存储桶

首先，让我们创建一个新的 S3 存储桶。导航到 [AWS S3 控制台](https://s3.console.aws.amazon.com/s3/home)并点击“创建存储桶”。给这个桶一个惟一的名称，并设置 AWS 区域。对其他一切使用默认配置。按“创建”。

### IAM 组和用户

导航到 [IAM 控制台](https://console.aws.amazon.com/iamv2/home)。在屏幕左侧，选择“用户组”。创建一个具有“AmazonS3FullAccess”权限的新组:

![AWS IAM Group](img/58a2f20fbabb15738eddb795ad86fa15.png)

然后，创建一个具有“编程访问”权限的新用户，并将该组分配给该用户:

![AWS IAM User](img/2bd0b7b2d1fbd0a8328784f4e24f3b85.png)

AWS 将为您生成认证凭证。下载提供的*。csv* 文件。在下一步中，我们需要将它们传递给我们的 Elastic Beanstalk 环境。

### 设置 EB 环境变量

接下来，我们需要设置以下环境变量:

```
`AWS_ACCESS_KEY_ID  -  your  ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY  -  your  SECRET_ACCESS_KEY AWS_S3_REGION_NAME  -  your  selected  S3  region AWS_STORAGE_BUCKET_NAME  -  your  bucket  name` 
```

导航到您的弹性豆茎控制台。点击“配置”。然后，在“软件”类别中，单击“编辑”并向下滚动到“环境属性”部分。添加四个变量。

![AWS Elastic Beanstalk Environment Variables](img/e20faf7825075f82a6c4f92d1e7526e1.png)

添加完所有变量后，点击“应用”。

接下来，为了让 Django 与我们的 S3 桶通信，我们需要安装 [django-storages](https://django-storages.readthedocs.io/en/latest/) 和 [boto3](https://aws.amazon.com/sdk-for-python/) 包。

将它们添加到 *requirements.txt* 文件中:

```
`Django==4.0.2
Pillow==9.0.1
django-tables2==2.4.1
django-crispy-forms==1.14.0
psycopg2-binary==2.9.3
boto3==1.21.3
django-storages==1.12.3` 
```

接下来，将新安装的 app 添加到 *core/settings.py* 中的`INSTALLED_APPS`:

```
`# core/settings.py

INSTALLED_APPS = [
    # ...
    'storages',
]` 
```

配置 django-storages 以使用由 Elastic Beanstalk 传递的环境变量:

```
`if 'AWS_STORAGE_BUCKET_NAME' in os.environ:
    STATICFILES_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
    DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'

    AWS_STORAGE_BUCKET_NAME = os.environ['AWS_STORAGE_BUCKET_NAME']
    AWS_S3_REGION_NAME = os.environ['AWS_S3_REGION_NAME']

    AWS_S3_ACCESS_KEY_ID = os.environ['AWS_ACCESS_KEY_ID']
    AWS_S3_SECRET_ACCESS_KEY = os.environ['AWS_SECRET_ACCESS_KEY']` 
```

最后，我们需要在部署完成后运行`collectstatic`命令，因此将以下内容添加到 *01_django.config* 的底部:

```
`# .ebextensions/01_django.config # ... container_commands: # ... 04_collectstatic: command:  "source /var/app/venv/*/bin/activate && python3 manage.py collectstatic --noinput" leader_only:  true` 
```

完整的文件现在应该如下所示:

```
`# .ebextensions/01_django.config option_settings: aws:elasticbeanstalk:application:environment: DJANGO_SETTINGS_MODULE:  "core.settings" PYTHONPATH:  "/var/app/current:$PYTHONPATH" aws:elasticbeanstalk:container:python: WSGIPath:  "core.wsgi:application" container_commands: 01_makemigrations: command:  "source /var/app/venv/*/bin/activate && python3 manage.py makemigrations --noinput" leader_only:  true 02_migrate: command:  "source /var/app/venv/*/bin/activate && python3 manage.py migrate --noinput" leader_only:  true 03_superuser: command:  "source /var/app/venv/*/bin/activate && python3 manage.py createsu" leader_only:  true 04_collectstatic: command:  "source /var/app/venv/*/bin/activate && python3 manage.py collectstatic --noinput" leader_only:  true` 
```

将更改提交给 git 并部署:

```
`$ git add .
$ git commit -m "updates for eb"

$ eb deploy` 
```

确认静态和媒体文件现在存储在 S3 上。

> 如果您得到一个`Signature mismatch`错误，您可能想要将以下设置添加到 *core/settings.py* : `AWS_S3_ADDRESSING_STYLE = "virtual"`。更多详情，请参见[本期 GitHub](https://github.com/jschneier/django-storages/issues/782)。
> 
> 要了解更多关于 AWS S3 上静态和媒体文件存储的信息，请看一下亚马逊 S3 上的文章[存储 Django 静态和媒体文件。](/blog/storing-django-static-and-media-files-on-amazon-s3/)

## HTTPS 与证书管理器

教程的这一部分要求您有一个域名。

> 需要一个便宜的域名来练习？几个域名注册商有特殊优惠。“xyz”域。或者，您可以在 [Freenom](https://www.freenom.com/) 创建一个免费域名。如果你没有域名，但仍然想使用 HTTPS，你可以[创建并签署一个 X509 证书](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/configuring-https-ssl.html)。

要通过 HTTPS 为您的申请提供服务，我们需要:

1.  请求并验证 SSL/TLS 证书
2.  把你的域名指向你的 EB CNAME
3.  修改负载平衡器以服务于 HTTPS
4.  修改您的应用程序设置

### 请求并验证 SSL/TLS 证书

导航到 [AWS 证书管理器控制台](https://console.aws.amazon.com/acm/)。单击“申请证书”。将证书类型设置为“公共”，然后单击“下一步”。在表单输入中输入您的[全限定域名](https://docs.aws.amazon.com/acm/latest/userguide/setup-domain.html)，设置“验证方式”为“DNS 验证”，点击“请求”。

![AWS Request Public Certificate](img/8dc728a1d52b26df9d274d7ac54a2402.png)

然后，您将被重定向到一个页面，在那里您可以看到您的所有证书。您刚刚创建的证书应该具有“待验证”状态。

为了让 AWS 颁发证书，你首先必须证明你是这个域名的所有者。在表中，单击证书以查看“证书详细信息”。注意“CNAME 的名字”和“CNAME 的价值”。要验证域的所有权，您需要在域的 DNS 设置中创建“CNAME 记录”。为此使用“CNAME 名称”和“CNAME 价值”。一旦完成，Amazon 将需要几分钟的时间来获取域更改并颁发证书。状态应该从“等待验证”更改为“已发布”。

### 将域名指向 EB CNAME

接下来，您需要将您的域(或子域)指向您的 EB 环境 CNAME。回到您的域名的 DNS 设置，添加另一个 CNAME 记录，其值为您的 EB CNAME -例如，`django-images-dev.us-west-2.elasticbeanstalk.com`。

等待几分钟，让您的 DNS 刷新，然后在浏览器中测试您的域名的`http://`风格。

### 修改负载平衡器以服务于 HTTPS

回到弹性豆茎控制台，点击“配置”。然后，在“负载平衡器”类别中，单击“编辑”。单击“添加监听程序”并使用以下详细信息创建监听程序:

1.  端口- 443
2.  议定书- HTTPS
3.  SSL 证书-选择您刚刚创建的证书

点击“添加”。然后，滚动到页面底部，单击“应用”。环境更新需要几分钟时间。

### 修改您的应用程序设置

接下来，我们需要对 Django 应用程序进行一些修改。

首先，将您的完全限定域添加到`ALLOWED_HOSTS`:

```
`# core/settings.py

ALLOWED_HOSTS = [
    # ...
    'yourdomain.com',
]` 
```

最后，我们需要将所有流量从 HTTP 重定向到 HTTPS。有多种方法可以做到这一点，但最简单的方法是将 [Apache](https://httpd.apache.org/) 设置为代理主机。我们可以通过在*中的`option_settings`末尾添加以下内容来编程实现这一点。EB extensions/01 _ django . config*:

```
`# .ebextensions/01_django.config

option_settings:
  # ...
  aws:elasticbeanstalk:environment:proxy:  # new
    ProxyServer: apache                    # new` 
```

您最终的 *01_django.config* 文件现在应该是这样的:

```
`# .ebextensions/01_django.config option_settings: aws:elasticbeanstalk:application:environment: DJANGO_SETTINGS_MODULE:  "core.settings" PYTHONPATH:  "/var/app/current:$PYTHONPATH" aws:elasticbeanstalk:container:python: WSGIPath:  "core.wsgi:application" aws:elasticbeanstalk:environment:proxy: ProxyServer:  apache container_commands: 01_makemigrations: command:  "source /var/app/venv/*/bin/activate && python3 manage.py makemigrations --noinput" leader_only:  true 02_migrate: command:  "source /var/app/venv/*/bin/activate && python3 manage.py migrate --noinput" leader_only:  true 03_superuser: command:  "source /var/app/venv/*/bin/activate && python3 manage.py createsu" leader_only:  true 04_collectstatic: command:  "source /var/app/venv/*/bin/activate && python3 manage.py collectstatic --noinput" leader_only:  true` 
```

接下来，创建一个”。平台"文件夹中，并添加以下文件和文件夹:

```
`└-- .platform
    └-- httpd
        └-- conf.d
            └-- ssl_rewrite.conf` 
```

*ssl_rewrite.conf* :

```
`# .platform/httpd/conf.d/ssl_rewrite.conf

RewriteEngine On
<If "-n '%{HTTP:X-Forwarded-Proto}' && %{HTTP:X-Forwarded-Proto} != 'https'">
RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [R,L]
</If>` 
```

您的项目结构现在应该如下所示:

```
`|-- .ebextensions
|   └-- 01_django.config
|-- .elasticbeanstalk
|   └-- config.yml
|-- .gitignore
|-- .platform
|   └-- httpd
|       └-- conf.d
|           └-- ssl_rewrite.conf
|-- README.md
|-- core
|   |-- __init__.py
|   |-- asgi.py
|   |-- settings.py
|   |-- urls.py
|   └-- wsgi.py
|-- db.sqlite3
|-- images
|   |-- __init__.py
|   |-- admin.py
|   |-- apps.py
|   |-- forms.py
│   ├── management
│   │   ├── __init__.py
│   │   └── commands
│   │       ├── __init__.py
│   │       └── createsu.py
|   |-- migrations
|   |   |-- 0001_initial.py
|   |   └-- __init__.py
|   |-- models.py
|   |-- tables.py
|   |-- templates
|   |   └-- images
|   |       └-- index.html
|   |-- tests.py
|   |-- urls.py
|   └-- views.py
|-- manage.py
└-- requirements.txt` 
```

将更改提交给 git 并部署:

```
`$ git add .
$ git commit -m "updates for eb"

$ eb deploy` 
```

现在，在你的浏览器中，你的应用程序的`https://`风格应该工作了。试试去`http://`味的。你应该被重定向到`https://`风味。确保证书也正确加载:

![secure app](img/25cb5abdd5b13c9bdd23ace9ffa8058b.png)

## 环境变量

在生产中，[最好将特定于环境的配置存储在环境变量](https://12factor.net/config)中。使用 Elastic Beanstalk，您可以用两种不同的方式设置自定义环境变量。

### 通过 EB CLI 的环境变量

让我们把 Django 的`SECRET_KEY`和`DEBUG`设置变成环境变量。

从跑步开始:

```
`$ eb setenv DJANGO_SECRET_KEY='<replace me with your own secret key>' \
            DJANGO_DEBUG='1'` 
```

> 您可以用一个命令设置多个环境变量，用空格分隔它们。这是推荐的方法，因为它只需要对 EB 环境进行一次更新。

相应地更改 *core/settings.py* :

```
`# core/settings.py

SECRET_KEY = os.environ.get(
    'DJANGO_SECRET_KEY',
    '<replace me with your own fallback secret key>'
)

DEBUG = os.environ.get('DJANGO_DEBUG', '1').lower() in ['true', 't', '1']` 
```

将更改提交给 git 并部署:

```
`$ git add .
$ git commit -m "updates for eb"

$ eb deploy` 
```

### 通过 EB 控制台的环境变量

通过`eb open`进入弹性豆茎控制台。导航至“配置”>“软件”>“编辑”。然后，向下滚动到“环境属性”。

![AWS Elastic Beanstalk Environment Variables](img/8169e58f9b6aa68700fe1de3250bccbf.png)

完成后，单击“应用”,您的环境将会更新。

然后，您可以通过`os.environ`在您的 Python 环境中访问这些变量。

例如:

```
`VARIABLE_NAME = os.environ['VARIABLE_NAME']` 
```

## 调试弹性豆茎

当使用 Elastic Beanstalk 时，如果您不知道如何访问日志文件，那么找出问题所在会非常令人沮丧。在这一节中，我们将会看到这一点。

有两种方法可以访问日志:

1.  弹性 Beanstalk CLI 或控制台
2.  SSH 到 EC2 实例

> 从个人经验来看，我已经能够用第一种方法解决所有问题。

### 弹性 Beanstalk CLI 或控制台

CLI:

该命令将从以下文件中获取最后 100 行:

```
`/var/log/web.stdout.log /var/log/eb-hooks.log /var/log/nginx/access.log /var/log/nginx/error.log /var/log/eb-engine.log` 
```

> 运行`eb logs`相当于登录 EB 控制台，导航到“日志”。

我建议将日志传送到 [CloudWatch](https://aws.amazon.com/cloudwatch/) 。运行以下命令来启用此功能:

```
`$ eb logs --cloudwatch-logs enable` 
```

您通常会在 */var/log/web.stdout.log* 或 */var/log/eb-engine.log* 中找到 Django 错误。

> 要了解更多关于弹性 Beanstalk 日志的信息，请查看来自 Amazon EC2 实例的日志。

### SSH 到 EC2 实例

要连接到运行 Django 应用程序的 EC2 实例，运行:

第一次会提示您将主机添加到已知主机。答应吧。这样，您现在就可以完全访问 EC2 实例了。请随意试验 Django 管理命令，并检查前一节中提到的一些日志文件。

> 请记住，Elastic Beanstalk 会自动伸缩和部署新的 EC2 实例。您在这个特定 EC2 实例上所做的更改不会反映在新启动的 EC2 实例上。一旦这个特定的 EC2 实例被替换，您的更改将被清除。

## 结论

在本教程中，我们介绍了将 Django 应用程序部署到 AWS Elastic Beanstalk 的过程。到目前为止，您应该对弹性豆茎的工作原理有了一个大致的了解。通过回顾本教程开头的目标，快速进行自我检查。

后续步骤:

1.  你应该考虑创建两个独立的 EB 环境(`dev`和`production`)。
2.  查看用于您的弹性 Beanstalk 环境的自动伸缩组,了解如何配置触发器来自动伸缩您的应用程序。

要删除我们在整个教程中创建的所有 AWS 资源，首先要终止 Elastic Beanstalk 环境:

您需要手动删除 S3 桶、SSL 证书以及 IAM 组和用户。

最后，你可以在 GitHub 上的[django-elastic-beanstalk](https://github.com/testdrivenio/django-elastic-beanstalk)repo 中找到代码的最终版本。