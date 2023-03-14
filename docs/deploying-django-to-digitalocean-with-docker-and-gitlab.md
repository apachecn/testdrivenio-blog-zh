# 与 Docker 和 GitLab 一起将 Django 持续部署到数字海洋

> 原文：<https://testdriven.io/blog/deploying-django-to-digitalocean-with-docker-and-gitlab/>

在本教程中，我们将了解如何配置 GitLab CI，以便将 Django 和 Docker 应用程序持续部署到 DigitalOcean。

*依赖关系*:

1.  Django v3.2.4
2.  Docker v20.04
3.  python 3 . 9 . 5 版

## 目标

本教程结束时，您将能够:

1.  使用 Docker 将 Django 部署到数字海洋
2.  配置 GitLab CI 以持续将 Django 部署到数字海洋
3.  设置无密码 SSH 登录
4.  为数据持久性配置数字海洋的托管数据库

## 项目设置

除了 Django 和 Docker，我们将使用的演示项目还包括 [Postgres](https://www.postgresql.org/) 、 [Nginx](https://www.nginx.com) 和 [Gunicorn](https://gunicorn.org/) 。

> 好奇这个项目是怎么开发出来的？查看 Postgres、Gunicorn 和 Nginx 的博客文章。

从克隆基础项目开始:

```
`$ git clone https://gitlab.com/testdriven/django-gitlab-digitalocean.git --branch base --single-branch
$ cd django-gitlab-digitalocean` 
```

要进行本地测试，构建映像并旋转容器:

```
`$ docker-compose up -d --build` 
```

导航到 [http://localhost:8000/](http://localhost:8000/) 。您应该看到:

## 数字海洋

让我们设置 DigitalOcean 来使用我们的应用程序。

首先，你需要[注册](https://m.do.co/c/d8f211a4b4c2)一个[数字海洋](https://www.digitalocean.com/)账户(如果你还没有的话)，然后[生成](https://www.digitalocean.com/docs/apis-clis/api/create-personal-access-token/)一个访问令牌，这样你就可以访问[数字海洋 API](https://developers.digitalocean.com/documentation/v2/) 。

将令牌添加到您的环境中:

```
`$ export DIGITAL_OCEAN_ACCESS_TOKEN=[your_digital_ocean_token]` 
```

### 微滴

接下来，使用预装的 Docker 创建一个新的 Droplet:

```
`$ curl -X POST \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    -d '{"name":"django-docker","region":"sfo3","size":"s-2vcpu-4gb","image":"docker-20-04"}' \
    "https://api.digitalocean.com/v2/droplets"` 
```

检查状态:

```
`$ curl \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    "https://api.digitalocean.com/v2/droplets?name=django-docker"` 
```

如果您安装了 [jq](https://stedolan.github.io/jq/) ，那么您可以像这样解析 JSON 响应:

```
`$ curl \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    "https://api.digitalocean.com/v2/droplets?name=django-docker" \
  | jq '.droplets[0].status'` 
```

root 密码应该会通过电子邮件发送给您。找回它。然后，一旦 droplet 的状态为`active`，就以 root 身份 SSH 到实例中，并在提示时更新密码。

接下来，生成一个新的 SSH 密钥:

将密钥保存到 */root/。ssh/id_rsa* 并且不设置密码。这将分别生成一个公钥和私钥- *id_rsa* 和 *id_rsa.pub* 。要设置无密码 SSH 登录，请将公钥复制到 [authorized_keys](https://security.stackexchange.com/questions/20706/what-is-the-difference-between-authorized-keys-and-known-hosts-file-for-ssh) 文件中，并设置适当的权限:

```
`$ cat ~/.ssh/id_rsa.pub
$ vi ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/id_rsa` 
```

复制私钥的内容:

将其设置为本地计算机上的环境变量:

```
`export PRIVATE_KEY='-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA04up8hoqzS1+APIB0RhjXyObwHQnOzhAk5Bd7mhkSbPkyhP1
...
iWlX9HNavcydATJc1f0DpzF0u4zY8PY24RVoW8vk+bJANPp1o2IAkeajCaF3w9nf
q/SyqAWVmvwYuIhDiHDaV2A==
-----END RSA PRIVATE KEY-----'` 
```

将密钥添加到 [ssh-agent](https://www.ssh.com/ssh/agent) 中:

```
`$ ssh-add - <<< "${PRIVATE_KEY}"` 
```

要进行测试，请运行:

然后，为应用程序创建一个新目录:

### 数据库ˌ资料库

接下来，让我们通过数字海洋的[托管数据库](https://www.digitalocean.com/products/managed-databases/)建立一个生产 Postgres 数据库:

```
`$ curl -X POST \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    -d '{"name":"django-docker-db","region":"sfo3","engine":"pg","version":"13","size":"db-s-2vcpu-4gb","num_nodes":1}' \
    "https://api.digitalocean.com/v2/databases"` 
```

检查状态:

```
`$ curl \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    "https://api.digitalocean.com/v2/databases?name=django-docker-db" \
  | jq '.databases[0].status'` 
```

它应该需要几分钟才能旋转起来。一旦状态为`online`，获取连接信息:

```
`$ curl \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    "https://api.digitalocean.com/v2/databases?name=django-docker-db" \
  | jq '.databases[0].connection'` 
```

示例响应:

```
`{
  "protocol": "postgresql",
  "uri": "postgresql://doadmin:[[email protected]](/cdn-cgi/l/email-protection)locean.com:25060/defaultdb?sslmode=require",
  "database": "defaultdb",
  "host": "django-docker-db-do-user-778274-0.a.db.ondigitalocean.com",
  "port": 25060,
  "user": "doadmin",
  "password": "na9tcfew9jw13a2m",
  "ssl": true
}` 
```

## GitLab CI

[注册](https://gitlab.com/users/sign_up)一个 GitLab 账号(如果需要的话)，然后[创建一个新项目](https://docs.gitlab.com/ee/user/project/working_with_projects.html#create-a-project)(再次，如果需要的话)。

### 构建阶段

接下来，添加一个名为*的 GitLab CI/CD 配置文件。gitlab-ci.yml* 到项目根:

```
`image: name:  docker/compose:1.29.1 entrypoint:  [""] services: -  docker:dind stages: -  build variables: DOCKER_HOST:  tcp://docker:2375 DOCKER_DRIVER:  overlay2 build: stage:  build before_script: -  export IMAGE=$CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME -  export WEB_IMAGE=$IMAGE:web -  export NGINX_IMAGE=$IMAGE:nginx script: -  apk add --no-cache bash -  chmod +x ./setup_env.sh -  bash ./setup_env.sh -  docker login -u $CI_REGISTRY_USER -p $CI_JOB_TOKEN $CI_REGISTRY -  docker pull $IMAGE:web || true -  docker pull $IMAGE:nginx || true -  docker-compose -f docker-compose.ci.yml build -  docker push $IMAGE:web -  docker push $IMAGE:nginx` 
```

在这里，我们定义了单个`build` [阶段](https://docs.gitlab.com/ee/ci/yaml/#stages)，在这里我们:

1.  设置`IMAGE`、`WEB_IMAGE`和`NGINX_IMAGE`环境变量
2.  安装 bash
3.  为 *setup_env.sh* 设置适当的权限
4.  运行 *setup_env.sh*
5.  登录到 [GitLab 容器注册表](https://docs.gitlab.com/ee/user/packages/container_registry/)
6.  如果图像存在，请提取图像
7.  构建图像
8.  将图像上传到注册表

将 *setup_env.sh* 文件添加到项目根目录:

```
`#!/bin/sh

echo DEBUG=0 >> .env
echo SQL_ENGINE=django.db.backends.postgresql >> .env
echo DATABASE=postgres >> .env

echo SECRET_KEY=$SECRET_KEY >> .env
echo SQL_DATABASE=$SQL_DATABASE >> .env
echo SQL_USER=$SQL_USER >> .env
echo SQL_PASSWORD=$SQL_PASSWORD >> .env
echo SQL_HOST=$SQL_HOST >> .env
echo SQL_PORT=$SQL_PORT >> .env` 
```

这个文件将创建所需的*。env* 文件，基于在 GitLab 项目的 CI/CD 设置中找到的环境变量(设置> CI / CD >变量)。根据连接信息添加变量。

例如:

1.  `SECRET_KEY` : `9zYGEFk2mn3mWB8Bmg9SAhPy6F4s7cCuT8qaYGVEnu7huGRKW9`
2.  `SQL_DATABASE` : `defaultdb`
3.  `SQL_HOST` : `django-docker-db-do-user-778274-0.a.db.ondigitalocean.com`
4.  `SQL_PASSWORD` : `na9tcfew9jw13a2m`
5.  `SQL_PORT` : `25060`
6.  `SQL_USER` : `doadmin`

![gitlab config](img/a4aa8a1640e761dc5cdd60d917cf6599.png)

完成后，提交代码并上传到 GitLab 以触发新的构建。确保它通过。您应该会在 GitLab 容器注册表中看到这些图像:

![gitlab config](img/7631d34c35ebed99e5716d3e2492f342.png)

### 部署阶段

接下来，给*增加一个`deploy`阶段。gitlab-ci.yml* 并创建一个用于两个阶段的全局`before_script`:

```
`image: name:  docker/compose:1.29.1 entrypoint:  [""] services: -  docker:dind stages: -  build -  deploy variables: DOCKER_HOST:  tcp://docker:2375 DOCKER_DRIVER:  overlay2 before_script: -  export IMAGE=$CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME -  export WEB_IMAGE=$IMAGE:web -  export NGINX_IMAGE=$IMAGE:nginx -  apk add --no-cache openssh-client bash -  chmod +x ./setup_env.sh -  bash ./setup_env.sh -  docker login -u $CI_REGISTRY_USER -p $CI_JOB_TOKEN $CI_REGISTRY build: stage:  build script: -  docker pull $IMAGE:web || true -  docker pull $IMAGE:nginx || true -  docker-compose -f docker-compose.ci.yml build -  docker push $IMAGE:web -  docker push $IMAGE:nginx deploy: stage:  deploy script: -  mkdir -p ~/.ssh -  echo "$PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa -  cat ~/.ssh/id_rsa -  chmod 700 ~/.ssh/id_rsa -  eval "$(ssh-agent -s)" -  ssh-add ~/.ssh/id_rsa -  ssh-keyscan -H 'gitlab.com' >> ~/.ssh/known_hosts -  chmod +x ./deploy.sh -  scp  -o StrictHostKeyChecking=no -r ./.env ./docker-compose.prod.yml [[email protected]](/cdn-cgi/l/email-protection)$DIGITAL_OCEAN_IP_ADDRESS:/app -  bash ./deploy.sh` 
```

因此，在`deploy`阶段，我们:

1.  将私有 ssh 密钥添加到 SSH 代理中
2.  复制完*。env* 和 *docker-compose.prod.yml* 文件到远程服务器
3.  为 *deploy.sh* 设置适当的权限
4.  运行 *deploy.sh*

将 *deploy.sh* 添加到项目根:

```
`#!/bin/sh

ssh -o StrictHostKeyChecking=no [[email protected]](/cdn-cgi/l/email-protection)$DIGITAL_OCEAN_IP_ADDRESS << 'ENDSSH'
 cd /app
 export $(cat .env | xargs)
 docker login -u $CI_REGISTRY_USER -p $CI_JOB_TOKEN $CI_REGISTRY
 docker pull $IMAGE:web
 docker pull $IMAGE:nginx
 docker-compose -f docker-compose.prod.yml up -d
ENDSSH` 
```

因此，在登录到服务器后，我们

1.  导航到部署目录
2.  添加环境变量
3.  登录 GitLab 容器注册表
4.  调出图像
5.  旋转容器

将`DIGITAL_OCEAN_IP_ADDRESS`和`PRIVATE_KEY`环境变量添加到 GitLab 中。

更新 *setup_env.sh* 文件:

```
`#!/bin/sh

echo DEBUG=0 >> .env
echo SQL_ENGINE=django.db.backends.postgresql >> .env
echo DATABASE=postgres >> .env

echo SECRET_KEY=$SECRET_KEY >> .env
echo SQL_DATABASE=$SQL_DATABASE >> .env
echo SQL_USER=$SQL_USER >> .env
echo SQL_PASSWORD=$SQL_PASSWORD >> .env
echo SQL_HOST=$SQL_HOST >> .env
echo SQL_PORT=$SQL_PORT >> .env
echo WEB_IMAGE=$IMAGE:web  >> .env
echo NGINX_IMAGE=$IMAGE:nginx  >> .env
echo CI_REGISTRY_USER=$CI_REGISTRY_USER   >> .env
echo CI_JOB_TOKEN=$CI_JOB_TOKEN  >> .env
echo CI_REGISTRY=$CI_REGISTRY  >> .env
echo IMAGE=$CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME >> .env` 
```

接下来，将服务器的 IP 添加到 Django 设置中的`ALLOWED_HOSTS`列表。

提交并推送您的代码以触发新的构建。构建通过后，导航到实例的 IP。您应该看到:

## 试验

最后，更新`deploy`阶段，使其仅在对`master`分支进行更改时运行:

```
`deploy: stage:  deploy script: -  mkdir -p ~/.ssh -  echo "$PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa -  cat ~/.ssh/id_rsa -  chmod 700 ~/.ssh/id_rsa -  eval "$(ssh-agent -s)" -  ssh-add ~/.ssh/id_rsa -  ssh-keyscan -H 'gitlab.com' >> ~/.ssh/known_hosts -  chmod +x ./deploy.sh -  scp  -o StrictHostKeyChecking=no -r ./.env ./docker-compose.prod.yml [[email protected]](/cdn-cgi/l/email-protection)$DIGITAL_OCEAN_IP_ADDRESS:/app -  bash ./deploy.sh only: -  master` 
```

为了测试，创建一个新的`develop`分支。在 *urls.py* 中的`world`后加一个感叹号:

```
`def home(request):
    return JsonResponse({'hello': 'world!'})` 
```

将您的更改提交并推送到 GitLab。确保只运行`build`阶段。一旦构建通过，打开一个针对`master`分支的 PR 并合并变更。这将触发一个包含两个阶段的新管道- `build`和`deploy`。确保部署按预期工作:

--

就是这样！你可以在 django-git lab-digital oceanrepo 中找到最终代码。