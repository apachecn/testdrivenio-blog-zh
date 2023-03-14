# 使用 Vault 和 Consul 管理秘密

> 原文：<https://testdriven.io/blog/managing-secrets-with-vault-and-consul/>

下面的教程详细介绍了如何设置和使用哈希公司的[金库](https://www.vaultproject.io/)和[领事](https://www.consul.io/)项目来安全地存储和管理机密。

我们将从在 Docker 容器中构建一个 Vault 实例开始，然后开始管理静态[和动态](https://learn.hashicorp.com/vault/secrets-management/sm-static-secrets)[秘密以及 Vault 的“加密即服务”特性。然后，我们将把 Consul 添加到这个组合中，看看如何扩展 Vault。](https://learn.hashicorp.com/vault/getting-started/dynamic-secrets)

> 这是一个中级教程。假设你对 [Docker](https://www.docker.com/) 有基本的工作知识。还建议您通读官方文档中的[简介](https://www.vaultproject.io/docs/what-is-vault)、[内部](https://www.vaultproject.io/docs/internals)和[基本概念](https://www.vaultproject.io/docs/concepts)指南，以便在开始之前熟悉跳马。

*主要依赖:*

*   文档 v20.10.8
*   坞站-复合 v1.29.2
*   保险库版本 1.8.2
*   领事 v1.10.2

## 目标

学完本教程后，您应该能够:

1.  解释什么是保险库，以及为什么您可能想要使用它
2.  描述基本的 Vault 架构以及动态和静态机密、各种后端(存储、机密、验证、审计)，以及如何将 Vault 用作“加密即服务”
3.  配置并运行 Vault，并咨询码头工人
4.  使用文件系统后端启动 Vault
5.  初始化并解封保险库
6.  根据保管库进行身份验证
7.  配置审核后端以记录与 Vault 的所有交互
8.  通过 CLI、HTTP API 和 UI 处理静态和动态机密
9.  创建存储策略以限制对特定路径的访问
10.  将传输后端用作“加密即服务”
11.  设置 Consul 使用 Vault 作为机密的存储后端
12.  定义机密的自定义租期，并在租期结束前撤销机密

## 什么是跳马？

[Vault](https://www.vaultproject.io/) 是一款开源工具，用于安全存储和管理机密。

> 什么是秘密？在本教程的上下文中，秘密是安全敏感的或个人可识别的信息，如数据库凭证、SSH 密钥、用户名和密码、AWS IAM 凭证、API 令牌、社会安全号、信用卡号等等。

花点时间想想您的团队目前是如何管理和传播秘密的:

1.  谁能接触到它们？
2.  谁管理他们？
3.  你如何控制谁可以访问它们？
4.  您的应用程序如何获得它们？
5.  它们是如何更新的？
6.  它们是如何被撤销的？

Vault 为这些问题提供了答案，并有助于解决以下与机密管理相关的问题:

| 问题 | 跳马的目标 |
| --- | --- |
| 秘密无处不在。 | 金库是所有秘密真相的唯一来源。 |
| 它们通常不加密。 | Vault 管理开箱即用的加密(在传输期间和静止时)。 |
| 很难动态地生成它们。 | 秘密可以动态生成。 |
| 租赁和撤销它们就更难了。 | 秘密可以出租和撤销。 |
| 没有审计记录。 | 秘密的产生和使用都有审计记录。 |

Vault 有许多可移动的部分，因此可能需要一些时间来适应整体架构。花点时间回顾一下[架构](https://www.vaultproject.io/docs/internals/architecture.html)指南，注意以下后端:

| 后端 | 使用 | 例子 |
| --- | --- | --- |
| [存储](https://www.vaultproject.io/docs/configuration/storage) | 秘密储存的地方 | 咨询`*`，文件系统`*`，内存中，PostgreSQL，S3 |
| [秘密](https://www.vaultproject.io/docs/secrets) | 处理静态或动态机密 | AWS `*`，数据库，键/值`*`，RabbitMQ，SSH |
| [认证](https://www.vaultproject.io/docs/auth) | 处理身份验证和授权 | AWS，Azure，Google Cloud，GitHub，令牌`*`，用户名&密码 |
| [审计](https://www.vaultproject.io/docs/audit) | 记录所有请求和响应 | 文件`*`，系统日志，套接字 |

本教程中使用的`*`

有了这些，我们开始使用 Vault。

## 文件系统后端

为了快速启动并运行，我们将使用[文件系统](https://www.vaultproject.io/docs/configuration/storage/filesystem.html)后端来存储静态秘密。

> 文件系统后端应仅用于本地开发或单服务器 Vault 部署，因为它不支持高可用性。

创建新的项目目录:

```
`$ mkdir vault-consul-docker && cd vault-consul-docker` 
```

然后添加以下文件夹:

```
`└── vault
    ├── config
    ├── data
    ├── logs
    └── policies` 
```

将 *Dockerfile* 添加到“vault”目录:

```
`# base image
FROM  alpine:3.14

# set vault version
ENV  VAULT_VERSION 1.8.2

# create a new directory
RUN  mkdir /vault

# download dependencies
RUN  apk --no-cache add \
      bash \
      ca-certificates \
      wget

# download and set up vault
RUN  wget --quiet --output-document=/tmp/vault.zip https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip && \
    unzip /tmp/vault.zip -d /vault && \
    rm -f /tmp/vault.zip && \
    chmod +x /vault

# update PATH
ENV  PATH="PATH=$PATH:$PWD/vault"

# add the config file
COPY  ./config/vault-config.json /vault/config/vault-config.json

# expose port 8200
EXPOSE  8200

# run vault
ENTRYPOINT  ["vault"]` 
```

接下来，将一个 *docker-compose.yml* 文件添加到项目根:

```
`version:  '3.8' services: vault: build: context:  ./vault dockerfile:  Dockerfile ports: -  8200:8200 volumes: -  ./vault/config:/vault/config -  ./vault/policies:/vault/policies -  ./vault/data:/vault/data -  ./vault/logs:/vault/logs environment: -  VAULT_ADDR=http://127.0.0.1:8200 -  VAULT_API_ADDR=http://127.0.0.1:8200 command:  server -config=/vault/config/vault-config.json cap_add: -  IPC_LOCK` 
```

将名为 *vault-config.json* 的配置文件添加到“vault/config”中:

```
`{ "backend":  { "file":  { "path":  "vault/data" } }, "listener":  { "tcp":{ "address":  "0.0.0.0:8200", "tls_disable":  1 } }, "ui":  true }` 
```

在这里，我们将 Vault 配置为使用文件系统后端，为 Vault 定义了[监听器](https://www.vaultproject.io/docs/configuration/listener/tcp.html),[禁用了 TLS](https://www.vaultproject.io/docs/configuration/listener/tcp.html#tls_disable) ，并启用了 [Vault UI](https://www.vaultproject.io/docs/configuration/ui) 。查看[文档](https://www.vaultproject.io/docs/configuration)了解更多关于配置保险库的信息。

现在我们可以构建图像并旋转容器:

```
`$ docker-compose up -d --build` 
```

调出 Docker 日志，确保构建中没有错误:

您应该会看到类似如下的内容:

```
`Attaching to vault-consul-docker_vault_1
vault_1  | ==> Vault server configuration:
vault_1  |
vault_1  |              Api Address: http://127.0.0.1:8200
vault_1  | 2021-09-08T14:48:35.014Z [INFO]  proxy environment: http_proxy="" https_proxy="" no_proxy=""
vault_1  |                      Cgo: disabled
vault_1  |          Cluster Address: https://127.0.0.1:8201
vault_1  |               Go Version: go1.16.7
vault_1  |               Listener 1: tcp (addr: "0.0.0.0:8200", cluster address: "0.0.0.0:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
vault_1  |                Log Level: info
vault_1  |                    Mlock: supported: true, enabled: true
vault_1  |            Recovery Mode: false
vault_1  |                  Storage: file
vault_1  |                  Version: Vault v1.8.2
vault_1  |              Version Sha: aca76f63357041a43b49f3e8c11d67358496959f
vault_1  |
vault_1  | ==> Vault server started! Log data will stream in below:
vault_1  |` 
```

## 初始化和解封

在运行的容器中启动 bash 会话:

```
`$ docker-compose exec vault bash` 
```

在 shell 中，初始化 Vault:

```
`bash-5.1# vault operator init` 
```

记下解封密钥和初始根令牌。每次重新密封或重新启动 Vault 服务器时，您都需要提供三个解封密钥。

> 为什么是三把钥匙？回顾[沙米尔的秘密分享](https://en.wikipedia.org/wiki/Shamir's_Secret_Sharing)。

现在，您可以使用以下三个密钥解封保险库:

```
`bash-5.1# vault operator unseal
Unseal Key (will be hidden):` 
```

运行这个命令两次以上，每次使用不同的密钥。一旦完成，确保`Sealed`是`false`:

```
`Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.8.2
Storage Type    file
Cluster Name    vault-cluster-8fcf9d05
Cluster ID      d86e0274-ad9c-d2c1-d6ec-baeab410797b
HA Enabled      false` 
```

使用根令牌，您现在可以进行身份验证:

```
`bash-5.1# vault login
Token (will be hidden):` 
```

您应该会看到类似如下的内容:

```
`Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.c0kYHWiOTqQvtR8JuSeTz6sZ
token_accessor       3FQJVxOY5C1brzlHHQSFaCdZ
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]` 
```

> 请记住，这使用了根策略。在生产中，您会希望设置具有不同访问级别的策略。我们很快就会看到如何做到这一点。

![vault init](img/de953946d93734de93c2b18b11654836.png)

保险库现已解封，可以使用了。

## 审计

在我们测试功能之前，让我们启用一个[审计设备](https://www.vaultproject.io/docs/audit):

```
`bash-5.1# vault audit enable file file_path=/vault/logs/audit.log

Success! Enabled the file audit device at: file/` 
```

现在，您应该能够在“vault/logs”中本地查看日志。要进行测试，请运行以下命令来查看所有启用的审核设备:

```
`bash-5.1# vault audit list

Path     Type    Description
----     ----    -----------
file/    file    n/a` 
```

请求和后续响应应记录在 *vault/logs/audit.log* 中。看一看。

## 秘密

跳马的秘密有两种:[静态](https://learn.hashicorp.com/vault/secrets-management/sm-static-secrets)和[动态](https://learn.hashicorp.com/vault/getting-started/dynamic-secrets)。

1.  静态机密(比如加密的 Redis 或 Memcached)有刷新间隔，但除非明确撤销，否则不会过期。它们预先用[键/值](https://www.vaultproject.io/docs/secrets/kv)后端(以前的“通用”后端)定义，然后共享。

    ![secure secret storage](img/22403a708a73706da2b51efe5c19ff60.png)

2.  **动态**秘密按需生成。他们有强制租约，通常在短期内到期。因为它们在被访问之前是不存在的，所以暴露的机会更少——所以动态秘密更安全。Vault 附带了许多动态后端——即 [AWS](https://www.vaultproject.io/docs/secrets/aws) 、[数据库](https://www.vaultproject.io/docs/secrets/databases)、[谷歌云](https://www.vaultproject.io/docs/secrets/gcp)、[领事](https://www.vaultproject.io/docs/secrets/consul)和 [RabbitMQ](https://www.vaultproject.io/docs/secrets/rabbitmq) 。

> 查看[为什么我们需要动态秘密](https://www.hashicorp.com/blog/why-we-need-dynamic-secrets)的博客文章，了解更多关于使用动态秘密的优势的信息。

## 静态秘密

可以通过 [CLI](https://www.vaultproject.io/docs/commands) 、 [HTTP API](https://www.vaultproject.io/api-docs) 或 [UI](https://www.vaultproject.io/docs/configuration/ui) 管理 Vault。

### 硬币指示器 （coin-levelindicator 的缩写）命令行界面（Command Line Interface for batch scripting）

仍然在容器的 bash 会话中，我们可以创建、读取、更新和删除秘密。我们还将了解如何版本化和回滚机密。

使用以下命令启用机密:

```
`bash-5.1# vault secrets enable kv

Success! Enabled the kv secrets engine at: kv/` 
```

在`kv/foo`路径中创建一个密钥为`bar`值为`precious`的新秘密:

```
`bash-5.1# vault kv put kv/foo bar=precious

Success! Data written to: kv/foo` 
```

阅读:

```
`bash-5.1# vault kv get kv/foo

=== Data ===
Key    Value
---    -----
bar    precious` 
```

要使用特定键的不同版本，我们需要升级到[键/值](https://www.vaultproject.io/docs/secrets/kv)后端的 [v2](https://www.vaultproject.io/docs/secrets/kv/kv-v2.html) :

```
`bash-5.1# vault kv enable-versioning kv/

Success! Tuned the secrets engine at: kv/` 
```

通过将值更新为`copper`来添加版本 2:

```
`bash-5.1# vault kv put kv/foo bar=copper

Key              Value
---              -----
created_time     2021-09-08T18:23:14.4154928Z
deletion_time    n/a
destroyed        false
version          2` 
```

阅读版本 1:

```
`bash-5.1# vault kv get -version=1 kv/foo

====== Metadata ======
Key              Value
---              -----
created_time     2021-09-08T18:22:37.2548824Z
deletion_time    n/a
destroyed        false
version          1

=== Data ===
Key    Value
---    -----
bar    precious` 
```

阅读版本 2:

```
`bash-5.1# vault kv get -version=2 kv/foo

====== Metadata ======
Key              Value
---              -----
created_time     2021-09-08T18:23:14.4154928Z
deletion_time    n/a
destroyed        false
version          2

=== Data ===
Key    Value
---    -----
bar    copper` 
```

删除最新版本(如版本 2):

```
`bash-5.1# vault kv delete kv/foo

Success! Data deleted (if it existed) at: kv/foo` 
```

删除版本 1:

```
`bash-5.1# vault kv delete -versions=1 kv/foo

Success! Data deleted (if it existed) at: kv/foo` 
```

您也可以取消删除:

```
`bash-5.1# vault kv undelete -versions=1 kv/foo

Success! Data written to: kv/undelete/foo` 
```

删除类似于软删除。如果您想删除底层元数据，您必须使用 [destroy](https://www.vaultproject.io/api/secret/kv/kv-v2.html#destroy-secret-versions) 命令:

```
`bash-5.1# vault kv destroy -versions=1 kv/foo

Success! Data written to: kv/destroy/foo` 
```

查看 [v1](https://www.vaultproject.io/api/secret/kv/kv-v1.html) 和 [v2](https://www.vaultproject.io/docs/secrets/kv) 以查看所有可用的命令。

> 记下审计日志。上面的每个请求都被记录了！

### 应用程序接口

您还可以通过 [HTTP API](https://learn.hashicorp.com/vault/getting-started/apis) 与 Vault 进行交互。我们将针对 API 的 [v2](https://www.vaultproject.io/api/secret/kv/kv-v2.html) 提出请求。打开一个新的终端选项卡，然后将根令牌设置为环境变量:

```
`$ export VAULT_TOKEN=your_token_goes_here` 
```

创建一个名为`foo`的新秘密，其值为`world`:

```
`$ curl \
    -H "X-Vault-Token: $VAULT_TOKEN" \
    -H "Content-Type: application/json" \
    -X POST \
    -d '{ "data": { "foo": "world" } }' \
    http://127.0.0.1:8200/v1/kv/data/hello` 
```

阅读秘密:

```
`$ curl \
    -H "X-Vault-Token: $VAULT_TOKEN" \
    -X GET \
    http://127.0.0.1:8200/v1/kv/data/hello` 
```

JSON 响应应该包含一个`data`键，其值类似于:

```
`"data": {
  "data":{
    "foo": "world"
  },
  "metadata": {
    "created_time": "2021-09-08T18:30:32.5140484Z",
    "deletion_time": "",
    "destroyed": false,
    "version": 1
  }
}` 
```

![vault api](img/be308cc2fe26afe28ece8bf830e7b7bf.png)

尝试自己添加、删除和销毁新版本。

### 用户界面

[UI](https://www.vaultproject.io/docs/configuration/ui) 应该在[http://localhost:8200/UI/vault](http://localhost:8200/ui/vault)上运行。使用根令牌登录。然后，自己探索键/值后端:

![vault ui](img/e33b57dd888feb76e904fe8f29c256ec.png)

## 政策

到目前为止，我们一直使用[根策略](https://www.vaultproject.io/docs/concepts/policies.html#root-policy)与 API 交互。让我们设置一个只有读权限的策略。

将名为 *app-policy.json* 的新配置文件添加到“vault/policies”中:

```
`{ "path":  { "kv/data/app/*":  { "policy":  "read" } } }` 
```

[在 bash 会话中创建](https://www.vaultproject.io/docs/concepts/policies.html#creating-policies)一个新策略:

```
`bash-5.1# vault policy write app /vault/policies/app-policy.json

Success! Uploaded policy: app` 
```

然后，创建一个新令牌:

```
`bash-5.1# vault token create -policy=app

Key                  Value
---                  -----
token                s.ZOUMx3RIhVRhI4ijlZg8KXRQ
token_accessor       TT53xOxbIfGjI7l4392gjXcg
token_duration       768h
token_renewable      true
token_policies       ["app" "default"]
identity_policies    []
policies             ["app" "default"]` 
```

在另一个新的终端选项卡中(现在应该有三个了)，添加带有新令牌的`VAULT_TOKEN`环境变量:

```
`$ export VAULT_TOKEN=your_token_goes_here` 
```

试着读出我们之前设定的`foo`秘密:

```
`$ curl \
    -H "X-Vault-Token: $VAULT_TOKEN" \
    -X GET \
    http://127.0.0.1:8200/v1/kv/data/hello` 
```

您应该没有查看该机密的正确权限:

```
`{
  "errors":[
    "1 error occurred:\n\t* permission denied\n\n"
  ]
}` 
```

为什么我们连它都看不懂？跳回到 *vault-config.json* 中的策略配置。`kv/data/app/*`表示策略只能从`app`路径读取。

> 你可能已经注意到了，Vault 中的几乎所有东西都是基于路径的。

回到容器中的 bash 会话，向`app/test`路径添加一个新的秘密:

```
`bash-5.1# vault kv put kv/app/test ping=pong

Key              Value
---              -----
created_time     2021-09-08T18:40:35.2694047Z
deletion_time    n/a
destroyed        false
version          1` 
```

您应该能够使用与`app`策略相关联的令牌来查看秘密:

```
`$ curl \
    -H "X-Vault-Token: $VAULT_TOKEN" \
    -X GET \
    http://127.0.0.1:8200/v1/kv/data/app/test` 
```

也可以从用户界面管理策略:

![vault ui](img/77a90a80de7a53d542a588909330be4b.png)

## 加密即服务

在我们研究动态秘密之前，让我们快速回顾一下[传输](https://www.vaultproject.io/docs/secrets/transit)后端，它可以作为“加密即服务”用于:

*   加密和解密“传输中”的数据，而不将其存储在保险库中
*   轻松将加密集成到您的应用程序工作流程中

回到容器中的 bash 会话，启用传输:

```
`bash-5.1# vault secrets enable transit

Success! Enabled the transit secrets engine at: transit/` 
```

配置命名加密密钥:

```
`bash-5.1# vault write -f transit/keys/foo

Success! Data written to: transit/keys/foo` 
```

加密:

```
`bash-5.1# vault write transit/encrypt/foo plaintext=$(base64 <<< "my precious")

Key           Value
---           -----
ciphertext    vault:v1:cFnk5AQLE9Mg+mZ7Ej17vRmYT5aqheikdZQ1FC4vre5jAod0L/uHDA==` 
```

解密:

```
`bash-5.1# vault write transit/decrypt/foo ciphertext=vault:v1:cFnk5AQLE9Mg+mZ7Ej17vRmYT5aqheikdZQ1FC4vre5jAod0L/uHDA==

Key          Value
---          -----
plaintext    bXkgcHJlY2lvdXMK` 
```

解码:

```
`bash-5.1# base64 -d <<< "bXkgcHJlY2lvdXMK"

my precious` 
```

也在用户界面中进行测试:

![vault ui](img/1905bd4058852765dca971feb9cd275a.png)

## 动态秘密

如前所述，Vault 支持许多动态秘密后端，用于在需要时动态生成秘密。例如，使用 [AWS](https://www.vaultproject.io/docs/secrets/aws) 和 [Google Cloud](https://www.vaultproject.io/docs/secrets/gcp) 后端，您可以基于 IAM 策略创建访问凭证。与此同时，[数据库](https://www.vaultproject.io/docs/secrets/databases)后端基于配置的角色生成数据库凭证。

*动态秘密:*

*   按需生成
*   基于角色具有有限的访问权限
*   被租赁一段时间
*   可以撤销
*   附带一份审计记录

让我们看看如何使用 AWS 后端生成 AWS 凭证。

#### AWS 凭据

启用 AWS 机密后端:

```
`bash-5.1# vault secrets enable -path=aws aws

Success! Enabled the aws secrets engine at: aws/` 
```

认证:

```
`bash-5.1# vault write aws/config/root access_key=foo secret_key=bar

Success! Data written to: aws/config/root` 
```

> 确保分别用您的 AWS 访问密钥 id 和秘密密钥替换`foo`和`bar`。

创建角色:

```
`bash-5.1# vault write aws/roles/ec2-read credential_type=iam_user policy_document=-<<EOF
{
 "Version": "2012-10-17",
 "Statement": [
 {
 "Sid": "Stmt1426528957000",
 "Effect": "Allow",
 "Action": [
 "ec2:*"
 ],
 "Resource": [
 "*"
 ]
 }
 ]
}
EOF

Success! Data written to: aws/roles/ec2-read` 
```

这里，我们基于`AmazonEC2ReadOnlyAccess`创建了一个新角色，这是一个 AWS 管理的[策略](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)。顾名思义，它给予用户对 EC2 控制台的只读访问权；他们不能执行任何操作或创建新资源。您还可以使用内嵌策略根据您的个人需求创建自定义角色。我们很快就会看到一个例子。更多信息请参考 [AWS 秘密引擎](https://www.vaultproject.io/docs/secrets/aws)文档。

> **记住**:动态秘密只有在被请求时才会生成(例如，一个 web 应用程序请求访问 S3)。在此之前，商店里没有这些东西。

创建一组新的凭据:

```
`bash-5.1# vault read aws/creds/ec2-read

Key                Value
---                -----
lease_id           aws/creds/ec2-read/9KdO6J7KVBiSwOPEvwrqqALG
lease_duration     768h
lease_renewable    true
access_key         AKIAZ4DZAKZKEULSDW5A
secret_key         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
security_token     <nil>` 
```

现在，您应该能够在 AWS 上的 [IAM 控制台](https://console.aws.amazon.com/iam)的“用户”部分看到该用户:

![aws iam](img/29349953e3bc194c7c0dfd9fd9cc271f.png)

## 租赁和撤销

在这一节中，我们将快速看一下如何定义一个自定义的租期，并在租期结束前撤销一个秘密。

创建新的 AWS 角色:

```
`bash-5.1# vault write aws/roles/foo credential_type=iam_user policy_document=-<<EOF
{
 "Version": "2012-10-17",
 "Statement": [
 {
 "Sid": "Stmt1426528957000",
 "Effect": "Allow",
 "Action": [
 "ec2:*"
 ],
 "Resource": [
 "*"
 ]
 }
 ]
}
EOF

Success! Data written to: aws/roles/foo` 
```

创建新的 AWS 凭证时，请注意`lease_duration`:

```
`bash-5.1# vault read aws/creds/foo

Key                Value
---                -----
lease_id           aws/creds/foo/F0oBbnBIHEoz0ywVVtbuJB7r
lease_duration     768h
lease_renewable    true
access_key         AKIAZ4DZAKZKLJKB7CPX
secret_key         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
security_token     <nil>` 
```

如果您只希望所有 AWS IAM 动态机密的租期为 30 分钟，会怎么样？

```
`bash-5.1# vault write aws/config/lease lease=1800s lease_max=1800s` 
```

在本例中，由于`lease_max`与`lease`相同，您将无法续订令牌。如果你将`lease_max`设置为`3600s`，你就可以续租一次。有关更多信息，请查看[令牌和租赁](https://learn.hashicorp.com/vault/identity-access-management/tokens)指南。

创建新凭据:

```
`bash-5.1# vault read aws/creds/foo

Key                Value
---                -----
lease_id           aws/creds/foo/xQlJpKDS1ljE9Awz0aywXgbB
lease_duration     30m
lease_renewable    true
access_key         AKIAZ4DZAKZKJPL5OM5W
secret_key         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
security_token     <nil>` 
```

想要快速吊销此凭据吗？抓住`lease_id`然后跑:

```
`bash-5.1# vault lease revoke aws/creds/foo/xQlJpKDS1ljE9Awz0aywXgbB` 
```

想要撤销所有 AWS 信用？

```
`bash-5.1# vault lease revoke -prefix aws/` 
```

有关这些概念的更多信息，请参考[租赁、续订和撤销](https://www.vaultproject.io/docs/concepts/lease)指南。

## 领事后端

到目前为止，我们一直在使用[文件系统](https://www.vaultproject.io/docs/configuration/storage/filesystem.html)后端。这将无法扩展到单台服务器之外，因此无法利用 Vault 的高可用性。幸运的是，有许多其他的[存储](https://www.vaultproject.io/docs/configuration/storage)后端，像[领事](https://www.vaultproject.io/docs/configuration/storage/consul.html)后端，是为分布式系统设计的。

要设置[consult](https://www.consul.io/)，首先要更新 *docker-compose.yml* 文件:

```
`version:  '3.8' services: vault: build: context:  ./vault dockerfile:  Dockerfile ports: -  8200:8200 volumes: -  ./vault/config:/vault/config -  ./vault/policies:/vault/policies -  ./vault/data:/vault/data -  ./vault/logs:/vault/logs environment: -  VAULT_ADDR=http://127.0.0.1:8200 -  VAULT_API_ADDR=http://127.0.0.1:8200 command:  server -config=/vault/config/vault-config.json cap_add: -  IPC_LOCK depends_on: -  consul consul: build: context:  ./consul dockerfile:  Dockerfile ports: -  8500:8500 command:  agent -server -bind 0.0.0.0 -client 0.0.0.0 -bootstrap-expect 1 -config-file=/consul/config/config.json volumes: -  ./consul/config/consul-config.json:/consul/config/config.json -  ./consul/data:/consul/data` 
```

在项目根目录中添加一个名为“consul”的新目录，然后在这个新创建的目录中添加一个新的 *Dockerfile* :

```
`# base image
FROM  alpine:3.14

# set consul version
ENV  CONSUL_VERSION 1.10.2

# create a new directory
RUN  mkdir /consul

# download dependencies
RUN  apk --no-cache add \
      bash \
      ca-certificates \
      wget

# download and set up consul
RUN  wget --quiet --output-document=/tmp/consul.zip https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip && \
    unzip /tmp/consul.zip -d /consul && \
    rm -f /tmp/consul.zip && \
    chmod +x /consul/consul

# update PATH
ENV  PATH="PATH=$PATH:$PWD/consul"

# add the config file
COPY  ./config/consul-config.json /consul/config/config.json

# expose ports
EXPOSE  8300 8400 8500 8600

# run consul
ENTRYPOINT  ["consul"]` 
```

接下来，在“consul”目录中添加两个新目录:“config”和“data”。然后，在“config”中，添加一个名为 *consul-config.json* 的配置文件:

```
`{ "datacenter":  "localhost", "data_dir":  "/consul/data", "log_level":  "DEBUG", "server":  true, "ui":  true, "ports":  { "dns":  53 } }` 
```

> 请务必查看咨询文档中的[配置](https://www.consul.io/docs/agent/options.html)选项，了解有关上述选项的更多信息。

“领事”目录现在应该是这样的:

```
`├── Dockerfile
├── config
│   └── consul-config.json
└── data` 
```

退出 bash 会话。关闭容器，然后更新 Vault 配置文件:

```
`{ "backend":  { "consul":  { "address":  "consul:8500", "path":  "vault/" } }, "listener":  { "tcp":{ "address":  "0.0.0.0:8200", "tls_disable":  1 } }, "ui":  true }` 
```

所以，现在我们使用的是[领事](https://www.vaultproject.io/docs/configuration/storage/consul.html)后端，而不是文件系统。我们使用服务名`consul`作为地址的一部分。`path`键定义了 Consul 的键/值存储中存储 Vault 数据的路径。

清除“vault/data”目录中的所有文件和文件夹，以删除文件系统后端。构建新映像并旋转容器:

```
`$ docker-compose down
$ docker-compose up -d --build` 
```

在浏览器中导航到 [http://localhost:8500/ui](http://localhost:8500/ui) ，确保一切正常:

![consul ui](img/b217468defbcd79975603bc54ca669ef.png)

从 CLI 或 UI 对此进行测试。

### 硬币指示器 （coin-levelindicator 的缩写）命令行界面（Command Line Interface for batch scripting）

在 Vault 容器中创建新的 bash 会话:

```
`$ docker-compose exec vault bash` 
```

然后，运行:

```
`# Init
bash-5.1# vault operator init

# Unseal
bash-5.1# vault operator unseal

# Authenticate
bash-5.1# vault login

# Enable secrets
bash-5.1# vault secrets enable kv

# Add a new static secret
bash-5.1# vault kv put kv/foo bar=precious

# Read it back
bash-5.1# vault kv get kv/foo` 
```

### 用户界面

![vault consul](img/563b10b4d53832df082d1b52026aea15.png)

> 请注意“vault/data”中没有文件或文件夹。你认为这是为什么？

想再添加一个 Consul 服务器吗？向 *docker-compose.yml* 添加新服务:

```
`consul-worker: build: context:  ./consul dockerfile:  Dockerfile command:  agent -server -join consul -config-file=/consul/config/config.json volumes: -  ./consul/config/consul-config.json:/consul/config/config.json depends_on: -  consul` 
```

这里，我们使用了 [join](https://www.consul.io/commands/join) 命令将这个代理连接到一个现有的集群。注意，我们只需要引用服务名:`consul`。

然后:

1.  退出 bash 会话(如有必要)
2.  把集装箱拿下来
3.  清空“领事/数据”中的数据目录(为什么？)
4.  旋转容器并测试

![consul ui](img/2ba57d3c3d75cd35608a1750e3c0ccdb.png)

## 结论

在本教程中，我们讨论了如何在 Docker 容器中设置和运行 Vault 和 Consul。现在，您应该对如何与 Vault 交互以及如何执行基本操作有了清晰的了解。

从[金库-领事-码头](https://github.com/testdrivenio/vault-consul-docker)仓库中获取最终代码。也可以看看[的展示](https://mherman.org/presentations/vault/)。

想要更多吗？看看下面的帖子吧:

1.  [部署金库和咨询](/blog/deploying-vault-and-consul/)
2.  [使用金库和烧瓶的动态秘密生成](/blog/dynamic-secret-generation-with-vault-and-flask/)