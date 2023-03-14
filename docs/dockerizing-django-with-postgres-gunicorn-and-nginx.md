# 用 Postgres、Gunicorn 和 Nginx 编写 Django 文档

> 原文：<https://testdriven.io/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/>

这是一个循序渐进的教程，详细介绍了如何配置 Django，使其在带有 Postgres 的 Docker 上运行。对于生产环境，我们将添加 Nginx 和 Gunicorn。我们还将看看如何通过 Nginx 提供 Django 静态和媒体文件。

*依赖关系*:

1.  Django v3.2.6
2.  文档 v20.10.8
3.  python 3 . 9 . 6 版

> 姜戈码头系列:
> 
> 1.  [用 Postgres、Gunicorn、Nginx](/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/) (本文！)
> 2.  [用加密保护容器化的 Django 应用](/blog/django-lets-encrypt/)
> 3.  [用 Docker 将 Django 部署到 AWS，让我们加密](/blog/django-docker-https-aws/)

## 项目设置

创建一个新的项目目录和一个新的 Django 项目:

```py
`$ mkdir django-on-docker && cd django-on-docker
$ mkdir app && cd app
$ python3.9 -m venv env
$ source env/bin/activate
(env)$

(env)$ pip install django==3.2.6
(env)$ django-admin.py startproject hello_django .
(env)$ python manage.py migrate
(env)$ python manage.py runserver` 
```

