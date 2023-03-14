# 部署 Django 应用程序进行渲染

> 原文：<https://testdriven.io/blog/django-render/>

在本教程中，我们将看看如何部署一个 [Django](https://www.djangoproject.com/) 应用程序来[渲染](https://render.com)。

## 目标

学完本教程后，您应该能够:

1.  解释什么是渲染以及它是如何工作的。
2.  部署一个 Django 应用程序来呈现。
3.  渲染时加速 PostgreSQL 实例。
4.  了解如何在渲染时提供静态和媒体文件。
5.  添加一个自定义域并在 HTTPS 上提供您的 web 应用程序。

## 什么是渲染？

Render 是一个易于使用的[平台即服务](https://en.wikipedia.org/wiki/Platform_as_a_service) (PaaS)解决方案，非常适合构建和运行您的所有应用程序和网站。它于 2019 年推出，此后越来越受欢迎。Render 允许您托管静态站点、web 服务、PostgreSQL 数据库和 Redis 实例。

它极其简单的用户界面/UX 和强大的 git 集成让你可以在几分钟内启动并运行一个应用。它具有对 Python、Node.js、Ruby、Elixir、Go 和 Rust 的原生支持。如果这些都不适合你，Render 还可以通过一个 [Dockerfile](https://docs.docker.com/engine/reference/builder/) 进行部署。

Render 的[自动缩放功能](https://render.com/docs/scaling#autoscaling)将确保你的应用程序总是以合适的价格拥有必要的资源。此外，Render 上托管的所有内容也可以获得免费的 TLS 证书。

> 参考他们的[官方文档](https://render.com/pricing)以获得更多关于他们定价的信息。

### 为什么渲染？

*   非常适合初学者
*   轻松设置和部署应用
*   基于实时 CPU 和内存使用情况的自动扩展
*   免费层(包括 web 服务、PostgreSQL、Redis)——非常适合原型开发
*   良好的客户支持

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

![django-images Application Preview](img/840725e76e1fd56daae6f89c5575a972.png)

## 配置 Django 项目

在本节教程中，我们将准备 Django 项目，以便进行渲染部署。

### 环境变量

我们不应该在源代码中存储秘密，所以让我们利用环境变量。最简单的方法是使用名为 [python-dotenv](https://saurabh-kumar.com/python-dotenv/) 的第三方包。首先将其添加到 *requirements.txt* :

> 随意使用不同的包来处理环境变量，如 [django-environ](https://github.com/joke2k/django-environ) 或 [python-decouple](https://github.com/henriquebastos/python-decouple/) 。

接下来，导航到您的 *settings.py* 并在文件顶部初始化 python-dotenv，如下所示:

```py
`# core/settings.py

from pathlib import Path

from dotenv import load_dotenv

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

load_dotenv(BASE_DIR / '.env')` 
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

不要忘记在文件顶部导入`os`:

### 数据库ˌ资料库

让我们把 Django 的默认 SQLite 换成 PostgreSQL。

通过将下面一行添加到 *requirements.txt* 来安装数据库适配器:

稍后，当我们启动 PostgreSQL 数据库时，Render 将为我们提供一个`DATABASE_URL`。这是一个受[十二因素应用](https://12factor.net/)启发的环境变量，包括连接数据库所需的所有参数。它将采用以下格式:

```py
`postgres://USER:PASSWORD@HOST:PORT/NAME` 
```

为了在 Django 中使用它，我们可以使用一个名为 [dj-database-url](https://pypi.org/project/dj-database-url/) 的包。这个包允许我们将数据库 URL 转换成 Django 数据库参数。

像这样添加到 *requirements.txt* 中:

接下来，导航到 *core/settings.py* ，将`DATABASES`更改如下:

```py
`# core/settings.py

DATABASES = {
    'default': dj_database_url.parse(os.environ.get('DATABASE_URL'), conn_max_age=600),
}` 
```

不要忘记重要的一点:

### 格尼科恩

接下来，让我们安装 [Gunicorn](https://gunicorn.org/) ，这是一个生产级的 WSGI 服务器，将用于生产，而不是 Django 的开发服务器。

添加到 *requirements.txt* :

### 构建脚本

为了收集静态文件和迁移数据库，我们将创建一个构建脚本。构建脚本允许我们在部署应用程序之前运行一系列命令。

在项目根目录下创建一个 *build.sh* 文件，内容如下:

```py
`#!/usr/bin/env bash

set -o errexit  # exit on error

pip install -r requirements.txt

python manage.py collectstatic --no-input
python manage.py migrate` 
```

将所有更改提交给 git，并推送到您的遥控器。

## 部署

在本节教程中，我们将启动一个 PostgreSQL 实例，并部署 Django 应用程序进行渲染。

> 如果你还没有渲染账户，就去[注册](https://dashboard.render.com/register?next=/)吧。

### 数据库ˌ资料库

在[渲染面板](https://dashboard.render.com/)中，点击屏幕右上角的“新建”，然后点击“PostgreSQL”。然后，使用以下参数创建一个 PostgreSQL 实例:

1.  名称:**自定义名称**
2.  数据库:**留空**
3.  用户:**留空**
4.  地区:**离你最近的地区**
5.  PostgreSQL 版本: **15**
6.  数据狗 API 键:**留空**
7.  计划类型:**适合您需求的计划**

> 请记住，免费渲染帐户有以下限制:
> 
> 1.  如果你不升级你的帐户，免费的 PostgreSQL 数据库会在 90 天后被删除。
> 2.  Render 只为免费的 PostgreSQL 数据库提供 1 GB 的存储空间。

等待数据库状态从“正在创建”变为“可用”，然后向下滚动到“连接”部分。记下“内部数据库 URL”。

![Render dashboard - database URL](img/a190a2d36252ff69f60983f76b4fd5c8.png)

太好了。数据库部分到此为止。

### 网络服务

接下来，让我们创建一个 web 服务。

再次单击屏幕右上角的“New ”,但这次选择“Web Service”。

将您的渲染帐户连接到 GitHub 或 GitLab 帐户。确保向您想要部署的存储库授予呈现权限。连接后，选择您的存储库。

输入以下详细信息:

1.  名称:**自定义名称**
2.  区域:**与您的数据库相同的区域**
3.  分支:**您的主要分支(例如，主/主要)**
4.  根目录:**留空**
5.  环境: **Python 3**
6.  构建命令: **`sh build.sh`**
7.  开始命令 **`gunicorn core.wsgi:application`**
8.  计划类型:**适合您需求的计划**

打开“高级”下拉菜单，添加以下环境变量:

1.  `PYTHON_VERSION` : `3.9.9`
2.  `SECRET_KEY`:点击“生成”
3.  `DEBUG` : `1`
4.  `ALLOWED_HOSTS` : `*`
5.  `DATABASE_URL` : `<your_internal_database_url>`

我们需要设置`PYTHON_VERSION`，因为 Render 的默认 Python 版本是`3.7`，而 Django 4 需要`3.8`或更高版本。我们还临时启用了调试模式，并允许所有主机。不要担心这一点，因为我们将在教程的后面更改它。

最后，单击“创建 Web 服务”。

Render 将检查您的源代码，准备环境，运行 *build.sh* ，生成容器，并部署它。

![Render dashboard - deploying](img/f2b8d9298d6fb64096fb1524ed166cc6.png)

等待几分钟，让部署状态变为“实时”，然后通过上传图像来测试应用程序。您的 web 应用程序的 URL 显示在 web 服务名称下(左上角)。

每次您将代码签入遥控器时，Render 都会自动重新部署您的应用程序。

## 静态文件

为了在生产中提供静态文件，我们可以使用一个名为[whiten noise](https://whitenoise.evans.io/en/latest/)的包。WhiteNoise 使我们的应用程序成为一个独立的单元，可以部署在任何地方，而不依赖于 Nginx、Apache 或任何其他外部服务。此外，它通过使用 [gzip](https://www.gnu.org/software/gzip/) 和 [Brotli 格式](https://github.com/google/brotli)来压缩我们的内容。

首先将以下两个包添加到 *requirements.txt* 中:

```py
`whitenoise==6.2.0
Brotli==1.0.9` 
```

> 添加 Brotli 支持是可选的，但是 Whitenoise 推荐的[以及 Render。](https://whitenoise.evans.io/en/latest/django.html#brotli-compression)

接下来，将 WhiteNoise 的中间件添加到 *settings.py* 中的`MIDDLEWARE`列表中。它应该放在除 Django 的`SecurityMiddleware`之外的所有其他中间件之上:

```py
`# core/settings.py

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',
    # ...
]` 
```

最后，向下滚动到 *settings.py* 的底部，添加:

```py
`# core/settings.py

STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'` 
```

就是这样。下次 Render 部署新版本的应用程序时，我们的静态文件将由 WhiteNoise 提供服务。

Render(以及许多其他类似的服务，如 Heroku)提供了一个短暂的文件系统。这意味着当应用程序关闭或重新部署时，您的数据不会持久，可能会消失。如果你的应用程序需要保留文件，这是非常糟糕的。

要启用持久存储，您可以使用:

1.  [渲染磁盘](https://render.com/docs/disks):带自动每日快照的高性能固态硬盘
2.  云对象存储，如 AWS S3 或类似的服务

我建议你设置 AWS S3，因为渲染磁盘不允许水平缩放，而且有点贵。

> 要了解如何使用 Django 设置 AWS S3，请看一下在亚马逊 S3 上存储 Django 静态和媒体文件的[。](https://testdriven.io/blog/storing-django-static-and-media-files-on-amazon-s3/)

然而，为了本教程的完整性，让我们设置渲染磁盘。

导航到您的渲染仪表板并选择您的 web 服务。单击边栏上的“Disks ”,并使用以下详细信息创建一个新磁盘:

1.  名称:**选择一个自定义名称**
2.  挂载路径:**/opt/render/project/src/media files**
3.  尺寸:**适合你的最小尺寸**

点击“创建”就完成了。您的媒体文件现在将保留。

## Django 管理访问

创建超级用户有两种方法:

1.  SSH 进入服务器并运行`createsuperuser`命令。
2.  创建一个 Django 命令来创建超级用户，并将其添加到 *build.sh* 。

我们将使用第二种方法，因为它允许我们自动化部署，而且 Render 不允许免费用户 SSH 到他们的 web 服务。

首先，在“images”应用程序中创建以下目录结构:

```py
`└-- images
    └-- management
        |-- __init__.py
        └-- commands
            |-- __init__.py
            └-- createsu.py` 
```

将以下内容放入 *createsu.py* 中:

```py
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

> 如果您不想在源代码中暴露您的超级用户凭证，请考虑从环境变量中加载它们。

将`createsu`命令添加到 *build.sh* 的末尾，如下所示:

```py
`#!/usr/bin/env bash

set -o errexit  # exit on error

pip install -r requirements.txt

python manage.py collectstatic --no-input
python manage.py migrate
python manage.py createsu  # new` 
```

最后，提交并推送代码。

Render 将自动重新部署您的应用程序。完成后，导航到您的 web 应用的管理仪表板并尝试登录。

## 自定义域

导航到您的[渲染仪表板](https://dashboard.render.com/)并选择您的 web 服务。选择边栏上的“设置”，然后向下滚动到“自定义域”部分。点击“添加”，输入您的域名，然后点击“保存”。

![Render dashboard - custom domain](img/c48171c2ae629f1d180ba8c0948c9bf7.png)

接下来，进入你的域名注册服务商 DNS 设置，添加一个新的“CNAME 记录”指向你的应用程序的主机名，如下所示:

```py
`+----------+--------------+-----------------------------------+-----------+ | Type     | Host         | Value                             | TTL       |
+----------+--------------+-----------------------------------+-----------+ | CNAME    | <some host> | <your_app_hostname> | Automatic |
+----------+--------------+-----------------------------------+-----------+` 
```

示例:

```py
`+----------+--------------+-----------------------------------+-----------+ | Type     | Host         | Value                             | TTL       |
+----------+--------------+-----------------------------------+-----------+ | CNAME    | render       | django-images-374w.onrender.com   | Automatic |
+----------+--------------+-----------------------------------+-----------+` 
```

等待几分钟，让 DNS 更改传播开来，然后单击“验证”。Render 将为您的域颁发 TLS 证书，您的应用现在可以在 HTTPS 上的自定义域中访问。

为确保其有效，请尝试访问您的 web 应用，网址为:

```py
`https://<your_custom_domain>

Example:
https://render.testdriven.io` 
```

将`ALLOWED_HOSTS`环境变量改为`<your_custom_domain>`(例如`render.testdriven.io`)。然后，触发最新提交的手动部署。

等待您的应用程序重新部署，您就大功告成了！

## 结论

在本教程中，我们已经成功地部署了一个 Django 应用程序来进行渲染。我们已经处理了 PostgreSQL 数据库、静态和媒体文件，添加了自定义域名并启用了 HTTPS。现在，您应该对 Render 的工作原理有了一个大致的了解，并且能够部署自己的应用程序了。

### 下一步是什么？

1.  考虑 AWS S3 或类似的服务，以更好、更安全的方式提供媒体文件。
2.  设置`DEBUG=0`禁用调试模式。请记住，该应用程序仅在启用调试模式时提供媒体文件。有关如何在生产中处理静态和媒体文件的更多信息，请参考[在 Django](https://testdriven.io/blog/django-static-files/) 中处理静态和媒体文件。
3.  看看 Render 的[缩放](https://render.com/docs/scaling)特性。