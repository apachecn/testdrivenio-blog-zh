# 装有 Postgres、Gunicorn 和 Nginx 的对接烧瓶

> 原文：<https://testdriven.io/blog/dockerizing-flask-with-postgres-gunicorn-and-nginx/>

这是一个循序渐进的教程，详细介绍了如何配置 Flask 运行在 Docker 与 Postgres。对于生产环境，我们将添加 Nginx 和 Gunicorn。我们还将看看如何通过 Nginx 提供静态和用户上传的媒体文件。

*依赖关系*:

1.  烧瓶 v2.2.2
2.  文档 v20.10.17
3.  python 3 . 10 . 7 版

## 项目设置

创建一个新的项目目录并安装 Flask:

```py
`$ mkdir flask-on-docker && cd flask-on-docker
$ mkdir services && cd services
$ mkdir web && cd web
$ mkdir project

$ python3.10 -m venv env
$ source env/bin/activate

(env)$ pip install flask==2.2.2` 
```

> 你可以随意把 virtualenv 和 Pip 换成诗歌[或](https://python-poetry.org) [Pipenv](https://github.com/pypa/pipenv) 。更多信息，请查看[现代 Python 环境](/blog/python-environments/)。

接下来，让我们创建一个新的 Flask 应用程序。

添加一个 *__init__。py* 文件到" project "目录并配置第一条路由:

```py
`from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def hello_world():
    return jsonify(hello="world")` 
```

然后，要配置 Flask CLI 工具从命令行运行和管理应用程序，请将一个 *manage.py* 文件添加到“web”目录:

```py
`from flask.cli import FlaskGroup

from project import app

cli = FlaskGroup(app)

if __name__ == "__main__":
    cli()` 
```

这里，我们创建了一个新的`FlaskGroup`实例，用与 Flask 应用程序相关的命令来扩展普通 CLI。

从“web”目录运行服务器:

```py
`(env)$ export FLASK_APP=project/__init__.py
(env)$ python manage.py run` 
```

导航到 [http://localhost:5000/](http://localhost:5000/) 。您应该看到:

一旦完成就杀死服务器。退出，然后也删除虚拟环境。

在“web”目录下创建一个 *requirements.txt* 文件，并添加 Flask 作为依赖项:

您的项目结构应该是这样的:

```py
`└── services
    └── web
        ├── manage.py
        ├── project
        │   └── __init__.py
        └── requirements.txt` 
```

## 码头工人

安装 [Docker](https://docs.docker.com/install/) ，如果你还没有，那么在“web”目录下添加一个 *Dockerfile* :

```py
`# pull official base image
FROM  python:3.10.7-slim-buster

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

# install dependencies
RUN  pip install --upgrade pip
COPY  ./requirements.txt /usr/src/app/requirements.txt
RUN  pip install -r requirements.txt

# copy project
COPY  . /usr/src/app/` 
```

所以，我们从 Python 3.10.7 的基于`slim-buster`的 [Docker 镜像](https://hub.docker.com/_/python/)开始。然后我们设置一个[工作目录](https://docs.docker.com/engine/reference/builder/#workdir)以及两个环境变量:

1.  `PYTHONDONTWRITEBYTECODE`:防止 Python 将 pyc 文件写入磁盘(相当于`python -B` [选项](https://docs.python.org/3/using/cmdline.html#cmdoption-B)
2.  `PYTHONUNBUFFERED`:防止 Python 缓冲 stdout 和 stderr(相当于`python -u` [选项](https://docs.python.org/3/using/cmdline.html#cmdoption-u)

最后，我们更新了 Pip，复制了 *requirements.txt* 文件，安装了依赖项，并复制了 Flask 应用程序本身。

> 查看 [Docker 针对 Python 开发人员的最佳实践](/blog/docker-best-practices/)，了解更多关于构造 Docker 文件的信息，以及为基于 Python 的开发配置 Docker 的一些最佳实践。

接下来，将一个 *docker-compose.yml* 文件添加到项目根:

```py
`version:  '3.8' services: web: build:  ./services/web command:  python manage.py run -h 0.0.0.0 volumes: -  ./services/web/:/usr/src/app/ ports: -  5000:5000 env_file: -  ./.env.dev` 
```

> 查看[合成文件参考](https://docs.docker.com/compose/compose-file/)，了解该文件如何工作的信息。

然后，在项目根目录下创建一个 *.env.dev* 文件来存储开发环境变量:

```py
`FLASK_APP=project/__init__.py
FLASK_DEBUG=1` 
```

建立形象:

构建映像后，运行容器:

导航到 [http://localhost:5000/](http://localhost:5000/) 再次查看 hello world 健全性检查。

> 如果这不起作用，通过`docker-compose logs -f`检查日志中的错误。

## Postgres

要配置 Postgres，我们需要在 *docker-compose.yml* 文件中添加一个新服务，设置 [Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com) ，安装 [Psycopg2](https://www.psycopg.org/) 。

首先，向 *docker-compose.yml* 添加一个名为`db`的新服务:

```py
`version:  '3.8' services: web: build:  ./services/web command:  python manage.py run -h 0.0.0.0 volumes: -  ./services/web/:/usr/src/app/ ports: -  5000:5000 env_file: -  ./.env.dev depends_on: -  db db: image:  postgres:13-alpine volumes: -  postgres_data:/var/lib/postgresql/data/ environment: -  POSTGRES_USER=hello_flask -  POSTGRES_PASSWORD=hello_flask -  POSTGRES_DB=hello_flask_dev volumes: postgres_data:` 
```

为了在容器的生命周期之外保存数据，我们配置了一个卷。这个配置将把`postgres_data`绑定到容器中的“/var/lib/postgresql/data/”目录。

我们还添加了一个环境键来定义默认数据库的名称，并设置用户名和密码。

> 查看 [Postgres Docker Hub 页面](https://hub.docker.com/_/postgres)的“环境变量”部分了解更多信息。

向 *.env.dev* 添加一个`DATABASE_URL`环境变量:

```py
`FLASK_APP=project/__init__.py FLASK_DEBUG=1 DATABASE_URL=postgresql://hello_flask:hello_flask@db:5432/hello_flask_dev` 
```

然后，将一个名为 *config.py* 的新文件添加到“项目”目录中，在这里我们将定义特定于环境的[配置](https://flask.palletsprojects.com/config/)变量:

```py
`import os

basedir = os.path.abspath(os.path.dirname(__file__))

class Config(object):
    SQLALCHEMY_DATABASE_URI = os.getenv("DATABASE_URL", "sqlite://")
    SQLALCHEMY_TRACK_MODIFICATIONS = False` 
```

这里，数据库是基于我们刚刚定义的`DATABASE_URL`环境变量配置的。记下默认值。

更新 *__init__。py* 在 init 上拉入配置:

```py
`from flask import Flask, jsonify

app = Flask(__name__)
app.config.from_object("project.config.Config")

@app.route("/")
def hello_world():
    return jsonify(hello="world")` 
```

将 [Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com) 和 [Psycopg2](https://www.psycopg.org/) 添加到 *requirements.txt* :

```py
`Flask==2.2.2
Flask-SQLAlchemy==2.5.1
psycopg2-binary==2.9.4` 
```

更新 *__init__。py* 再次创建一个新的`SQLAlchemy`实例并定义一个数据库模型:

```py
`from flask import Flask, jsonify
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config.from_object("project.config.Config")
db = SQLAlchemy(app)

class User(db.Model):
    __tablename__ = "users"

    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(128), unique=True, nullable=False)
    active = db.Column(db.Boolean(), default=True, nullable=False)

    def __init__(self, email):
        self.email = email

@app.route("/")
def hello_world():
    return jsonify(hello="world")` 
```

最后，更新 *manage.py* :

```py
`from flask.cli import FlaskGroup

from project import app, db

cli = FlaskGroup(app)

@cli.command("create_db")
def create_db():
    db.drop_all()
    db.create_all()
    db.session.commit()

if __name__ == "__main__":
    cli()` 
```

这向 CLI 注册了一个新命令`create_db`，以便我们可以从命令行运行它，稍后我们将使用它将模型应用到数据库。

构建新的映像并旋转两个容器:

```py
`$ docker-compose up -d --build` 
```

创建表格:

```py
`$ docker-compose exec web python manage.py create_db` 
```

> 得到以下错误？
> 
> ```py
> sqlalchemy.exc.OperationalError: (psycopg2.OperationalError)
> FATAL:  database "hello_flask_dev" does not exist 
> ```
> 
> 运行`docker-compose down -v`移除卷和容器。然后，重新构建映像，运行容器，并应用迁移。

确保`users`表已创建:

```py
`$ docker-compose exec db psql --username=hello_flask --dbname=hello_flask_dev

psql (13.8)
Type "help" for help.

hello_flask_dev=# \l
                                        List of databases
      Name       |    Owner    | Encoding |  Collate   |   Ctype    |      Access privileges
-----------------+-------------+----------+------------+------------+-----------------------------
 hello_flask_dev | hello_flask | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres        | hello_flask | UTF8     | en_US.utf8 | en_US.utf8 |
 template0       | hello_flask | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_flask             +
                 |             |          |            |            | hello_flask=CTc/hello_flask
 template1       | hello_flask | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_flask             +
                 |             |          |            |            | hello_flask=CTc/hello_flask
(4 rows)

hello_flask_dev=# \c hello_flask_dev
You are now connected to database "hello_flask_dev" as user "hello_flask".

hello_flask_dev=# \dt
          List of relations
 Schema | Name  | Type  |    Owner
--------+-------+-------+-------------
 public | users | table | hello_flask
(1 row)

hello_flask_dev=# \q` 
```

您也可以通过运行以下命令来检查该卷是否已创建:

```py
`$ docker volume inspect flask-on-docker_postgres_data` 
```

您应该会看到类似如下的内容:

```py
`[
    {
        "CreatedAt": "2022-10-11T14:43:49Z",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "flask-on-docker",
            "com.docker.compose.version": "2.10.2",
            "com.docker.compose.volume": "postgres_data"
        },
        "Mountpoint": "/var/lib/docker/volumes/flask-on-docker_postgres_data/_data",
        "Name": "flask-on-docker_postgres_data",
        "Options": null,
        "Scope": "local"
    }
]` 
```

接下来，将一个 *entrypoint.sh* 文件添加到“web”目录，以在创建数据库表并运行 Flask 开发服务器之前，验证 Postgres 是否已启动*和*是否健康*:*

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

python manage.py create_db

exec "[[email protected]](/cdn-cgi/l/email-protection)"` 
```

记下环境变量。

在本地更新文件权限:

```py
`$ chmod +x services/web/entrypoint.sh` 
```

然后，更新 Docker 文件安装 [Netcat](http://netcat.sourceforge.net/) ，复制 *entrypoint.sh* 文件，运行该文件作为 Docker [entrypoint](https://docs.docker.com/engine/reference/builder/#entrypoint) 命令:

```py
`# pull official base image
FROM  python:3.10.7-slim-buster

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

# install system dependencies
RUN  apt-get update && apt-get install -y netcat

# install dependencies
RUN  pip install --upgrade pip
COPY  ./requirements.txt /usr/src/app/requirements.txt
RUN  pip install -r requirements.txt

# copy project
COPY  . /usr/src/app/

# run entrypoint.sh
ENTRYPOINT  ["/usr/src/app/entrypoint.sh"]` 
```

将 *entrypoint.sh* 脚本的`SQL_HOST`、`SQL_PORT`和`DATABASE`环境变量添加到 *.env.dev* :

```py
`FLASK_APP=project/__init__.py FLASK_DEBUG=1 DATABASE_URL=postgresql://hello_flask:hello_flask@db:5432/hello_flask_dev SQL_HOST=db SQL_PORT=5432 DATABASE=postgres` 
```

再次测试:

1.  重建图像
2.  运行容器
3.  试试 [http://localhost:5000/](http://localhost:5000/)

让我们添加一个 CLI 种子命令，用于将示例用户添加到 *manage.py* 中的`users`表中:

```py
`from flask.cli import FlaskGroup

from project import app, db, User

cli = FlaskGroup(app)

@cli.command("create_db")
def create_db():
    db.drop_all()
    db.create_all()
    db.session.commit()

@cli.command("seed_db")
def seed_db():
    db.session.add(User(email="[[email protected]](/cdn-cgi/l/email-protection)"))
    db.session.commit()

if __name__ == "__main__":
    cli()` 
```

尝试一下:

```py
`$ docker-compose exec web python manage.py seed_db

$ docker-compose exec db psql --username=hello_flask --dbname=hello_flask_dev

psql (13.8)
Type "help" for help.

hello_flask_dev=# \c hello_flask_dev
You are now connected to database "hello_flask_dev" as user "hello_flask".

hello_flask_dev=# select * from users;
 id |        email        | active
----+---------------------+--------
  1 | [[email protected]](/cdn-cgi/l/email-protection) | t
(1 row)

hello_flask_dev=# \q` 
```

> 尽管添加了 Postgres，我们仍然可以通过不设置`DATABASE_URL`环境变量来为 Flask 创建一个独立的 Docker 映像。要进行测试，构建一个新的映像，然后运行一个新的容器:
> 
> ```py
> $ docker build -f ./services/web/Dockerfile -t hello_flask:latest ./services/web
> $ docker run -p 5001:5000 \
>     -e "FLASK_APP=project/__init__.py" -e "FLASK_DEBUG=1" \
>     hello_flask python /usr/src/app/manage.py run -h 0.0.0.0 
> ```
> 
> 您应该能够在 http://localhost:5001 查看 hello world 健全性检查。

## 格尼科恩

接下来，对于生产环境，让我们将 [Gunicorn](https://gunicorn.org/) ，一个生产级的 WSGI 服务器，添加到需求文件中:

```py
`Flask==2.2.2
Flask-SQLAlchemy==2.5.1
gunicorn==20.1.0
psycopg2-binary==2.9.4` 
```

因为我们仍然希望在开发中使用 Flask 的内置服务器，所以为生产创建一个名为 *docker-compose.prod.yml* 的新合成文件:

```py
`version:  '3.8' services: web: build:  ./services/web command:  gunicorn --bind 0.0.0.0:5000 manage:app ports: -  5000:5000 env_file: -  ./.env.prod depends_on: -  db db: image:  postgres:13-alpine volumes: -  postgres_data_prod:/var/lib/postgresql/data/ env_file: -  ./.env.prod.db volumes: postgres_data_prod:` 
```

> 如果您有多个环境，您可能希望使用一个[docker-compose . override . yml](https://docs.docker.com/compose/extends/)配置文件。使用这种方法，您可以将您的基本配置添加到一个 *docker-compose.yml* 文件中，然后使用一个*docker-compose . override . yml*文件根据环境覆盖那些配置设置。

记下默认值`command`。我们运行的是 Gunicorn，而不是 Flask 开发服务器。我们还从`web`服务中删除了这个卷，因为我们在生产中不需要它。最后，我们使用[单独的环境变量文件](https://docs.docker.com/compose/env-file/)来定义两个服务的环境变量，它们将在运行时传递给容器。

*.env.prod* :

```py
`FLASK_APP=project/__init__.py FLASK_DEBUG=0 DATABASE_URL=postgresql://hello_flask:hello_flask@db:5432/hello_flask_prod SQL_HOST=db SQL_PORT=5432 DATABASE=postgres` 
```

*.env.prod.db* :

```py
`POSTGRES_USER=hello_flask
POSTGRES_PASSWORD=hello_flask
POSTGRES_DB=hello_flask_prod` 
```

将这两个文件添加到项目根目录。您可能想让它们不受版本控制，所以将它们添加到一个*中。gitignore* 文件。

将[下放到](https://docs.docker.com/compose/reference/down/)开发容器(以及带有`-v`标志的相关卷):

然后，构建生产映像并启动容器:

```py
`$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

验证`hello_flask_prod`数据库是与`users`表一起创建的。测试出 [http://localhost:5000/](http://localhost:5000/) 。

> 同样，如果容器启动失败，通过`docker-compose -f docker-compose.prod.yml logs -f`检查日志中的错误。

## 生产文档

您是否注意到我们仍然在运行`create_db`命令，每次运行容器时，该命令会删除所有现有的表，然后从模型中创建表。这在开发中很好，但是让我们为生产创建一个新的入口点文件。

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

> 或者，不创建新的入口点文件，您可以修改现有的文件，如下所示:
> 
> ```py
> #!/bin/sh
> 
> if [ "$DATABASE" = "postgres" ]
> then
>     echo "Waiting for postgres..."
> 
>     while ! nc -z $SQL_HOST $SQL_PORT; do
>       sleep 0.1
>     done
> 
>     echo "PostgreSQL started"
> fi
> 
> if [ "$FLASK_DEBUG" = "1" ]
> then
>     echo "Creating the database tables..."
>     python manage.py create_db
>     echo "Tables created"
> fi
> 
> exec "[[email protected]](/cdn-cgi/l/email-protection)" 
> ```

在本地更新文件权限:

```py
`$ chmod +x services/web/entrypoint.prod.sh` 
```

要使用这个文件，创建一个名为 *Dockerfile.prod* 的新 Dockerfile，用于生产构建:

```py
`###########
# BUILDER #
###########

# pull official base image
FROM  python:3.10.7-slim-buster  as  builder

# set work directory
WORKDIR  /usr/src/app

# set environment variables
ENV  PYTHONDONTWRITEBYTECODE 1
ENV  PYTHONUNBUFFERED 1

# install system dependencies
RUN  apt-get update && \
    apt-get install -y --no-install-recommends gcc

# lint
RUN  pip install --upgrade pip
RUN  pip install flake8==5.0.4
COPY  . /usr/src/app/
RUN  flake8 --ignore=E501,F401 .

# install python dependencies
COPY  ./requirements.txt .
RUN  pip wheel --no-cache-dir --no-deps --wheel-dir /usr/src/app/wheels -r requirements.txt

#########
# FINAL #
#########

# pull official base image
FROM  python:3.10.7-slim-buster

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
RUN  apt-get update && apt-get install -y --no-install-recommends netcat
COPY  --from=builder /usr/src/app/wheels /wheels
COPY  --from=builder /usr/src/app/requirements.txt .
RUN  pip install --upgrade pip
RUN  pip install --no-cache /wheels/*

# copy entrypoint-prod.sh
COPY  ./entrypoint.prod.sh $APP_HOME

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
`web: build: context:  ./services/web dockerfile:  Dockerfile.prod command:  gunicorn --bind 0.0.0.0:5000 manage:app ports: -  5000:5000 env_file: -  ./.env.prod depends_on: -  db` 
```

尝试一下:

```py
`$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py create_db` 
```

## Nginx

接下来，让我们将 Nginx 添加进来，充当 Gunicorn 的反向代理来处理客户端请求以及提供静态文件。

将服务添加到 *docker-compose.prod.yml* :

```py
`nginx: build:  ./services/nginx ports: -  1337:80 depends_on: -  web` 
```

然后，在“服务”目录中，创建以下文件和文件夹:

```py
`└── nginx
    ├── Dockerfile
    └── nginx.conf` 
```

*Dockerfile* :

```py
`FROM  nginx:1.23-alpine

RUN  rm /etc/nginx/conf.d/default.conf
COPY  nginx.conf /etc/nginx/conf.d` 
```

*engine . conf*:

```py
`upstream hello_flask {
    server web:5000;
}

server {

    listen 80;

    location / {
        proxy_pass http://hello_flask;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

}` 
```

> 回顾[如何为 Flask Web 应用配置 NGINX](https://www.patricksoftwareblog.com/how-to-configure-nginx-for-a-flask-web-application/)以获得更多关于配置 NGINX 与 Flask 一起工作的信息。

然后，更新`web`服务，在 *docker-compose.prod.yml* 中，用`expose`替换`ports`:

```py
`web: build: context:  ./services/web dockerfile:  Dockerfile.prod command:  gunicorn --bind 0.0.0.0:5000 manage:app expose: -  5000 env_file: -  ./.env.prod depends_on: -  db` 
```

现在，端口 5000 只在内部对其他 Docker 服务公开。该端口将不再发布到主机上。

> 关于端口与暴露的更多信息，请查看[这个](https://stackoverflow.com/questions/40801772/what-is-the-difference-between-docker-compose-ports-vs-expose)堆栈溢出问题。

再次测试:

```py
`$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py create_db` 
```

确保应用程序在 [http://localhost:1337](http://localhost:1337) 启动并运行。

您的项目结构现在应该看起来像这样:

```py
`├── .env.dev
├── .env.prod
├── .env.prod.db
├── .gitignore
├── docker-compose.prod.yml
├── docker-compose.yml
└── services
    ├── nginx
    │   ├── Dockerfile
    │   └── nginx.conf
    └── web
        ├── Dockerfile
        ├── Dockerfile.prod
        ├── entrypoint.prod.sh
        ├── entrypoint.sh
        ├── manage.py
        ├── project
        │   ├── __init__.py
        │   └── config.py
        └── requirements.txt` 
```

完成后将容器拿下来:

```py
`$ docker-compose -f docker-compose.prod.yml down -v` 
```

由于 Gunicorn 是一个应用服务器，它不会提供静态文件。那么，在这种特定的配置中，应该如何处理静态文件和媒体文件呢？

## 静态文件

首先在“服务/web/项目”文件夹中创建以下文件和文件夹:

给 *hello.txt* 添加一些文字:

向 *__init__ 添加一个新的路由处理程序。py* :

```py
`@app.route("/static/<path:filename>")
def staticfiles(filename):
    return send_from_directory(app.config["STATIC_FOLDER"], filename)` 
```

不要忘记导入[发送自目录](https://flask.palletsprojects.com/api/#flask.send_from_directory):

```py
`from flask import Flask, jsonify, send_from_directory` 
```

最后，将`STATIC_FOLDER`配置添加到*服务/web/project/config.py*

```py
`import os

basedir = os.path.abspath(os.path.dirname(__file__))

class Config(object):
    SQLALCHEMY_DATABASE_URI = os.getenv("DATABASE_URL", "sqlite://")
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    STATIC_FOLDER = f"{os.getenv('APP_FOLDER')}/project/static"` 
```

### 发展

将`APP_FOLDER`环境变量添加到 *.env.dev* :

```py
`FLASK_APP=project/__init__.py FLASK_DEBUG=1 DATABASE_URL=postgresql://hello_flask:hello_flask@db:5432/hello_flask_dev SQL_HOST=db SQL_PORT=5432 DATABASE=postgres APP_FOLDER=/usr/src/app` 
```

为了进行测试，首先重新构建映像，并像往常一样旋转新的容器。一旦完成，确保[http://localhost:5000/static/hello . txt](http://localhost:5000/static/hello.txt)正确地提供文件。

### 生产

对于生产，向 *docker-compose.prod.yml* 中的`web`和`nginx`服务添加一个卷，这样每个容器将共享一个名为“static”的目录:

```py
`version:  '3.8' services: web: build: context:  ./services/web dockerfile:  Dockerfile.prod command:  gunicorn --bind 0.0.0.0:5000 manage:app volumes: -  static_volume:/home/app/web/project/static expose: -  5000 env_file: -  ./.env.prod depends_on: -  db db: image:  postgres:13-alpine volumes: -  postgres_data_prod:/var/lib/postgresql/data/ env_file: -  ./.env.prod.db nginx: build:  ./services/nginx volumes: -  static_volume:/home/app/web/project/static ports: -  1337:80 depends_on: -  web volumes: postgres_data_prod: static_volume:` 
```

接下来，更新 Nginx 配置，将静态文件请求路由到“static”文件夹:

```py
`upstream hello_flask {
    server web:5000;
}

server {

    listen 80;

    location / {
        proxy_pass http://hello_flask;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        alias /home/app/web/project/static/;
    }

}` 
```

将`APP_FOLDER`环境变量添加到 *.env.prod* :

```py
`FLASK_APP=project/__init__.py FLASK_DEBUG=0 DATABASE_URL=postgresql://hello_flask:hello_flask@db:5432/hello_flask_prod SQL_HOST=db SQL_PORT=5432 DATABASE=postgres APP_FOLDER=/home/app/web` 
```

> 这个目录路径来自哪里？将该路径与添加到 *.env.dev* 的路径进行比较。它们为什么不同呢？

降低开发容器的转速:

测试:

```py
`$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

同样，对`http://localhost:1337/static/*`的请求将由“静态”目录提供服务。

导航到[http://localhost:1337/static/hello . txt](http://localhost:1337/static/hello.txt)并确保静态资产被正确加载。

您还可以通过`docker-compose -f docker-compose.prod.yml logs -f`在日志中验证对静态文件的请求是否通过 Nginx 成功提供:

```py
`192.168.80.1 - - [11/Oct/2022:15:20:25 +0000] "GET /static/hello.txt HTTP/1.1" 200 4 "-"
"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36" "-"` 
```

完成后带上容器:

```py
`$ docker-compose -f docker-compose.prod.yml down -v` 
```

为了测试对用户上传的媒体文件的处理，向 *__init__ 添加两个新的路由处理程序。py* :

```py
`@app.route("/media/<path:filename>")
def mediafiles(filename):
    return send_from_directory(app.config["MEDIA_FOLDER"], filename)

@app.route("/upload", methods=["GET", "POST"])
def upload_file():
    if request.method == "POST":
        file = request.files["file"]
        filename = secure_filename(file.filename)
        file.save(os.path.join(app.config["MEDIA_FOLDER"], filename))
    return """
 <!doctype html>
 <title>upload new File</title>
 <form action="" method=post enctype=multipart/form-data>
 <p><input type=file name=file><input type=submit value=Upload>
 </form>
 """` 
```

也更新导入:

```py
`import os

from flask import (
    Flask,
    jsonify,
    send_from_directory,
    request,
)
from flask_sqlalchemy import SQLAlchemy
from werkzeug.utils import secure_filename` 
```

将`MEDIA_FOLDER`配置添加到*services/web/project/config . py*:

```py
`import os

basedir = os.path.abspath(os.path.dirname(__file__))

class Config(object):
    SQLALCHEMY_DATABASE_URI = os.getenv("DATABASE_URL", "sqlite://")
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    STATIC_FOLDER = f"{os.getenv('APP_FOLDER')}/project/static"
    MEDIA_FOLDER = f"{os.getenv('APP_FOLDER')}/project/media"` 
```

最后，在“项目”文件夹中创建一个名为“媒体”的新文件夹。

### 发展

测试:

```py
`$ docker-compose up -d --build` 
```

你应该可以在[http://localhost:5000/upload](http://localhost:5000/upload)上传一张图片，然后在[http://localhost:5000/media/IMAGE _ FILE _ NAME](http://localhost:5000/media/IMAGE_FILE_NAME)查看图片。

### 生产

对于生产，向`web`和`nginx`服务添加另一个卷:

```py
`version:  '3.8' services: web: build: context:  ./services/web dockerfile:  Dockerfile.prod command:  gunicorn --bind 0.0.0.0:5000 manage:app volumes: -  static_volume:/home/app/web/project/static -  media_volume:/home/app/web/project/media expose: -  5000 env_file: -  ./.env.prod depends_on: -  db db: image:  postgres:13-alpine volumes: -  postgres_data_prod:/var/lib/postgresql/data/ env_file: -  ./.env.prod.db nginx: build:  ./services/nginx volumes: -  static_volume:/home/app/web/project/static -  media_volume:/home/app/web/project/media ports: -  1337:80 depends_on: -  web volumes: postgres_data_prod: static_volume: media_volume:` 
```

接下来，更新 Nginx 配置，将媒体文件请求路由到“media”文件夹:

```py
`upstream hello_flask {
    server web:5000;
}

server {

    listen 80;

    location / {
        proxy_pass http://hello_flask;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        alias /home/app/web/project/static/;
    }

    location /media/ {
        alias /home/app/web/project/media/;
    }

}` 
```

重建:

```py
`$ docker-compose down -v

$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py create_db` 
```

最后一次测试:

1.  在[http://localhost:1337/upload](http://localhost:1337/upload)上传一张图片。
2.  然后在[http://localhost:1337/media/IMAGE _ FILE _ NAME](http://localhost:1337/media/IMAGE_FILE_NAME)查看图片。

## 结论

在本教程中，我们介绍了如何用 Postgres 封装 Flask 应用程序以进行开发。我们还创建了一个生产就绪的 Docker Compose 文件，将 Gunicorn 和 Nginx 添加到混合文件中，以处理静态和媒体文件。现在，您可以在本地测试生产设置。

就生产环境的实际部署而言，您可能希望使用:

1.  完全托管的数据库服务——像 [RDS](https://aws.amazon.com/rds/) 或[云 SQL](https://cloud.google.com/sql/)——而不是在一个容器中管理你自己的 Postgres 实例。
2.  `db`和`nginx`服务的非根用户

你可以在[码头上的烧瓶](https://github.com/testdrivenio/flask-on-docker)报告中找到代码。

感谢阅读！