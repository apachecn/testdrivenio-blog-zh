# 通过 Docker 和 GitHub 操作不断将 Django 部署到数字海洋

> 原文：<https://testdriven.io/blog/deploying-django-to-digitalocean-with-docker-and-github-actions/>

在本教程中，我们将了解如何配置 GitHub 操作，以便将 Django 和 Docker 应用程序持续部署到 DigitalOcean。

*依赖关系*:

1.  Django v3.2.4
2.  Docker v20.04
3.  python 3 . 9 . 5 版

## 目标

本教程结束时，您将能够:

1.  使用 Docker 将 Django 部署到数字海洋
2.  配置 GitHub 操作以持续将 Django 部署到数字海洋
3.  使用 GitHub 包来存储 Docker 图像
4.  设置无密码 SSH 登录
5.  为数据持久性配置数字海洋的托管数据库

## 项目设置

除了 Django 和 Docker，我们将使用的演示项目还包括 [Postgres](https://www.postgresql.org/) 、 [Nginx](https://www.nginx.com) 和 [Gunicorn](https://gunicorn.org/) 。

> 好奇这个项目是怎么开发出来的？查看 Postgres、Gunicorn 和 Nginx 的博客文章。

从克隆基础项目开始:

```py
`$ git clone https://github.com/testdrivenio/django-github-digitalocean.git --branch base --single-branch
$ cd django-github-digitalocean` 
```

要进行本地测试，构建映像并旋转容器:

```py
`$ docker-compose up -d --build` 
```

导航到 [http://localhost:8000/](http://localhost:8000/) 。您应该看到:

## GitHub 包

[GitHub Packages](https://github.com/features/packages) 是一个包管理服务，与 GitHub 完全集成。它允许你公开或私下托管你的软件包，在你的 GitHub 项目中使用。我们将使用它来存储 Docker 图像。

假设您在 GitHub 上有一个帐户，为这个项目创建一个新的存储库，并更新本地项目的 origin remote 以指向您刚刚创建的存储库。

要进行本地测试，您需要[创建](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)一个个人访问令牌。在您的[开发者设置](https://github.com/settings/developers)中，点击“个人访问令牌”。然后，单击“生成新令牌”。提供描述性注释并选择以下范围:

1.  `write:packages`
2.  `read:packages`
3.  `delete:packages`

![github personal access token](img/2543c6146eb7a25f207ffa2afa584b44.png)

记下令牌。

构建并标记图像:

```py
`$ docker build -f app/Dockerfile -t ghcr.io/<USERNAME>/<REPOSITORY_NAME>/web:latest ./app

# example:
# docker build -f app/Dockerfile -t ghcr.io/testdrivenio/django-github-digitalocean/web:latest ./app` 
```

接下来，使用您的个人访问令牌，[向 Docker 认证](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#authenticating-to-the-container-registry)GitHub 包:

```py
`$ docker login ghcr.io -u <USERNAME> -p <TOKEN>

# example:
# docker login ghcr.io -u testdrivenio -p ce70f1d4a3a906ce8ac24caa6870fd29f2273d30` 
```

将图像推送到 GitHub 包的[容器注册表](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry):

```py
`$ docker push ghcr.io/<USERNAME>/<REPOSITORY_NAME>/web:latest

# example:
# docker push ghcr.io/testdrivenio/django-github-digitalocean/web:latest` 
```

现在，您应该可以在以下网址之一看到该包(取决于您使用的是个人帐户还是组织):

```py
`https://github.com/orgs/<USERNAME>/packages

https://github.com/<USERNAME>?tab=packages` 
```

![github packages](img/af3064d0ce2fd070daa43ac6bc9f4f25.png)

## 数字海洋

让我们设置 DigitalOcean 来使用我们的应用程序。

首先，你需要[注册](https://m.do.co/c/d8f211a4b4c2)一个[数字海洋](https://www.digitalocean.com/)账户(如果你还没有的话)，然后[生成](https://www.digitalocean.com/docs/apis-clis/api/create-personal-access-token/)一个访问令牌，这样你就可以访问[数字海洋 API](https://developers.digitalocean.com/documentation/v2/) 。

将令牌添加到您的环境中:

```py
`$ export DIGITAL_OCEAN_ACCESS_TOKEN=[your_digital_ocean_token]` 
```

### 微滴

接下来，使用预装的 Docker 创建一个新的 Droplet:

```py
`$ curl -X POST \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    -d '{"name":"django-docker","region":"sfo3","size":"s-2vcpu-4gb","image":"docker-20-04"}' \
    "https://api.digitalocean.com/v2/droplets"` 
```

检查状态:

```py
`$ curl \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    "https://api.digitalocean.com/v2/droplets?name=django-docker"` 
```

如果您安装了 [jq](https://stedolan.github.io/jq/) ，那么您可以像这样解析 JSON 响应:

```py
`$ curl \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    "https://api.digitalocean.com/v2/droplets?name=django-docker" \
  | jq '.droplets[0].status'` 
```

root 密码应该会通过电子邮件发送给您。找回它。然后，一旦 droplet 的状态为`active`，就以 root 身份 SSH 到实例中，并在提示时更新密码。

接下来，生成一个新的 SSH 密钥:

将密钥保存到 */root/。ssh/id_rsa* 并且不设置密码。这将分别生成一个公钥和私钥- *id_rsa* 和 *id_rsa.pub* 。要设置无密码 SSH 登录，请将公钥复制到 [authorized_keys](https://security.stackexchange.com/questions/20706/what-is-the-difference-between-authorized-keys-and-known-hosts-file-for-ssh) 文件中，并设置适当的权限:

```py
`$ cat ~/.ssh/id_rsa.pub
$ vi ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/id_rsa` 
```

复制私钥的内容:

退出 SSH 会话，然后将密钥设置为本地计算机上的环境变量:

```py
`export PRIVATE_KEY='-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA04up8hoqzS1+APIB0RhjXyObwHQnOzhAk5Bd7mhkSbPkyhP1
...
iWlX9HNavcydATJc1f0DpzF0u4zY8PY24RVoW8vk+bJANPp1o2IAkeajCaF3w9nf
q/SyqAWVmvwYuIhDiHDaV2A==
-----END RSA PRIVATE KEY-----'` 
```

将密钥添加到 [ssh-agent](https://www.ssh.com/ssh/agent) 中:

```py
`$ ssh-add - <<< "${PRIVATE_KEY}"` 
```

要进行测试，请运行:

然后，为应用程序创建一个新目录:

### 数据库ˌ资料库

接下来，让我们通过数字海洋的[托管数据库](https://www.digitalocean.com/products/managed-databases/)建立一个生产 Postgres 数据库:

```py
`$ curl -X POST \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    -d '{"name":"django-docker-db","region":"sfo3","engine":"pg","version":"13","size":"db-s-2vcpu-4gb","num_nodes":1}' \
    "https://api.digitalocean.com/v2/databases"` 
```

检查状态:

```py
`$ curl \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    "https://api.digitalocean.com/v2/databases?name=django-docker-db" \
  | jq '.databases[0].status'` 
```

它应该需要几分钟才能旋转起来。一旦状态为`online`，获取连接信息:

```py
`$ curl \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$DIGITAL_OCEAN_ACCESS_TOKEN'' \
    "https://api.digitalocean.com/v2/databases?name=django-docker-db" \
  | jq '.databases[0].connection'` 
```

示例响应:

```py
`{
  "protocol": "postgresql",
  "uri": "postgresql://doadmin:[[email protected]](/cdn-cgi/l/email-protection)locean.com:25060/defaultdb?sslmode=require",
  "database": "defaultdb",
  "host": "django-docker-db-do-user-778274-0.a.db.ondigitalocean.com",
  "port": 25060,
  "user": "doadmin",
  "password": "v60qcyaito1i0h66",
  "ssl": true
}` 
```

## GitHub 操作

要配置 [GitHub 动作](https://github.com/features/actions)，首先添加一个名为。github”在您的项目的根目录中。在该目录中添加另一个名为“workflows”的目录。现在，要配置由一个或多个作业组成的工作流，在“工作流”目录中创建一个名为 *main.yml* 的新文件。

### 构建作业

```py
`name:  Continuous Integration and Delivery on:  [push] env: WEB_IMAGE:  ghcr.io/$(echo $GITHUB_REPOSITORY | tr '[:upper:]' '[:lower:]')/web NGINX_IMAGE:  ghcr.io/$(echo $GITHUB_REPOSITORY | tr '[:upper:]' '[:lower:]')/nginx jobs: build: name:  Build Docker Images runs-on:  ubuntu-latest steps: -  name:  Checkout master uses:  actions/[[email protected]](/cdn-cgi/l/email-protection) -  name:  Add environment variables to .env run:  | echo DEBUG=0 >> .env echo SQL_ENGINE=django.db.backends.postgresql >> .env echo DATABASE=postgres >> .env echo SECRET_KEY=${{ secrets.SECRET_KEY }} >> .env echo SQL_DATABASE=${{ secrets.SQL_DATABASE }} >> .env echo SQL_USER=${{ secrets.SQL_USER }} >> .env echo SQL_PASSWORD=${{ secrets.SQL_PASSWORD }} >> .env echo SQL_HOST=${{ secrets.SQL_HOST }} >> .env echo SQL_PORT=${{ secrets.SQL_PORT }} >> .env -  name:  Set environment variables run:  | echo "WEB_IMAGE=$(echo ${{env.WEB_IMAGE}} )" >> $GITHUB_ENV echo "NGINX_IMAGE=$(echo ${{env.NGINX_IMAGE}} )" >> $GITHUB_ENV -  name:  Log in to GitHub Packages run:  echo ${PERSONAL_ACCESS_TOKEN} | docker login ghcr.io -u ${{ secrets.NAMESPACE }} --password-stdin env: PERSONAL_ACCESS_TOKEN:  ${{ secrets.PERSONAL_ACCESS_TOKEN }} -  name:  Pull images run:  | docker pull ${{ env.WEB_IMAGE }} || true docker pull ${{ env.NGINX_IMAGE }} || true -  name:  Build images run:  | docker-compose -f docker-compose.ci.yml build -  name:  Push images run:  | docker push ${{ env.WEB_IMAGE }} docker push ${{ env.NGINX_IMAGE }}` 
```

这里，我们定义了一个`build` [作业](https://help.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobs)，它将在我们:

1.  设置全局环境变量`WEB_IMAGE`和`NGINX_IMAGE`
2.  签出存储库，以便作业可以访问它
3.  给一个*添加环境变量。环境*文件
4.  [用设置`WEB_IMAGE`和`NGINX_IMAGE`环境变量](https://help.github.com/en/actions/reference/workflow-commands-for-github-actions#setting-an-environment-variable)，这样就可以在 Docker 合成文件中访问它们
5.  登录 GitHub 包
6.  如果图像存在，请提取图像
7.  构建图像
8.  将图片上传到 GitHub 包

你注意到那些秘密了吗？

1.  `secrets.SECRET_KEY`
2.  `secrets.SQL_DATABASE`
3.  `secrets.SQL_USER`
4.  `secrets.SQL_PASSWORD`
5.  `secrets.SQL_HOST`
6.  `secrets.SQL_PORT`
7.  `secrets.NAMESPACE`
8.  `secrets.PERSONAL_ACCESS_TOKEN`

这些需要在你的储存库的[秘密](https://help.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets#creating-encrypted-secrets)中设置(设置>秘密)。根据上面的数据库连接信息，添加前六个变量。

例如:

1.  `SECRET_KEY` : `9zYGEFk2mn3mWB8Bmg9SAhPy6F4s7cCuT8qaYGVEnu7huGRKW9`
2.  `SQL_DATABASE` : `defaultdb`
3.  `SQL_HOST` : `django-docker-db-do-user-778274-0.a.db.ondigitalocean.com`
4.  `SQL_PORT` : `25060`
5.  `SQL_USER` : `doadmin`
6.  `SQL_PASSWORD` : `v60qcyaito1i0h66`

对`NAMESPACE`使用您的 Github 用户名或您的组织名称，对`PERSONAL_ACCESS_TOKEN`变量使用您的个人访问令牌。

![github secrets](img/bc3ed618e72aec266e8888e6e46135ff.png)

完成后，提交代码并上传到 GitHub 以触发新的构建。确保它通过。您应该会看到 GitHub 包中的图像:

![github packages](img/c7ca3a01beccda7b915f6f28595e5030.png)

### 部署作业

接下来，添加一个`deploy`任务:

```py
`deploy: name:  Deploy to DigitalOcean runs-on:  ubuntu-latest needs:  build steps: -  name:  Checkout master uses:  actions/[[email protected]](/cdn-cgi/l/email-protection) -  name:  Add environment variables to .env run:  | echo DEBUG=0 >> .env echo SQL_ENGINE=django.db.backends.postgresql >> .env echo DATABASE=postgres >> .env echo SECRET_KEY=${{ secrets.SECRET_KEY }} >> .env echo SQL_DATABASE=${{ secrets.SQL_DATABASE }} >> .env echo SQL_USER=${{ secrets.SQL_USER }} >> .env echo SQL_PASSWORD=${{ secrets.SQL_PASSWORD }} >> .env echo SQL_HOST=${{ secrets.SQL_HOST }} >> .env echo SQL_PORT=${{ secrets.SQL_PORT }} >> .env echo WEB_IMAGE=${{ env.WEB_IMAGE }} >> .env echo NGINX_IMAGE=${{ env.NGINX_IMAGE }} >> .env echo NAMESPACE=${{ secrets.NAMESPACE }} >> .env echo PERSONAL_ACCESS_TOKEN=${{ secrets.PERSONAL_ACCESS_TOKEN }} >> .env -  name:  Add the private SSH key to the ssh-agent env: SSH_AUTH_SOCK:  /tmp/ssh_agent.sock run:  | mkdir -p ~/.ssh ssh-agent -a $SSH_AUTH_SOCK > /dev/null ssh-keyscan github.com >> ~/.ssh/known_hosts ssh-add - <<< "${{ secrets.PRIVATE_KEY }}" -  name:  Build and deploy images on DigitalOcean env: SSH_AUTH_SOCK:  /tmp/ssh_agent.sock run:  | scp  -o StrictHostKeyChecking=no -r ./.env ./docker-compose.prod.yml [[email protected]](/cdn-cgi/l/email-protection)${{ secrets.DIGITAL_OCEAN_IP_ADDRESS }}:/app ssh -o StrictHostKeyChecking=no [[email protected]](/cdn-cgi/l/email-protection)${{ secrets.DIGITAL_OCEAN_IP_ADDRESS }} << 'ENDSSH' cd /app source .env docker login ghcr.io -u $NAMESPACE -p $PERSONAL_ACCESS_TOKEN docker pull $WEB_IMAGE docker pull $NGINX_IMAGE docker-compose -f docker-compose.prod.yml up -d ENDSSH` 
```

因此，在`deploy`作业中，只有当`build`作业成功完成时才会运行(通过`needs: build`，我们:

1.  签出存储库，以便作业可以访问它
2.  给一个*添加环境变量。环境*文件
3.  将私有 SSH 密钥添加到 ssh-agent 并运行代理
4.  复制完*。env* 和 *docker-compose.prod.yml* 文件到远程服务器
5.  SSH 到数字海洋上的删除服务器
6.  导航到部署目录并设置环境变量
7.  登录 GitHub 包
8.  调出图像
9.  旋转容器
10.  结束 SSH 会话

将`DIGITAL_OCEAN_IP_ADDRESS`和`PRIVATE_KEY`秘密添加到 GitHub。然后，将服务器的 IP 地址添加到 Django 设置中的`ALLOWED_HOSTS`列表中。

提交并推送您的代码以触发新的构建。构建通过后，导航到实例的 IP。您应该看到:

## 试验

最后，通过在`needs: build`下面添加`if: github.ref == 'refs/heads/master'`来更新`deploy`作业，以便它只在对`master`分支进行更改时运行:

```py
`deploy: name:  Deploy to DigitalOcean runs-on:  ubuntu-latest needs:  build if:  github.ref == 'refs/heads/master' steps: -  name:  Checkout master uses:  actions/[[email protected]](/cdn-cgi/l/email-protection) -  name:  Add environment variables to .env run:  | echo DEBUG=0 >> .env echo SQL_ENGINE=django.db.backends.postgresql >> .env echo DATABASE=postgres >> .env echo SECRET_KEY=${{ secrets.SECRET_KEY }} >> .env echo SQL_DATABASE=${{ secrets.SQL_DATABASE }} >> .env echo SQL_USER=${{ secrets.SQL_USER }} >> .env echo SQL_PASSWORD=${{ secrets.SQL_PASSWORD }} >> .env echo SQL_HOST=${{ secrets.SQL_HOST }} >> .env echo SQL_PORT=${{ secrets.SQL_PORT }} >> .env echo WEB_IMAGE=${{ env.WEB_IMAGE }} >> .env echo NGINX_IMAGE=${{ env.NGINX_IMAGE }} >> .env echo NAMESPACE=${{ secrets.NAMESPACE }} >> .env echo PERSONAL_ACCESS_TOKEN=${{ secrets.PERSONAL_ACCESS_TOKEN }} >> .env -  name:  Add the private SSH key to the ssh-agent env: SSH_AUTH_SOCK:  /tmp/ssh_agent.sock run:  | mkdir -p ~/.ssh ssh-agent -a $SSH_AUTH_SOCK > /dev/null ssh-keyscan github.com >> ~/.ssh/known_hosts ssh-add - <<< "${{ secrets.PRIVATE_KEY }}" -  name:  Build and deploy images on DigitalOcean env: SSH_AUTH_SOCK:  /tmp/ssh_agent.sock run:  | scp  -o StrictHostKeyChecking=no -r ./.env ./docker-compose.prod.yml [[email protected]](/cdn-cgi/l/email-protection)${{ secrets.DIGITAL_OCEAN_IP_ADDRESS }}:/app ssh -o StrictHostKeyChecking=no [[email protected]](/cdn-cgi/l/email-protection)${{ secrets.DIGITAL_OCEAN_IP_ADDRESS }} << 'ENDSSH' cd /app source .env docker login ghcr.io -u $NAMESPACE -p $PERSONAL_ACCESS_TOKEN docker pull $WEB_IMAGE docker pull $NGINX_IMAGE docker-compose -f docker-compose.prod.yml up -d ENDSSH` 
```

为了测试，创建一个新的`develop`分支。在 *urls.py* 中的`world`后加一个感叹号:

```py
`def home(request):
    return JsonResponse({"hello": "world!"})` 
```

将您的更改提交并推送到 GitHub。确保仅运行`build`作业。一旦构建通过，打开一个针对`master`分支的 PR 并合并变更。这将触发两个阶段的新构建- `build`和`deploy`。确保部署按预期工作:

--

就是这样！你可以在 django-github-digital oceanrepo 中找到最终代码。