# 使用 Docker 和 GitLab 将 Django 持续部署到 AWS EC2

> 原文：<https://testdriven.io/blog/deploying-django-to-ec2-with-docker-and-gitlab/>

在本教程中，我们将了解如何配置 GitLab CI，以便将 Django 和 Docker 应用程序持续部署到 Amazon Web Services (AWS) EC2。

*依赖关系*:

1.  Django v3.2.5
2.  文档 v20.10.4
3.  python 3 . 9 . 6 版

## 目标

本教程结束时，您将能够:

1.  设置新的 EC2 实例
2.  配置 AWS 安全组
3.  在 EC2 实例上安装 Docker
4.  设置无密码 SSH 登录
5.  为数据持久性配置 AWS RDS
6.  使用 Docker 将 Django 部署到 AWS EC2
7.  配置 GitLab CI 以持续将 Django 部署到 EC2

## 项目设置

除了 Django 和 Docker，我们将使用的演示项目还包括 [Postgres](https://www.postgresql.org/) 、 [Nginx](https://www.nginx.com) 和 [Gunicorn](https://gunicorn.org/) 。

> 好奇这个项目是怎么开发出来的？查看 Postgres、Gunicorn 和 Nginx 的博客文章。

从克隆基础项目开始:

```
`$ git clone https://gitlab.com/testdriven/django-gitlab-ec2.git --branch base --single-branch
$ cd django-gitlab-ec2` 
```

要进行本地测试，构建映像并旋转容器:

```
`$ docker-compose up -d --build` 
```

导航到 [http://localhost:8000/](http://localhost:8000/) 。您应该看到:

## AWS 设置

让我们首先设置一个 EC2 实例来部署我们的应用程序，并配置 RDS。

首先，你需要[注册](https://portal.aws.amazon.com/billing/signup)一个 [AWS](https://aws.amazon.com/) 账户(如果你还没有的话)。

> 设置您的第一个 AWS 帐户？
> 
> 创建一个非根 IAM 用户是一个好主意，具有“管理员访问”和“计费”策略，并通过 CloudWatch 发出计费警报，以便在您的 AWS 使用成本超过一定金额时提醒您。有关更多信息，请查看[锁定您的 AWS 帐户 Root 用户访问密钥](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#lock-away-credentials)和[分别创建计费警报](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/tracking-free-tier-usage.html)。

### EC2

登录 [AWS 控制台](https://console.aws.amazon.com/console/home)，导航到 [EC2 控制台](https://console.aws.amazon.com/ec2/v2/home)，点击左侧边栏的“实例”。然后，单击“启动实例”按钮:

![new ec2 instance](img/3ac50ee3d20ba9dfcb560329a29a5879.png)

接下来，使用基本的[亚马逊 Linux AMI](https://aws.amazon.com/amazon-linux-ami/) 和`t2.micro` [实例类型](https://aws.amazon.com/ec2/instance-types/):

![new ec2 instance](img/ccc0fbdec433bf61b30686a7aba067c2.png)

![new ec2 instance](img/030b3dca35b94f57b8c8389ba23d333a.png)

单击“下一步:配置实例详细信息”。为了使本教程简单，我们将坚持使用默认的 VPC，但也可以随时更新。

![new ec2 instance](img/06457a8e2beab9b1fcabae9975d4983f.png)

多次单击“下一步”按钮，直到进入“配置安全组”步骤。创建一个名为`django-security-group`的新安全组(类似于防火墙)，确保至少 HTTP 80 和 SSH 22 是开放的。

![new ec2 instance](img/da07cce274470a4bc2f55978cbbc2c8c.png)

点击“查看并启动”。

在下一页，单击“启动”。在 modal 上，创建一个新的[密钥对](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)，这样就可以通过 SSH 连接到实例。保存这个*。pem* 文件放在安全的地方。

![new ec2 instance](img/addccb8657cd6538150c8a52b3785615.png)

> 在 Mac 或 Linux 机器上？建议保存*。pem* 文件到“/用户/$用户/”。ssh "目录。一定要设置适当的权限，例如`chmod 400 ~/.ssh/django.pem`。

单击“启动实例”创建新实例。在“启动状态”页面上，单击“查看实例”。然后，在主实例页面上，获取新创建的实例的公共 IP:

![new ec2 instance](img/d9982ef1d24e7b5b0378c2e31b2e1b70.png)

### 码头工人

实例启动并运行后，我们现在可以在其上安装 Docker 了。

使用您的密钥对 SSH 到实例中，如下所示:

首先安装并启动最新版本的 Docker 和版本 1.29.2 的 Docker Compose:

```
`[ec2-user]$ sudo yum update -y
[ec2-user]$ sudo yum install -y docker
[ec2-user]$ sudo service docker start

[ec2-user]$ sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" \
    -o /usr/local/bin/docker-compose
[ec2-user]$ sudo chmod +x /usr/local/bin/docker-compose

[ec2-user]$ docker --version
Docker version 20.10.4, build d3cb89e

[ec2-user]$ docker-compose --version
docker-compose version 1.29.2, build 5becea4c` 
```

将`ec2-user`添加到`docker`组，这样您就可以执行 Docker 命令，而不必使用`sudo`:

```
`[ec2-user]$ sudo usermod -a -G docker ec2-user` 
```

接下来，生成一个新的 SSH 密钥:

```
`[ec2-user]$ ssh-keygen -t rsa` 
```

将密钥保存到 */home/ec2-user/。ssh/id_rsa* 并且不设置密码。这将分别生成一个公钥和私钥- *id_rsa* 和 *id_rsa.pub* 。要设置无密码 SSH 登录，请将公钥复制到 [authorized_keys](https://security.stackexchange.com/questions/20706/what-is-the-difference-between-authorized-keys-and-known-hosts-file-for-ssh) 文件中，并设置适当的权限:

```
`[ec2-user]$ cat ~/.ssh/id_rsa.pub
[ec2-user]$ vi ~/.ssh/authorized_keys
[ec2-user]$ chmod 600 ~/.ssh/authorized_keys
[ec2-user]$ chmod 600 ~/.ssh/id_rsa` 
```

复制私钥的内容:

```
`[ec2-user]$ cat ~/.ssh/id_rsa` 
```

退出远程 SSH 会话。将密钥设置为本地计算机上的环境变量:

```
`$ export PRIVATE_KEY='-----BEGIN RSA PRIVATE KEY-----
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

```
`$ ssh -o StrictHostKeyChecking=no [[email protected]](/cdn-cgi/l/email-protection)<YOUR_INSTANCE_IP> mkdir /home/ec2-user/app

# example:
# ssh -o StrictHostKeyChecking=no [[email protected]](/cdn-cgi/l/email-protection).143 mkdir /home/ec2-user/app` 
```

### 无线电数据系统

接下来，让我们通过 AWS 关系数据库服务(RDS)构建一个生产 Postgres 数据库。

导航到 [Amazon RDS](https://console.aws.amazon.com/rds) ，点击侧边栏上的“数据库”，然后点击“创建数据库”按钮。

![new rds instance](img/d871a2924ed119a7e5b005f7629ecca9.png)

对于“引擎选项”，选择“PostgreSQL”引擎和`PostgreSQL 12.7-R1`版本。

使用“自由层”模板。

> 有关免费层的更多信息，请查看 [AWS 免费层](https://aws.amazon.com/free/)指南。

![new rds instance](img/4251f9961a84513dcf2323b2d7632268.png)

在“设置”下:

1.  “数据库实例标识符”:`djangodb`
2.  “主用户名”:`webapp`
3.  “主密码”:勾选“自动生成密码”

![new rds instance](img/030e29459e130a9eafae70e58140eb1c.png)

向下滚动到“连接”部分。坚持使用默认的“VPC”并选择`django-security-group`安全组。关闭“公共可访问性”。

![new rds instance](img/671e169561da9279f52a315ef49dda20.png)

在“附加配置”下，将“初始数据库名称”更改为`django_prod`，然后创建新数据库。

![new rds instance](img/871fde9a3b2adee4ac0b0e293cc512fb.png)

单击“查看凭据详细信息”按钮查看生成的密码。记下它。

RDS 实例启动需要几分钟时间。一旦它可用，记下端点。例如:

```
`djangodb.c7kxiqfnzo9e.us-west-1.rds.amazonaws.com` 
```

完整的 URL 如下所示:

```
`postgres://webapp:YOUR_PASSWORD@djangodb.c7kxiqfnzo9e.us-west-1.rds.amazonaws.com:5432/django_prod` 
```

> 请记住，您无法在 VPC 之外访问数据库。因此，如果您想直接连接到它，您需要使用 [SSH 隧道](https://en.wikipedia.org/wiki/Tunneling_protocol#Secure_Shell_tunneling)通过 SSH 进入 EC2 实例并从那里连接到数据库。我们很快就会看到如何做到这一点。

## GitLab CI

[注册](https://gitlab.com/users/sign_up)一个 GitLab 账号(如果需要的话)，然后[创建一个新项目](https://docs.gitlab.com/ee/user/project/working_with_projects.html#create-a-project)(再次，如果需要的话)。

### 构建阶段

接下来，添加一个名为*的 GitLab CI/CD 配置文件。gitlab-ci.yml* 到项目根:

```
`image: name:  docker/compose:1.29.2 entrypoint:  [""] services: -  docker:dind stages: -  build variables: DOCKER_HOST:  tcp://docker:2375 DOCKER_DRIVER:  overlay2 build: stage:  build before_script: -  export IMAGE=$CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME -  export WEB_IMAGE=$IMAGE:web -  export NGINX_IMAGE=$IMAGE:nginx script: -  apk add --no-cache bash -  chmod +x ./setup_env.sh -  bash ./setup_env.sh -  docker login -u $CI_REGISTRY_USER -p $CI_JOB_TOKEN $CI_REGISTRY -  docker pull $IMAGE:web || true -  docker pull $IMAGE:nginx || true -  docker-compose -f docker-compose.ci.yml build -  docker push $IMAGE:web -  docker push $IMAGE:nginx` 
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

这个文件将创建所需的*。env* 文件，基于在 GitLab 项目的 CI/CD 设置中找到的环境变量(设置> CI / CD >变量)。根据上面的 RDS 连接信息添加变量。

例如:

1.  `SECRET_KEY` : `9zYGEFk2mn3mWB8Bmg9SAhPy6F4s7cCuT8qaYGVEnu7huGRKW9`
2.  `SQL_DATABASE` : `djangodb`
3.  `SQL_HOST` : `djangodb.c7kxiqfnzo9e.us-west-1.rds.amazonaws.com`
4.  `SQL_PASSWORD` : `3ZQtN4vxkZp2kAa0vinV`
5.  `SQL_PORT` : `5432`
6.  `SQL_USER` : `webapp`

![gitlab config](img/9c577cb7297fdc35bba36b2059a89f45.png)

完成后，提交代码并上传到 GitLab 以触发新的构建。确保它通过。您应该会在 GitLab 容器注册表中看到这些图像:

![gitlab config](img/6a6a45665b60343ee65999958b1b224b.png)

## AWS 安全组

接下来，在将部署添加到 CI 流程之前，我们需要更新“安全组”的入站端口，以便可以从 EC2 实例访问端口 5432。为什么这是必要的？转到 *app/entrypoint.prod.sh* :

```
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

在这里，我们在启动 Gunciorn 之前，通过测试与 [netcat](https://en.wikipedia.org/wiki/Netcat) 的连接，等待 Postgres 实例变得健康。如果端口 5432 没有打开，循环将永远继续下去。

因此，再次导航到 [EC2 控制台](https://console.aws.amazon.com/ec2/v2/home)，点击左侧边栏上的“安全组”。选择`django-security-group`安全组，点击“编辑入站规则”:

![security group](img/133fc8d427758be7b1f2383c504061b7.png)

单击“添加规则”。在类型下，选择“PostgreSQL ”,并在源下选择`django-security-group`安全组:

![security group](img/eb28a77acd9418a4163d3d745b18e4e5.png)

现在，与该组相关联的任何 AWS 服务都可以通过端口 5432 访问 RDS 实例。单击“保存规则”。

## GitLab CI:部署阶段

接下来，给*增加一个`deploy`阶段。gitlab-ci.yml* 并创建一个用于两个阶段的全局`before_script`:

```
`image: name:  docker/compose:1.29.2 entrypoint:  [""] services: -  docker:dind stages: -  build -  deploy variables: DOCKER_HOST:  tcp://docker:2375 DOCKER_DRIVER:  overlay2 before_script: -  export IMAGE=$CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME -  export WEB_IMAGE=$IMAGE:web -  export NGINX_IMAGE=$IMAGE:nginx -  apk add --no-cache openssh-client bash -  chmod +x ./setup_env.sh -  bash ./setup_env.sh -  docker login -u $CI_REGISTRY_USER -p $CI_JOB_TOKEN $CI_REGISTRY build: stage:  build script: -  docker pull $IMAGE:web || true -  docker pull $IMAGE:nginx || true -  docker-compose -f docker-compose.ci.yml build -  docker push $IMAGE:web -  docker push $IMAGE:nginx deploy: stage:  deploy script: -  mkdir -p ~/.ssh -  echo "$PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa -  cat ~/.ssh/id_rsa -  chmod 700 ~/.ssh/id_rsa -  eval "$(ssh-agent -s)" -  ssh-add ~/.ssh/id_rsa -  ssh-keyscan -H 'gitlab.com' >> ~/.ssh/known_hosts -  chmod +x ./deploy.sh -  scp  -o StrictHostKeyChecking=no -r ./.env ./docker-compose.prod.yml [[email protected]](/cdn-cgi/l/email-protection)$EC2_PUBLIC_IP_ADDRESS:/home/ec2-user/app -  bash ./deploy.sh` 
```

因此，在`deploy`阶段，我们:

1.  将私有 ssh 密钥添加到 SSH 代理中
2.  复制完*。env* 和 *docker-compose.prod.yml* 文件到远程服务器
3.  为 *deploy.sh* 设置适当的权限
4.  运行 *deploy.sh*

将 *deploy.sh* 添加到项目根:

```
`#!/bin/sh

ssh -o StrictHostKeyChecking=no [[email protected]](/cdn-cgi/l/email-protection)$EC2_PUBLIC_IP_ADDRESS << 'ENDSSH'
 cd /home/ec2-user/app
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

将`EC2_PUBLIC_IP_ADDRESS`和`PRIVATE_KEY`环境变量添加到 GitLab 中。

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

## 通过 SSH 隧道的 PostgreSQL

需要访问数据库？

宋承宪入框:

安装 Postgres:

```
`[ec2-user]$ sudo amazon-linux-extras install postgresql12 -y` 
```

然后，运行`psql`，像这样:

```
`[ec2-user]$ psql -h <YOUR_RDS_ENDPOINT> -U webapp -d django_prod

# example:
# psql -h djangodb.c7vzuyfvhlgz.us-east-1.rds.amazonaws.com -U webapp -d django_prod` 
```

输入密码。

```
`psql (12.7)
SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
Type "help" for help.

django_prod=> \l
                                   List of databases
    Name     |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-------------+----------+----------+-------------+-------------+-----------------------
 django_prod | webapp   | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres    | webapp   | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 rdsadmin    | rdsadmin | UTF8     | en_US.UTF-8 | en_US.UTF-8 | rdsadmin=CTc/rdsadmin
 template0   | rdsadmin | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/rdsadmin          +
             |          |          |             |             | rdsadmin=CTc/rdsadmin
 template1   | webapp   | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/webapp            +
             |          |          |             |             | webapp=CTc/webapp
(5 rows)

django_prod=> \q` 
```

完成后，退出 SSH 会话。

## 更新 GitLab CI

最后，更新`deploy`阶段，使其仅在对`master`分支进行更改时运行:

```
`deploy: stage:  deploy script: -  mkdir -p ~/.ssh -  echo "$PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa -  cat ~/.ssh/id_rsa -  chmod 700 ~/.ssh/id_rsa -  eval "$(ssh-agent -s)" -  ssh-add ~/.ssh/id_rsa -  ssh-keyscan -H 'gitlab.com' >> ~/.ssh/known_hosts -  chmod +x ./deploy.sh -  scp  -o StrictHostKeyChecking=no -r ./.env ./docker-compose.prod.yml [[email protected]](/cdn-cgi/l/email-protection)$EC2_PUBLIC_IP_ADDRESS:/home/ec2-user/app -  bash ./deploy.sh only: -  master` 
```

为了测试，创建一个新的`develop`分支。在 *urls.py* 中的`world`后加一个感叹号:

```
`def home(request):
    return JsonResponse({'hello': 'world!'})` 
```

将您的更改提交并推送到 GitLab。确保只运行`build`阶段。一旦构建通过，打开一个针对`master`分支的 PR 并合并变更。这将触发一个包含两个阶段的新管道- `build`和`deploy`。确保部署按预期工作:

## 后续步骤

本教程介绍了如何配置 GitLab CI，以便将 Django 和 Docker 应用程序持续部署到 AWS EC2。

此时，您可能希望使用域名而不是 IP 地址。为此，您需要:

1.  设置一个静态 IP 地址并[将](https://aws.amazon.com/premiumsupport/knowledge-center/ec2-associate-static-public-ip/)关联到您的 EC2 实例
2.  通过亚马逊证书管理器创建一个 SSL 证书
3.  安装新的[弹性负载平衡器](https://aws.amazon.com/elasticloadbalancing/)并在其上安装证书

想挑战吗？为了使整个过程自动化，因此您不需要每次都手动提供一个新实例并在其上安装 Docker，设置[弹性容器服务](https://aws.amazon.com/ecs/)。有关这方面的更多信息，请查看[将 Flask 和 React 微服务部署到 AWS ECS](https://testdriven.io/courses/aws-flask-react/) 课程和[使用 Terraform 将 Django 部署到 AWS ECS](/blog/deploying-django-to-ecs-with-terraform/)教程。

你可以在 django-gitlab-ec2 repo 中找到最终代码。