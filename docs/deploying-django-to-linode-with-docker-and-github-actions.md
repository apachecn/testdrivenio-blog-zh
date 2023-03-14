# 通过 Docker 和 GitHub 操作不断将 Django 部署到 Linode

> 原文：<https://testdriven.io/blog/deploying-django-to-linode-with-docker-and-github-actions/>

在本教程中，我们将了解如何配置 [GitHub 动作](https://github.com/features/actions)来持续地将 [Django](https://www.djangoproject.com/) 和 [Docker](https://www.docker.com/) 应用程序部署到 [Linode](https://www.linode.com/) 。

## 目标

学完本教程后，您应该能够:

1.  将 Django 应用程序部署到 Linode
2.  使用 GitHub 包来存储 Docker 图像
3.  建立一个 Linode CPU 实例和一个 Linode 管理的数据库
4.  启用无密码 SSH 验证
5.  使用 GitHub 操作持续部署您的应用程序

## 项目设置

首先克隆 [GitHub repo](https://github.com/testdrivenio/django-github-linode) 并将当前目录更改为其根目录:

```
`$ git clone [[email protected]](/cdn-cgi/l/email-protection):testdrivenio/django-github-linode.git --branch base --single-branch
$ cd django-github-linode` 
```

> 好奇这个项目是怎么开发出来的？查看 Postgres、Gunicorn 和 Nginx 的博客文章。

要进行本地测试，构建映像并旋转容器:

```
`$ docker-compose up -d --build` 
```

> 如果你的容器以`exec /usr/src/app/entrypoint.sh: no such file or directory`退出，打开 *app/entrypoint.sh* 脚本，执行从`CR LF`到`LF`的 EOL 转换。之后再运行`docker-compose up -d --build`。

导航到 [http://localhost:8000/](http://localhost:8000/) 。您应该看到:

## GitHub 包

[GitHub Packages](https://github.com/features/packages) 是一个托管和管理包的平台，包括容器和其他依赖项。它允许你在源代码旁边的 GitHub 上托管你的软件包。我们将使用它来存储 Docker 图像。

要使用 GitHub 包，首先必须创建一个个人访问令牌。导航至[开发者设置](https://github.com/settings/apps)并选择“个人访问令牌”。点击“生成新令牌”。

给它一个名称/注释-例如，“GitHub actions”-并添加以下权限:

*   `write:packages`
*   `read:packages`
*   `delete:packages`

接下来，单击“生成令牌”按钮。

![GitHub Personal Access Token](img/fd18e5f2856be39580053f0011699577.png)

您将只能看到令牌一次，因此请将其存放在安全的地方。

构建并标记容器:

```
`$ docker build -f app/Dockerfile -t ghcr.io/<USERNAME>/<REPOSITORY_NAME>/web:latest ./app

# Example
$ docker build -f app/Dockerfile -t ghcr.io/testdrivenio/django-github-linode/web:latest ./app` 
```

接下来，使用您的个人访问令牌登录 GitHub 包:

```
`$ docker login ghcr.io -u <USERNAME> -p <TOKEN>

# Example
$ docker login ghcr.io -u testdrivenio -p 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` 
```

最后，将图像推送到 GitHub 包的[容器注册表](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)中。

```
`$ docker push ghcr.io/<USERNAME>/<REPOSITORY_NAME>/web:latest

# Example
$ docker push ghcr.io/testdrivenio/django-github-linode/web:latest` 
```

现在，您应该可以在以下网址之一看到该包(取决于您使用的是个人帐户还是组织):

```
`# Personal account
https://github.com/<USERNAME>?tab=packages

# Organization
https://github.com/orgs/<ORGANIZATION_NAME>/packages` 
```

![Linode Create Database Cluster](img/055ffe85058e8f6a8ee432f940f634cb.png)

## Linode CPU

> 如果您还没有 Linode 帐户，[导航到他们的网站](https://www.linode.com/)并点击“注册”。

首先，我们需要创建一个 [Linode CPU 实例](https://www.linode.com/products/dedicated-cpu/)。

登录 [Linode 仪表盘](https://cloud.linode.com/)并选择侧边栏上的“Linodes”。然后点击“创建 Linode”。

![Linode Create](img/fe05db07820fbdf50d2dc574a47ca224.png)

要创建一个预装 Docker 的 Linode，我们可以使用 [Linode Marketplace](https://www.linode.com/marketplace/) 。在导航栏中选择“市场”，在应用程序中选择“Docker”

![Linode Create Marketplace](img/387911074a4d3a0498a8b4a4011098cf.png)

向下滚动到“选择图像”。对于图像，选择“LTS Ubuntu 20.04”。对于地区，请选择离您的客户最近的地区。然后根据你的项目需求选择一个“Linode 计划”。

> 如果您正在部署 CPU 密集型 web 应用程序，请选择专用 CPU。如果您预算有限，并且正在部署一个简单的 web 应用程序，请放心使用共享 CPU。有关更多信息，请查看[专用与共享 CPU 实例](https://www.linode.com/content/linode-dedicated-cpus-explained-dedicated-vs-shared-cpu-instances/)。

![Linode Marketplace Image](img/3302973c37c095fb41ad871ac402828f.png)

让我们把我们的 Linode 命名为“django-docker”。将标签留空。然后选择一个强根密码。您应该记下来，但是没有必要记住它，因为我们很快就会启用无密码 SSH。

出于身份验证的目的，我们将使用 SSH 密钥。要生成密钥，请运行:

将密钥保存到 */root/。ssh/id_rsa* 并且不设置密码短语。这将分别生成一个公钥和私钥- *id_rsa* 和 *id_rsa.pub* 。要设置无密码 SSH 登录，请将公钥复制到 *authorized_keys* 文件中，并设置适当的权限:

> 如果您愿意，可以随意使用现有的 SSH 密钥。

```
`$ cat ~/.ssh/id_rsa.pub
$ vi ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/id_rsa` 
```

导航回到 Linode 仪表板，点击“添加 SSH 密钥”。给它一个标签并粘贴你的公钥。

![Linode Create Security](img/7e0673cef486e32ea0596c2203305ea4.png)

将其他内容留空，滚动到底部。点击“创建 Linode”。

您将被重定向到您的 Linode 详细信息。Linode 需要几分钟的时间进行配置。准备就绪后，其状态将变为“正在运行”。

![Linode Details](img/0fed2e1986f79231af9064268674029c.png)

获取 Linode 的 IP，让我们测试一下是否一切正常。

要连接到新创建的 Linode 实例并检查 Docker 版本，请运行:

```
`$ ssh -o StrictHostKeyChecking=no [[email protected]](/cdn-cgi/l/email-protection)<YOUR_INSTANCE_IP> docker --version

Docker version 20.10.17, build 100c701` 
```

> 实例运行后，Docker 可能还需要几分钟才能完成安装。

然后，为应用程序创建一个新目录:

很好，Linode CPU 实例现在已经准备好了。在下一步中，我们将设置一个托管数据库。

## 数据库ˌ资料库

为了托管我们的数据库，我们将使用 [Linode 数据库](https://www.linode.com/products/databases/)。

在侧边栏上，导航到“数据库”并单击“创建数据库集群”。

![Linode Create Database Cluster](img/e87ed0a8223ce38e33b76e55af7a291c.png)

将其标记为“django-docker-db”，选择“PostgreSQL v13.2”作为引擎，并为该地区选择最接近您的客户的引擎。根据您的项目需求选择 CPU 计划。

> 对于一个玩具项目，使用共享的 CPU，因为专用的 CPU 数据库非常昂贵。

![Linode Create Database Cluster](img/055ffe85058e8f6a8ee432f940f634cb.png)

一个节点应该绰绰有余。对于访问控制，添加您的 Linode CPU 实例 IP 地址。

> 要允许任何 IP 连接到数据库，您可以将`0.0.0.0/0`添加到访问控制列表中。请记住，这是一种糟糕的安全做法，只应该出于测试目的而这样做。不要像这样公开生产数据。

单击“创建数据库集群”。数据库启动大约需要 10-15 分钟。一旦它准备好了，就获取它的连接细节。

![Linode Database Details](img/9284d1f07ffc23365546d1743e616750.png)

## GitHub 操作

[GitHub Actions](https://docs.github.com/en/actions) 允许您在 GitHub 存储库中自动化、定制和执行软件开发工作流。当代码被推送到 GitHub repo 时，我们将使用它来构建和部署 Docker 映像。

要配置 GitHub 动作，首先需要创建一个“.github”目录。接下来，在该目录中创建“工作流”目录，并在“工作流”中创建 *main.yml* :

```
`.github
└── workflows
    └── main.yml` 
```

### 构建作业

让我们从创建构建作业开始。

将以下内容放入*。github/workflows/main.yml* :

```
`name:  Continuous  Integration  and  Delivery on:  [push] env: WEB_IMAGE:  ghcr.io/$(echo  $GITHUB_REPOSITORY  |  tr  '[:upper:]'  '[:lower:]')/web NGINX_IMAGE:  ghcr.io/$(echo  $GITHUB_REPOSITORY  |  tr  '[:upper:]'  '[:lower:]')/nginx jobs: build: name:  Build  Docker  Images runs-on:  ubuntu-latest steps: -  name:  Checkout  master uses:  actions/checkout@v1 -  name:  Add  environment  variables  to  .env run:  | echo  "DEBUG=0"  >>  .env echo  "SQL_ENGINE=django.db.backends.postgresql"  >>  .env echo  "DATABASE=postgres"  >>  .env echo  "SECRET_KEY=${{ secrets.SECRET_KEY }}"  >>  .env echo  "SQL_DATABASE=${{ secrets.SQL_DATABASE }}"  >>  .env echo  "SQL_USER=${{ secrets.SQL_USER }}"  >>  .env echo  "SQL_PASSWORD=${{ secrets.SQL_PASSWORD }}"  >>  .env echo  "SQL_HOST=${{ secrets.SQL_HOST }}"  >>  .env echo  "SQL_PORT=${{ secrets.SQL_PORT }}"  >>  .env -  name:  Set  environment  variables run:  | echo  "WEB_IMAGE=$(echo ${{env.WEB_IMAGE}} )"  >>  $GITHUB_ENV echo  "NGINX_IMAGE=$(echo ${{env.NGINX_IMAGE}} )"  >>  $GITHUB_ENV -  name:  Log  in  to  GitHub  Packages run:  echo  ${PERSONAL_ACCESS_TOKEN}  |  docker  login  ghcr.io  -u  ${{  secrets.NAMESPACE  }}  --password-stdin env: PERSONAL_ACCESS_TOKEN:  ${{  secrets.PERSONAL_ACCESS_TOKEN  }} -  name:  Pull  images run:  | docker  pull  ${{  env.WEB_IMAGE  }}  ||  true docker  pull  ${{  env.NGINX_IMAGE  }}  ||  true -  name:  Build  images run:  | docker-compose  -f  docker-compose.ci.yml  build -  name:  Push  images run:  | docker  push  ${{  env.WEB_IMAGE  }} docker  push  ${{  env.NGINX_IMAGE  }}` 
```

首先，我们用`on`命名工作流并定义它何时运行。接下来，我们设置环境变量并定义`build`作业。一个[任务](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#jobs)由一个或多个顺序运行的命令组成。

为了避免泄露像密码和密钥这样的私有变量，我们使用了[秘密](https://docs.github.com/en/actions/security-guides/encrypted-secrets)。因此，为了成功完成这项工作，您需要向 GitHub 库添加一些秘密。

导航至:

```
`https://github.com/<USERNAME|ORGANIZATION_NAME>/<REPOSITORY_NAME></settings

# Example
https://github.com/testdrivenio/django-github-linode/settings` 
```

点击“秘密”,然后点击“操作”,添加以下秘密:

1.  `SECRET_KEY` : `^8!w90zymm9_0z3h4!_n637hw$^-7g%5-l0npq+zbmqz!v22q9`
2.  `SQL_DATABASE` : `postgres`
3.  `SQL_USER` : `linpostgres`
4.  `SQL_PASSWORD` : `yjNajtqytZU1mp1t`
5.  `SQL_HOST` : `lin-7098-1320-pgsql-primary.servers.linodedb.net`
6.  `SQL_PORT` : `5432`
7.  `NAMESPACE`:您的 GitHub 用户名或您的组织名称
8.  `PERSONAL_ACCESS_TOKEN`:您的 GitHub 个人访问令牌

确保用您自己的凭据替换 SQL 连接信息。

![GitHub Repository Secrets](img/083649db7fe05e01f906d2d614e8faef.png)

接下来，在 Django 设置中将您的 Linode 的 IP 地址添加到`ALLOWED_HOSTS`:

```
`# app/hello_django/settings.py

ALLOWED_HOSTS = ['localhost', '127.0.0.1', '<your Linode IP>']` 
```

提交您的更改并将您的代码上传到 GitHub 以触发新的构建。确保它成功完成。

![Build Job Details](img/4bfc86f3e653fa29b3a2e4e76f5a40df.png)

如果您检查您的包，您会注意到创建了两个映像，`web`和`nginx`。 [NGINX](https://www.nginx.com/) 作为反向代理服务于 web 应用和静态/媒体文件。

### 部署作业

接下来，在`build`作业之后添加一个名为`deploy`的新作业:

```
`deploy: name:  Deploy  to  Linode runs-on:  ubuntu-latest needs:  build steps: -  name:  Checkout  master uses:  actions/checkout@v1 -  name:  Add  environment  variables  to  .env run:  | echo  "DEBUG=0"  >>  .env echo  "SQL_ENGINE=django.db.backends.postgresql"  >>  .env echo  "DATABASE=postgres"  >>  .env echo  "SECRET_KEY=${{ secrets.SECRET_KEY }}"  >>  .env echo  "SQL_DATABASE=${{ secrets.SQL_DATABASE }}"  >>  .env echo  "SQL_USER=${{ secrets.SQL_USER }}"  >>  .env echo  "SQL_PASSWORD=${{ secrets.SQL_PASSWORD }}"  >>  .env echo  "SQL_HOST=${{ secrets.SQL_HOST }}"  >>  .env echo  "SQL_PORT=${{ secrets.SQL_PORT }}"  >>  .env echo  "WEB_IMAGE=${{ env.WEB_IMAGE }}"  >>  .env echo  "NGINX_IMAGE=${{ env.NGINX_IMAGE }}"  >>  .env echo  "NAMESPACE=${{ secrets.NAMESPACE }}"  >>  .env echo  "PERSONAL_ACCESS_TOKEN=${{ secrets.PERSONAL_ACCESS_TOKEN }}"  >>  .env -  name:  Add  the  private  SSH  key  to  the  ssh-agent env: SSH_AUTH_SOCK:  /tmp/ssh_agent.sock run:  | mkdir  -p  ~/.ssh ssh-agent  -a  $SSH_AUTH_SOCK  >  /dev/null ssh-keyscan  github.com  >>  ~/.ssh/known_hosts ssh-add  -  <<<  "${{ secrets.PRIVATE_KEY }}" -  name:  Build  and  deploy  images  on  Linode env: SSH_AUTH_SOCK:  /tmp/ssh_agent.sock run:  | scp  -o  StrictHostKeyChecking=no  -r  ./.env  ./docker-compose.prod.yml  root@${{  secrets.LINODE_IP_ADDRESS  }}:/app ssh  -o  StrictHostKeyChecking=no  root@${{  secrets.LINODE_IP_ADDRESS  }}  <<  'ENDSSH' cd  /app source  .env docker  login  ghcr.io  -u  $NAMESPACE  -p  $PERSONAL_ACCESS_TOKEN docker  pull  $WEB_IMAGE docker  pull  $NGINX_IMAGE sudo  docker-compose  -f  docker-compose.prod.yml  up  -d ENDSSH` 
```

只有当`build`任务成功完成(通过`needs: build`)时，该任务才会运行。

作业步骤:

1.  检查存储库。
2.  将环境变量(包括秘密)添加到一个*中。env* 文件。
3.  初始化 SSH 代理并添加 Linode 的私有 SSH 密钥。
4.  复制*。env* 和 *docker-compose.prod* 到 Linode。
5.  嘘去里诺德。
6.  更改活动目录。
7.  登录到容器注册表并提取图像。
8.  使用 Docker Compose 部署图像。

将以下两个秘密添加到您的回购中:

1.  `LINODE_IP_ADDRESS`:您的链接的 IP 地址
2.  `PRIVATE_KEY`:你的 SSH 私钥

为了测试，提交然后推送你的代码。

确保两个作业都成功完成。然后，导航到您的网站。您应该看到:

## 试验

最后，为了确保部署作业只在对主分支进行更改时运行，在`needs: build`下面添加`if: github.ref == 'refs/heads/master'`。

> 如果你使用`main`作为默认分支，确保用`main`替换`master`。

```
`deploy: name:  Deploy  to  Linode runs-on:  ubuntu-latest needs:  build if:  github.ref  ==  'refs/heads/master' steps: -  name:  Checkout  master uses:  actions/checkout@v1 -  name:  Add  environment  variables  to  .env run:  | echo  "DEBUG=0"  >>  .env echo  "SQL_ENGINE=django.db.backends.postgresql"  >>  .env echo  "DATABASE=postgres"  >>  .env echo  "SECRET_KEY=${{ secrets.SECRET_KEY }}"  >>  .env echo  "SQL_DATABASE=${{ secrets.SQL_DATABASE }}"  >>  .env echo  "SQL_USER=${{ secrets.SQL_USER }}"  >>  .env echo  "SQL_PASSWORD=${{ secrets.SQL_PASSWORD }}"  >>  .env echo  "SQL_HOST=${{ secrets.SQL_HOST }}"  >>  .env echo  "SQL_PORT=${{ secrets.SQL_PORT }}"  >>  .env echo  "WEB_IMAGE=${{ env.WEB_IMAGE }}"  >>  .env echo  "NGINX_IMAGE=${{ env.NGINX_IMAGE }}"  >>  .env echo  "NAMESPACE=${{ secrets.NAMESPACE }}"  >>  .env echo  "PERSONAL_ACCESS_TOKEN=${{ secrets.PERSONAL_ACCESS_TOKEN }}"  >>  .env -  name:  Add  the  private  SSH  key  to  the  ssh-agent env: SSH_AUTH_SOCK:  /tmp/ssh_agent.sock run:  | mkdir  -p  ~/.ssh ssh-agent  -a  $SSH_AUTH_SOCK  >  /dev/null ssh-keyscan  github.com  >>  ~/.ssh/known_hosts ssh-add  -  <<<  "${{ secrets.PRIVATE_KEY }}" -  name:  Build  and  deploy  images  on  Linode env: SSH_AUTH_SOCK:  /tmp/ssh_agent.sock run:  | scp  -o  StrictHostKeyChecking=no  -r  ./.env  ./docker-compose.prod.yml  root@${{  secrets.LINODE_IP_ADDRESS  }}:/app ssh  -o  StrictHostKeyChecking=no  root@${{  secrets.LINODE_IP_ADDRESS  }}  <<  'ENDSSH' cd  /app source  .env docker  login  ghcr.io  -u  $NAMESPACE  -p  $PERSONAL_ACCESS_TOKEN docker  pull  $WEB_IMAGE docker  pull  $NGINX_IMAGE sudo  docker-compose  -f  docker-compose.prod.yml  up  -d ENDSSH` 
```

为了测试它是否工作，创建一个新的`dev`分支。然后将 *app/hello_django/urls.py* 中的`hello world`消息改为`hello linode`:

```
`def home(request):
    return JsonResponse({"hello": "linode"})` 
```

将您的更改提交并推送到 GitHub。确保仅运行`build`作业。一旦构建通过，打开一个针对`master`(或者`main`)分支的 PR 并合并变更。这将触发两个阶段的新构建，`build`和`deploy`。确保部署按预期工作:

--

就是这样。你可以从[django-github-Li node](https://github.com/testdrivenio/django-github-linode)repo 中抓取最终的源代码。