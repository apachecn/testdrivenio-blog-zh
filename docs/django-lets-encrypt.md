# 用加密保护容器化的 Django 应用程序

> 原文：<https://testdriven.io/blog/django-lets-encrypt/>

如何为 Django 应用程序设置 SSL 证书？

在本教程中，我们将看看如何使用[加密](https://letsencrypt.org/) SSL 证书来保护运行在 HTTPS Nginx 代理后面的容器化 Django 应用程序。

本教程建立在用 Postgres、Gunicorn 和 Nginx 构建 Django 的基础上。它假设您了解如何将 Django 应用程序与 Postgres、Nginx 和 Gunicorn 一起容器化。

如今，你根本无法让你的应用程序运行在 HTTP 之上。没有 HTTPS，你的网站就不那么安全和可信。“让我们加密”简化了获取和安装 SSL 证书的过程，再也没有理由不使用 HTTPS 了。

> 姜戈码头系列:
> 
> 1.  [用 Postgres、Gunicorn 和 Nginx 将 Django 归档](/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/)
> 2.  [使用 Let's Encrypt](/blog/django-lets-encrypt/) (本文！)
> 3.  [用 Docker 将 Django 部署到 AWS，让我们加密](/blog/django-docker-https-aws/)

## 先决条件

要学习本教程，您需要:

> 需要一个便宜的域名来练习？几个域名注册商有特殊优惠。“xyz”域。或者，您可以在 [Freenom](https://www.freenom.com/) 创建一个免费域名。

## 方法

有许多不同的方法来保护 HTTPS 的容器化 Django 应用程序。可以说，最流行的方法是向 Docker Compose 文件添加一个新的服务，该服务利用 [Certbot](https://certbot.eff.org/) 来发布和更新 SSL 证书。虽然这是完全正确的，但我们将采用稍微不同的方法，并使用以下项目:

1.  [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) -用于自动构建运行容器的 nginx 代理配置，其中每个容器都被视为单个虚拟主机
2.  Let sencrypt-nginx-proxy-companion-用于为 nginx-proxy 代理的每个容器颁发和更新加密 SSL 证书

总之，这些项目简化了 Nginx 配置和 SSL 证书的管理。

> 另一种选择是使用 [Traefik](https://traefik.io/) 而不是 Nginx。简而言之，Traefik 使用 Let's Encrypt 来发布和更新证书。更多信息，请查看[Dockerizing Django with Postgres、Gunicorn 和 Traefik](/blog/django-docker-traefik/) 。

## 让我们加密

首次部署应用程序时，您应该遵循以下两个步骤来避免证书问题:

1.  首先从 Let's Encrypt 的[登台环境](https://letsencrypt.org/docs/staging-environment/)中颁发证书
2.  然后，当一切按预期运行时，切换到 Let's Encrypt 的生产环境

为什么？

为了保护他们的服务器，让我们对他们的生产验证系统实施[速率限制](https://letsencrypt.org/docs/rate-limits/):

1.  每个帐户、每个主机名每小时 5 次验证失败
2.  每个域每周可以创建 50 个证书

如果您在域名或 DNS 条目或任何类似的地方输入错误，您的请求将失败，这将计入您的费率限制，您将不得不尝试颁发新的证书。

为了避免速率受限，在开发和测试期间，您应该使用 Let's Encrypt 的登台环境来测试他们的验证系统。在分段环境中，速率限制要高得多，这对测试来说更好。请注意，暂存中颁发的证书并不公开可信，因此一旦一切正常，您应该切换到他们的生产环境。

## 项目设置

首先，克隆 GitHub 项目报告的内容:

```py
`$ git clone https://github.com/testdrivenio/django-on-docker django-on-docker-letsencrypt
$ cd django-on-docker-letsencrypt` 
```

这个库包含了部署 Dockerized Django 应用程序所需的一切，但不包括 SSL 证书，我们将在本教程中添加 SSL 证书。

## Django 配置

首先，要在 HTTPS 代理后面运行 Django 应用程序，您需要将 [SECURE_PROXY_SSL_HEADER](https://docs.djangoproject.com/en/3.2/ref/settings/#secure-proxy-ssl-header) 设置添加到 *settings.py* :

```py
`SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")` 
```

在这个元组中，当`X-Forwarded-Proto`被设置为`https`时，请求是安全的。

## 复合坞站

是时候配置 Docker Compose 了。

让我们添加一个新的 Docker Compose 文件用于测试，名为*Docker-Compose . staging . yml*:

```py
`version:  '3.8' services: web: build: context:  ./app dockerfile:  Dockerfile.prod command:  gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000 volumes: -  static_volume:/home/app/web/staticfiles -  media_volume:/home/app/web/mediafiles expose: -  8000 env_file: -  ./.env.staging depends_on: -  db db: image:  postgres:13.0-alpine volumes: -  postgres_data:/var/lib/postgresql/data/ env_file: -  ./.env.staging.db nginx-proxy: container_name:  nginx-proxy build:  nginx restart:  always ports: -  443:443 -  80:80 volumes: -  static_volume:/home/app/web/staticfiles -  media_volume:/home/app/web/mediafiles -  certs:/etc/nginx/certs -  html:/usr/share/nginx/html -  vhost:/etc/nginx/vhost.d -  /var/run/docker.sock:/tmp/docker.sock:ro depends_on: -  web nginx-proxy-letsencrypt: image:  jrcs/letsencrypt-nginx-proxy-companion env_file: -  ./.env.staging.proxy-companion volumes: -  /var/run/docker.sock:/var/run/docker.sock:ro -  certs:/etc/nginx/certs -  html:/usr/share/nginx/html -  vhost:/etc/nginx/vhost.d -  acme:/etc/acme.sh depends_on: -  nginx-proxy volumes: postgres_data: static_volume: media_volume: certs: html: vhost: acme:` 
```

为`db`容器添加一个 *.env.staging.db* 文件:

```py
`POSTGRES_USER=hello_django
POSTGRES_PASSWORD=hello_django
POSTGRES_DB=hello_django_prod` 
```

> 更改`POSTGRES_USER`和`POSTGRES_PASSWORD`的值，以匹配您的用户和密码。

我们已经在之前的[教程](https://testdriven.io/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/)中看到了`web`和`db`服务，所以让我们深入研究一下`nginx-proxy`和`nginx-proxy-letsencrypt`服务。

> 数据库是关键服务。增加额外的层，例如 Docker，会增加生产中不必要的风险。为了简化小版本更新、定期备份和扩展等任务，建议使用托管服务，如 [AWS RDS](https://aws.amazon.com/rds/) 、 [Google Cloud SQL](https://cloud.google.com/sql) 或 [DigitalOcean 托管数据库](https://www.digitalocean.com/products/managed-databases/)。

## Nginx 代理服务

对于这个服务， [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) 项目用于为使用虚拟主机进行路由的`web`容器生成反向代理配置。

> 请务必查看关于 [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) repo 的自述文件。

一旦启动，与`nginx-proxy`相关联的容器自动检测设置了`VIRTUAL_HOST`环境变量的容器(在同一网络中),并动态更新其虚拟主机配置。

继续为`web`容器添加一个 *.env.staging* 文件:

```py
`DEBUG=0
SECRET_KEY=change_me
DJANGO_ALLOWED_HOSTS=<YOUR_DOMAIN.COM>
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_prod
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
VIRTUAL_HOST=<YOUR_DOMAIN.COM>
VIRTUAL_PORT=8000
LETSENCRYPT_HOST=<YOUR_DOMAIN.COM>` 
```

注意事项:

1.  将`<YOUR_DOMAIN.COM>`更改为您的实际域，并将`SQL_USER`和`SQL_PASSWORD`的默认值更改为匹配`POSTGRES_USER`和`POSTGRES_PASSWORD`(来自 *.env.staging.db* )。
2.  如上所述，`nginx-proxy`需要`VIRTUAL_HOST`(和`VIRTUAL_PORT`)来自动创建反向代理配置。
3.  `LETSENCRYPT_HOST`有没有办法让`nginx-proxy-companion`为你的域名颁发加密证书。
4.  由于 Django 应用程序将监听端口 8000，我们还设置了`VIRTUAL_PORT`环境变量。
5.  *docker-compose . staging . yml*中的`/var/run/docker.sock:/tmp/docker.sock:ro`卷用于监听新注册/注销的容器。

> 出于测试/调试的目的，您可能希望在第一次部署时使用一个`*`来代替`DJANGO_ALLOWED_HOSTS`,以简化事情。只是不要忘记在测试完成后限制允许的主机。

因此，对指定域的请求将由[代理](https://timothy-quinn.com/using-nginx-as-a-reverse-proxy-for-multiple-sites/)到容器，该容器将域设置为`VIRTUAL_HOST`环境变量。

接下来，让我们更新“nginx”文件夹中的 Nginx 配置。

首先，添加名为“vhost.d”的目录。然后，在该目录中添加一个名为 *default* 的文件来提供静态和媒体文件:

```py
`location /static/ {
  alias /home/app/web/staticfiles/;
  add_header Access-Control-Allow-Origin *;
}

location /media/ {
  alias /home/app/web/mediafiles/;
  add_header Access-Control-Allow-Origin *;
}` 
```

符合任何这些模式的请求将从静态或媒体文件夹中得到服务。它们不会被代理到其他容器。`web`和`nginx-proxy`容器共享静态和媒体文件所在的卷:

```py
`static_volume:/home/app/web/staticfiles
media_volume:/home/app/web/mediafiles` 
```

将一个 *custom.conf* 文件添加到“nginx”文件夹中，以保存自定义代理范围的配置:

```py
`client_max_body_size 10M;` 
```

更新 *nginx/Dockerfile* :

```py
`FROM  jwilder/nginx-proxy:0.9
COPY  vhost.d/default /etc/nginx/vhost.d/default
COPY  custom.conf /etc/nginx/conf.d/custom.conf` 
```

移除 *nginx.conf* 。

您的“nginx”目录现在应该如下所示:

```py
`└── nginx
    ├── Dockerfile
    ├── custom.conf
    └── vhost.d
        └── default` 
```

## 让我们加密 Nginx 代理配套服务

当`nginx-proxy`服务处理路由时，`nginx-proxy-letsencrypt`(通过[lets Encrypt-nginx-proxy-companion](https://github.com/nginx-proxy/docker-letsencrypt-nginx-proxy-companion))处理代理 Docker 容器的证书的创建、更新和使用。

要为代理容器颁发和更新证书，需要为每个容器添加`LETSENCRYPT_HOST`环境变量(我们已经完成了)。它还必须具有与`VIRTUAL_HOST`相同的值。

该容器必须与`nginx-proxy`共享以下卷:

1.  `certs:/etc/nginx/certs`存储证书、私钥和 ACME 帐户密钥
2.  `html:/usr/share/nginx/html`写入 [http-01](https://tools.ietf.org/html/draft-ietf-acme-acme-03#section-7.2) 挑战文件
3.  `vhost:/etc/nginx/vhost.d`更改虚拟主机的配置

> 有关更多信息，请查看官方文档。

添加一个*. env . staging . proxy-companion*文件:

```py
`DEFAULT_EMAIL=youremail@yourdomain.com ACME_CA_URI=https://acme-staging-v02.api.letsencrypt.org/directory NGINX_PROXY_CONTAINER=nginx-proxy` 
```

注意事项:

1.  `DEFAULT_EMAIL`是我们将用来向您发送证书(包括续订)通知的电子邮件。
2.  `ACME_CA_URI`是用于颁发证书的 URL。再次，使用阶段直到你 100%确定一切正常。
3.  `NGINX_PROXY_CONTAINER`是`nginx-proxy`容器的名称。

## 运行容器

一切准备就绪，随时可以部署。

是时候转移到您的 Linux 实例了。

假设在您的实例上创建了一个项目目录，如*/home/my user/django-on-docker*，用 SCP 复制文件和文件夹:

```py
`$ scp -r $(pwd)/{app,nginx,.env.staging,.env.staging.db,.env.staging.proxy-companion,docker-compose.staging.yml} [[email protected]](/cdn-cgi/l/email-protection):/path/to/django-on-docker` 
```

通过 SSH 连接到您的实例，并移动到项目目录:

这时，您就可以构建映像并旋转容器了:

```py
`$ docker-compose -f docker-compose.staging.yml up -d --build` 
```

一旦容器启动并运行，在浏览器中导航到您的域。您应该会看到类似这样的内容:

![HTTPS certificate not secure](img/d83157ae320af80a164075c62af84cc2.png)

这是意料之中的。显示该屏幕是因为证书是从[暂存环境](https://letsencrypt.org/docs/staging-environment/)中颁发的，该环境同样没有与生产环境相同的[速率限制](https://letsencrypt.org/docs/staging-environment/#rate-limits)。它类似于一个[自签名的 HTTPS 证书](https://en.wikipedia.org/wiki/Self-signed_certificate)。始终使用试运行环境，直到您确定一切都按预期运行。

你怎么知道一切是否正常？

点击“高级”,然后点击“继续”。您现在应该可以看到您的应用程序了。上传图像，然后确保您可以在`https://yourdomain.com/mediafiles/IMAGE_FILE_NAME`查看图像。

## 颁发生产证书

现在，一切都按预期运行，我们可以切换到 Let's Encrypt 的生产环境。

关闭现有容器并退出实例:

```py
`$ docker-compose -f docker-compose.staging.yml down -v
$ exit` 
```

回到您的本地机器上，更新*docker-composite . product . yml*:

```py
`version:  '3.8' services: web: build: context:  ./app dockerfile:  Dockerfile.prod command:  gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000 volumes: -  static_volume:/home/app/web/staticfiles -  media_volume:/home/app/web/mediafiles expose: -  8000 env_file: -  ./.env.prod depends_on: -  db db: image:  postgres:13.0-alpine volumes: -  postgres_data:/var/lib/postgresql/data/ env_file: -  ./.env.prod.db nginx-proxy: container_name:  nginx-proxy build:  nginx restart:  always ports: -  443:443 -  80:80 volumes: -  static_volume:/home/app/web/staticfiles -  media_volume:/home/app/web/mediafiles -  certs:/etc/nginx/certs -  html:/usr/share/nginx/html -  vhost:/etc/nginx/vhost.d -  /var/run/docker.sock:/tmp/docker.sock:ro depends_on: -  web nginx-proxy-letsencrypt: image:  jrcs/letsencrypt-nginx-proxy-companion env_file: -  ./.env.prod.proxy-companion volumes: -  /var/run/docker.sock:/var/run/docker.sock:ro -  certs:/etc/nginx/certs -  html:/usr/share/nginx/html -  vhost:/etc/nginx/vhost.d -  acme:/etc/acme.sh depends_on: -  nginx-proxy volumes: postgres_data: static_volume: media_volume: certs: html: vhost: acme:` 
```

与*docker-compose . staging . yml*相比，这里唯一的区别是我们使用了不同的环境文件。

*.env.prod* :

```py
`DEBUG=0
SECRET_KEY=change_me
DJANGO_ALLOWED_HOSTS=<YOUR_DOMAIN.COM>
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_prod
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
VIRTUAL_HOST=<YOUR_DOMAIN.COM>
VIRTUAL_PORT=8000
LETSENCRYPT_HOST=<YOUR_DOMAIN.COM>` 
```

*.env.prod.db* :

```py
`POSTGRES_USER=hello_django
POSTGRES_PASSWORD=hello_django
POSTGRES_DB=hello_django_prod` 
```

*. env . prod . proxy-companion*:

```py
`DEFAULT_EMAIL=youremail@yourdomain.co NGINX_PROXY_CONTAINER=nginx-proxy` 
```

适当地更新它们。

你发现了不同版本的区别了吗？没有设置`ACME_CA_URI`环境变量，因为默认情况下`letsencrypt-nginx-proxy-companion`映像使用 Let's Encrypt 的生产环境。

使用 SCP 将新文件和文件夹复制到您的实例中:

```py
`$ scp $(pwd)/{.env.prod,.env.prod.db,.env.prod.proxy-companion,docker-compose.prod.yml} [[email protected]](/cdn-cgi/l/email-protection):/path/to/django-on-docker` 
```

像前面一样，通过 SSH 连接到您的实例，并移动到项目目录:

构建图像并旋转容器:

```py
`$ docker-compose -f docker-compose.prod.yml up -d --build` 
```

再次导航到您的域。您应该不会再看到警告。

恭喜你。您现在使用的是生产加密证书。

> 想要查看证书创建过程的运行情况，请查看日志:
> 
> ```py
> `$ docker-compose -f docker-compose.prod.yml logs nginx-proxy-letsencrypt` 
> ```

## 结论

总之，一旦您将 Docker Compose 配置为运行 Django，要设置 HTTPS，您需要在 Docker Compose 文件中添加(并配置)服务`nginx-proxy`和`nginx-proxy-letsencrypt`。现在，您可以通过配置`VIRTUAL_HOST`(路由)和`LETSENCRYPT_HOST`(证书)环境变量来添加更多的容器。和往常一样，一定要先用 Let's Encrypt 的登台环境进行测试。

你可以在 django-on-docker-letsencryptrepo 中找到代码。

> 姜戈码头系列:
> 
> 1.  [用 Postgres、Gunicorn 和 Nginx 将 Django 归档](/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/)
> 2.  [使用 Let's Encrypt](/blog/django-lets-encrypt/) (本文！)
> 3.  [用 Docker 将 Django 部署到 AWS，让我们加密](/blog/django-docker-https-aws/)