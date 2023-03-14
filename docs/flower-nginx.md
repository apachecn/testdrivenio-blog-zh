# 生产中的流水作业

> 原文：<https://testdriven.io/blog/flower-nginx/>

在本教程中，我们将看看如何用 Docker 配置 [Flower](https://flower.readthedocs.io/) 在 Nginx 后面运行。我们还将设置基本身份验证。

## 码头工人

假设您使用 Redis 作为您的消息代理，您的 Docker 编写配置将类似于:

```
`version:  '3.7' services: redis: image:  redis expose: -  6379 worker: build: context:  . dockerfile:  Dockerfile command:  ['celery',  '-A',  'app.app',  'worker',  '-l',  'info'] environment: -  BROKER_URL=redis://redis:6379 -  RESULT_BACKEND=redis://redis:6379 depends_on: -  redis flower: image:  mher/flower:0.9.7 command:  ['flower',  '--broker=redis://redis:6379',  '--port=5555'] ports: -  5557:5555 depends_on: -  redis` 
```

> 截至发稿时，[官方 Flower Docker 图像](https://hub.docker.com/r/mher/flower)还没有版本> 0.9.7 的标签，这就是为什么使用了`0.9.7`标签。更多信息见[1 . 0 . 0 版本的 Docker 标签](https://github.com/mher/flower/issues/1140)。
> 
> 想检查正在使用的版本吗？运行`docker-compose exec flower pip freeze`。
> 
> 本教程用花版本 [0.9.7](https://github.com/mher/flower/releases/tag/v0.9.7) 和芹菜 [5.2.7](https://github.com/celery/celery/releases/tag/v5.2.7) 进行了测试。

您可以在 GitHub 上的[芹菜花码头](https://github.com/testdrivenio/celery-flower-docker) repo 中查看这个样本代码。

*app.py* :

```
`import os

from celery import Celery

os.environ.setdefault('CELERY_CONFIG_MODULE', 'celery_config')

app = Celery('app')
app.config_from_envvar('CELERY_CONFIG_MODULE')

@app.task
def add(x, y):
    return x + y` 
```

*芹菜 _ 配置. py* :

```
`from os import environ

broker_url = environ['BROKER_URL']
result_backend = environ['RESULT_BACKEND']` 
```

*Dockerfile* :

```
`FROM  python:3.10

WORKDIR  /usr/src/app

RUN  pip install celery[redis]==5.2.7

COPY  app.py .
COPY  celery_config.py .` 
```

快速测试:

```
`$ docker-compose build
$ docker-compose up -d --build
$ docker-compose exec worker python

>> from app import add
>> task = add.delay(5, 5)
>>>
>>> task.status
'SUCCESS'
>>> task.result
10` 
```

您应该能够在 [http://localhost:5557/](http://localhost:5557/) 查看仪表板。

## Nginx

要在 [Nginx](https://www.nginx.com) 之后运行 flower，首先将 Nginx 添加到 Docker Compose 配置中:

```
`version:  '3.7' services: redis: image:  redis expose: -  6379 worker: build: context:  . dockerfile:  Dockerfile command:  ['celery',  '-A',  'app.app',  'worker',  '-l',  'info'] environment: -  BROKER_URL=redis://redis:6379 -  RESULT_BACKEND=redis://redis:6379 depends_on: -  redis flower: image:  mher/flower:0.9.7 command:  ['flower',  '--broker=redis://redis:6379',  '--port=5555'] expose:  # new -  5555 depends_on: -  redis # new nginx: image:  nginx:latest volumes: -  ./nginx.conf:/etc/nginx/nginx.conf ports: -  80:80 depends_on: -  flower` 
```

*engine . conf*:

```
`events  {} http  { server  { listen  80; #  server_name  your.server.url; location  /  { proxy_pass  http://flower:5555; proxy_set_header  Host  $host; proxy_redirect  off; proxy_http_version  1.1; proxy_set_header  Upgrade  $http_upgrade; proxy_set_header  Connection  "upgrade"; } } }` 
```

快速测试:

```
`$ docker-compose down
$ docker-compose build
$ docker-compose up -d --build
$ docker-compose exec worker python

>> from app import add
>> task = add.delay(7, 7)
>>>
>>> task.status
'SUCCESS'
>>> task.result
14` 
```

这一次，仪表板应该可以在 [http://localhost/](http://localhost/) 上看到。

## 证明

要添加基本认证，首先创建一个 *htpasswd* 文件。例如:

```
`$ htpasswd -c htpasswd michael` 
```

接下来，向`nginx`服务添加另一个卷，以将 *htpasswd* 从主机挂载到 */etc/nginx/。容器中的 htpasswd* :

```
`nginx: image:  nginx:latest volumes: -  ./nginx.conf:/etc/nginx/nginx.conf -  ./htpasswd:/etc/nginx/.htpasswd  # new ports: -  80:80 depends_on: -  flower` 
```

最后，为了保护“/”路由，将 [auth_basic](http://nginx.org/en/docs/http/ngx_http_auth_basic_module.html#auth_basic) 和 [auth_basic_user_file](http://nginx.org/en/docs/http/ngx_http_auth_basic_module.html#auth_basic_user_file) 指令添加到位置块:

```
`events  {} http  { server  { listen  80; #  server_name  your.server.url; location  /  { proxy_pass  http://flower:5555; proxy_set_header  Host  $host; proxy_redirect  off; proxy_http_version  1.1; proxy_set_header  Upgrade  $http_upgrade; proxy_set_header  Connection  "upgrade"; auth_basic  "Restricted"; auth_basic_user_file  /etc/nginx/.htpasswd; } } }` 
```

> 有关使用 Nginx 设置基本身份验证的更多信息，请查看使用 HTTP 基本身份验证限制访问指南。

最终测试:

```
`$ docker-compose down
$ docker-compose build
$ docker-compose up -d --build
$ docker-compose exec worker python

>> from app import add
>> task = add.delay(9, 9)
>>>
>>> task.status
'SUCCESS'
>>> task.result
18` 
```

再次导航到 [http://localhost/](http://localhost/) 。这一次应该会提示您输入用户名和密码。