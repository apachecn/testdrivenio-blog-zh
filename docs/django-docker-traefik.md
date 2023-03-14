# 将 Django 与 Postgres、Gunicorn 和 Traefik 一起归档

> 原文：<https://testdriven.io/blog/django-docker-traefik/>

在本教程中，我们将看看如何用 Postgres 和 Docker 设置 Django。对于生产环境，我们将添加 Gunicorn、Traefik，并进行加密。

## 项目设置

首先创建一个项目目录:

```py
`$ mkdir django-docker-traefik && cd django-docker-traefik
$ mkdir app && cd app
$ python3.11 -m venv venv
$ source venv/bin/activate` 
```

> 你可以随意把 virtualenv 和 Pip 换成诗歌[或](https://python-poetry.org) [Pipenv](https://github.com/pypa/pipenv) 。更多信息，请查看[现代 Python 环境](/blog/python-environments/)。

接下来，让我们安装 Django 并创建一个简单的 Django 应用程序:

```py
`(venv)$ pip install django==4.1.6
(venv)$ django-admin startproject config .
(venv)$ python manage.py migrate` 
```

运行应用程序:

```py
`(venv)$ python manage.py runserver` 
```

导航到 [http://localhost:8000/](http://localhost:8000/) 查看 Django 欢迎屏幕。完成后，关闭服务器并退出虚拟环境。也删除虚拟环境。我们现在有了一个简单的 Django 项目。

在“app”目录下创建一个 *requirements.txt* 文件，并添加 Django 作为依赖项:

因为我们将转移到 Postgres，所以继续从“app”目录中删除 *db.sqlite3* 文件。

您的项目目录应该如下所示:

```py
`└── app
    ├── config
    │   ├── __init__.py
    │   ├── asgi.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    └── requirements.txt` 
```

## 码头工人

安装 [Docker](https://docs.docker.com/install/) ，如果你还没有，那么在“app”目录下添加一个 *Dockerfile* :

```py
`# app/Dockerfile

# pull the official docker image
FROM  python:3.11.2-slim

# set work directory
WORKDIR  /app

# set env variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

# install dependencies
COPY  requirements.txt .
RUN  pip install -r requirements.txt

# copy project
COPY  . .` 
```

所以，我们从 Python 3.11.2 的`slim` [Docker 镜像](https://hub.docker.com/_/python/)开始。然后我们设置了一个[工作目录](https://docs.docker.com/engine/reference/builder/#workdir)以及两个环境变量:

1.  `PYTHONDONTWRITEBYTECODE`:防止 Python 将 pyc 文件写入磁盘(相当于`python -B` [选项](https://docs.python.org/3/using/cmdline.html#cmdoption-B)
2.  `PYTHONUNBUFFERED`:防止 Python 缓冲 stdout 和 stderr(相当于`python -u` [选项](https://docs.python.org/3/using/cmdline.html#cmdoption-u)

最后，我们复制了 *requirements.txt* 文件，安装了依赖项，并复制了项目。

> 查看 [Docker 针对 Python 开发人员的最佳实践](/blog/docker-best-practices/)，了解更多关于构造 Docker 文件的信息，以及为基于 Python 的开发配置 Docker 的一些最佳实践。

接下来，将一个 *docker-compose.yml* 文件添加到项目根:

```py
`# docker-compose.yml version:  '3.8' services: web: build:  ./app command:  python manage.py runserver 0.0.0.0:8000 volumes: -  ./app:/app ports: -  8008:8000 environment: -  DEBUG=1` 
```

> 查看[合成文件参考](https://docs.docker.com/compose/compose-file/)，了解该文件如何工作的信息。

建立形象:

构建映像后，运行容器:

导航到 [http://localhost:8008](http://localhost:8008) 再次查看欢迎页面。

> 如果这不起作用，通过`docker-compose logs -f`检查日志中的错误。

## Postgres

要配置 Postgres，我们需要向 *docker-compose.yml* 文件添加一个新服务，更新 Django 设置，并安装 [Psycopg2](https://www.psycopg.org/) 。

首先，向 *docker-compose.yml* 添加一个名为`db`的新服务:

```py
`# docker-compose.yml version:  '3.8' services: web: build:  ./app command:  bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; python manage.py runserver 0.0.0.0:8000' volumes: -  ./app:/app ports: -  8008:8000 environment: -  DEBUG=1 -  DATABASE_URL=postgresql://django_traefik:[[email protected]](/cdn-cgi/l/email-protection):5432/django_traefik depends_on: -  db db: image:  postgres:15-alpine volumes: -  postgres_data:/var/lib/postgresql/data/ expose: -  5432 environment: -  POSTGRES_USER=django_traefik -  POSTGRES_PASSWORD=django_traefik -  POSTGRES_DB=django_traefik volumes: postgres_data:` 
```

为了在容器的生命周期之外保存数据，我们配置了一个卷。这个配置将把`postgres_data`绑定到容器中的“/var/lib/postgresql/data/”目录。

我们还添加了一个环境键来定义默认数据库的名称，并设置用户名和密码。

> 查看 [Postgres Docker Hub 页面](https://hub.docker.com/_/postgres)的“环境变量”部分了解更多信息。

注意`web`服务中的新命令:

```py
`bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; python manage.py runserver 0.0.0.0:8000'` 
```

将持续到 Postgres 完成。一旦启动，`python manage.py runserver 0.0.0.0:8000`就会运行。

要配置 Postgres，添加 [django-environ](https://django-environ.readthedocs.io/en/latest/) ，加载/读取环境变量，添加 [Psycopg2](https://github.com/psycopg/psycopg2) 到 *requirements.txt* :

```py
`Django==4.1.6
django-environ==0.9.0
psycopg2-binary==2.9.5` 
```

初始化 *config/settings.py* 顶部的 environ:

```py
`# config/settings.py

import environ

env = environ.Env()` 
```

然后，更新`DATABASES`字典:

```py
`# config/settings.py

DATABASES = {
    'default': env.db(),
}` 
```

django-environ 将自动解析我们添加到 *docker-compose.yml* 中的数据库连接 URL 字符串:

```py
`DATABASE_URL=postgresql://django_traefik:django_traefik@db:5432/django_traefik` 
```

同样更新`DEBUG`变量:

```py
`# config/settings.py

DEBUG = env('DEBUG')` 
```

构建新的映像并旋转两个容器:

```py
`$ docker-compose up -d --build` 
```

运行初始迁移:

```py
`$ docker-compose exec web python manage.py migrate --noinput` 
```

确保创建了默认的 Django 表:

```py
`$ docker-compose exec db psql --username=django_traefik --dbname=django_traefik

psql (15.2)
Type "help" for help.

django_traefik=# \l
                                            List of databases
      Name      |     Owner      | Encoding |  Collate   |   Ctype    |         Access privileges
----------------+----------------+----------+------------+------------+-----------------------------------
 django_traefik | django_traefik | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres       | django_traefik | UTF8     | en_US.utf8 | en_US.utf8 |
 template0      | django_traefik | UTF8     | en_US.utf8 | en_US.utf8 | =c/django_traefik                +
                |                |          |            |            | django_traefik=CTc/django_traefik
 template1      | django_traefik | UTF8     | en_US.utf8 | en_US.utf8 | =c/django_traefik                +
                |                |          |            |            | django_traefik=CTc/django_traefik
(4 rows)

django_traefik=# \c django_traefik
You are now connected to database "django_traefik" as user "django_traefik".

django_traefik=# \dt
                      List of relations
 Schema |            Name            | Type  |     Owner
--------+----------------------------+-------+----------------
 public | auth_group                 | table | django_traefik
 public | auth_group_permissions     | table | django_traefik
 public | auth_permission            | table | django_traefik
 public | auth_user                  | table | django_traefik
 public | auth_user_groups           | table | django_traefik
 public | auth_user_user_permissions | table | django_traefik
 public | django_admin_log           | table | django_traefik
 public | django_content_type        | table | django_traefik
 public | django_migrations          | table | django_traefik
 public | django_session             | table | django_traefik
(10 rows)

django_traefik=# \q` 
```

您也可以通过运行以下命令来检查该卷是否已创建:

```py
`$ docker volume inspect django-docker-traefik_postgres_data` 
```

您应该会看到类似如下的内容:

```py
`[
    {
        "CreatedAt": "2023-02-11T18:01:42Z",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "django-docker-traefik",
            "com.docker.compose.version": "2.12.2",
            "com.docker.compose.volume": "postgres_data"
        },
        "Mountpoint": "/var/lib/docker/volumes/django-docker-traefik_postgres_data/_data",
        "Name": "django-docker-traefik_postgres_data",
        "Options": null,
        "Scope": "local"
    }
]` 
```

## 格尼科恩

接下来，对于生产环境，让我们将 [Gunicorn](https://gunicorn.org/) ，一个生产级的 WSGI 服务器，添加到需求文件中:

```py
`Django==4.1.6
django-environ==0.9.0
psycopg2-binary==2.9.5
gunicorn==20.1.0` 
```

由于我们仍然希望在开发中使用 Django 的内置服务器，因此为生产创建一个名为 *docker-compose.prod.yml* 的新合成文件:

```py
`# docker-compose.prod.yml version:  '3.8' services: web: build:  ./app command:  bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; gunicorn --bind 0.0.0.0:8000 config.wsgi' ports: -  8008:8000 environment: -  DEBUG=0 -  DATABASE_URL=postgresql://django_traefik:[[email protected]](/cdn-cgi/l/email-protection):5432/django_traefik depends_on: -  db db: image:  postgres:15-alpine volumes: -  postgres_data_prod:/var/lib/postgresql/data/ expose: -  5432 environment: -  POSTGRES_USER=django_traefik -  POSTGRES_PASSWORD=django_traefik -  POSTGRES_DB=django_traefik volumes: postgres_data_prod:` 
```

> 如果您有多个环境，您可能希望使用一个[docker-compose . override . yml](https://docs.docker.com/compose/extends/)配置文件。使用这种方法，您可以将基本配置添加到 docker-compose.yml 文件中，然后使用 docker-compose.override.yml 文件根据环境覆盖这些配置设置。

记下默认值`command`。我们运行的是 Gunicorn，而不是 Django 开发服务器。我们还从`web`服务中删除了这个卷，因为我们在生产中不需要它。

将[下放到](https://docs.docker.com/compose/reference/down/)开发容器(以及带有`-v`标志的相关卷):

然后，构建生产映像并启动容器:

```py
`$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

运行迁移:

```py
`$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput` 
```

验证`django_traefik`数据库是和默认的 Django 表一起创建的。在[http://localhost:8008/admin](http://localhost:8008/admin)测试管理页面。静态文件加载不正确。这是意料之中的。我们会尽快解决这个问题。

> 同样，如果容器启动失败，通过`docker-compose -f docker-compose.prod.yml logs -f`检查日志中的错误。

## 生产文档

创建一个名为 *Dockerfile.prod* 的新 Dockerfile，用于生产构建:

```py
`# app/Dockerfile.prod

###########
# BUILDER #
###########

# pull official base image
FROM  python:3.11-slim  as  builder

# set work directory
WORKDIR  /app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

# install system dependencies
RUN  apt-get update && \
    apt-get install -y --no-install-recommends gcc

# lint
RUN  pip install --upgrade pip
RUN  pip install flake8==6.0.0
COPY  . .
RUN  flake8 --ignore=E501,F401 .

# install python dependencies
COPY  requirements.txt .
RUN  pip wheel --no-cache-dir --no-deps --wheel-dir /usr/src/app/wheels -r requirements.txt

#########
# FINAL #
#########

# pull official base image
FROM  python:3.11-slim

# create directory for the app user
RUN  mkdir -p /home/app

# create the app user
RUN  addgroup --system app && adduser --system --group app

# create the appropriate directories
ENV  HOME=/home/app
ENV  APP_HOME=/home/app/web
RUN  mkdir $APP_HOME
WORKDIR  $APP_HOME

# install dependencies
COPY  --from=builder /usr/src/app/wheels /wheels
COPY  --from=builder /app/requirements.txt .
RUN  pip install --upgrade pip
RUN  pip install --no-cache /wheels/*

# copy project
COPY  . $APP_HOME

# chown all the files to the app user
RUN  chown -R app:app $APP_HOME

# change to the app user
USER  app` 
```

在这里，我们使用了一个 Docker [多阶段构建](https://docs.docker.com/develop/develop-images/multistage-build/)来缩小最终的图像尺寸。本质上，`builder`是一个用于构建 Python 轮子的临时图像。然后车轮被复制到最终产品图像中，而`builder`图像被丢弃。

> 您可以将[多阶段构建方法](https://stackoverflow.com/a/53101932/1799408)更进一步，使用单个 docker 文件，而不是创建两个 docker 文件。思考在两个不同的文件上使用这种方法的利弊。

您是否注意到我们创建了一个非 root 用户？默认情况下，Docker 在容器内部以 root 用户身份运行容器进程。这是一种不好的做法，因为如果攻击者设法突破容器，他们可以获得 Docker 主机的根用户访问权限。如果您是容器中的 root 用户，那么您将是主机上的 root 用户。

更新 *docker-compose.prod.yml* 文件中的`web`服务，用 *Dockerfile.prod* 构建:

```py
`web: build: context:  ./app dockerfile:  Dockerfile.prod command:  bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; gunicorn --bind 0.0.0.0:8000 config.wsgi' ports: -  8008:8000 environment: -  DEBUG=0 -  DATABASE_URL=postgresql://django_traefik:[[email protected]](/cdn-cgi/l/email-protection):5432/django_traefik depends_on: -  db` 
```

现在，让我们重建生产映像并启动容器:

```py
`$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput` 
```

## Traefik

接下来，让我们添加 [Traefik](https://traefik.io/traefik/) ，一个[反向代理](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/)。

> 刚接触 Traefik？查看官方[入门](https://doc.traefik.io/traefik/getting-started/concepts/)指南。
> 
> **Traefik vs Nginx** : Traefik 是一个现代的、HTTP 反向代理和负载平衡器。它经常被比作 [Nginx](https://www.nginx.com) ，一个网络服务器和反向代理。由于 Nginx 主要是一个网络服务器，它可以用来提供网页，也可以作为一个反向代理和负载平衡器。总的来说，Traefik 的启动和运行更简单，而 Nginx 的功能更丰富。
> 
> **Traefik** :
> 
> 1.  反向代理和负载平衡器
> 2.  通过开箱即用的[让我们加密](https://letsencrypt.org/)，自动发布和更新 SSL 证书
> 3.  将 Traefik 用于简单的、基于 Docker 的微服务
> 
> **Nginx** :
> 
> 1.  Web 服务器、反向代理和负载平衡器
> 2.  比 trafik 稍快
> 3.  对复杂的服务使用 Nginx

添加一个名为 *traefik.dev.toml* 的新文件:

```py
`# traefik.dev.toml # listen on port 80 [entryPoints] [entryPoints.web] address  =  ":80" # Traefik dashboard over http [api] insecure  =  true [log] level  =  "DEBUG" [accessLog] # containers are not discovered automatically [providers] [providers.docker] exposedByDefault  =  false` 
```

在这里，由于我们不想公开`db`服务，我们将 [exposedByDefault](https://doc.traefik.io/traefik/providers/docker/#exposedbydefault) 设置为`false`。要手动公开服务，我们可以将`"traefik.enable=true"`标签添加到 Docker 组合文件中。

接下来，更新 *docker-compose.yml* 文件，以便 Traefik 发现我们的`web`服务并添加一个新的`traefik`服务:

```py
`# docker-compose.yml version:  '3.8' services: web: build:  ./app command:  bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; python manage.py runserver 0.0.0.0:8000' volumes: -  ./app:/app expose:  # new -  8000 environment: -  DEBUG=1 -  DATABASE_URL=postgresql://django_traefik:[[email protected]](/cdn-cgi/l/email-protection):5432/django_traefik depends_on: -  db labels:  # new -  "traefik.enable=true" -  "traefik.http.routers.django.rule=Host(`django.localhost`)" db: image:  postgres:15-alpine volumes: -  postgres_data:/var/lib/postgresql/data/ expose: -  5432 environment: -  POSTGRES_USER=django_traefik -  POSTGRES_PASSWORD=django_traefik -  POSTGRES_DB=django_traefik traefik:  # new image:  traefik:v2.9.6 ports: -  8008:80 -  8081:8080 volumes: -  "$PWD/traefik.dev.toml:/etc/traefik/traefik.toml" -  "/var/run/docker.sock:/var/run/docker.sock:ro" volumes: postgres_data:` 
```

首先，`web`服务只对端口`8000`上的其他容器公开。我们还为`web`服务添加了以下标签:

1.  `traefik.enable=true`使 Traefik 能够发现服务
2.  `traefik.http.routers.django.rule=Host(`django.localhost`)`当请求有`Host=django.localhost`时，请求被重定向到该服务

记下`traefik`服务中的卷:

1.  将本地配置文件映射到容器中的配置文件，以便保持设置同步
2.  `"/var/run/docker.sock:/var/run/docker.sock:ro"`使 Traefik 能够发现其他容器

要进行测试，首先取下任何现有的容器:

```py
`$ docker-compose down -v
$ docker-compose -f docker-compose.prod.yml down -v` 
```

构建新的开发映像并启动容器:

```py
`$ docker-compose up -d --build` 
```

导航到[http://django.localhost:8008/](http://django.localhost:8008/)你应该看到 Django 的欢迎页面。

接下来，在 [django.localhost:8081](http://django.localhost:8081) 查看[仪表盘](https://doc.traefik.io/traefik/operations/dashboard/):

![traefik dashboard](img/c388e49cdf834296ca304b677e08a153.png)

完成后，将容器和体积拿下来:

## 让我们加密

我们已经成功地在开发模式中创建了 Django、Docker 和 Traefik 的工作示例。对于生产，您需要配置 Traefik 来通过 Let's Encrypt 管理 TLS 证书。简而言之，Traefik 将自动联系证书颁发机构来颁发和续订证书。

因为 Let's Encrypt 不会为`localhost`颁发证书，所以你需要在云计算实例(比如 [DigitalOcean](https://m.do.co/c/d8f211a4b4c2) droplet 或 AWS EC2 实例)上运行你的生产容器。您还需要一个有效的域名。如果你没有，你可以在 [Freenom](https://www.freenom.com/) 创建一个免费域名。

> 我们使用一个 [DigitalOcean](https://m.do.co/c/d8f211a4b4c2) droplet 为 Docker 提供一个计算实例，并部署生产容器来测试 Traefik 配置。

假设您配置了一个计算实例并设置了一个自由域，那么现在就可以在生产模式下设置 Traefik 了。

首先将 Traefik 配置的生产版本添加到名为 *traefik.prod.toml* 的文件中:

```py
`# traefik.prod.toml [entryPoints] [entryPoints.web] address  =  ":80" [entryPoints.web.http] [entryPoints.web.http.redirections] [entryPoints.web.http.redirections.entryPoint] to  =  "websecure" scheme  =  "https" [entryPoints.websecure] address  =  ":443" [accessLog] [api] dashboard  =  true [providers] [providers.docker] exposedByDefault  =  false [certificatesResolvers.letsencrypt.acme] email  =  "[[email protected]](/cdn-cgi/l/email-protection)" storage  =  "/certificates/acme.json" [certificatesResolvers.letsencrypt.acme.httpChallenge] entryPoint  =  "web"` 
```

> 确保用您的实际电子邮件地址替换`[[email protected]](/cdn-cgi/l/email-protection)`。

这里发生了什么:

1.  将我们不安全的 HTTP 应用程序的入口点设置为端口 80
2.  将我们的安全 HTTPS 应用程序的入口点设置为端口 443
3.  将所有不安全的请求重定向到安全端口
4.  `exposedByDefault = false`取消所有服务
5.  `dashboard = true`启用监控仪表板

最后，请注意:

```py
`[certificatesResolvers.letsencrypt.acme] email  =  "[[email protected]](/cdn-cgi/l/email-protection)" storage  =  "/certificates/acme.json" [certificatesResolvers.letsencrypt.acme.httpChallenge] entryPoint  =  "web"` 
```

这是让我们加密配置的地方。我们定义了证书的存储位置(T1)和验证类型(T3)，这是一个 HTTP 挑战(T5)。

接下来，假设您更新了域名的 DNS 记录，创建两个新的 A 记录，它们都指向您的计算实例的公共 IP:

1.  `django-traefik.your-domain.com` -用于网络服务
2.  `dashboard-django-traefik.your-domain.com` -用于 Traefik 仪表板

> 确保用您的实际域名替换`your-domain.com`。

接下来，像这样更新 *docker-compose.prod.yml* :

```py
`# docker-compose.prod.yml version:  '3.8' services: web: build: context:  ./app dockerfile:  Dockerfile.prod command:  bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; gunicorn --bind 0.0.0.0:8000 config.wsgi' expose:  # new -  8000 environment: -  DEBUG=0 -  DATABASE_URL=postgresql://django_traefik:[[email protected]](/cdn-cgi/l/email-protection):5432/django_traefik -  DJANGO_ALLOWED_HOSTS=.your-domain.com depends_on: -  db labels:  # new -  "traefik.enable=true" -  "traefik.http.routers.django.rule=Host(`django-traefik.your-domain.com`)" -  "traefik.http.routers.django.tls=true" -  "traefik.http.routers.django.tls.certresolver=letsencrypt" db: image:  postgres:15-alpine volumes: -  postgres_data_prod:/var/lib/postgresql/data/ expose: -  5432 environment: -  POSTGRES_USER=django_traefik -  POSTGRES_PASSWORD=django_traefik -  POSTGRES_DB=django_traefik traefik:  # new build: context:  . dockerfile:  Dockerfile.traefik ports: -  80:80 -  443:443 volumes: -  "/var/run/docker.sock:/var/run/docker.sock:ro" -  "./traefik-public-certificates:/certificates" labels: -  "traefik.enable=true" -  "traefik.http.routers.dashboard.rule=Host(`dashboard-django-traefik.your-domain.com`)" -  "traefik.http.routers.dashboard.tls=true" -  "traefik.http.routers.dashboard.tls.certresolver=letsencrypt" -  "[[email protected]](/cdn-cgi/l/email-protection)" -  "traefik.http.routers.dashboard.middlewares=auth" -  "traefik.http.middlewares.auth.basicauth.users=testuser:$$apr1$$jIKW.bdS$$eKXe4Lxjgy/rH65wP1iQe1" volumes: postgres_data_prod: traefik-public-certificates:` 
```

> 同样，确保用您的实际域名替换`your-domain.com`。

这里有什么新鲜事？

在`web`服务中，我们添加了以下标签:

1.  `traefik.http.routers.django.rule=Host(`django-traefik.your-domain.com`)`将主机更改为实际的域
2.  `traefik.http.routers.django.tls=true`启用 HTTPS
3.  `traefik.http.routers.django.tls.certresolver=letsencrypt`将证书颁发者设置为让我们加密

接下来，对于`traefik`服务，我们为证书目录添加了适当的端口和一个卷。该卷确保即使容器关闭，证书仍然有效。

至于标签:

1.  `traefik.http.routers.dashboard.rule=Host(`dashboard-django-traefik.your-domain.com`)`定义仪表板主机，因此可以在`$Host/dashboard/`访问
2.  `traefik.http.routers.dashboard.tls=true`启用 HTTPS
3.  `traefik.http.routers.dashboard.tls.certresolver=letsencrypt`将证书解析器设置为“让我们加密”
4.  `traefik.http.routers.dashboard.middlewares=auth`启用`HTTP BasicAuth`中间件
5.  `traefik.http.middlewares.auth.basicauth.users`定义用于登录的用户名和散列密码

您可以使用 htpasswd 实用程序创建新的密码哈希:

```py
`# username: testuser
# password: password

$ echo $(htpasswd -nb testuser password) | sed -e s/\\$/\\$\\$/g
testuser:$$apr1$$jIKW.bdS$$eKXe4Lxjgy/rH65wP1iQe1` 
```

随意使用一个`env_file`来存储用户名和密码作为环境变量

```py
`USERNAME=testuser
HASHED_PASSWORD=$$apr1$$jIKW.bdS$$eKXe4Lxjgy/rH65wP1iQe1` 
```

接下来，更新 *config/settings.py* 中的`ALLOWED_HOSTS`环境变量，如下所示:

```py
`# config/settings.py

ALLOWED_HOSTS = env('DJANGO_ALLOWED_HOSTS', default=[])` 
```

最后，添加一个名为 *Dockerfile.traefik* 的新 Dockerfile:

```py
`# Dockerfile.traefik

FROM  traefik:v2.9.6

COPY  ./traefik.prod.toml ./etc/traefik/traefik.toml` 
```

接下来，旋转新容器:

```py
`$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

确保这两个 URL 有效:

1.  [https://django-traefik.your-domain.com](https://django-traefik.your-domain.com)
2.  [https://dashboard-django-traefik.your-domain.com/dashboard/](https://dashboard-django-traefik.your-domain.com/dashboard/)

此外，请确保当您访问上述网址的 HTTP 版本时，您会被重定向到 HTTPS 版本。

最后，让我们加密有效期为 [90 天](https://letsencrypt.org/2015/11/09/why-90-days.html)的证书。Treafik 将在后台自动为您处理证书更新，这样您就少了一件担心的事情！

## 静态文件

由于 Traefik 不提供静态文件，我们将使用[whiten noise](http://whitenoise.evans.io)来管理静态资产。

首先，将包添加到 *requirements.txt* 文件中:

```py
`Django==4.1.6
django-environ==0.9.0
psycopg2-binary==2.9.5
gunicorn==20.1.0
whitenoise==6.3.0` 
```

像这样更新 *config/settings.py* 中的中间件:

```py
`# config/settings.py

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # new
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]` 
```

然后用`STATIC_ROOT`配置静态文件的处理:

```py
`# config/settings.py

STATIC_ROOT = BASE_DIR / 'staticfiles'` 
```

最后，添加压缩和缓存支持:

```py
`# config/settings.py

STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'` 
```

要进行测试，请更新图像和容器:

```py
`$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

收集静态文件:

```py
`$ docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic` 
```

确保静态文件在[https://django-traefik.your-domain.com/admin](https://django-traefik.your-domain.com/admin)被正确提供。

## 结论

在本教程中，我们介绍了如何使用 Postgres 将 Django 应用程序容器化以进行开发。我们还创建了一个生产就绪的 Docker 组合文件，设置了 Traefik 和 Let's Encrypt 来通过 HTTPS 为应用程序提供服务，并启用了一个安全的仪表板来监控我们的服务。

就生产环境的实际部署而言，您可能希望使用:

1.  完全托管的数据库服务——像 [RDS](https://aws.amazon.com/rds/) 或[云 SQL](https://cloud.google.com/sql/)——而不是在一个容器中管理你自己的 Postgres 实例。
2.  服务的非根用户

你可以在 django-docker-traefik 报告中找到代码。