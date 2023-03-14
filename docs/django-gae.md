# 将 Django 应用程序部署到 Google 应用程序引擎

> 原文：<https://testdriven.io/blog/django-gae/>

在本教程中，我们将看看如何将一个 [Django](https://www.djangoproject.com/) 应用安全地部署到[谷歌应用引擎](https://cloud.google.com/appengine)。

## 目标

学完本教程后，您应该能够:

1.  解释什么是谷歌应用引擎，它是如何工作的。
2.  将 Django 应用程序部署到 Google App Engine。
3.  在[云 SQL](https://cloud.google.com/sql/) 上运行 Postgres 实例。
4.  利用[秘密管理器](https://cloud.google.com/secret-manager)处理环境变量和秘密。
5.  使用[云存储](https://cloud.google.com/storage)为静态和媒体文件设置持久存储。
6.  将域名链接到您的应用程序，并在 HTTPS 上提供您的应用程序。

## 什么是谷歌应用引擎？

Google App Engine (GAE)是一个完全托管的无服务器平台，用于大规模开发和托管网络应用。它具有强大的内置自动扩展功能，可以根据需求自动分配更多/更少的资源。GAE 原生支持用 Python、Node.js、Java、Ruby、C#、Go 和 PHP 编写的应用程序。或者，它通过[定制运行时](https://cloud.google.com/appengine/docs/flexible/custom-runtimes)或 Dockerfiles 提供对其他语言的支持。

它具有强大的应用程序诊断功能，您可以将它与[云监控](https://cloud.google.com/monitoring)和[日志记录](https://cloud.google.com/logging)相结合，以监控您的应用程序的健康状况和性能。此外，GAE 允许你的应用扩展到零，这意味着如果没有人使用你的服务，你不用支付任何费用。

在撰写本文时，谷歌[为新用户提供了](https://cloud.google.com/free)300 美元的免费积分来试用他们的平台。积分将在 90 天后到期。

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

![django-images Application Preview](img/88771f42d782008ae0c84689b4971be6.png)

## 安装 Google Cloud CLI

要使用[谷歌云平台](https://cloud.google.com/) (GCP)，首先安装[谷歌云 CLI](https://cloud.google.com/sdk/gcloud) (gcloud CLI)。gcloud CLI 允许您创建和管理您的 Google 云资源和服务。

根据您的操作系统和处理器架构，安装过程会有所不同。继续按照[官方安装指南](https://cloud.google.com/sdk/docs/install#installation_instructions)为您的操作系统和 CPU 安装。

要验证安装是否成功，请运行:

```py
`$ gcloud version

Google Cloud SDK 415.0.0
bq 2.0.84
core 2023.01.20
gcloud-crc32c 1.0.0
gsutil 5.18` 
```

## 配置 Django 项目

在教程的这一部分，我们将配置 Django 项目，以便与 GAE 一起工作。

### 环境变量

我们不应该在源代码中存储秘密，所以让我们利用环境变量。最简单的方法是使用名为 [django-environ](https://django-environ.readthedocs.io/en/latest/) 的第三方 Python 包。首先将其添加到 *requirements.txt* :

> 我建议您继续使用 django-environ，因为它是专门针对 django 的，并且支持数据库 URL 中的 Unix 套接字路径。

对于 Django 来说，要初始化环境更改，请更新 *settings.py* 的顶部，如下所示:

```py
`# core/settings.py

import os
import environ

from pathlib import Path

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

env = environ.Env(DEBUG=(bool, False))
env_file = os.path.join(BASE_DIR, '.env')
env.read_env(env_file)` 
```

接下来，从环境中加载`SECRET_KEY`和`DEBUG`:

```py
`# core/settings.py

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = env('SECRET_KEY')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = env('DEBUG')` 
```

为了设置`ALLOWED_HOSTS`和`CSRF_TRUSTED_ORIGINS`，我们可以使用来自 GAE 文档的以下[代码片段](https://cloud.google.com/python/django/appengine#csrf_configurations):

```py
`# core/settings.py

APPENGINE_URL = env('APPENGINE_URL', default=None)
if APPENGINE_URL:
    # ensure a scheme is present in the URL before it's processed.
    if not urlparse(APPENGINE_URL).scheme:
        APPENGINE_URL = f'https://{APPENGINE_URL}'

    ALLOWED_HOSTS = [urlparse(APPENGINE_URL).netloc]
    CSRF_TRUSTED_ORIGINS = [APPENGINE_URL]
    SECURE_SSL_REDIRECT = True
else:
    ALLOWED_HOSTS = ['*']` 
```

这段代码从环境中获取`APPENGINE_URL`，并自动配置`ALLOWED_HOSTS`和`CSRF_TRUSTED_ORIGINS`。此外，它使`SECURE_SSL_REDIRECT`能够执行 HTTPS。

不要忘记在文件顶部添加导入:

```py
`from urllib.parse import urlparse` 
```

### 数据库ˌ资料库

要使用 Postgres 代替 SQLite，我们首先需要安装数据库适配器。

将下面一行添加到 *requirements.txt* 中:

在本教程的后面，我们将创建一个 Postgres 实例，为我们提供形成一个受[十二因素应用](https://12factor.net/)启发的数据库 URL 所需的细节。`DATABASE_URL`将采用以下格式:

```py
`postgres://USER:[[email protected]](/cdn-cgi/l/email-protection)//cloudsql/PROJECT_ID:REGION:INSTANCE_NAME/DATABASE_NAME` 
```

为了将`DATABASE_URL`用于 Django，我们可以像这样使用 django-environ 的`db()`方法:

```py
`# core/settings.py

DATABASES = {'default': env.db()}` 
```

### 格尼科恩

接下来，让我们安装 [Gunicorn](https://gunicorn.org/) ，这是一个生产级的 WSGI 服务器，将用于生产，而不是 Django 的开发服务器。

添加到 *requirements.txt* :

### app.yaml

Google App Engine 的 [app.yaml](https://cloud.google.com/appengine/docs/standard/python3/configuring-your-app-with-app-yaml) 配置文件用于配置 web 应用程序的运行时环境。 *app.yaml* 文件包含运行时、URL 处理程序和环境变量等信息。

首先在项目根目录下创建一个名为 *app.yaml* 的新文件，包含以下内容:

```py
`# app.yaml runtime:  python39 env:  standard entrypoint:  gunicorn -b :$PORT core.wsgi:application handlers: -  url:  /.* script:  auto runtime_config: python_version:  3` 
```

注意事项:

1.  我们定义了启动 WSGI 服务器的`entrypoint`命令。
2.  [env](https://cloud.google.com/appengine/docs/the-appengine-environments#app-engine-environments) 有两个选项:`standard`和`flexible`。我们选择了 standard，因为它更容易启动和运行，适用于较小的应用程序，并且支持 Python 3.9 开箱即用。
3.  最后，`handlers`定义不同的 URL 是如何路由的。我们将在教程的后面定义静态和媒体文件的处理程序。

> 有关 *app.yaml* 的更多信息，请查看[文档](https://cloud.google.com/appengine/docs/flexible/reference/app-yaml?tab=python)。

### 。gcloudnignore

一个[。gcloudnignore](https://cloud.google.com/sdk/gcloud/reference/topic/gcloudignore)file 允许您在部署应用程序时指定不想上传到 GAE 的文件。它的工作原理类似于*。gitignore* 文件。

继续创建一个*。项目根中的 gcloudnignore*文件包含以下内容:

```py
`# .gcloudignore .gcloudignore # Ignore local .env file .env # If you would like to upload your .git directory, .gitignore file, or files # from your .gitignore file, remove the corresponding line # below: .git .gitignore # Python pycache: __pycache__/ # Ignore collected static and media files mediafiles/ staticfiles/ # Ignore the local DB db.sqlite3 # Ignored by the build system /setup.cfg venv/ # Ignore IDE files .idea/` 
```

## 部署应用程序

在本节教程中，我们将把应用程序部署到 Google App Engine。

### 项目初始化

如果您尚未初始化 gcloud CLI，请继续操作:

CLI 将打开您的浏览器，要求您登录并接受一些权限。

之后，你必须选择你的项目。我建议您创建一个新项目，因为删除一个项目比单独删除所有服务和资源更容易。

对于该地区，选择离您最近的地区。

### 创建应用程序

要创建 App Engine 应用程序，请转到项目根目录并运行:

```py
`$ gcloud app create

You are creating an app for project [indigo-griffin-376011].
WARNING: Creating an App Engine application for a project is irreversible and the region
cannot be changed.

Please choose the region where you want your App Engine application located:

 ...
 [13] europe-west3  (supports standard and flexible and search_api)
 [14] europe-west6  (supports standard and flexible and search_api)
 [15] northamerica-northeast1 (supports standard and flexible and search_api)
 [16] southamerica-east1 (supports standard and flexible and search_api)
 [17] us-central    (supports standard and flexible and search_api)
 [18] us-east1      (supports standard and flexible and search_api)
 ...
 [24] cancel
Please enter your numeric choice:  13

Creating App Engine application in project [indigo-griffin-376011] and region [europe-west3]....done.
Success! The app is now created. Please use `gcloud app deploy` to deploy your first app.` 
```

同样，选择离你最近的地区。

### 数据库ˌ资料库

#### 规定

导航到[云 SQL 仪表板](https://console.cloud.google.com/sql/)并使用以下参数创建一个新的 Postgres 实例:

*   实例 ID:**mydb-实例**
*   密码:**输入自定义密码或生成密码**
*   数据库版本: **PostgreSQL 14**
*   配置:**由你决定**
*   地区:**与您的应用相同的地区**
*   区域可用性:**由您决定**

> 您可能还需要启用“计算引擎 API”来创建 SQL 实例。

设置数据库需要几分钟时间。同时，通过搜索“云 SQL 管理 API”并点击“启用”，继续启用[云 SQL 管理 API](https://console.cloud.google.com/marketplace/product/google/sqladmin.googleapis.com) 。我们需要启用它来测试数据库连接。

一旦提供了数据库，您应该会被重定向到数据库详细信息。记下“连接名称”:

![SQL Connection Name](img/908026a618bcdd040b15ac9738f0ac89.png)

接下来，选择侧边栏上的“Databases”并创建一个新的数据库。

最后，选择侧边栏上的“Users”并创建一个新用户。生成一个密码并记下它。

就是这样。数据库现在已经准备好了！

#### 云 SQL 代理

为了测试数据库连接和迁移数据库，我们将使用[云 SQL 身份验证代理](https://cloud.google.com/sql/docs/postgres/connect-instance-auth-proxy)。云 SQL 身份验证代理提供对云 SQL 实例的安全访问，无需授权网络或配置 SSL。

首先，验证并获取 API 的凭证:

```py
`$ gcloud auth application-default login` 
```

接下来，下载云 SQL 身份验证代理并使其可执行:

```py
`$ wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
$ chmod +x cloud_sql_proxy` 
```

> 如果您不在 Linux 上，请遵循[安装指南](https://cloud.google.com/sql/docs/postgres/connect-instance-auth-proxy#install-proxy)来安装云 SQL 代理。

安装完成后，打开一个新的终端窗口，使用您的连接详细信息启动代理，如下所示:

```py
`$ ./cloud_sql_proxy.exe -instances="PROJECT_ID:REGION:INSTANCE_NAME"=tcp:5432

# Example:
# cloud_sql_proxy.exe -instances="indigo-35:europe-west3:mydb-instance"=tcp:5432

2023/01/30 13:45:22 Listening on 127.0.0.1:5432 for indigo-35:europe-west3:mydb-instance
2023/01/30 13:45:22 Ready for new connections
2023/01/30 13:45:22 Generated RSA key in 110.0168ms` 
```

现在您可以像在本地机器上运行 Postgres 一样连接到`localhost:5432`。

#### 迁移数据库

因为 GAE 不允许我们在服务器上执行命令，所以我们必须从本地机器上迁移数据库。

如果您还没有安装，请继续安装这些要求:

```py
`(venv)$ pip install -r requirements.txt` 
```

接下来，创建一个*。项目根目录中的 env* 文件，带有所需的环境变量:

```py
`# .env DEBUG=1 SECRET_KEY=+an@of0zh--q%vypb^9x@vgecoda5o!m!l9sqno)vz^n!euncl DATABASE_URL=postgres://DB_USER:DB_PASS@localhost/DB_NAME # Example `DATABASE_URL`: # DATABASE_URL=postgres://django-images:[[email protected]](/cdn-cgi/l/email-protection)/mydb` 
```

> 确保用您的实际凭证替换`DB_USER`、`DB_PASS`和`DB_NAME`。

最后，迁移数据库:

```py
`(venv)$ python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, images, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  ...
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying images.0001_initial... OK
  Applying sessions.0001_initial... OK` 
```

#### 创建超级用户

要创建超级用户，请运行:

```py
`(venv)$ python manage.py createsuperuser` 
```

并按照提示进行操作。

### 秘密经理

为了安全地管理我们的秘密和环境文件，我们将使用 [Secret Manager](https://console.cloud.google.com/security/secret-manager) 。

导航到[秘密管理器仪表板](https://console.cloud.google.com/security/secret-manager)并启用 API，如果你还没有的话。接下来，创建一个名为`django_settings`的秘密，内容如下:

```py
`DEBUG=1 SECRET_KEY=+an@of0zh--q%vypb^9x@vgecoda5o!m!l9sqno)vz^n!euncl DATABASE_URL=postgres://DB_USER:DB_PASS@//cloudsql/PROJECT_ID:REGION:INSTANCE_NAME/DB_NAME GS_BUCKET_NAME=django-images-bucket # Example `DATABASE_URL`: # postgres://django-images:[[email protected]](/cdn-cgi/l/email-protection)//cloudsql/indigo-35:europe-west3:mydb-instance/mydb` 
```

> 确保相应地更改`DATABASE_URL`。`PROJECT_ID:REGION:INSTANCE_NAME`等于您的数据库连接细节。
> 
> 你不用担心`GS_BUCKET_NAME`。这只是我们稍后要创建和使用的一个存储桶的名称。

回到您的项目，将以下内容添加到 *requirements.txt* :

```py
`google-cloud-secret-manager==2.15.1` 
```

要从 Secret Manager 加载环境变量，我们可以使用下面的[官方代码片段](https://cloud.google.com/python/django/appengine#understanding-secrets):

```py
`# core/settings.py

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

env = environ.Env(DEBUG=(bool, False))
env_file = os.path.join(BASE_DIR, '.env')

if os.path.isfile(env_file):
    # read a local .env file
    env.read_env(env_file)
elif os.environ.get('GOOGLE_CLOUD_PROJECT', None):
    # pull .env file from Secret Manager
    project_id = os.environ.get('GOOGLE_CLOUD_PROJECT')

    client = secretmanager.SecretManagerServiceClient()
    settings_name = os.environ.get('SETTINGS_NAME', 'django_settings')
    name = f'projects/{project_id}/secrets/{settings_name}/versions/latest'
    payload = client.access_secret_version(name=name).payload.data.decode('UTF-8')

    env.read_env(io.StringIO(payload))
else:
    raise Exception('No local .env or GOOGLE_CLOUD_PROJECT detected. No secrets found.')` 
```

不要忘记导入文件顶部的`io`和`secretmanager`:

```py
`import io
from google.cloud import secretmanager` 
```

太好了！终于到了部署我们应用的时候了。为此，请运行:

```py
`$ gcloud app deploy

Services to deploy:

descriptor:                  [C:\Users\Nik\PycharmProjects\django-images-new\app.yaml]
source:                      [C:\Users\Nik\PycharmProjects\django-images-new]
target project:              [indigo-griffin-376011]
target service:              [default]
target version:              [20230130t135926]
target url:                  [https://indigo-griffin-376011.ey.r.appspot.com]

Do you want to continue (Y/n)?  y

Beginning deployment of service [default]...
#============================================================#
#= Uploading 21 files to Google Cloud Storage               =#
#============================================================#
File upload done.
Updating service [default]...done.
Setting traffic split for service [default]...done.
Deployed service [default] to [https://indigo-griffin-376011.ey.r.appspot.com]

You can stream logs from the command line by running:
  $ gcloud app logs tail -s default` 
```

在浏览器中打开您的 web 应用程序，并测试它是否正常工作:

> 如果你得到一个`502 Bad Gateway`错误，你可以导航到[日志浏览器](https://console.cloud.google.com/logs/)来查看你的日志。
> 
> 如果有一个`403 Permission 'secretmanager.versions.access' denied`错误，导航到`django_settings` secret permissions 并确保默认的 App Engine 服务帐户可以访问此机密。参见[栈顶溢出](https://stackoverflow.com/a/67331344/13058252)的解决方案。

如果您尝试上传图像，您应该会看到以下错误:

```py
`[Errno 30] Read-only file system: '/workspace/mediafiles'` 
```

这是因为 GAE 文件是只读的。别担心。我们将在下一节中解决它。

## 持久存储

Google App Engine(以及许多其他类似的服务，如 Heroku)提供了一个短暂的文件系统。这意味着您的数据不是持久的，可能会在应用程序关闭或重新部署时消失。此外，GAE 文件是只读的，这使得您无法将媒体文件直接上传到 GAE。

正因为如此，我们将使用[云存储](https://cloud.google.com/storage)来设置持久存储。

### 服务帐户

要使用云存储，我们首先需要创建一个专用的服务帐户，该帐户具有足够的权限来读/写和签署云存储中的文件。

导航至[服务账户仪表板](https://console.cloud.google.com/iam-admin/serviceaccounts)并点击“创建服务账户”。将其命名为“django-images-bucket ”,并将其他内容保留为默认值。提交表单后，您应该会在表格中看到一个新的服务帐户。记下您的新服务帐户的电子邮件。

接下来，单击您的服务帐户旁边的三个点，然后单击“管理详细信息”:

![Service Account Details](img/f27b8075390d2806758ed7acee81070a.png)

在导航中选择“Keys”和“Create a new key”。将其导出为 JSON。

![Create New Service Account Key](img/e0dfd442bcd8d1688f3b9f3ad85d6c08.png)

一旦 JSON 密匙被下载，给它一个更易读的名字，比如 *gcpCredentials.json* ，并把它放在您的项目根目录中。

> 确保将该文件添加到您的*中。gitignore* 以防止您的服务帐户凭证意外泄露给版本控制。

### 水桶

导航到您的[云存储桶](https://console.cloud.google.com/storage/)，并使用以下详细信息创建一个新桶:

*   名称:**姜戈-图像-桶**
*   位置类型:**地区**
*   位置:**与您的应用程序相同的地区**

将其他内容保留为默认值，然后单击“创建”。

如果您收到一条消息说“公共访问将被阻止”，请取消选中“在此桶上实施公共访问阻止”和“确认”。我们必须允许公众访问，因为我们正在部署一个图像托管网站，并希望上传的图像可以被每个人访问。

> 有关更多信息，请查看[公共访问防护](https://cloud.google.com/storage/docs/public-access-prevention)。

要授予我们的新服务帐户对存储桶的权限，请查看您的存储桶详细信息，然后在导航中选择“权限”。之后，点击“授权访问”:

![Bucket Permissions](img/277a2e4f6bf6c4c980f338ce2a23057c.png)

添加新主体:

*   电子邮件:**您的服务帐户电子邮件**
*   角色:**云存储>存储管理员**

太好了！我们现在有了一个存储桶和一个可以使用该存储桶的服务帐户。

### 配置 Django

为了利用 Django 的云存储，我们将使用一个名为 [django-storages](https://django-storages.readthedocs.io/) 的第三方包。

在本地安装软件包，并将下面两行添加到 *requirements.txt* :

```py
`django-storages[google]==1.13.2 google-cloud-storage==2.7.0` 
```

接下来，配置 django-storages 以使用云存储和您的服务帐户:

```py
`# core/settings.py

GS_CREDENTIALS = service_account.Credentials.from_service_account_file(
    os.path.join(BASE_DIR, 'gcpCredentials.json')
)

DEFAULT_FILE_STORAGE = 'storages.backends.gcloud.GoogleCloudStorage'
STATICFILES_STORAGE = 'storages.backends.gcloud.GoogleCloudStorage'
GS_BUCKET_NAME = env('GS_BUCKET_NAME')` 
```

不要忘记重要的一点:

```py
`from google.oauth2 import service_account` 
```

使用 *app.yaml* 中的处理程序使 App Engine 服务于静态和媒体文件:

```py
`# app.yaml handlers: -  url:  /static  # new static_dir:  staticfiles/  # new -  url:  /media  # new static_dir:  mediafiles/  # new -  url:  /.* script:  auto` 
```

确保它们被放置在`/.*`之前。

> 如果你正在部署自己的应用程序，确保你的`STATIC_URL`、`STATIC_ROOT`、`MEDIA_URL`和`MEDIA_ROOT`设置正确([示例](https://github.com/duplxey/django-images/blob/889062793da0a7ccca7f88836f22244d574fb368/core/settings.py#L130))。

收集静态文件到 GAE:

```py
`(venv)$ python manage.py collectstatic

You have requested to collect static files at the destination
location as specified in your settings.

This will overwrite existing files!
Are you sure you want to do this?

Type 'yes' to continue, or 'no' to cancel: yes

141 static files copied.` 
```

重新部署你的应用程序:

在浏览器中打开应用程序:

导航到`/admin`并确保静态文件已被收集和加载。

## 自定义域

要将自定义域链接到您的 web 应用，首先导航到[应用引擎仪表板](https://console.cloud.google.com/appengine)。在边栏中选择“设置”，然后选择“自定义域”。最后，单击“添加自定义域”。

选择您想要使用的域，或者添加并验证新域。如果您添加一个新域，请确保输入裸域名。示例:

```py
`testdriven.io       <-- good
app.testdriven.io   <-- bad` 
```

单击“继续”,然后输入要映射到你的 GAE 应用的域名和子域名。然后单击“保存映射”。

接下来，转到您的域名注册商的 DNS 设置，添加一个指向`ghs.googlehosted.com`的新“CNAME 记录”,如下所示:

```py
`+----------+--------------+----------------------------+-----------+ | Type     | Host         | Value                      | TTL       |
+----------+--------------+----------------------------+-----------+ | CNAME    | <some host> | ghs.googlehosted.co        | Automatic |
+----------+--------------+----------------------------+-----------+` 
```

示例:

```py
`+----------+--------------+----------------------------+-----------+ | Type     | Host         | Value                      | TTL       |
+----------+--------------+----------------------------+-----------+ | CNAME    | app          | ghs.googlehosted.com       | Automatic |
+----------+--------------+----------------------------+-----------+` 
```

返回到您的自定义域设置，然后单击“完成”。

您已成功添加了自定义域名。等待 DNS 更改传播，等待 Google 发布 SSL 证书。要耐心...有时需要 24 小时。

在您验证您的应用程序可通过 HTTPS 访问后，将以下内容添加到 *app.yaml* 的末尾:

```py
`# app.yaml env_variables: APPENGINE_URL:  <the domain you mapped> # Example: # env_variables: #   APPENGINE_URL: app.testdriven.io` 
```

这添加了一个新的环境变量，由 *settings.py* 用来配置`ALLOWED_HOSTS`、`CSRF_TRUSTED_ORIGINS`，并强制 SSL 重定向。

最后，再次重新部署您的应用:

## 结论

在本教程中，我们已经成功地将 Django 应用程序部署到 Google App Engine。我们已经处理了 Postgres 数据库、静态和媒体文件，添加了自定义域名，并启用了 HTTPS。现在，您应该能够将自己的应用程序部署到 App Engine。

从 [django-app-engine](https://github.com/duplxey/django-app-engine) repo 中获取最终的源代码。

> 如果您希望删除我们在整个教程中创建的所有服务和资源，请查看文档中的[清理](https://cloud.google.com/appengine/docs/standard/python3/building-app/cleaning-up)资源。

### 未来的步骤

1.  设置[秘密管理器](https://console.cloud.google.com/security/secret-manager)中的`DEBUG=0`禁用调试模式。
2.  看看[如何管理实例](https://cloud.google.com/appengine/docs/legacy/standard/python/how-instances-are-managed)以更好地理解 GAE 伸缩。
3.  了解如何使用[日志浏览器](https://cloud.google.com/logging/docs/view/logs-explorer-interface)。