# 带有 Postgres、Gunicorn 和 Traefik 的多用途烧瓶

> 原文：<https://testdriven.io/blog/flask-docker-traefik/>

在本教程中，我们将看看如何用 Postgres 和 Docker 设置 Flask。对于生产环境，我们将添加 Gunicorn、Traefik，并进行加密。

## 项目设置

首先创建一个项目目录:

```
`$ mkdir flask-docker-traefik && cd flask-docker-traefik
$ python3.9 -m venv venv
$ source venv/bin/activate
(venv)$` 
```

> 你可以随意把 virtualenv 和 Pip 换成诗歌[或](https://python-poetry.org) [Pipenv](https://github.com/pypa/pipenv) 。更多信息，请查看[现代 Python 环境](https://testdriven.io/blog/python-environments/)。

然后，创建以下文件和文件夹:

```
`└── services
    └── web
        ├── manage.py
        ├── project
        │   └── __init__.py
        └── requirements.txt` 
```

将[烧瓶](https://flask.palletsprojects.com/en/2.0.x/)添加到*要求. txt* 中:

从“服务/web”安装软件包:

```
`(venv)$ pip install -r requirements.txt` 
```

接下来，让我们在 *__init.py__* 中创建一个简单的 Flask 应用程序:

```
`from flask import Flask, jsonify

app = Flask(__name__)

@app.get("/")
def read_root():
    return jsonify(hello="world")` 
```

然后，要配置 Flask CLI 工具从命令行运行和管理应用程序，请将以下内容添加到 *services/web/manage.py* :

```
`from flask.cli import FlaskGroup

from project import app

cli = FlaskGroup(app)

if __name__ == "__main__":
    cli()` 
```

这里，我们创建了一个新的`FlaskGroup`实例，用与 Flask 应用程序相关的命令来扩展普通 CLI。

从“web”目录运行服务器:

```
`(venv)$ export FLASK_APP=project/__init__.py
(venv)$ python manage.py run` 
```

导航到 [127.0.0.1:5000](http://127.0.0.1:5000/) ，您应该看到:

一旦完成就杀死服务器。退出虚拟环境，并将其删除。

## 码头工人

安装 [Docker](https://docs.docker.com/install/) ，如果你还没有，那么在“web”目录下添加一个 *Dockerfile* :

```
`# pull the official docker image
FROM  python:3.9.5-slim

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

所以，我们从 Python 3.9.5 的基于`slim`的 [Docker 镜像](https://hub.docker.com/_/python/)开始。然后我们设置一个[工作目录](https://docs.docker.com/engine/reference/builder/#workdir)以及两个环境变量:

1.  `PYTHONDONTWRITEBYTECODE`:防止 Python 将 pyc 文件写入磁盘(相当于`python -B` [选项](https://docs.python.org/3/using/cmdline.html#cmdoption-B)
2.  `PYTHONUNBUFFERED`:防止 Python 缓冲 stdout 和 stderr(相当于`python -u` [选项](https://docs.python.org/3/using/cmdline.html#cmdoption-u)

最后，我们复制了 *requirements.txt* 文件，安装了依赖项，并复制了 Flask 应用程序本身。

> 查看[Docker for Python Developers](https://mherman.org/presentations/dockercon-2018)了解更多关于构造 Docker 文件的信息，以及为基于 Python 的开发配置 Docker 的一些最佳实践。

接下来，将一个 *docker-compose.yml* 文件添加到项目根:

```
`version:  '3.8' services: web: build:  ./services/web command:  python manage.py run -h 0.0.0.0 volumes: -  ./services/web/:/app ports: -  5000:5000 environment: -  FLASK_APP=project/__init__.py -  FLASK_ENV=development` 
```

> 查看[合成文件参考](https://docs.docker.com/compose/compose-file/)，了解该文件如何工作的信息。

建立形象:

构建映像后，运行容器:

导航到 [http://127.0.0.1:5000/](http://127.0.0.1:5000/) 再次查看 hello world 健全性检查。

> 如果这不起作用，通过`docker-compose logs -f`检查日志中的错误。

## Postgres

要配置 Postgres，我们需要在 *docker-compose.yml* 文件中添加一个新服务，设置 [Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com/) ，安装 [Psycopg2](https://www.psycopg.org/) 。

首先，向 *docker-compose.yml* 添加一个名为`db`的新服务:

```
`version:  '3.8' services: web: build:  ./services/web command:  bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; python manage.py run -h 0.0.0.0' volumes: -  ./services/web/:/app ports: -  5000:5000 environment: -  FLASK_APP=project/__init__.py -  FLASK_ENV=development -  DATABASE_URL=postgresql://hello_flask:[[email protected]](/cdn-cgi/l/email-protection):5432/hello_flask_dev depends_on: -  db db: image:  postgres:13-alpine volumes: -  postgres_data:/var/lib/postgresql/data/ environment: -  POSTGRES_USER=hello_flask -  POSTGRES_PASSWORD=hello_flask -  POSTGRES_DB=hello_flask_dev volumes: postgres_data:` 
```

为了在容器的生命周期之外保存数据，我们配置了一个卷。这个配置将把`postgres_data`绑定到容器中的“/var/lib/postgresql/data/”目录。

我们还添加了一个环境键来定义默认数据库的名称，并设置用户名和密码。

> 查看 [Postgres Docker Hub 页面](https://hub.docker.com/_/postgres)的“环境变量”部分了解更多信息。

注意`web`服务中的新命令:

```
`bash  -c  'while !</dev/tcp/db/5432; do sleep 1; done; python manage.py run -h 0.0.0.0'` 
```

将持续到 Postgres 完成。一旦启动，`python manage.py run -h 0.0.0.0`就会运行。

然后，将一个名为 *config.py* 的新文件添加到“项目”目录中，在这里我们将定义特定于环境的[配置](https://flask.palletsprojects.com/config/)变量:

```
`import os

class Config(object):
    SQLALCHEMY_DATABASE_URI = os.getenv("DATABASE_URL", "sqlite://")
    SQLALCHEMY_TRACK_MODIFICATIONS = False` 
```

这里，数据库是基于我们刚刚定义的`DATABASE_URL`环境变量配置的。记下默认值。

更新 *__init__。py* 在 init 上拉入配置:

```
`from flask import Flask, jsonify

app = Flask(__name__)
app.config.from_object("project.config.Config")

@app.get("/")
def read_root():
    return jsonify(hello="world")` 
```

将 [Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com/) 和 [Psycopg2](https://www.psycopg.org/) 添加到 *requirements.txt* :

```
`Flask==2.0.1
Flask-SQLAlchemy==2.5.1
psycopg2-binary==2.8.6` 
```

更新 *__init__。py* 再次创建一个新的`SQLAlchemy`实例并定义一个数据库模型:

```
`from dataclasses import dataclass

from flask import Flask, jsonify
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config.from_object("project.config.Config")
db = SQLAlchemy(app)

@dataclass
class User(db.Model):
    id: int = db.Column(db.Integer, primary_key=True)
    email: str = db.Column(db.String(120), unique=True, nullable=False)
    active: bool = db.Column(db.Boolean(), default=True, nullable=False)

    def __init__(self, email: str) -> None:
        self.email = email

@app.get("/")
def read_root():
    users = User.query.all()
    return jsonify(users)` 
```

在数据库模型上使用 [dataclass](https://docs.python.org/3/library/dataclasses.html) decorator 有助于我们序列化数据库对象。

最后，更新 *manage.py* :

```
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

```
`$ docker-compose up -d --build` 
```

创建表格:

```
`$ docker-compose exec web python manage.py create_db` 
```

> 得到以下错误？
> 
> ```
> sqlalchemy.exc.OperationalError: (psycopg2.OperationalError)
> FATAL:  database "hello_flask_dev" does not exist 
> ```
> 
> 运行`docker-compose down -v`移除卷和容器。然后，重新构建映像，运行容器，并应用迁移。

确保`users`表已创建:

```
`$ docker-compose exec db psql --username=hello_flask --dbname=hello_flask_dev

psql (13.3)
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
 Schema | Name | Type  |    Owner
--------+------+-------+-------------
 public | user | table | hello_flask
(1 row)

hello_flask_dev=# \q` 
```

您也可以通过运行以下命令来检查该卷是否已创建:

```
`$ docker volume inspect flask-docker-traefik_postgres_data` 
```

您应该会看到类似如下的内容:

```
`[
    {
        "CreatedAt": "2021-06-05T14:12:52Z",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "flask-docker-traefik",
            "com.docker.compose.version": "1.29.1",
            "com.docker.compose.volume": "postgres_data"
        },
        "Mountpoint": "/var/lib/docker/volumes/flask-docker-traefik_postgres_data/_data",
        "Name": "flask-docker-traefik_postgres_data",
        "Options": null,
        "Scope": "local"
    }
]` 
```

导航到 [http://127.0.0.1:5000](http://127.0.0.1:5000) 。健全性检查显示一个空列表。这是因为我们还没有填充`users`表。让我们添加一个 CLI 种子命令，用于将样本`users`添加到 *manage.py* 中的用户表:

```
`from flask.cli import FlaskGroup

from project import User, app, db

cli = FlaskGroup(app)

@cli.command("create_db")
def create_db():
    db.drop_all()
    db.create_all()
    db.session.commit()

@cli.command("seed_db") # new
def seed_db():
    db.session.add(User(email="[[email protected]](/cdn-cgi/l/email-protection)"))
    db.session.add(User(email="[[email protected]](/cdn-cgi/l/email-protection)"))
    db.session.commit()

if __name__ == "__main__":
    cli()` 
```

尝试一下:

```
`$ docker-compose exec web python manage.py seed_db` 
```

再次导航到 [http://127.0.0.1:5000](http://127.0.0.1:5000) 。您现在应该看到:

## 格尼科恩

接下来，对于生产环境，让我们将 [Gunicorn](https://gunicorn.org/) ，一个生产级的 WSGI 服务器，添加到需求文件中:

```
`Flask==2.0.1
Flask-SQLAlchemy==2.5.1
gunicorn==20.1.0
psycopg2-binary==2.8.6` 
```

因为我们仍然希望在开发中使用 Flask 的内置服务器，所以在项目根目录中创建一个名为 *docker-compose.prod.yml* 的新合成文件用于生产:

```
`version:  '3.8' services: web: build:  ./services/web command:  bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; gunicorn --bind 0.0.0.0:5000 manage:app' ports: -  5000:5000 environment: -  FLASK_APP=project/__init__.py -  FLASK_ENV=production -  DATABASE_URL=postgresql://hello_flask:[[email protected]](/cdn-cgi/l/email-protection):5432/hello_flask_prod depends_on: -  db db: image:  postgres:13-alpine volumes: -  postgres_data_prod:/var/lib/postgresql/data/ environment: -  POSTGRES_USER=hello_flask -  POSTGRES_PASSWORD=hello_flask -  POSTGRES_DB=hello_flask_prod volumes: postgres_data_prod:` 
```

> 如果您有多个环境，您可能希望使用一个[docker-compose . override . yml](https://docs.docker.com/compose/extends/)配置文件。使用这种方法，您可以将您的基本配置添加到一个 *docker-compose.yml* 文件中，然后使用一个*docker-compose . override . yml*文件根据环境覆盖那些配置设置。

记下默认值`command`。我们运行的是 Gunicorn，而不是 Flask 开发服务器。我们还从`web`服务中删除了这个卷，因为我们在生产中不需要它。

关闭开发容器(以及带有-v 标志的相关卷):

然后，构建生产映像并启动容器:

```
`$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

创建表格并应用种子:

```
`$ docker-compose -f docker-compose.prod.yml exec web python manage.py create_db
$ docker-compose -f docker-compose.prod.yml exec web python manage.py seed_db` 
```

验证`hello_flask_prod`数据库是与`users`表一起创建的。测试出 [http://127.0.0.1:5000/](http://127.0.0.1:5000/) 。

> 同样，如果容器启动失败，通过`docker-compose -f docker-compose.prod.yml logs -f`检查日志中的错误。

## 生产文档

在“web”目录中创建一个名为 *Dockerfile.prod* 的新 Dockerfile，用于生产构建:

```
`###########
# BUILDER #
###########

# pull official base image
FROM  python:3.9.5-slim  as  builder

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
RUN  pip install flake8==3.9.1
COPY  . .
RUN  flake8 --ignore=E501,F401 .

# install python dependencies
COPY  ./requirements.txt .
RUN  pip wheel --no-cache-dir --no-deps --wheel-dir /usr/src/app/wheels -r requirements.txt

#########
# FINAL #
#########

# pull official base image
FROM  python:3.9.5-slim

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

用 *Dockerfile.prod* 更新 docker-compose.prod.yml 文件中的`web`服务:

```
`web: build: context:  ./services/web dockerfile:  Dockerfile.prod command:  bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; gunicorn --bind 0.0.0.0:5000 manage:app' ports: -  5000:5000 environment: -  FLASK_APP=project/__init__.py -  FLASK_ENV=production -  DATABASE_URL=postgresql://hello_flask:[[email protected]](/cdn-cgi/l/email-protection):5432/hello_flask_prod depends_on: -  db` 
```

尝试一下:

```
`$ docker-compose -f docker-compose.prod.yml down -v
$ docker-compose -f docker-compose.prod.yml up -d --build
$ docker-compose -f docker-compose.prod.yml exec web python manage.py create_db
$ docker-compose -f docker-compose.prod.yml exec web python manage.py seed_db` 
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
> 2.  比 Traefik 稍快
> 3.  对复杂的服务使用 Nginx

将名为“traefik”的新文件夹与以下文件一起添加到“services”目录中:

```
`traefik
├── Dockerfile.traefik
├── traefik.dev.toml
└── traefik.prod.toml` 
```

您的项目结构现在应该如下所示:

```
`├── docker-compose.prod.yml
├── docker-compose.yml
└── services
    ├── traefik
    │   ├── Dockerfile.traefik
    │   ├── traefik.dev.toml
    │   └── traefik.prod.toml
    └── web
        ├── Dockerfile
        ├── Dockerfile.prod
        ├── manage.py
        ├── project
        │   ├── __init__.py
        │   └── config.py
        └── requirements.txt` 
```

将以下内容添加到 *traefik.dev.toml* 中:

```
`# listen on port 80 [entryPoints] [entryPoints.web] address  =  ":80" # Traefik dashboard over http [api] insecure  =  true [log] level  =  "DEBUG" [accessLog] # containers are not discovered automatically [providers] [providers.docker] exposedByDefault  =  false` 
```

在这里，由于我们不想公开`db`服务，我们将 [exposedByDefault](https://doc.traefik.io/traefik/providers/docker/#exposedbydefault) 设置为`false`。要手动公开服务，我们可以将`"traefik.enable=true"`标签添加到 Docker 组合文件中。

接下来，更新 *docker-compose.yml* 文件，以便 Traefik 发现我们的`web`服务并添加一个新的`traefik`服务:

```
`version:  '3.8' services: web: build:  ./services/web command:  bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; python manage.py run -h 0.0.0.0' volumes: -  ./services/web/:/app expose:  # new -  5000 environment: -  FLASK_APP=project/__init__.py -  FLASK_ENV=development -  DATABASE_URL=postgresql://hello_flask:[[email protected]](/cdn-cgi/l/email-protection):5432/hello_flask_dev depends_on: -  db labels:  # new -  "traefik.enable=true" -  "traefik.http.routers.flask.rule=Host(`flask.localhost`)" db: image:  postgres:13-alpine volumes: -  postgres_data:/var/lib/postgresql/data/ environment: -  POSTGRES_USER=hello_flask -  POSTGRES_PASSWORD=hello_flask -  POSTGRES_DB=hello_flask_dev traefik:  # new image:  traefik:v2.2 ports: -  80:80 -  8081:8080 volumes: -  "./services/traefik/traefik.dev.toml:/etc/traefik/traefik.toml" -  "/var/run/docker.sock:/var/run/docker.sock:ro" volumes: postgres_data:` 
```

首先，`web`服务只对端口`5000`上的其他容器公开。我们还为`web`服务添加了以下标签:

1.  `traefik.enable=true`使 Traefik 能够发现服务
2.  `traefik.http.routers.flask.rule=Host(`flask.localhost`)`当请求有`Host=flask.localhost`时，请求被重定向到该服务

记下`traefik`服务中的卷:

1.  将本地配置文件映射到容器中的配置文件，以便保持设置同步
2.  `/var/run/docker.sock:/var/run/docker.sock:ro`使 Traefik 能够发现其他容器

要进行测试，首先取下任何现有的容器:

```
`$ docker-compose down -v
$ docker-compose -f docker-compose.prod.yml down -v` 
```

构建新的开发映像并启动容器:

```
`$ docker-compose up -d --build` 
```

创建表格并应用种子:

```
`$ docker-compose exec web python manage.py create_db
$ docker-compose exec web python manage.py seed_db` 
```

导航到 [http://flask.localhost](http://flask.localhost) 。您应该看到:

您也可以通过 cURL 进行测试:

```
`$ curl -H Host:flask.localhost http://0.0.0.0` 
```

接下来，在[查看](http://flask.localhost:8081)[仪表盘](https://doc.traefik.io/traefik/operations/dashboard/)http://flask . localhost:8081:

![traefik dashboard](img/c388e49cdf834296ca304b677e08a153.png)

完成后，将容器和体积拿下来:

## 让我们加密

我们已经在开发模式下成功地创建了 Flask、Docker 和 Traefik 的工作示例。对于生产，您需要配置 Traefik 来通过 Let's Encrypt 管理 TLS 证书。简而言之，Traefik 将自动联系证书颁发机构来颁发和续订证书。

因为 Let's Encrypt 不会为`localhost`颁发证书，所以你需要在云计算实例(比如 [DigitalOcean](https://m.do.co/c/d8f211a4b4c2) droplet 或 AWS EC2 实例)上运行你的生产容器。您还需要一个有效的域名。如果你没有，你可以在 [Freenom](https://www.freenom.com/) 创建一个免费域名。

> 我们使用了一个 [DigitalOcean](https://m.do.co/c/d8f211a4b4c2) droplet 和 Docker machine 来快速配置 Docker 的计算实例，并部署了生产容器来测试 Traefik 配置。查看 Docker 文档中的 [DigitalOcean 示例](https://docs.docker.com/machine/examples/ocean/)，了解更多关于使用 Docker 机器供应 droplet 的信息。

假设您配置了一个计算实例并设置了一个自由域，那么现在就可以在生产模式下设置 Traefik 了。

首先将 Traefik 配置的生产版本添加到 *traefik.prod.toml* :

```
`[entryPoints] [entryPoints.web] address  =  ":80" [entryPoints.web.http] [entryPoints.web.http.redirections] [entryPoints.web.http.redirections.entryPoint] to  =  "websecure" scheme  =  "https" [entryPoints.websecure] address  =  ":443" [accessLog] [api] dashboard  =  true [providers] [providers.docker] exposedByDefault  =  false [certificatesResolvers.letsencrypt.acme] email  =  "[[email protected]](/cdn-cgi/l/email-protection)" storage  =  "/certificates/acme.json" [certificatesResolvers.letsencrypt.acme.httpChallenge] entryPoint  =  "web"` 
```

> 确保用您的实际电子邮件地址替换`[[email protected]](/cdn-cgi/l/email-protection)`。

这里发生了什么:

1.  将我们不安全的 HTTP 应用程序的入口点设置为端口 80
2.  将我们的安全 HTTPS 应用程序的入口点设置为端口 443
3.  将所有不安全的请求重定向到安全端口
4.  `exposedByDefault = false`取消所有服务
5.  `dashboard = true`启用监控仪表板

最后，请注意:

```
`[certificatesResolvers.letsencrypt.acme] email  =  "[[email protected]](/cdn-cgi/l/email-protection)" storage  =  "/certificates/acme.json" [certificatesResolvers.letsencrypt.acme.httpChallenge] entryPoint  =  "web"` 
```

这是让我们加密配置的地方。我们定义了证书的存储位置(T1)和验证类型(T3)，这是一个 HTTP 挑战(T5)。

接下来，假设您更新了域名的 DNS 记录，创建两个新的 A 记录，它们都指向您的计算实例的公共 IP:

1.  `flask-traefik.your-domain.com` -用于网络服务
2.  `dashboard-flask-traefik.your-domain.com` -用于 Traefik 仪表板

> 确保用您的实际域名替换`your-domain.com`。

接下来，像这样更新 *docker-compose.prod.yml* :

```
`version:  '3.8' services: web: build: context:  ./services/web dockerfile:  Dockerfile.prod command:  bash -c 'while !</dev/tcp/db/5432; do sleep 1; done; gunicorn --bind 0.0.0.0:5000 manage:app' expose:  # new -  5000 environment: -  FLASK_APP=project/__init__.py -  FLASK_ENV=production -  DATABASE_URL=postgresql://hello_flask:[[email protected]](/cdn-cgi/l/email-protection):5432/hello_flask_prod depends_on: -  db labels:  # new -  "traefik.enable=true" -  "traefik.http.routers.flask.rule=Host(`flask-traefik.your-domain.com`)" -  "traefik.http.routers.flask.tls=true" -  "traefik.http.routers.flask.tls.certresolver=letsencrypt" db: image:  postgres:13-alpine volumes: -  postgres_data_prod:/var/lib/postgresql/data/ environment: -  POSTGRES_USER=hello_flask -  POSTGRES_PASSWORD=hello_flask -  POSTGRES_DB=hello_flask_prod traefik:  # new build: context:  ./services/traefik dockerfile:  Dockerfile.traefik ports: -  80:80 -  443:443 volumes: -  "/var/run/docker.sock:/var/run/docker.sock:ro" -  "./traefik-public-certificates:/certificates" labels: -  "traefik.enable=true" -  "traefik.http.routers.dashboard.rule=Host(`dashboard-flask-traefik.your-domain.com`)" -  "traefik.http.routers.dashboard.tls=true" -  "traefik.http.routers.dashboard.tls.certresolver=letsencrypt" -  "[[email protected]](/cdn-cgi/l/email-protection)" -  "traefik.http.routers.dashboard.middlewares=auth" -  "traefik.http.middlewares.auth.basicauth.users=testuser:$$apr1$$jIKW.bdS$$eKXe4Lxjgy/rH65wP1iQe1" volumes: postgres_data_prod: traefik-public-certificates:` 
```

> 同样，确保用您的实际域名替换`your-domain.com`。

这里有什么新鲜事？

在`web`服务中，我们添加了以下标签:

1.  `traefik.http.routers.flask.rule=Host(`flask-traefik.your-domain.com`)`将主机更改为实际的域
2.  `traefik.http.routers.flask.tls=true`启用 HTTPS
3.  `traefik.http.routers.flask.tls.certresolver=letsencrypt`将证书颁发者设置为让我们加密

接下来，对于`traefik`服务，我们为证书目录添加了适当的端口和一个卷。该卷确保即使容器关闭，证书仍然有效。

至于标签:

1.  `traefik.http.routers.dashboard.rule=Host(`dashboard-flask-traefik.your-domain.com`)`定义仪表板主机，因此可以在`$Host/dashboard/`访问
2.  `traefik.http.routers.dashboard.tls=true`启用 HTTPS
3.  `traefik.http.routers.dashboard.tls.certresolver=letsencrypt`将证书解析器设置为“让我们加密”
4.  `traefik.http.routers.dashboard.middlewares=auth`启用`HTTP BasicAuth`中间件
5.  `traefik.http.middlewares.auth.basicauth.users`定义用于登录的用户名和散列密码

您可以使用 htpasswd 实用程序创建新的密码哈希:

```
`# username: testuser
# password: password

$ echo $(htpasswd -nb testuser password) | sed -e s/\\$/\\$\\$/g
testuser:$$apr1$$jIKW.bdS$$eKXe4Lxjgy/rH65wP1iQe1` 
```

随意使用一个`env_file`来存储用户名和密码作为环境变量

```
`USERNAME=testuser
HASHED_PASSWORD=$$apr1$$jIKW.bdS$$eKXe4Lxjgy/rH65wP1iQe1` 
```

Update *Dockerfile.traefik* :

```
`FROM  traefik:v2.2

COPY  ./traefik.prod.toml ./etc/traefik/traefik.toml` 
```

接下来，旋转新容器:

```
`$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

创建表格并应用种子:

```
`$ docker-compose -f docker-compose.prod.yml exec web python manage.py create_db
$ docker-compose -f docker-compose.prod.yml exec web python manage.py seed_db` 
```

确保这两个 URL 有效:

1.  [https://flask-traefik.your-domain.com](https://flask-traefik.your-domain.com)
2.  [https://dashboard-flask-traefik.your-domain.com/dashboard/](https://dashboard-flask-traefik.your-domain.com/dashboard/)

此外，请确保当您访问上述网址的 HTTP 版本时，您会被重定向到 HTTPS 版本。

最后，让我们加密有效期为 [90 天](https://letsencrypt.org/2015/11/09/why-90-days.html)的证书。Treafik 将在后台自动为您处理证书更新，这样您就少了一件担心的事情！

## 结论

在本教程中，我们介绍了如何用 Postgres 封装 Flask 应用程序以进行开发。我们还创建了一个生产就绪的 Docker 组合文件，设置了 Traefik 和 Let's Encrypt 来通过 HTTPS 为应用程序提供服务，并启用了一个安全的仪表板来监控我们的服务。

就生产环境的实际部署而言，您可能希望使用:

1.  完全托管的数据库服务——像 [RDS](https://aws.amazon.com/rds/) 或[云 SQL](https://cloud.google.com/sql/)——而不是在一个容器中管理你自己的 Postgres 实例。
2.  服务的非根用户

你可以在[flask-docker-traefik](https://github.com/testdrivenio/flask-docker-traefik)repo 中找到代码。