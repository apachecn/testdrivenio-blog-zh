# 基于 Python 的应用程序的 Heroku 替代方案

> 原文：<https://testdriven.io/blog/heroku-alternatives/>

Heroku 改变了开发人员构建和部署软件的方式，使得构建、部署和扩展应用程序变得更加容易和快速。他们为如何管理云服务制定了各种标准和方法——即[十二因素应用](https://12factor.net/)——这些标准和方法在今天仍然与基于微服务和云原生应用高度相关。不幸的是，从 2022 年 11 月 28 日开始，Heroku 将停止其免费层。这意味着您将不再能够利用免费的 dynos、Postgres 数据库和 Redis 实例。

> 关于 Heroku 停止其免费产品层的更多信息，请查看 Heroku 的下一章和[对 Heroku 免费资源的反对](https://devcenter.heroku.com/changelog-items/2461)。

在本文中，您将了解最佳的 Heroku 替代方案(及其优缺点)。

## 什么是 Heroku？

Heroku 成立于 2007 年，是一个为网络应用提供托管服务的云计算平台。它们提供了抽象的环境，您无需管理底层基础设施，从而轻松管理、部署和扩展 web 应用程序。只需几次点击，您就可以启动并运行您的应用程序，准备接收流量。

在 Heroku 出现之前，运行一个 web 应用程序的过程是相当具有挑战性的，主要是留给经验丰富的系统操作专业人员而不是开发人员。Heroku 提供了一个自以为是的层，抽象出了 web 服务器所需的大部分配置。大多数 web 应用程序可以(并且仍然可以)利用这样的环境，因此较小的公司和团队可以专注于应用程序开发，而不是配置 web 服务器、安装 Linux 包、设置负载平衡器以及在传统服务器上进行基础设施管理的所有其他事情。

## Heroku 的利与弊

尽管 Heroku 很受欢迎，但多年来它还是受到了很多批评。

> 如果你已经熟悉 Heroku，可以跳过这一部分。

### 赞成的意见

#### 易用性

Heroku 可以说是最用户友好的 PaaS 平台。您只需定义运行 web 应用程序所需的命令，Heroku 就会为您完成剩下的工作，而无需花费数天时间来设置和配置 web 服务器和底层基础设施。您可以在几分钟内启动并运行您的应用程序！

此外，Heroku 利用 [git](https://devcenter.heroku.com/articles/git) 来版本化和部署应用程序，这使得部署和回滚变得容易。

最后，与大多数 PaaS 平台不同，Heroku 提供了优秀的错误日志，使得调试相对容易。

#### 流行

在成立的头五年，Heroku 几乎没有竞争对手。他们的用户/开发者体验远远领先于其他人，公司需要一段时间来适应。这一点，再加上它们巨大的免费层，意味着大多数面向开发人员的教程都使用 Heroku 作为他们的部署平台。即使到今天，绝大多数 web 开发教程、书籍和课程仍然利用 Heroku 进行部署。

Heroku 还对一些最流行的语言和运行时(通过[build pack](https://devcenter.heroku.com/articles/buildpacks))提供一流的支持，如 Python、Ruby、Node.js、PHP、Go、Java、Scala 和 Clojure。虽然这些都是官方支持的语言，但您仍然可以将自己的语言或自定义运行时带到 Heroku 平台上。

#### 集成和附加组件

经常被忽视的是，Heroku 提供了数百个附加工具和服务——从数据存储和缓存到监控和分析，再到数据和视频处理。只需点击一个按钮，您就可以通过提供第三方云服务来扩展您的应用，而无需手动安装或配置。

#### 缩放比例

Heroku 允许开发者轻松地纵向和横向扩展他们的应用。可以通过 Heroku 的[仪表盘](https://dashboard.heroku.com/)或 [CLI](https://devcenter.heroku.com/articles/heroku-cli) 实现缩放。此外，如果你运行更高性能的 dyno，你可以利用免费的自动缩放功能，这将根据当前的流量增加 web dynos 的数量。

### 骗局

#### 费用

与市场上的其他 PaaS 相比，Heroku 相当昂贵。虽然他们的起始计划是每个 dyno 每月 7 美元，但随着你的应用程序的扩展，你很快就必须升级到更好的 dyno，这需要花费相当多的钱。由于更高性能的 dynos 的价格，Heroku 可能不适合大型高流量应用程序。

Heroku 与 AWS EC2 相比大约贵五倍。

> 但是请记住，Heroku 是一个 PaaS，它为您做了很多繁重的工作，而 EC2 只是一个 Linux 实例，您必须自己管理它。

#### 缺乏控制和灵活性

Heroku 没有提供足够的控制，也缺乏透明度。通过使用他们的服务，你将高度依赖他们的技术和设计决策。它们的一些限制阻碍了可伸缩性——例如，应用程序只能监听单个端口，函数的最大源代码大小为 500 MB，并且没有办法微调数据库。Heroku 也高度依赖 AWS，这意味着如果一个 AWS 区域宕机，您的服务(托管在该区域)也会宕机。

类似地，Heroku 实际上是为普通的 RESTful APIs 设计的。如果你的应用包括繁重的计算，或者你需要调整基础设施来满足你的特定需求，Heroku 可能不是一个很好的选择。

#### 缺少地区

Heroku 提供两种类型的运行时:

1.  [公共运行时](https://devcenter.heroku.com/articles/dyno-runtime#common-runtime) -面向非企业用户
2.  [私有空间运行时](https://devcenter.heroku.com/articles/dyno-runtime#private-spaces-runtime) -面向企业用户

公共运行时只支持两个地区，美国和欧盟，而私有空间运行时支持 [6 个地区](https://devcenter.heroku.com/articles/private-spaces#regions)。

这意味着，如果你不是企业用户，你只能在美国(弗吉尼亚州)或欧盟地区(爱尔兰都柏林)托管你的应用程序。

```
`$ heroku regions

ID         Location                 Runtime
─────────  ───────────────────────  ──────────────
eu         Europe                   Common Runtime
us         United States            Common Runtime
dublin     Dublin, Ireland          Private Spaces
frankfurt  Frankfurt, Germany       Private Spaces
oregon     Oregon, United States    Private Spaces
sydney     Sydney, Australia        Private Spaces
tokyo      Tokyo, Japan             Private Spaces
virginia   Virginia, United States  Private Spaces` 
```

#### 缺乏新功能

当今世界，发展趋势变化比以往任何时候都快。这迫使托管服务追随潮流，吸引寻找尖端技术的团队。Heroku 的许多竞争对手，我们很快就会谈到，正在推进和增加新功能，如无服务器、边缘计算等。另一方面，Heroku 认为稳定性比特性开发更重要。这并不意味着他们没有增加新功能；他们只是添加新功能的速度比一些竞争对手慢得多。

> 如果你想知道 Heroku 接下来会发生什么，看看他们的路线图吧。

#### 锁住

一旦你在 Heroku 上运行生产代码，就很难迁移到不同的主机提供商。

> 请记住，如果您不再使用 PaaS，您将不得不处理 Heroku 自己处理的所有事情，所以请准备好雇用一两个 SysAdmin 或 DevOps。

## Heroku 的核心特征

在这一节中，我们将看看 Heroku 的核心特性，这样你就可以了解在寻找替代方案时应该寻找什么。

> 同样，如果你已经熟悉 Heroku 的特性，可以跳过这一部分。

| 特征 | 描述 |
| --- | --- |
| [Heroku 运行时](https://www.heroku.com/platform/runtime) | Heroku 运行时负责提供和编排 dyno，管理和监控 dyno 的生命周期，提供适当的网络配置、HTTP 路由、日志聚合等等。 |
| [CI/CD 系统](https://www.heroku.com/continuous-integration) | 易于使用的 CI/CD，负责构建、测试、部署、增量应用程序更新等。 |
| [基于 git 的部署](https://devcenter.heroku.com/articles/git) | 使用 git 管理应用部署。 |
| [数据持久性](https://www.heroku.com/managed-data-services) | 完全托管的数据服务，如 [Postgres](https://www.heroku.com/postgres) 、 [Redis](https://devcenter.heroku.com/articles/heroku-redis) 和 [Apache Kafka](https://www.heroku.com/kafka) 。 |
| [缩放功能](https://www.heroku.com/dynos/scaling) | 易于使用的工具，使开发人员能够按需进行水平和垂直扩展。 |
| [日志记录和应用指标](https://devcenter.heroku.com/articles/logging) | 日志记录、监控和应用程序指标。 |
| [协作功能](https://devcenter.heroku.com/categories/collaboration) | 与他人轻松协作。协作者可以对您的应用程序进行部署、缩放和访问数据等操作。 |
| [附加组件](https://elements.heroku.com/addons) | 数百种附加工具和服务–从数据存储和缓存到监控和分析，再到数据和视频处理，无所不包。 |

在寻找替代方案时，您应该优先考虑这些功能。你不可能找到 Heroku 的 1:1 替代品，所以一定要确定哪些功能是“必须拥有”而不是“最好拥有”。

例如:

**必备物品**

1.  实心用户界面
2.  构建包
3.  基于 git 的部署
4.  经过战斗考验的
5.  简单缩放

**拥有美好事物**

1.  应用和基础设施监控
2.  使用 AWS
3.  自由层
4.  Add-ons
5.  CI/CD 系统

## Heroku 替代方案

最后，在这一节中，我们将看看最好的 Heroku 替代方案以及它们的优缺点。

### 数字海洋应用平台

[App Platform](https://www.digitalocean.com/products/app-platform) 是 DigitalOcean 用于将应用部署到云的完全托管解决方案。它集成了 CI/CD，可以很好地与 GitHub 和 GitLab 兼容。它原生支持流行的语言和框架，如 Python、Node.js、Django、go 和 PHP。或者，它允许你通过 Docker 部署应用程序。

其他重要特性:

*   水平和垂直缩放
*   内置警报、监控和洞察力
*   零停机部署和回滚

该平台的用户界面/UX 简单明了，给人一种类似 Heroku 的感觉。

> DigitalOcean 应用程序平台起价为 5 美元/月，1 个 CPU 和 512 MB 内存。要了解更多关于他们定价的信息，请查看官方定价页面。

#### 赞成的意见

*   使用方便
*   最便宜的 PaaS 之一
*   免费计划，允许您托管多达 3 个静态网站
*   体面的区域支持(8 个区域)
*   托管应用上的 SSL 保护
*   DDoS 缓解

#### 骗局

*   相对较新的 PaaS(成立于 2020 年)
*   构建通常需要 15 分钟
*   不支持重复作业(如 cron)
*   缺少文件

> 想学习如何将 Django 应用程序部署到 DigitalOcean 的应用程序平台？在 DigitalOcean 的 App 平台上查看[运行 Django。](/blog/django-digitalocean-app-platform/)

### 提供；给予

2019 年推出的 [Render](https://render.com/) 是 Heroku 的绝佳替代品。它允许你完全免费地托管静态站点、web 服务、PostgreSQL 数据库和 Redis 实例。它极其简单的用户界面/UX 和强大的 git 集成让你可以在几分钟内运行一个应用程序。它具有对 Python、Node.js、Ruby、Elixir、Go 和 Rust 的原生支持。如果这些都不适合您，Render 还可以通过 docker 文件进行部署。

Render 的免费自动缩放功能将确保您的应用程序总是以合适的成本获得必要的资源。此外，Render 上托管的所有内容也可以获得免费的 TLS 证书。

> 参考他们的官方文件以获得更多关于他们免费计划的信息。

#### 赞成的意见

*   非常适合初学者
*   轻松设置和部署应用
*   自由层
*   与 Heroku 相比，预算适中(便宜约 50%)
*   基于实时 CPU 和内存使用情况的自动扩展
*   出色的客户支持

#### 骗局

*   相对较新的 PaaS(成立于 2019 年)
*   有限的地区支持(仅限俄勒冈州、法兰克福、俄亥俄州和新加坡)
*   免费层应用程序需要很长时间才能启动和运行
*   没有构建包(看看这个问题)
*   缺乏附加产品生态系统

> 想了解如何部署应用程序进行渲染？查看我们的教程:
> 
> 1.  [部署 Django 应用程序进行渲染](/blog/django-render/)
> 2.  [部署 Flask App 渲染](/blog/flask-render-deployment/)

### Fly.io

Fly.io 是一个流行的、灵活的 PaaS。他们不是转售 [AWS](https://aws.amazon.com/) 或 [GCP](https://cloud.google.com/) 服务，而是将你的应用程序托管在运行于世界各地[的物理专用服务器之上](https://fly.io/docs/reference/regions/)。正因为如此，他们能够提供比其他 PaaS 更便宜的主机服务，比如 Heroku。他们的主要关注点是尽可能靠近他们的客户部署应用程序(你可以在 22 个地区中选择)。Fly.io 支持三种构建器:Dockerfile、 [buildpacks](https://buildpacks.io/) ，或者预构建的 Docker 映像。

他们还提供[缩放和自动缩放功能](https://fly.io/docs/reference/scaling/)。

与其他 PaaS 相比，Fly.io 采用不同的方法来管理您的资源。它没有花哨的管理仪表板；相反，所有的工作都是通过他们名为 [flyctl](https://fly.io/docs/hands-on/install-flyctl/) 的 CLI 来完成的。

他们的[免费计划](https://community.fly.io/t/i-dont-understand-free-tier/5145/2)包括:

*   多达 3 个共享 cpu-1x 256 MB 虚拟机
*   3GB 永久卷存储(总计)
*   160GB 出站数据传输

这应该足够运行一些小应用程序来测试他们的平台了。

#### 赞成的意见

*   小型项目的免费计划
*   巨大的区域支持(在编写本报告时有 22 个区域)
*   出色的文档和完整的 API 文档
*   轻松实现水平和垂直缩放

#### 骗局

*   只能通过 CLI 管理(可能不适合初学者)
*   没有现成的 GitHub 或 GitLab 集成
*   每个地区的定价不同

> 想学习如何在 Fly.io 上部署 Django 应用程序吗？查看[部署 Django 应用程序来飞行。](/blog/django-fly/)

### 谷歌应用引擎

Google App Engine (GAE)是一个完全托管的无服务器平台，用于大规模开发和托管网络应用。它具有强大的内置自动扩展功能，可以根据需求自动分配更多/更少的资源。GAE 原生支持用 Python、Node.js、Java、Ruby、C#、Go 和 PHP 编写的应用程序。或者，它通过[定制运行时](https://cloud.google.com/appengine/docs/flexible/custom-runtimes)或 Dockerfiles 提供对其他语言的支持。

它具有强大的应用程序诊断功能，您可以结合云[监控](https://cloud.google.com/monitoring)和[日志](https://cloud.google.com/logging)来监控您的应用程序的健康和性能。

谷歌为新客户提供 300 美元的免费积分，可以为小应用服务几年。

#### 赞成的意见

*   300 美元免费信贷
*   稳定且经过测试，成立于 2008 年
*   强大的应用程序诊断功能(可与其他 GCP/谷歌服务结合使用)
*   它可以扩展到零，这意味着如果没有人使用你的服务，你不用支付任何费用
*   强大的客户支持

#### 骗局

*   相当昂贵
*   如果你不熟悉 GCP，学习曲线会很陡
*   由于谷歌的专有软件，供应商被锁定
*   他们的定价可能更直截了当

> 想学习如何在 Google App Engine 上部署 Django 应用程序吗？查看[将 Django 应用部署到 Google 应用引擎](/blog/django-gae/)。

### Platform.sh

[Platform.sh](https://platform.sh/) 是一个专门为持续部署而构建的平台即服务。它允许您在云上托管 web 应用程序，同时使您的开发和测试工作流更加高效。它与 GitHub 直接集成，允许开发人员立即从 GitHub 库进行部署。它支持现代开发语言，如 Python、Java、PHP 和 Go，以及许多不同的框架。

Platform.sh 不提供免费计划。他们的开发者计划(不适合生产)起价 10 美元/月。他们的生产就绪计划起价为每月 50 美元。

#### 赞成的意见

*   出色的 CI/CD 以及与 GitHub 的集成
*   您的 GitHub 分支(开发/阶段/生产)反映在 Platform.sh 上
*   通过自动缩放轻松扩展
*   良好的文档
*   出色的客户支持

#### 骗局

*   没有空闲层
*   随着网站的发展，费用会越来越高
*   可能不适合小型企业

### AWS 弹性豆茎

AWS Elastic Beanstalk (EB)是一个易于使用的服务，用于部署和扩展 web 应用程序。它连接多个 AWS 服务，例如计算实例( [EC2](https://aws.amazon.com/ec2/) )、数据库( [RDS](https://aws.amazon.com/rds/) )、负载平衡器([应用负载平衡器](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html))和文件存储系统( [S3](https://aws.amazon.com/s3/) )，等等。EB 可以让你快速部署用 Python、Go、Java、.Net、Node.js、PHP 和 Ruby。它还支持 Docker。

Elastic Beanstalk 通过抽象出底层架构，使应用部署变得更加容易，同时仍然允许实例和数据库的低级配置。它与 git 集成得很好，并允许您进行增量部署。它还支持负载平衡和自动伸缩。

弹性豆茎的伟大之处在于它不需要额外的费用。您只需为应用程序消耗的资源(EC2 实例、RDS 等)付费。).

#### 赞成的意见

*   预算友好
*   如果你已经熟悉 AWS，那就太好了
*   高度可定制，提供高级别的控制
*   自动扩展和多个可用性区域，最大限度地提高应用的可靠性
*   出色的支持

#### 骗局

*   不适合小型项目
*   与其他 PaaS 提供商相比，设置和运营相对困难
*   没有部署失败通知
*   复杂的文档

> 想了解如何将应用程序部署到 Elastic Beanstalk 吗？查看我们的教程:

### 微软 Azure 应用服务

[Azure App Service](https://azure.microsoft.com/services/app-service/) 允许您快速轻松地为任何平台或设备创建企业级 web 和移动应用，并将其部署在可扩展且可靠的云基础设施上。它本身支持 Python。网，。NET Core，Node.js，Java，PHP，容器。他们有内置的 CI/CD 和零停机部署。

其他重要特性:

*   用于跟踪和故障排除的日志收集和失败请求跟踪
*   使用 [Azure 活动目录](https://azure.microsoft.com/services/active-directory/)进行身份验证
*   监控和警报

如果你是新客户，你可以获得 200 美元的免费积分来测试 Azure。

#### 赞成的意见

*   与 Visual Studio 很好地集成
*   [Azure Autoscale](https://azure.microsoft.com/products/virtual-machines/autoscale/) 可以帮助您优化成本
*   内置 SSL/TLS 证书
*   通过 [Azure Monitor](https://azure.microsoft.com/products/monitor/) 轻松调试和分析
*   稳定，99.95%的正常运行时间

#### 骗局

*   昂贵的
*   不是最直观的 PaaS
*   如果你不熟悉 Azure，学习曲线会很陡
*   复杂的文档

> 想学习如何在 Azure App Service 上部署 Django 应用？查看[将 Django 应用部署到 Azure 应用服务](/blog/django-azure-app-service/)。

### 数字海洋水滴上的 Dokku

Dokku 声称是你见过的最小的 PaaS 实现。它允许您构建和管理应用程序从构建到扩展的生命周期。它基本上是一个迷你的 Heroku，你可以在你的 Linux 机器上自己运行。Dokku 由 Docker 提供支持，并与 git 很好地集成。

> Dokku 提供了一个名为 Dokku PRO 的高级计划，该计划具有用户友好的界面和其他功能。你可以在他们的[官网](https://pro.dokku.com/)了解更多。

Dokku 的最低系统要求是 1 GB 内存。这意味着你可以以每月 6 美元的价格在 DigitalOcean Droplet 上运行它。

#### 赞成的意见

*   完全免费和开源
*   易于部署的应用
*   丰富的命令行界面
*   [各种插件](https://dokku.com/docs/community/plugins/)

#### 骗局

*   必须是自托管的
*   需要初始配置
*   文件可以改进
*   缩放并不容易

> 想学习如何在 Dokku 上部署 Django 应用程序吗？查看[在数字海洋水滴](/blog/django-dokku/)上将 Django 应用程序部署到 Dokku。

### 皮顿 Anywhere

[PythonAnywhere](https://www.pythonanywhere.com/) 是基于 Python 编程语言的在线集成开发环境(IDE)和 web 托管服务(PaaS)。它为 [Django](https://www.djangoproject.com/) 、 [web2py](http://www.web2py.com/) 、[烧瓶](https://flask.palletsprojects.com/en/2.2.x/)和[瓶子](https://bottlepy.org/docs/dev/)提供了开箱即用的部署选项。与列表中的其他 PaaS 相比，PythonAnywhere 的行为更像一个传统的 web 服务器。您可以访问它的文件系统，并可以通过 SSH 进入控制台来查看日志等等。

它提供了一个[免费计划](https://blog.pythonanywhere.com/206/)，这对初学者或者只是想测试不同 Python 框架的人来说是很棒的。免费计划允许您在`your_username.pythonanywhere.com`托管一个 web 应用。您还可以使用免费计划来启动 MySQL 实例。

> 其他相对便宜的付费计划可以在[他们的定价页面](https://www.pythonanywhere.com/pricing/)上看到。

#### 赞成的意见

*   一个小项目的免费托管
*   好用，基本没有学习曲线
*   为 Django、web2py、烧瓶和瓶子预先配置
*   一键免费 SSL
*   强大的客户支持

#### 骗局

*   不支持 CI/CD
*   仅支持 Python 应用程序
*   没有 ASGI 支持
*   无自动缩放

### 发动机场

[Engine Yard](https://www.engineyard.com/) 是一个 PaaS 解决方案，允许开发人员在云中规划、构建、部署和管理应用。Engine Yard 还为部署、管理 AWS、支持数据库和微服务容器开发提供服务。它主要关注 Ruby on Rails，但也支持其他语言，如 Python、PHP 和 Node.js。

Engine Yard 通过自动对托管环境进行堆栈更新和安全修补，简化了云上的应用管理。还可以通过应用指标来扩展应用的资源。

#### 赞成的意见

*   可以部署到任何 AWS 区域
*   快速简单的部署
*   自动化数据库管理
*   旨在扩展
*   良好的客户支持

#### 骗局

*   昂贵的
*   免费试用仅 14 天
*   Python 不是主要焦点，因为他们关注的是 Ruby on Rails 应用程序

### 维克塞尔

[Vercel](https://vercel.com/) 是一个静态站点和无服务器功能的云平台。它主要用于前端项目，但也支持 Python、Node.js、Ruby、Go 和 Docker。Vercel 使开发人员能够托管即时部署、自动扩展、几乎不需要监管的网站和 web 服务——所有这些都不需要配置。它也有一个漂亮而直观的用户界面。

Vercel 提供[免费计划](https://vercel.com/guides/does-vercel-offer-free-trial)，包括:

*   100 GB 带宽
*   内置 CI/CD
*   自动 HTTPS/SSL
*   每次 git 推送的预览

#### 赞成的意见

#### 骗局

*   不支持许多框架
*   Python、Go、Ruby 只能作为无服务器函数使用
*   不提供太多的控制

### Netlify

Netlify 是一个面向网络开发者和企业的基于云的开发平台。它允许开发者托管静态站点和无服务器功能。它支持 Python、Node.js、Go、PHP、Ruby、Rust 和 Swift。它无疑是前端项目使用最多的托管平台之一。

Netlify 有一个直观的用户界面，非常容易使用，因为它不需要任何配置。

其免费计划包括:

*   100 GB 带宽
*   每月 300 分钟构建时间
*   现场预览
*   即时回滚到任何版本
*   静态资产和动态无服务器功能的部署

#### 赞成的意见

#### 骗局

*   不支持许多框架
*   它的大多数本地支持的语言只能用作无服务器功能
*   有限控制

### 铁路. app

[Railway.app](https://railway.app/) 是一个鲜为人知的基础设施平台，允许您提供基础设施，在本地使用该基础设施进行开发，然后将其部署到云中。无论项目规模大小，它都适用于每种语言。

其特点包括:

*   自动缩放
*   使用指标
*   自动构建
*   协作功能

#### 赞成的意见

*   适合开发和原型制作
*   与 GitHub 的良好集成
*   [模板](https://railway.app/templates)适用于几乎所有的框架

#### 骗局

*   相对较新的平台
*   不如本文中的其他 PaaS 解决方案流行
*   由于公司名称的原因，很难找到任何与 Railway.app 相关的内容

### 红帽 OpenShift

[OpenShift](https://www.redhat.com/technologies/cloud-computing/openshift) 是红帽的云计算 PaaS 产品。这是一个构建在云中 Kubernetes 之上的应用程序平台，应用程序开发人员和团队可以在其中构建、测试、部署和运行他们的应用程序。

OpenShift 拥有无缝的 DevOps 工作流，可以水平和垂直扩展，并且可以自动扩展。

#### 赞成的意见

*   稳定，2011 年发布
*   与 GitHub 和 Docker 的强大集成
*   直观的 UI

#### 骗局

*   可能会很贵
*   可以改进监测和故障排除
*   缓慢的客户支持

### 应用程序

[Appliku](https://appliku.com/) 是一个 PaaS 平台，它使用您的云服务器来部署您的应用。您可以通过 Appliku 的仪表板链接您的 DigitalOcean 或 AWS 帐户并配置服务器。虽然他们的主要焦点是基于 Python 的应用程序，但是您可以通过利用 Docker 来部署用任何语言构建的应用程序。Appliku 的定价基于托管服务器的数量，因此您可以根据需要部署任意多的应用。他们确实提供免费层。

#### 赞成的意见

*   专为 Python/Django 构建
*   经济高效地运行多个应用
*   使用任何云提供商
*   CI/CD、GitHub 和 GitLab 集成
*   让我们加密集成
*   轻松访问服务器日志

#### 骗局

*   更新的平台(2019 年)
*   不如其他一些平台受欢迎

## 结论

Heroku 是一个成熟、久经考验且稳定的平台。它为您做了很多繁重的工作，并且将为您节省大量的时间和金钱，尤其是对于小型团队。Heroku 让您可以专注于您的产品，而不是摆弄您的服务器的配置选项和雇用 DevOps 工程师或系统管理员。

它可能不是最便宜的选择，但它仍然是市场上最好的 PaaS 之一。因此，如果你已经在使用 Heroku，你应该有一个很强的理由放弃它。

虽然市场上有许多替代方案，但没有一个能与 Heroku 的开发人员体验相匹配。目前，Heroku 最有希望的替代品是数字海洋应用平台和 T2 渲染。这两个平台的问题是它们相对较新，还没有经过实战考验。如果你只是想找一个地方来免费托管你的应用程序，那就用 Render 吧。