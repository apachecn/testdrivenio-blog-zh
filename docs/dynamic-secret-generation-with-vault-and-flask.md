# 基于金库和烧瓶的动态秘密生成

> 原文：<https://testdriven.io/blog/dynamic-secret-generation-with-vault-and-flask/>

在本教程中，我们将查看一个快速的真实示例，使用哈希公司的 [Vault](https://www.vaultproject.io/) 和[consult](https://www.consul.io/)为 Flask web 应用程序创建动态 Postgres 凭证。

## 先决条件

开始之前，您应该:

1.  对金库和顾问的秘密管理有基本的工作知识。请参考[用 Vault 管理秘密和咨询](/managing-secrets-with-vault-and-consul)博客帖子了解更多信息。
2.  使用[存储后端](https://www.vaultproject.io/docs/configuration/storage/index.html)部署的 Vault 实例。查看[部署金库和领事](/deploying-vault-and-consul)的帖子，了解如何通过 Docker Swarm 将金库和领事部署到数字海洋。保险存储也应该初始化和解封。
3.  部署了 Postgres 服务器。如果您没有运行 Postgres，请使用 [AWS RDS 自由层](https://aws.amazon.com/rds/free/)。
4.  以前和 Flask 和 Docker 合作过。查看[使用 Python、Flask 和 Docker 的测试驱动开发](/courses/tdd-flask/)课程，了解更多信息。

## 入门指南

让我们从一个基本的 Flask web 应用程序开始。

如果你想继续，复制下[金库-领事-烧瓶](https://github.com/testdrivenio/vault-consul-flask)回购，然后查看 [v1](https://github.com/testdrivenio/vault-consul-flask/tree/v1) 分支:

```py
`$ git clone https://github.com/testdrivenio/vault-consul-flask --branch v1 --single-branch
$ cd vault-consul-flask` 
```

快速浏览一下代码:

```py
`├── .gitignore
├── Dockerfile
├── README.md
├── docker-compose.yml
├── manage.py
├── project
│   ├── __init__.py
│   ├── api
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── models.py
│   │   └── users.py
│   └── config.py
└── requirements.txt` 
```

本质上，为了让这个应用程序工作，我们需要向一个[添加以下环境变量。env](https://docs.docker.com/compose/environment-variables/#the-env-file) 文件(我们将很快完成):

1.  `DB_USER`
2.  `DB_PASSWORD`
3.  `DB_SERVER`

*项目/配置文件*:

```py
`import os

USER = os.environ.get('DB_USER')
PASSWORD = os.environ.get('DB_PASSWORD')
SERVER = os.environ.get('DB_SERVER')

class ProductionConfig():
    """Production configuration"""
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SQLALCHEMY_DATABASE_URI = f'postgresql://{USER}:{PASSWORD}@{SERVER}:5432/users_db'` 
```

## 配置保管库

> 同样，如果您想跟进，您应该有一个部署了[存储后端](https://www.vaultproject.io/docs/configuration/storage/index.html)的 Vault 实例。这个实例也应该被初始化和解封。想要快速启动并运行集群吗？从 [vault-consul-swarm](https://github.com/testdrivenio/vault-consul-swarm) 运行 *deploy.sh* 脚本，将 vault 和 consul 集群部署到三个 DigitalOcean droplets。调配和部署只需不到五分钟的时间！

首先，登录 Vault(如有必要)，然后从 Vault [CLI](https://www.vaultproject.io/docs/commands/index.html) 启用数据库[秘密后端](https://www.vaultproject.io/docs/secrets/databases/index.html):

```py
`$ vault secrets enable database

Success! Enabled the database secrets engine at: database/` 
```

添加 Postgres 连接以及数据库引擎[插件](https://www.vaultproject.io/docs/secrets/databases/postgresql.html)信息:

```py
`$ vault write database/config/users_db \
    plugin_name="postgresql-database-plugin" \
    connection_url="postgresql://{{username}}:{{password}}@<ENDPOINT>:5432/users_db" \
    allowed_roles="mynewrole" \
    username="<USERNAME>" \
    password="<PASSWORD>"` 
```

> 您是否注意到 URL 中有`username`和`password`的模板？这用于防止对密码的直接读取访问，并启用凭据轮换。

确保更新数据库端点以及用户名和密码。例如:

```py
`$ vault write database/config/users_db \
    plugin_name="postgresql-database-plugin" \
    connection_url="postgresql://{{username}}:{{password}}@users-db.c7vzuyfvhlgz.us-east-1.rds.amazonaws.com:5432/users_db" \
    allowed_roles="mynewrole" \
    username="vault" \
    password="lOfon7BA3uzZzxGGv36j"` 
```

这在“database/config/users_db”中创建了一个新的机密路径:

```py
`$ vault list database/config

Keys
----
users_db` 
```

接下来，创建一个名为`mynewrole`的新角色:

```py
`$ vault write database/roles/mynewrole \
    db_name=users_db \
    creation_statements="CREATE ROLE \"{{name}}\" \
 WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
 GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"

Success! Data written to: database/roles/mynewrole` 
```

这里，我们将 Vault 中的`mynewrole`名称映射到一个 SQL 语句，该语句在运行时将创建一个拥有数据库中所有权限的新用户。请记住，这实际上还没有创建新用户。记下默认值和最大 TTL。

现在我们准备创建新用户。

## 创建凭据

快速查看一下`psql`中有哪些用户:

在项目根目录下创建一个名为 *run.sh* 的新文件:

```py
`#!/bin/sh

rm -f .env

echo DB_SERVER=<DB_ENDPOINT> >> .env

user=$(curl  -H "X-Vault-Token: $VAULT_TOKEN" \
        -X GET http://<VAULT_ENDPOINT>:8200/v1/database/creds/mynewrole)
echo DB_USER=$(echo $user | jq -r .data.username) >> .env
echo DB_PASSWORD=$(echo $user | jq -r .data.password) >> .env

docker-compose up -d --build` 
```

因此，这将调用 Vault API 从`/creds`端点生成一组新的凭证。随后的响应通过 JQ 进行解析，并且凭证被添加到一个*中。env* 文件。确保更新数据库(`DB_ENDPOINT`)和 Vault ( `VAULT_ENDPOINT`)端点。

添加`VAULT_TOKEN`环境变量:

```py
`$ export VAULT_TOKEN=<YOUR_VAULT_TOKEN>` 
```

构建映像并旋转容器:

验证环境变量是否已成功添加:

```py
`$ docker-compose exec web env` 
```

您还应该在数据库中看到该用户:

```py
`Role name                                   | Attributes                                  | Member of
--------------------------------------------+---------------------------------------------+----------
 v-root-mynewrol-jC8Imdx2sMTZj03-1533704364 | Password valid until 2018-08-08 05:59:29+00 | {}` 
```

创建并植入数据库`users`表:

```py
`$ docker-compose run web python manage.py recreate-db
$ docker-compose run web python manage.py seed-db` 
```

在浏览器中进行测试，网址为[http://localhost:5000/users](http://localhost:5000/users):

```py
`{ "status":  "success", "users":  [{ "active":  true, "admin":  false, "email":  "[[email protected]](/cdn-cgi/l/email-protection)", "id":  1, "username":  "michael" }] }` 
```

完成后取下容器:

## 结论

就是这样！

请记住，在这个示例中，凭据仅在一个小时内有效。这非常适合短暂的、动态的、一次性的任务。如果您有更长的任务，您可以设置一个 cron 作业来每小时触发一次 run.sh 脚本来获取新的凭证。请记住，最大 TTL 设置为 24 小时。

> 您可能还想看看如何使用 [envconsul](https://github.com/hashicorp/envconsul) 将凭证放入 Flask 的环境中。它甚至可以在凭证更新时重启 Flask。

你可以在[金库-领事-烧瓶](https://github.com/testdrivenio/vault-consul-flask)回购中找到最终代码。