> 你可以随意把 virtualenv 和 Pip 换成诗歌[或](https://python-poetry.org) [Pipenv](https://github.com/pypa/pipenv) 。更多信息，请查看[现代 Python 环境](/blog/python-environments/)。

导航到 [http://localhost:8000/](http://localhost:8000/) 查看 Django 欢迎屏幕。一旦完成就杀死服务器。然后，退出并删除虚拟环境。我们现在有了一个简单的 Django 项目。

在“app”目录下创建一个 *requirements.txt* 文件，并添加 Django 作为依赖项:

因为我们将转移到 Postgres，所以继续从“app”目录中删除 *db.sqlite3* 文件。

您的项目目录应该如下所示:

```py
`└── app
    ├── hello_django
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
`# pull official base image
FROM  python:3.9.6-alpine

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

# install dependencies
RUN  pip install --upgrade pip
COPY  ./requirements.txt .
RUN  pip install -r requirements.txt

# copy project
COPY  . .` 
```

所以，我们从 Python 3.9.6 的基于 [Alpine](https://github.com/gliderlabs/docker-alpine) 的 [Docker 镜像](https://hub.docker.com/_/python/)开始。然后我们设置一个[工作目录](https://docs.docker.com/engine/reference/builder/#workdir)以及两个环境变量:

1.  `PYTHONDONTWRITEBYTECODE`:防止 Python 将 pyc 文件写入磁盘(相当于`python -B` [选项](https://docs.python.org/3/using/cmdline.html#cmdoption-B)
2.  `PYTHONUNBUFFERED`:防止 Python 缓冲 stdout 和 stderr(相当于`python -u` [选项](https://docs.python.org/3/using/cmdline.html#cmdoption-u)

最后，我们更新了 Pip，复制了 *requirements.txt* 文件，安装了依赖项，并复制了 Django 项目本身。

> 查看[Docker for Python Developers](https://mherman.org/presentations/dockercon-2018)了解更多关于构造 Docker 文件的信息，以及为基于 Python 的开发配置 Docker 的一些最佳实践。

接下来，将一个 *docker-compose.yml* 文件添加到项目根:

```py
`version:  '3.8' services: web: build:  ./app command:  python manage.py runserver 0.0.0.0:8000 volumes: -  ./app/:/usr/src/app/ ports: -  8000:8000 env_file: -  ./.env.dev` 
```

> 查看[合成文件参考](https://docs.docker.com/compose/compose-file/)，了解该文件如何工作的信息。

更新 *settings.py* 中的`SECRET_KEY`、`DEBUG`和`ALLOWED_HOSTS`变量:

```py
`SECRET_KEY = os.environ.get("SECRET_KEY")

DEBUG = int(os.environ.get("DEBUG", default=0))

# 'DJANGO_ALLOWED_HOSTS' should be a single string of hosts with a space between each.
# For example: 'DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]'
ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS").split(" ")` 
```

确保将导入添加到顶部:

然后，在项目根目录下创建一个 *.env.dev* 文件来存储开发环境变量:

```py
`DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]` 
```

建立形象:

构建映像后，运行容器:

导航到 [http://localhost:8000/](http://localhost:8000/) 再次查看欢迎屏幕。

> 如果这不起作用，通过`docker-compose logs -f`检查日志中的错误。

## Postgres

要配置 Postgres，我们需要向 *docker-compose.yml* 文件添加一个新服务，更新 Django 设置，并安装 [Psycopg2](https://www.psycopg.org/) 。

首先，向 *docker-compose.yml* 添加一个名为`db`的新服务:

```py
`version:  '3.8' services: web: build:  ./app command:  python manage.py runserver 0.0.0.0:8000 volumes: -  ./app/:/usr/src/app/ ports: -  8000:8000 env_file: -  ./.env.dev depends_on: -  db db: image:  postgres:13.0-alpine volumes: -  postgres_data:/var/lib/postgresql/data/ environment: -  POSTGRES_USER=hello_django -  POSTGRES_PASSWORD=hello_django -  POSTGRES_DB=hello_django_dev volumes: postgres_data:` 
```

为了在容器的生命周期之外保存数据，我们配置了一个卷。这个配置将把`postgres_data`绑定到容器中的“/var/lib/postgresql/data/”目录。

我们还添加了一个环境键来定义默认数据库的名称，并设置用户名和密码。

> 查看 [Postgres Docker Hub 页面](https://hub.docker.com/_/postgres)的“环境变量”部分了解更多信息。

我们还需要为`web`服务添加一些新的环境变量，所以像这样更新 *.env.dev* :

```py
`DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_dev
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432` 
```

更新 *settings.py* 中的`DATABASES` dict:

```py
`DATABASES = {
    "default": {
        "ENGINE": os.environ.get("SQL_ENGINE", "django.db.backends.sqlite3"),
        "NAME": os.environ.get("SQL_DATABASE", BASE_DIR / "db.sqlite3"),
        "USER": os.environ.get("SQL_USER", "user"),
        "PASSWORD": os.environ.get("SQL_PASSWORD", "password"),
        "HOST": os.environ.get("SQL_HOST", "localhost"),
        "PORT": os.environ.get("SQL_PORT", "5432"),
    }
}` 
```

这里，数据库是基于我们刚刚定义的环境变量进行配置的。记下默认值。

更新 docker 文件以安装 Psycopg2 所需的相应软件包:

```py
`# pull official base image
FROM  python:3.9.6-alpine

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN  apk update \
    && apk add postgresql-dev gcc python3-dev musl-dev

# install dependencies
RUN  pip install --upgrade pip
COPY  ./requirements.txt .
RUN  pip install -r requirements.txt

# copy project
COPY  . .` 
```

将 Psycopg2 添加到 *requirements.txt* :

```py
`Django==3.2.6
psycopg2-binary==2.9.1` 
```

> 查看[本期 GitHub](https://github.com/psycopg/psycopg2/issues/684)了解更多关于在基于 Alpine 的 Docker 映像中安装 Psycopg2 的信息。

构建新的映像并旋转两个容器:

```py
`$ docker-compose up -d --build` 
```

运行迁移:

```py
`$ docker-compose exec web python manage.py migrate --noinput` 
```

> 得到以下错误？
> 
> ```py
> django.db.utils.OperationalError: FATAL:  database "hello_django_dev" does not exist 
> ```
> 
> 运行`docker-compose down -v`移除卷和容器。然后，重新构建映像，运行容器，并应用迁移。

确保创建了默认的 Django 表:

```py
`$ docker-compose exec db psql --username=hello_django --dbname=hello_django_dev

psql (13.0)
Type "help" for help.

hello_django_dev=# \l
                                          List of databases
       Name       |    Owner     | Encoding |  Collate   |   Ctype    |       Access privileges
------------------+--------------+----------+------------+------------+-------------------------------
 hello_django_dev | hello_django | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres         | hello_django | UTF8     | en_US.utf8 | en_US.utf8 |
 template0        | hello_django | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_django              +
                  |              |          |            |            | hello_django=CTc/hello_django
 template1        | hello_django | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_django              +
                  |              |          |            |            | hello_django=CTc/hello_django
(4 rows)

hello_django_dev=# \c hello_django_dev
You are now connected to database "hello_django_dev" as user "hello_django".

hello_django_dev=# \dt
                     List of relations
 Schema |            Name            | Type  |    Owner
--------+----------------------------+-------+--------------
 public | auth_group                 | table | hello_django
 public | auth_group_permissions     | table | hello_django
 public | auth_permission            | table | hello_django
 public | auth_user                  | table | hello_django
 public | auth_user_groups           | table | hello_django
 public | auth_user_user_permissions | table | hello_django
 public | django_admin_log           | table | hello_django
 public | django_content_type        | table | hello_django
 public | django_migrations          | table | hello_django
 public | django_session             | table | hello_django
(10 rows)

hello_django_dev=# \q` 
```

您也可以通过运行以下命令来检查该卷是否已创建:

```py
`$ docker volume inspect django-on-docker_postgres_data` 
```

您应该会看到类似如下的内容:

```py
`[
    {
        "CreatedAt": "2021-08-23T15:49:08Z",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "django-on-docker",
            "com.docker.compose.version": "1.29.2",
            "com.docker.compose.volume": "postgres_data"
        },
        "Mountpoint": "/var/lib/docker/volumes/django-on-docker_postgres_data/_data",
        "Name": "django-on-docker_postgres_data",
        "Options": null,
        "Scope": "local"
    }
]` 
```

接下来，在应用迁移并运行 Django 开发服务器之前，将 *entrypoint.sh* 文件添加到“app”目录中，以验证 Postgres 是否健康*:*

```py
`#!/bin/sh

if [ "$DATABASE" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

python manage.py flush --no-input
python manage.py migrate

exec "[[email protected]](/cdn-cgi/l/email-protection)"` 
```

在本地更新文件权限:

```py
`$ chmod +x app/entrypoint.sh` 
```

然后，更新 Docker 文件以复制覆盖 *entrypoint.sh* 文件，并将其作为 Docker [entrypoint](https://docs.docker.com/engine/reference/builder/#entrypoint) 命令运行:

```py
`# pull official base image
FROM  python:3.9.6-alpine

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN  apk update \
    && apk add postgresql-dev gcc python3-dev musl-dev

# install dependencies
RUN  pip install --upgrade pip
COPY  ./requirements.txt .
RUN  pip install -r requirements.txt

# copy entrypoint.sh
COPY  ./entrypoint.sh .
RUN  sed -i 's/\r$//g' /usr/src/app/entrypoint.sh
RUN  chmod +x /usr/src/app/entrypoint.sh

# copy project
COPY  . .

# run entrypoint.sh
ENTRYPOINT  ["/usr/src/app/entrypoint.sh"]` 
```

将`DATABASE`环境变量添加到 *.env.dev* :

```py
`DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_dev
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres` 
```

再次测试:

1.  重建图像
2.  运行容器
3.  试试 [http://localhost:8000/](http://localhost:8000/)

### 笔记

首先，尽管添加了 Postgres，只要`DATABASE`环境变量没有设置为`postgres`，我们仍然可以为 Django 创建一个独立的 Docker 映像。要进行测试，构建一个新的映像，然后运行一个新的容器:

```py
`$ docker build -f ./app/Dockerfile -t hello_django:latest ./app
$ docker run -d \
    -p 8006:8000 \
    -e "SECRET_KEY=please_change_me" -e "DEBUG=1" -e "DJANGO_ALLOWED_HOSTS=*" \
    hello_django python /usr/src/app/manage.py runserver 0.0.0.0:8000` 
```

您应该能够在 [http://localhost:8006](http://localhost:8006) 查看欢迎页面

其次，您可能希望注释掉 *entrypoint.sh* 脚本中的数据库刷新和迁移命令，这样它们就不会在每次容器启动或重新启动时运行:

```py
`#!/bin/sh

if [ "$DATABASE" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

# python manage.py flush --no-input
# python manage.py migrate

exec "[[email protected]](/cdn-cgi/l/email-protection)"` 
```

相反，您可以在容器旋转后手动运行它们，如下所示:

```py
`$ docker-compose exec web python manage.py flush --no-input
$ docker-compose exec web python manage.py migrate` 
```

## 格尼科恩

接下来，对于生产环境，让我们将 [Gunicorn](https://gunicorn.org/) ，一个生产级的 WSGI 服务器，添加到需求文件中:

```py
`Django==3.2.6
gunicorn==20.1.0
psycopg2-binary==2.9.1` 
```

> 对 WSGI 和 Gunicorn 很好奇？回顾来自[构建你自己的 Python Web 框架](https://testdriven.io/courses/python-web-framework/)课程的 [WSGI](https://testdriven.io/courses/python-web-framework/wsgi/) 章节。

由于我们仍然希望在开发中使用 Django 的内置服务器，因此为生产创建一个名为 *docker-compose.prod.yml* 的新合成文件:

```py
`version:  '3.8' services: web: build:  ./app command:  gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000 ports: -  8000:8000 env_file: -  ./.env.prod depends_on: -  db db: image:  postgres:13.0-alpine volumes: -  postgres_data:/var/lib/postgresql/data/ env_file: -  ./.env.prod.db volumes: postgres_data:` 
```

> 如果您有多个环境，您可能希望使用一个[docker-compose . override . yml](https://docs.docker.com/compose/extends/)配置文件。使用这种方法，您可以将您的基本配置添加到一个 *docker-compose.yml* 文件中，然后使用一个*docker-compose . override . yml*文件根据环境覆盖那些配置设置。

记下默认值`command`。我们运行的是 Gunicorn，而不是 Django 开发服务器。我们还从`web`服务中删除了这个卷，因为我们在生产中不需要它。最后，我们使用[单独的环境变量文件](https://docs.docker.com/compose/env-file/)来定义两个服务的环境变量，它们将在运行时传递给容器。

*.env.prod* :

```py
`DEBUG=0
SECRET_KEY=change_me
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_prod
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres` 
```

*.env.prod.db* :

```py
`POSTGRES_USER=hello_django
POSTGRES_PASSWORD=hello_django
POSTGRES_DB=hello_django_prod` 
```

将这两个文件添加到项目根目录。您可能想让它们不受版本控制，所以将它们添加到一个*中。gitignore* 文件。

将[下放到](https://docs.docker.com/compose/reference/down/)开发容器(以及带有`-v`标志的相关卷):

然后，构建生产映像并启动容器:

```py
`$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

验证`hello_django_prod`数据库是和默认的 Django 表一起创建的。在[http://localhost:8000/admin](http://localhost:8000/admin)测试管理页面。静态文件不再被加载。这是意料之中的，因为调试模式已关闭。我们会尽快解决这个问题。

> 同样，如果容器启动失败，通过`docker-compose -f docker-compose.prod.yml logs -f`检查日志中的错误。

## 生产文档

您是否注意到我们仍然在运行数据库 [flush](https://docs.djangoproject.com/en/2.2/ref/django-admin/#flush) (清空数据库)并在每次运行容器时迁移命令？这在开发中很好，但是让我们为生产创建一个新的入口点文件。

entry point . prod . sh:

```py
`#!/bin/sh

if [ "$DATABASE" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

exec "[[email protected]](/cdn-cgi/l/email-protection)"` 
```

在本地更新文件权限:

```py
`$ chmod +x app/entrypoint.prod.sh` 
```

要使用这个文件，创建一个名为 *Dockerfile.prod* 的新 Dockerfile，用于生产构建:

```py
`###########
# BUILDER #
###########

# pull official base image
FROM  python:3.9.6-alpine  as  builder

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN  apk update \
    && apk add postgresql-dev gcc python3-dev musl-dev

# lint
RUN  pip install --upgrade pip
RUN  pip install flake8==3.9.2
COPY  . .
RUN  flake8 --ignore=E501,F401 .

# install dependencies
COPY  ./requirements.txt .
RUN  pip wheel --no-cache-dir --no-deps --wheel-dir /usr/src/app/wheels -r requirements.txt

#########
# FINAL #
#########

# pull official base image
FROM  python:3.9.6-alpine

# create directory for the app user
RUN  mkdir -p /home/app

# create the app user
RUN  addgroup -S app && adduser -S app -G app

# create the appropriate directories
ENV  HOME=/home/app
ENV  APP_HOME=/home/app/web
RUN  mkdir $APP_HOME
WORKDIR  $APP_HOME

# install dependencies
RUN  apk update && apk add libpq
COPY  --from=builder /usr/src/app/wheels /wheels
COPY  --from=builder /usr/src/app/requirements.txt .
RUN  pip install --no-cache /wheels/*

# copy entrypoint.prod.sh
COPY  ./entrypoint.prod.sh .
RUN  sed -i 's/\r$//g'  $APP_HOME/entrypoint.prod.sh
RUN  chmod +x  $APP_HOME/entrypoint.prod.sh

# copy project
COPY  . $APP_HOME

# chown all the files to the app user
RUN  chown -R app:app $APP_HOME

# change to the app user
USER  app

# run entrypoint.prod.sh
ENTRYPOINT  ["/home/app/web/entrypoint.prod.sh"]` 
```

在这里，我们使用了一个 Docker [多阶段构建](https://docs.docker.com/develop/develop-images/multistage-build/)来缩小最终的图像尺寸。本质上，`builder`是一个用于构建 Python 轮子的临时图像。然后车轮被复制到最终产品图像中，而`builder`图像被丢弃。

> 你可以将[多阶段构建方法](https://stackoverflow.com/a/53101932/1799408)更进一步，使用单个*docker 文件*而不是创建两个 docker 文件。思考在两个不同的文件上使用这种方法的利弊。

您是否注意到我们创建了一个非 root 用户？默认情况下，Docker 在容器内部以 root 用户身份运行容器进程。这是一种不好的做法，因为如果攻击者设法突破容器，他们可以获得 Docker 主机的根用户访问权限。如果您是容器中的 root 用户，那么您将是主机上的 root 用户。

更新 *docker-compose.prod.yml* 文件中的`web`服务，用 *Dockerfile.prod* 构建:

```py
`web: build: context:  ./app dockerfile:  Dockerfile.prod command:  gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000 ports: -  8000:8000 env_file: -  ./.env.prod depends_on: -  db` 
```

尝试一下:

```py
`$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput` 
```

## Nginx

接下来，让我们将 Nginx 添加进来，充当 Gunicorn 的反向代理来处理客户端请求以及提供静态文件。

将服务添加到 *docker-compose.prod.yml* :

```py
`nginx: build:  ./nginx ports: -  1337:80 depends_on: -  web` 
```

然后，在本地项目根目录中，创建以下文件和文件夹:

```py
`└── nginx
    ├── Dockerfile
    └── nginx.conf` 
```

*Dockerfile* :

```py
`FROM  nginx:1.21-alpine

RUN  rm /etc/nginx/conf.d/default.conf
COPY  nginx.conf /etc/nginx/conf.d` 
```

*engine . conf*:

```py
`upstream hello_django {
    server web:8000;
}

server {

    listen 80;

    location / {
        proxy_pass http://hello_django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

}` 
```

> 查看[使用 NGINX 和 NGINX Plus 作为 uWSGI 和 Django 的应用网关](https://docs.nginx.com/nginx/admin-guide/web-server/app-gateway-uwsgi-django/),了解更多关于配置 NGINX 与 Django 协同工作的信息。

然后，更新`web`服务，在 *docker-compose.prod.yml* 中，用`expose`替换`ports`:

```py
`web: build: context:  ./app dockerfile:  Dockerfile.prod command:  gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000 expose: -  8000 env_file: -  ./.env.prod depends_on: -  db` 
```

现在，端口 8000 只在内部对其他 Docker 服务公开。该端口将不再发布到主机上。

> 关于端口与暴露的更多信息，请查看[这个](https://stackoverflow.com/questions/40801772/what-is-the-difference-between-docker-compose-ports-vs-expose)堆栈溢出问题。

再测试一次。

```py
`$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput` 
```

确保应用程序在 [http://localhost:1337](http://localhost:1337) 启动并运行。

您的项目结构现在应该看起来像这样:

```py
`├── .env.dev
├── .env.prod
├── .env.prod.db
├── .gitignore
├── app
│   ├── Dockerfile
│   ├── Dockerfile.prod
│   ├── entrypoint.prod.sh
│   ├── entrypoint.sh
│   ├── hello_django
│   │   ├── __init__.py
│   │   ├── asgi.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   ├── manage.py
│   └── requirements.txt
├── docker-compose.prod.yml
├── docker-compose.yml
└── nginx
    ├── Dockerfile
    └── nginx.conf` 
```

完成后将容器拿下来:

```py
`$ docker-compose -f docker-compose.prod.yml down -v` 
```

由于 Gunicorn 是一个应用服务器，它不会提供静态文件。那么，在这种特定的配置中，应该如何处理静态文件和媒体文件呢？

## 静态文件

更新 *settings.py* :

```py
`STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"` 
```

### 发展

现在，对`http://localhost:8000/static/*`的任何请求都将从“staticfiles”目录得到服务。

为了进行测试，首先重新构建映像，并像往常一样旋转新的容器。确保静态文件仍然在[http://localhost:8000/admin](http://localhost:8000/admin)上被正确地提供。

### 生产

对于生产，向 *docker-compose.prod.yml* 中的`web`和`nginx`服务添加一个卷，这样每个容器将共享一个名为“staticfiles”的目录:

```py
`version:  '3.8' services: web: build: context:  ./app dockerfile:  Dockerfile.prod command:  gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000 volumes: -  static_volume:/home/app/web/staticfiles expose: -  8000 env_file: -  ./.env.prod depends_on: -  db db: image:  postgres:13.0-alpine volumes: -  postgres_data:/var/lib/postgresql/data/ env_file: -  ./.env.prod.db nginx: build:  ./nginx volumes: -  static_volume:/home/app/web/staticfiles ports: -  1337:80 depends_on: -  web volumes: postgres_data: static_volume:` 
```

我们还需要在 *Dockerfile.prod* 中创建“/home/app/web/staticfiles”文件夹:

```py
`...

# create the appropriate directories
ENV  HOME=/home/app
ENV  APP_HOME=/home/app/web
RUN  mkdir $APP_HOME
RUN  mkdir $APP_HOME/staticfiles
WORKDIR  $APP_HOME

...` 
```

为什么这是必要的？

Docker Compose 通常将命名的卷挂载为根卷。由于我们使用的是非根用户，如果目录不存在，那么当运行`collectstatic`命令时，我们会得到一个权限被拒绝的错误

要解决这个问题，您可以:

1.  在 Dockerfile ( [source](https://github.com/docker/compose/issues/3270#issuecomment-206214034) )中创建文件夹
2.  挂载后更改目录的权限( [source](https://stackoverflow.com/a/40510068/1799408) )

我们用的是前者。

接下来，更新 Nginx 配置，将静态文件请求路由到“staticfiles”文件夹:

```py
`upstream hello_django {
    server web:8000;
}

server {

    listen 80;

    location / {
        proxy_pass http://hello_django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        alias /home/app/web/staticfiles/;
    }

}` 
```

降低开发容器的转速:

测试:

```py
`$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
$ docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear` 
```

同样，对`http://localhost:1337/static/*`的请求将由“staticfiles”目录提供。

导航到[http://localhost:1337/admin](http://localhost:1337/admin)并确保静态资产正确加载。

您还可以通过`docker-compose -f docker-compose.prod.yml logs -f`在日志中验证对静态文件的请求是否通过 Nginx 成功提供:

```py
`nginx_1  | 192.168.144.1 - - [23/Aug/2021:20:11:00 +0000] "GET /admin/ HTTP/1.1" 302 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36" "-"
nginx_1  | 192.168.144.1 - - [23/Aug/2021:20:11:00 +0000] "GET /admin/login/?next=/admin/ HTTP/1.1" 200 2214 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36" "-"
nginx_1  | 192.168.144.1 - - [23/Aug/2021:20:11:00 +0000] "GET /static/admin/css/base.css HTTP/1.1" 304 0 "http://localhost:1337/admin/login/?next=/admin/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36" "-"
nginx_1  | 192.168.144.1 - - [23/Aug/2021:20:11:00 +0000] "GET /static/admin/css/nav_sidebar.css HTTP/1.1" 304 0 "http://localhost:1337/admin/login/?next=/admin/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36" "-"
nginx_1  | 192.168.144.1 - - [23/Aug/2021:20:11:00 +0000] "GET /static/admin/css/responsive.css HTTP/1.1" 304 0 "http://localhost:1337/admin/login/?next=/admin/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36" "-"
nginx_1  | 192.168.144.1 - - [23/Aug/2021:20:11:00 +0000] "GET /static/admin/css/login.css HTTP/1.1" 304 0 "http://localhost:1337/admin/login/?next=/admin/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36" "-"
nginx_1  | 192.168.144.1 - - [23/Aug/2021:20:11:00 +0000] "GET /static/admin/js/nav_sidebar.js HTTP/1.1" 304 0 "http://localhost:1337/admin/login/?next=/admin/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36" "-"
nginx_1  | 192.168.144.1 - - [23/Aug/2021:20:11:00 +0000] "GET /static/admin/css/fonts.css HTTP/1.1" 304 0 "http://localhost:1337/static/admin/css/base.css" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36" "-"
nginx_1  | 192.168.144.1 - - [23/Aug/2021:20:11:00 +0000] "GET /static/admin/fonts/Roboto-Regular-webfont.woff HTTP/1.1" 304 0 "http://localhost:1337/static/admin/css/fonts.css" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36" "-"
nginx_1  | 192.168.144.1 - - [23/Aug/2021:20:11:00 +0000] "GET /static/admin/fonts/Roboto-Light-webfont.woff HTTP/1.1" 304 0 "http://localhost:1337/static/admin/css/fonts.css" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36" "-"` 
```

完成后带上容器:

```py
`$ docker-compose -f docker-compose.prod.yml down -v` 
```

要测试媒体文件的处理，首先要创建一个新的 Django 应用程序:

```py
`$ docker-compose up -d --build
$ docker-compose exec web python manage.py startapp upload` 
```

将新应用添加到 *settings.py* 中的`INSTALLED_APPS`列表:

```py
`INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",

    "upload",
]` 
```

*app/upload/views.py* :

```py
`from django.shortcuts import render
from django.core.files.storage import FileSystemStorage

def image_upload(request):
    if request.method == "POST" and request.FILES["image_file"]:
        image_file = request.FILES["image_file"]
        fs = FileSystemStorage()
        filename = fs.save(image_file.name, image_file)
        image_url = fs.url(filename)
        print(image_url)
        return render(request, "upload.html", {
            "image_url": image_url
        })
    return render(request, "upload.html")` 
```

在“app/upload”目录下添加一个“templates”，然后添加一个名为*upload.html*的新模板:

```py
`{% block content %}

  <form action="{% url "upload" %}" method="post" enctype="multipart/form-data">
    {% csrf_token %}
    <input type="file" name="image_file">
    <input type="submit" value="submit" />
  </form>

  {% if image_url %}
    <p>File uploaded at: <a href="{{ image_url }}">{{ image_url }}</a></p>
  {% endif %}

{% endblock %}` 
```

*app/hello_django/urls.py* :

```py
`from django.contrib import admin
from django.urls import path
from django.conf import settings
from django.conf.urls.static import static

from upload.views import image_upload

urlpatterns = [
    path("", image_upload, name="upload"),
    path("admin/", admin.site.urls),
]

if bool(settings.DEBUG):
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)` 
```

*app/hello _ django/settings . py*:

```py
`MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "mediafiles"` 
```

### 发展

测试:

```py
`$ docker-compose up -d --build` 
```

你应该可以在 [http://localhost:8000/](http://localhost:8000/) 上传一张图片，然后在[http://localhost:8000/media/IMAGE _ FILE _ NAME](http://localhost:8000/media/IMAGE_FILE_NAME)查看图片。

### 生产

对于生产，向`web`和`nginx`服务添加另一个卷:

```py
`version:  '3.8' services: web: build: context:  ./app dockerfile:  Dockerfile.prod command:  gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000 volumes: -  static_volume:/home/app/web/staticfiles -  media_volume:/home/app/web/mediafiles expose: -  8000 env_file: -  ./.env.prod depends_on: -  db db: image:  postgres:13.0-alpine volumes: -  postgres_data:/var/lib/postgresql/data/ env_file: -  ./.env.prod.db nginx: build:  ./nginx volumes: -  static_volume:/home/app/web/staticfiles -  media_volume:/home/app/web/mediafiles ports: -  1337:80 depends_on: -  web volumes: postgres_data: static_volume: media_volume:` 
```

在 *Dockerfile.prod* 中创建“/home/app/web/mediafiles”文件夹:

```py
`...

# create the appropriate directories
ENV  HOME=/home/app
ENV  APP_HOME=/home/app/web
RUN  mkdir $APP_HOME
RUN  mkdir $APP_HOME/staticfiles
RUN  mkdir $APP_HOME/mediafiles
WORKDIR  $APP_HOME

...` 
```

再次更新 Nginx 配置:

```py
`upstream hello_django {
    server web:8000;
}

server {

    listen 80;

    location / {
        proxy_pass http://hello_django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        alias /home/app/web/staticfiles/;
    }

    location /media/ {
        alias /home/app/web/mediafiles/;
    }

}` 
```

重建:

```py
`$ docker-compose down -v

$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py migrate --noinput
$ docker-compose -f docker-compose.prod.yml exec web python manage.py collectstatic --no-input --clear` 
```

最后一次测试:

1.  在[上传一张图片 http://localhost:1337/](http://localhost:1337/) 。
2.  然后在[http://localhost:1337/media/IMAGE _ FILE _ NAME](http://localhost:1337/media/IMAGE_FILE_NAME)查看图片。

> 如果您看到一个`413 Request Entity Too Large`错误，您将需要[在 Nginx 配置中的服务器或位置上下文中增加客户端请求主体](https://stackoverflow.com/a/28476755/1799408)的最大允许大小。
> 
> 示例:
> 
> ```py
> location / {
>     proxy_pass http://hello_django;
>     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
>     proxy_set_header Host $host;
>     proxy_redirect off;
>     client_max_body_size 100M;
> } 
> ```

## 结论

在本教程中，我们介绍了如何使用 Postgres 来封装 Django web 应用程序以进行开发。我们还创建了一个生产就绪的 Docker Compose 文件，将 Gunicorn 和 Nginx 添加到混合文件中，以处理静态和媒体文件。现在，您可以在本地测试生产设置。

就生产环境的实际部署而言，您可能希望使用:

1.  完全托管的数据库服务——像 [RDS](https://aws.amazon.com/rds/) 或[云 SQL](https://cloud.google.com/sql/)——而不是在一个容器中管理你自己的 Postgres 实例。
2.  `db`和`nginx`服务的非根用户

关于其他生产技巧，请查看[本讨论](https://www.reddit.com/r/django/comments/bjgod8/dockerizing_django_with_postgres_gunicorn_and/)。

你可以在 django-on-docker repo 中找到代码。

> 这里还有一个更老的 Pipenv 版本的代码。

感谢阅读！

> 姜戈码头系列:
> 
> 1.  [用 Postgres、Gunicorn、Nginx](/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/) (本文！)
> 2.  [用加密保护容器化的 Django 应用](/blog/django-lets-encrypt/)
> 3.  [用 Docker 将 Django 部署到 AWS，让我们加密](/blog/django-docker-https-aws/)