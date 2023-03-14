# 用 Postgres、Gunicorn 和 Nginx 对 Masonite 进行对接

> 原文：<https://testdriven.io/blog/dockerizing-masonite-with-postgres-gunicorn-and-nginx/>

这是一个循序渐进的教程，详细介绍了如何配置 [Masonite](https://docs.masoniteproject.com/) ，一个基于 Python 的 web 框架，在 Docker 和 Postgres 上运行。对于生产环境，我们将添加 Nginx 和 Gunicorn。我们还将看看如何通过 Nginx 提供静态和用户上传的媒体文件。

> Masonite 是一个现代的、以开发者为中心的、包含电池的 Python web 框架。如果你是这个框架的新手，可以看看[人们选择 Masonite 而不是 Django 的 5 个原因](https://dev.to/masonite/5-reasons-why-people-are-choosing-masonite-over-django-ic3)文章。

*依赖关系*:

1.  mason ite 4 . 16 . 2 版
2.  文档 v20.10.17
3.  python 3 . 10 . 5 版

## 项目设置

创建项目目录，安装 Masonite，并创建新的 Masonite 项目:

```
`$ mkdir masonite-on-docker && cd masonite-on-docker
$ python3.10 -m venv env
$ source env/bin/activate

(env)$ pip install masonite==4.16.2
(env)$ project start web
(env)$ cd web
(env)$ project install
(env)$ python craft serve` 
```

> 你可以随意把 virtualenv 和 Pip 换成诗歌[或](https://python-poetry.org) [Pipenv](https://github.com/pypa/pipenv) 。更多信息，请查看[现代 Python 环境](/blog/python-environments/)。

导航到 [http://localhost:8000/](http://localhost:8000/) 查看 Masonite 欢迎屏幕。完成后，关闭服务器并退出虚拟环境。继续并删除虚拟环境。我们现在有了一个简单的 Masonite 项目。

接下来，在添加 Docker 之前，让我们稍微清理一下项目结构:

1.  拆下*。环境示例*和*。gitignore* 来自“web”目录的文件
2.  移动*。env* 文件到项目根目录，并将其重命名为 *.env.dev* 。

您的项目结构现在应该如下所示:

```
`├── .env.dev
└── web
    ├── .env.testing
    ├── Kernel.py
    ├── app
    │   ├── __init__.py
    │   ├── controllers
    │   │   ├── WelcomeController.py
    │   │   └── __init__.py
    │   ├── middlewares
    │   │   ├── AuthenticationMiddleware.py
    │   │   ├── VerifyCsrfToken.py
    │   │   └── __init__.py
    │   ├── models
    │   │   └── User.py
    │   └── providers
    │       ├── AppProvider.py
    │       └── __init__.py
    ├── config
    │   ├── __init__.py
    │   ├── application.py
    │   ├── auth.py
    │   ├── broadcast.py
    │   ├── cache.py
    │   ├── database.py
    │   ├── exceptions.py
    │   ├── filesystem.py
    │   ├── mail.py
    │   ├── notification.py
    │   ├── providers.py
    │   ├── queue.py
    │   ├── security.py
    │   └── session.py
    ├── craft
    ├── databases
    │   ├── migrations
    │   │   ├── 2021_01_09_033202_create_password_reset_table.py
    │   │   └── 2021_01_09_043202_create_users_table.py
    │   └── seeds
    │       ├── __init__.py
    │       ├── database_seeder.py
    │       └── user_table_seeder.py
    ├── makefile
    ├── package.json
    ├── pyproject.toml
    ├── requirements.txt
    ├── resources
    │   ├── css
    │   │   └── app.css
    │   └── js
    │       ├── app.js
    │       └── bootstrap.js
    ├── routes
    │   └── web.py
    ├── setup.cfg
    ├── storage
    │   ├── .gitignore
    │   └── public
    │       ├── favicon.ico
    │       ├── logo.png
    │       └── robots.txt
    ├── templates
    │   ├── __init__.py
    │   ├── base.html
    │   ├── errors
    │   │   ├── 403.html
    │   │   ├── 404.html
    │   │   └── 500.html
    │   ├── maintenance.html
    │   └── welcome.html
    ├── tests
    │   ├── TestCase.py
    │   ├── __init__.py
    │   └── unit
    │       └── test_basic_testcase.py
    ├── webpack.mix.js
    └── wsgi.py` 
```

## 码头工人

安装 [Docker](https://docs.docker.com/install/) ，如果你还没有，那么在“web”目录下添加一个 *Dockerfile* :

```
`# pull official base image
FROM  python:3.10.5-alpine

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1
ENV  TZ=UTC

# install system dependencies
RUN  apk update && apk --no-cache add \
    libressl-dev libffi-dev gcc musl-dev python3-dev openssl-dev cargo

# install dependencies
RUN  pip install --upgrade pip
COPY  ./requirements.txt .
RUN  pip install -r requirements.txt

# copy project
COPY  . .` 
```

所以，我们从 Python 3.10.5 的基于 [Alpine](https://github.com/gliderlabs/docker-alpine) 的 [Docker 镜像](https://hub.docker.com/_/python/)开始。然后我们设置一个[工作目录](https://docs.docker.com/engine/reference/builder/#workdir)以及三个环境变量:

1.  `PYTHONDONTWRITEBYTECODE`:防止 Python 将 pyc 文件写入磁盘(相当于`python -B` [选项](https://docs.python.org/3/using/cmdline.html#cmdoption-B)
2.  `PYTHONUNBUFFERED`:防止 Python 缓冲 stdout 和 stderr(相当于`python -u` [选项](https://docs.python.org/3/using/cmdline.html#cmdoption-u)
3.  `TZ=UTC`将容器中的时区设置为 UTC，这是日志记录所必需的

接下来，我们安装了 Python 所需的一些系统级依赖项。记下`openssl-dev`和`cargo`的依赖关系。这些是必需的，因为[密码术](https://cryptography.io/)库[现在依赖于 Rust](https://github.com/pyca/cryptography/issues/5771) 。更多信息，请查看[在 Linux 上构建加密技术](https://cryptography.io/en/latest/installation.html#building-cryptography-on-linux)。

最后，我们更新了 Pip，复制了 *requirements.txt* 文件，安装了依赖项，并复制了 Masonite 应用程序本身。

> 查看 [Docker 针对 Python 开发人员的最佳实践](/blog/docker-best-practices/)，了解更多关于构造 Docker 文件的信息，以及为基于 Python 的开发配置 Docker 的一些最佳实践。

接下来，将一个 *docker-compose.yml* 文件添加到项目根:

```
`version:  '3.8' services: web: build:  ./web command:  python craft serve -p 8000 -b 0.0.0.0 volumes: -  ./web/:/usr/src/app/ ports: -  8000:8000 env_file: -  .env.dev` 
```

> 查看[合成文件参考](https://docs.docker.com/compose/compose-file/)，了解该文件如何工作的信息。

让我们通过删除任何未使用的变量来简化 *.env.dev* :

```
`APP_DEBUG=True
APP_ENV=local
APP_KEY=zWDMwC0aNfVk8Ao1NyVJC_LiGD9tHJtVn_uCPeaaTNY=
APP_URL=http://localhost:8000
HASHING_FUNCTION=bcrypt

MAIL_DRIVER=terminal

DB_CONNECTION=sqlite
SQLITE_DB_DATABASE=masonite.sqlite3
DB_HOST=127.0.0.1
DB_USERNAME=root
DB_PASSWORD=root
DB_DATABASE=masonite
DB_PORT=3306
DB_LOG=True

QUEUE_DRIVER=async` 
```

建立形象:

构建映像后，运行容器:

导航到 [http://localhost:8000/](http://localhost:8000/) 再次查看欢迎屏幕。

> 如果这不起作用，通过`docker-compose logs -f`检查日志中的错误。

要测试自动重新加载，首先打开 Docker 日志- `docker-compose logs -f` -然后在本地对 *web/routes/web.py* 进行更改:

保存后，您应该会看到应用程序在您的终端中重新加载，如下所示:

```
`* Detected change in '/usr/src/app/routes/web.py', reloading
* Restarting with watchdog (inotify)` 
```

确保[http://localhost:8000/sample](http://localhost:8000/sample)按预期工作。

## Postgres

要配置 Postgres，我们需要向 *docker-compose.yml* 文件添加一个新服务，更新环境变量，并安装 [Psycopg2](https://www.psycopg.org/) 。

首先，向 *docker-compose.yml* 添加一个名为`db`的新服务:

```
`version:  '3.8' services: web: build:  ./web command:  python craft serve -p 8000 -b 0.0.0.0 volumes: -  ./web/:/usr/src/app/ ports: -  8000:8000 env_file: -  .env.dev depends_on: -  db db: image:  postgres:14.4-alpine volumes: -  postgres_data_dev:/var/lib/postgresql/data/ environment: -  POSTGRES_USER=hello_masonite -  POSTGRES_PASSWORD=hello_masonite -  POSTGRES_DB=hello_masonite_dev volumes: postgres_data_dev:` 
```

为了在容器的生命周期之外保存数据，我们配置了一个卷。这个配置将把`postgres_data_dev`绑定到容器中的“/var/lib/postgresql/data/”目录。

我们还添加了一个环境键来定义默认数据库的名称，并设置用户名和密码。

> 查看 [Postgres Docker Hub 页面](https://hub.docker.com/_/postgres)的“环境变量”部分了解更多信息。

在 *.env.dev* 文件中更新以下与数据库相关的环境变量:

```
`DB_CONNECTION=postgres
DB_HOST=db
DB_PORT=5432
DB_DATABASE=hello_masonite_dev
DB_USERNAME=hello_masonite
DB_PASSWORD=hello_masonite` 
```

> 查看 *web/config/database.py* 文件，了解如何根据为 Masonite 项目定义的环境变量来配置数据库。

更新 docker 文件以安装 Psycopg2 所需的相应软件包:

```
`# pull official base image
FROM  python:3.10.5-alpine

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1
ENV  TZ=UTC

# install system dependencies
RUN  apk update && apk --no-cache add \
    libressl-dev libffi-dev gcc musl-dev python3-dev openssl-dev cargo \
    postgresql-dev

# install dependencies
RUN  pip install --upgrade pip
COPY  ./requirements.txt .
RUN  pip install -r requirements.txt

# copy project
COPY  . .` 
```

将 Psycopg2 添加到 *web/requirements.txt* :

```
`masonite>=4.0,<5.0
masonite-orm>=2.0,<3.0
psycopg2-binary==2.9.3` 
```

> 查看[本期 GitHub](https://github.com/psycopg/psycopg2/issues/684)了解更多关于在基于 Alpine 的 Docker 映像中安装 Psycopg2 的信息。

构建新的映像并旋转两个容器:

```
`$ docker-compose up -d --build` 
```

应用迁移(从“web/数据库/迁移”文件夹):

```
`$ docker-compose exec web python craft migrate` 
```

您应该看到:

```
`Migrating: 2021_01_09_033202_create_password_reset_table
Migrated: 2021_01_09_033202_create_password_reset_table (0.01s)
Migrating: 2021_01_09_043202_create_users_table
Migrated: 2021_01_09_043202_create_users_table (0.02s)` 
```

确保`users`表已创建:

```
`$ docker-compose exec db psql --username=hello_masonite --dbname=hello_masonite_dev

psql (14.4)
Type "help" for help.

hello_masonite_dev=# \l
                                              List of databases
        Name        |     Owner      | Encoding |  Collate   |   Ctype    |         Access privileges
--------------------+----------------+----------+------------+------------+-----------------------------------
 hello_masonite_dev | hello_masonite | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres           | hello_masonite | UTF8     | en_US.utf8 | en_US.utf8 |
 template0          | hello_masonite | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_masonite                +
                    |                |          |            |            | hello_masonite=CTc/hello_masonite
 template1          | hello_masonite | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_masonite                +
                    |                |          |            |            | hello_masonite=CTc/hello_masonite
(4 rows)

hello_masonite_dev=# \c hello_masonite_dev
You are now connected to database "hello_masonite_dev" as user "hello_masonite".

hello_masonite_dev=# \dt
              List of relations
 Schema |    Name         | Type  |     Owner
--------+-----------------+-------+----------------
 public | migrations      | table | hello_masonite
 public | password_resets | table | hello_masonite
 public | users           | table | hello_masonite
(3 rows)

hello_masonite_dev=# \q` 
```

您也可以通过运行以下命令来检查该卷是否已创建:

```
`$ docker volume inspect masonite-on-docker_postgres_data_dev` 
```

您应该会看到类似如下的内容:

```
`[
    {
        "CreatedAt": "2022-07-22T20:07:50Z",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "masonite-on-docker",
            "com.docker.compose.version": "2.6.1",
            "com.docker.compose.volume": "postgres_data_dev"
        },
        "Mountpoint": "/var/lib/docker/volumes/masonite-on-docker_postgres_data_dev/_data",
        "Name": "masonite-on-docker_postgres_data_dev",
        "Options": null,
        "Scope": "local"
    }
]` 
```

接下来，将一个 *entrypoint.sh* 文件添加到“web”目录，以在应用迁移和运行 Masonite 开发服务器之前，验证 Postgres 是否启动*和*健康*:*

```
`#!/bin/sh

if [ "$DB_CONNECTION" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $DB_HOST $DB_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

python craft migrate:refresh  # you may want to remove this
python craft migrate

exec "[[email protected]](/cdn-cgi/l/email-protection)"` 
```

记下环境变量。

然后，更新 Dockerfile 来运行 *entrypoint.sh* 文件，作为 Docker [entrypoint](https://docs.docker.com/engine/reference/builder/#entrypoint) 命令:

```
`# pull official base image
FROM  python:3.10.5-alpine

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1
ENV  TZ=UTC

# install system dependencies
RUN  apk update && apk --no-cache add \
    libressl-dev libffi-dev gcc musl-dev python3-dev openssl-dev cargo \
    postgresql-dev

# install dependencies
RUN  pip install --upgrade pip
COPY  ./requirements.txt .
RUN  pip install -r requirements.txt

# copy project
COPY  . .

# run entrypoint.sh
ENTRYPOINT  ["/usr/src/app/entrypoint.sh"]` 
```

在本地更新文件权限:

```
`$ chmod +x web/entrypoint.sh` 
```

再次测试:

1.  重建图像
2.  运行容器
3.  试试 [http://localhost:8000/](http://localhost:8000/)

想播种一些用户？

```
`$ docker-compose exec web python craft seed:run

$ docker-compose exec db psql --username=hello_masonite --dbname=hello_masonite_dev

psql (14.4)
Type "help" for help.

hello_masonite_dev=# \c hello_masonite_dev
You are now connected to database "hello_masonite_dev" as user "hello_masonite".

hello_masonite_dev=# select count(*) from users;
 count
-------
     1
(1 row)

hello_masonite_dev=# \q` 
```

## 格尼科恩

接下来，对于生产环境，让我们将 [Gunicorn](https://gunicorn.org/) ，一个生产级的 WSGI 服务器，添加到需求文件中:

```
`masonite>=4.0,<5.0
masonite-orm>=2.0,<3.0
psycopg2-binary==2.9.3
gunicorn==20.1.0` 
```

由于我们仍然希望在开发中使用 Masonite 的内置服务器，因此为生产创建一个名为 *docker-compose.prod.yml* 的新合成文件:

```
`version:  '3.8' services: web: build:  ./web command:  gunicorn --bind 0.0.0.0:8000 wsgi:application ports: -  8000:8000 env_file: -  .env.prod depends_on: -  db db: image:  postgres:14.4-alpine volumes: -  postgres_data_prod:/var/lib/postgresql/data/ env_file: -  .env.prod.db volumes: postgres_data_prod:` 
```

> 如果您有多个环境，您可能希望使用一个[docker-compose . override . yml](https://docs.docker.com/compose/extends/)配置文件。使用这种方法，您可以将您的基本配置添加到一个 *docker-compose.yml* 文件中，然后使用一个*docker-compose . override . yml*文件根据环境覆盖那些配置设置。

记下默认值`command`。我们运行的是 Gunicorn 而不是 Masonite 开发服务器。我们还从`web`服务中删除了这个卷，因为我们在生产中不需要它。最后，我们使用[单独的环境变量文件](https://docs.docker.com/compose/env-file/)来定义两个服务的环境变量，它们将在运行时传递给容器。

*.env.prod* :

```
`APP_DEBUG=False
APP_ENV=prod
APP_KEY=GM28x-FeI1sM72tgtsgikLcT-AryyVOiY8etOGr7q7o=
APP_URL=http://localhost:8000
HASHING_FUNCTION=bcrypt

MAIL_DRIVER=terminal

DB_CONNECTION=postgres
DB_HOST=db
DB_PORT=5432
DB_DATABASE=hello_masonite_prod
DB_USERNAME=hello_masonite
DB_PASSWORD=hello_masonite
DB_LOG=True

QUEUE_DRIVER=async` 
```

*.env.prod.db* :

```
`POSTGRES_USER=hello_masonite
POSTGRES_PASSWORD=hello_masonite
POSTGRES_DB=hello_masonite_prod` 
```

将这两个文件添加到项目根目录。您可能想让它们不受版本控制，所以将它们添加到一个*中。gitignore* 文件。

将[下放到](https://docs.docker.com/compose/reference/down/)开发容器(以及带有`-v`标志的相关卷):

然后，构建生产映像并启动容器:

```
`$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

验证`hello_masonite_prod`数据库是与`users`表一起创建的。测试出 [http://localhost:8000/](http://localhost:8000/) 。

> 同样，如果容器启动失败，通过`docker-compose -f docker-compose.prod.yml logs -f`检查日志中的错误。

## 生产文档

您是否注意到，每次运行容器时，我们仍然在运行 [migrate:refresh](https://orm.masoniteproject.com/schema-and-migrations#refreshing-migrations) (清空数据库)和 migrate 命令？这在开发中很好，但是让我们为生产创建一个新的入口点文件。

entry point . prod . sh:

```
`#!/bin/sh

if [ "$DB_CONNECTION" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z $DB_HOST $DB_PORT; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

exec "[[email protected]](/cdn-cgi/l/email-protection)"` 
```

> 或者，不创建新的入口点文件，您可以修改现有的文件，如下所示:
> 
> ```
> #!/bin/sh
> 
> if [ "$DB_CONNECTION" = "postgres" ]
> then
>     echo "Waiting for postgres..."
> 
>     while ! nc -z $DB_HOST $DB_PORT; do
>       sleep 0.1
>     done
> 
>     echo "PostgreSQL started"
> fi
> 
> if [ "$APP_ENV" = "local" ]
> then
>     echo "Refreshing the database..."
>     craft migrate:refresh  # you may want to remove this
>     echo "Applying migrations..."
>     craft migrate
>     echo "Tables created"
> fi
> 
> exec "[[email protected]](/cdn-cgi/l/email-protection)" 
> ```

要使用这个文件，创建一个名为 *Dockerfile.prod* 的新 Dockerfile，用于生产构建:

```
`###########
# BUILDER #
###########

# pull official base image
FROM  python:3.10.5-alpine  as  builder

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1
ENV  TZ=UTC

# install system dependencies
RUN  apk update && apk --no-cache add \
    libressl-dev libffi-dev gcc musl-dev python3-dev openssl-dev cargo \
    postgresql-dev

# lint
RUN  pip install --upgrade pip
RUN  pip install flake8==4.0.1
COPY  . .
RUN  flake8 --ignore=E501,F401,E303,E402 .

# install python dependencies
COPY  ./requirements.txt .
RUN  pip wheel --no-cache-dir --no-deps --wheel-dir /usr/src/app/wheels -r requirements.txt

#########
# FINAL #
#########

# pull official base image
FROM  python:3.10.5-alpine

# create directory for the app user
RUN  mkdir -p /home/app

# create the app user
RUN  addgroup -S app && adduser -S app -G app

# create the appropriate directories
ENV  HOME=/home/app
ENV  APP_HOME=/home/app/web
RUN  mkdir $APP_HOME
WORKDIR  $APP_HOME

# set environment variables
ENV  TZ=UTC

# install dependencies
RUN  apk update && apk --no-cache add \
    libressl-dev libffi-dev gcc musl-dev python3-dev openssl-dev cargo \
    postgresql-dev
COPY  --from=builder /usr/src/app/wheels /wheels
COPY  --from=builder /usr/src/app/requirements.txt .
RUN  pip install --no-cache /wheels/*

# copy project
COPY  . $APP_HOME
RUN  chmod +x /home/app/web/entrypoint.prod.sh

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

```
`web: build: context:  ./web dockerfile:  Dockerfile.prod command:  gunicorn --bind 0.0.0.0:8000 wsgi:application ports: -  8000:8000 env_file: -  .env.prod depends_on: -  db` 
```

尝试一下:

```
`$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python craft migrate` 
```

## Nginx

接下来，让我们将 Nginx 添加进来，充当 Gunicorn 的反向代理来处理客户端请求以及提供静态文件。

将服务添加到 *docker-compose.prod.yml* :

```
`nginx: build:  ./nginx ports: -  1337:80 depends_on: -  web` 
```

然后，在本地项目根目录中，创建以下文件和文件夹:

```
`└── nginx
    ├── Dockerfile
    └── nginx.conf` 
```

*Dockerfile* :

```
`FROM  nginx:1.23.1-alpine

RUN  rm /etc/nginx/conf.d/default.conf
COPY  nginx.conf /etc/nginx/conf.d` 
```

*engine . conf*:

```
`upstream hello_masonite {
    server web:8000;
}

server {

    listen 80;

    location / {
        proxy_pass http://hello_masonite;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

}` 
```

> 查看[了解 Nginx 配置文件结构和配置上下文](https://www.digitalocean.com/community/tutorials/understanding-the-nginx-configuration-file-structure-and-configuration-contexts)以了解 Nginx 配置文件的更多信息。

然后，更新`web`服务，在 *docker-compose.prod.yml* 中，用`expose`替换`ports`:

```
`web: build: context:  ./web dockerfile:  Dockerfile.prod command:  gunicorn --bind 0.0.0.0:8000 wsgi:application expose: -  8000 env_file: -  .env.prod depends_on: -  db` 
```

现在，端口 8000 只在内部对其他 Docker 服务公开。该端口将不再发布到主机上。

> 关于端口与暴露的更多信息，请查看[这个](https://stackoverflow.com/questions/40801772/what-is-the-difference-between-docker-compose-ports-vs-expose)堆栈溢出问题。

再测试一次。

```
`$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python craft migrate` 
```

确保应用程序在 [http://localhost:1337](http://localhost:1337) 启动并运行。

您的项目结构现在应该看起来像这样:

```
`├── .env.dev
├── .env.prod
├── .env.prod.db
├── .gitignore
├── docker-compose.prod.yml
├── docker-compose.yml
├── nginx
│   ├── Dockerfile
│   └── nginx.conf
└── web
    ├── .env.testing
    ├── Dockerfile
    ├── Dockerfile.prod
    ├── Kernel.py
    ├── app
    │   ├── __init__.py
    │   ├── controllers
    │   │   ├── WelcomeController.py
    │   │   └── __init__.py
    │   ├── middlewares
    │   │   ├── AuthenticationMiddleware.py
    │   │   ├── VerifyCsrfToken.py
    │   │   └── __init__.py
    │   ├── models
    │   │   └── User.py
    │   └── providers
    │       ├── AppProvider.py
    │       └── __init__.py
    ├── config
    │   ├── __init__.py
    │   ├── application.py
    │   ├── auth.py
    │   ├── broadcast.py
    │   ├── cache.py
    │   ├── database.py
    │   ├── exceptions.py
    │   ├── filesystem.py
    │   ├── mail.py
    │   ├── notification.py
    │   ├── providers.py
    │   ├── queue.py
    │   ├── security.py
    │   └── session.py
    ├── craft
    ├── databases
    │   ├── migrations
    │   │   ├── 2021_01_09_033202_create_password_reset_table.py
    │   │   └── 2021_01_09_043202_create_users_table.py
    │   └── seeds
    │       ├── __init__.py
    │       ├── database_seeder.py
    │       └── user_table_seeder.py
    ├── entrypoint.prod.sh
    ├── entrypoint.sh
    ├── makefile
    ├── package.json
    ├── pyproject.toml
    ├── requirements.txt
    ├── resources
    │   ├── css
    │   │   └── app.css
    │   └── js
    │       ├── app.js
    │       └── bootstrap.js
    ├── routes
    │   └── web.py
    ├── setup.cfg
    ├── storage
    │   ├── .gitignore
    │   └── public
    │       ├── favicon.ico
    │       ├── logo.png
    │       └── robots.txt
    ├── templates
    │   ├── __init__.py
    │   ├── base.html
    │   ├── errors
    │   │   ├── 403.html
    │   │   ├── 404.html
    │   │   └── 500.html
    │   ├── maintenance.html
    │   └── welcome.html
    ├── tests
    │   ├── TestCase.py
    │   ├── __init__.py
    │   └── unit
    │       └── test_basic_testcase.py
    ├── webpack.mix.js
    └── wsgi.py` 
```

完成后将容器拿下来:

```
`$ docker-compose -f docker-compose.prod.yml down -v` 
```

由于 Gunicorn 是一个应用服务器，它不会提供静态文件。那么，在这种特定的配置中，应该如何处理静态文件和媒体文件呢？

## 静态文件

首先，更新 *web/config/filesystem.py* 中的`STATICFILES`配置:

```
`STATICFILES = {
    # folder          # template alias
    "storage/static": "static/",
    "storage/compiled": "static/",
    "storage/uploads": "uploads/",
    "storage/public": "/",
}` 
```

本质上，存储在“存储/静态”(常规 CSS 和 JS 文件)和“存储/编译”(SASS 和更少的文件)目录中的所有静态文件都将从`/static/` URL 提供。

为了启用资产编译，假设您已经安装了 [NPM](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm) ，安装依赖项:

然后，要编译资产，运行:

> 有关资产复杂性的更多信息，请查看 Masonite 文档中的[编译资产](https://docs.masoniteproject.com/features/compiling-assets)。

接下来，为了测试一个常规的静态资产，向“web/storage/static”添加一个名为 *hello.txt* 的文本文件:

### 发展

为了进行测试，首先重新构建映像，并像往常一样旋转新的容器。完成后，确保正确加载以下静态资产:

1.  [http://localhost:8000/robots . txt](http://localhost:8000/robots.txt)([root](https://docs.masoniteproject.com/the-basics/static-files#serving-root-files)静态资产)
2.  [http://localhost:8000/static/hello . txt](http://localhost:8000/static/hello.txt)(常规静态资产)
3.  [http://localhost:8000/static/CSS/app . CSS](http://localhost:8000/static/css/app.css)(编译后的静态资产)

### 生产

对于生产，向 *docker-compose.prod.yml* 中的`web`和`nginx`服务添加一个卷，这样每个容器将共享“存储”目录:

```
`version:  '3.8' services: web: build: context:  ./web dockerfile:  Dockerfile.prod command:  gunicorn --bind 0.0.0.0:8000 wsgi:application volumes: -  storage_volume:/home/app/web/storage expose: -  8000 env_file: -  .env.prod depends_on: -  db db: image:  postgres:14.4-alpine volumes: -  postgres_data_prod:/var/lib/postgresql/data/ env_file: -  .env.prod.db nginx: build:  ./nginx volumes: -  storage_volume:/home/app/web/storage ports: -  1337:80 depends_on: -  web volumes: postgres_data_prod: storage_volume:` 
```

接下来，更新 Nginx 配置，将静态文件请求路由到适当的文件夹:

```
`upstream hello_masonite {
    server web:8000;
}

server {

    listen 80;

    location /static/ {
        alias /home/app/web/storage/static/;
    }

    location ~ ^/(favicon.ico|robots.txt)/  {
        alias /home/app/web/storage/public/;
    }

    location / {
        proxy_pass http://hello_masonite;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

}` 
```

现在，对静态文件的请求将被适当地提供:

| 请求 URL | 文件夹 |
| --- | --- |
| /static/* | “存储/静态”、“存储/编译” |
| /favicon.ico，/robots.txt | "存储/公共" |

降低开发容器的转速:

测试:

```
`$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

同样，请确保正确加载了以下静态资产:

1.  [http://localhost:1337/robots . txt](http://localhost:1337/robots.txt)
2.  [http://localhost:1337/static/hello . txt](http://localhost:1337/static/hello.txt)
3.  [http://localhost:1337/static/CSS/app . CSS](http://localhost:1337/static/css/app.css)

您还可以通过`docker-compose -f docker-compose.prod.yml logs -f`在日志中验证对静态文件的请求是否通过 Nginx 成功提供:

```
`nginx_1  | 172.28.0.1 - - [2022-07-20:01:39:43 +0000] "GET /robots.txt HTTP/1.1" 200 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.96 Safari/537.36" "-"
nginx_1  | 172.28.0.1 - - [2022-07-20:01:39:52 +0000] "GET /static/hello.txt HTTP/1.1" 200 4 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.96 Safari/537.36" "-"
nginx_1  | 172.28.0.1 - - [2022-07-20:01:39:59 +0000] "GET /static/css/app.css HTTP/1.1" 200 649 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.96 Safari/537.36" "-"` 
```

完成后带上容器:

```
`$ docker-compose -f docker-compose.prod.yml down -v` 
```

要测试对用户上传的媒体文件的处理，请更新*web/templates/welcome . html*模板中的内容块:

```
`{% block content %}
<html>
  <body>
  <form action="/" method="POST" enctype="multipart/form-data">
    {{ csrf_field }}
    <input type="file" name="image_upload">
    <input type="submit" value="submit" />
  </form>
  {% if image_url %}
    <p>File uploaded at: <a href="{{ image_url }}">{{ image_url }}</a></p>
  {% endif %}
  </body>
</html>
{% endblock %}` 
```

在*web/app/controllers/welcome controller . py*中给`WelcomeController`添加一个名为`upload`的新方法:

```
`def upload(self, storage: Storage, view: View, request: Request):
    filename = storage.disk("local").put_file("image_upload", request.input("image_upload"))
    return view.render("welcome", {"image_url": f"/framework/filesystem/{filename}"})` 
```

不要忘记进口:

```
`from masonite.filesystem import Storage
from masonite.request import Request` 
```

接下来，将控制器连接到 *web/routes/web.py* 中的新路线:

### 发展

测试:

```
`$ docker-compose up -d --build` 
```

你应该可以在 [http://localhost:8000/](http://localhost:8000/) 上传一张图片，然后在[http://localhost:8000/uploads/IMAGE _ FILE _ NAME](http://localhost:8000/uploads/IMAGE_FILE_NAME)查看图片。

### 生产

对于生产，更新 Nginx 配置以将媒体文件请求路由到“上传”文件夹:

```
`upstream hello_masonite {
    server web:8000;
}

server {

    listen 80;

    location /static/ {
        alias /home/app/web/storage/static/;
    }

    location ~ ^/(favicon.ico|robots.txt)/  {
        alias /home/app/web/storage/public/;
    }

    location /uploads/ {
        alias /home/app/web/storage/framework/filesystem/image_upload/;
    }

    location / {
        proxy_pass http://hello_masonite;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

}` 
```

重建:

```
`$ docker-compose down -v

$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

最后一次测试:

1.  在 [http://localhost:1337](http://localhost:1337) 上传一张图片。
2.  然后在[http://localhost:1337/uploads/IMAGE _ FILE _ NAME](http://localhost:1337/uploads/IMAGE_FILE_NAME)查看图片。

## 结论

在本教程中，我们介绍了如何使用 Postgres 来封装 Masonite 应用程序以进行开发。我们还创建了一个生产就绪的 Docker Compose 文件，将 Gunicorn 和 Nginx 添加到混合文件中，以处理静态和媒体文件。现在，您可以在本地测试生产设置。

就生产环境的实际部署而言，您可能希望使用:

1.  完全托管的数据库服务——像 [RDS](https://aws.amazon.com/rds/) 或[云 SQL](https://cloud.google.com/sql/)——而不是在一个容器中管理你自己的 Postgres 实例。
2.  `db`和`nginx`服务的非根用户

你可以在 [masonite-on-docker](https://github.com/testdrivenio/masonite-on-docker) repo 中找到代码。

感谢阅读！