# 2023 年 Django vs. Flask:选择哪个框架

> 原文：<https://testdriven.io/blog/django-vs-flask/>

根据 [2021 JetBrains Python 开发者调查](https://lp.jetbrains.com/python-developers-survey-2021/#FrameworksLibraries)，Django 和 Flask 是目前最流行的两个 Python web 框架。虽然 Django 传统上是最受欢迎的 Python web 框架，但 Flask 在几年前超过了 Django，占据了头把交椅，考虑到 web 开发行业在过去七年左右的时间里一直倾向于更小的框架、微服务和“无服务器”平台，这并不奇怪。

> 或者这可能与行业趋势关系不大，而与 JetBrains 用户关系更大？

Django 和 Flask 拥有成熟的社区，得到了广泛的支持和欢迎，并提供了高效的应用程序开发方法，让您可以将时间和精力集中在应用程序的独特部分，而不是核心构架上。最后，这两个框架都用于开发 web 应用程序。关键的区别在于他们如何实现这个目标。把 Django 想象成一辆汽车，把 Flask 想象成一辆自行车。两者都可以把你从 A 点带到 B 点，但是他们的方法是完全不同的。每种都有自己的最佳用例。姜戈和弗拉斯克也一样。

在本文中，我们将从教育和开发的角度来看 Django 和 Flask 的最佳用例，以及它们的独特之处。

## 哲学

Django 和 Flask 都是免费的、开源的、基于 Python 的 web 框架，旨在构建 web 应用程序。

与 Flask 相比，Django 支持稳定性以及“包含电池”的方法，即开箱即用地提供大量电池(例如，工具、模式、特性和功能)。在稳定性方面，Django 通常有更长、更严格的发布周期。因此，虽然 Django 版本的新特性较少，但它们往往具有更强的向后兼容性。

基于 [Werkzeug](/blog/what-is-werkzeug/) ，Flask 很好的处理了核心脚手架。开箱即用，您可以获得 URL 路由、请求和错误处理、模板、cookies、单元测试支持、调试器和开发服务器。因为大多数 web 应用程序需要更多的东西(比如 ORM、认证和授权等等)，所以您可以决定如何构建您的应用程序。无论是利用第三方扩展还是自己定制代码，Flask 都不会干涉。比 Django 灵活多了。如果你需要打开引擎盖查看源代码，那么暴露在攻击面前的表面区域也少得多，需要审查的[代码也少得多。](http://www.grokcode.com/864/snakefooding-python-code-for-complexity-visualization/)

> 我强烈建议你阅读并回顾一下 [Flask 源代码](https://github.com/pallets/flask/tree/main/src/flask)。干净、清晰、简洁——这是结构良好的 Python 代码的一个很好的例子。

## 特征

接下来，让我们根据核心框架附带的特性来比较 Flask 和 Django。

### 数据库ˌ资料库

Django 包括一个简单而强大的 ORM(对象关系映射),支持许多现成的关系数据库(T2 ): SQLite、PostgreSQL、MySQL、MariaDB 和 Oracle。ORM 为生成和管理数据库[迁移](https://docs.djangoproject.com/en/4.1/topics/migrations/)提供支持。基于数据模型创建表单、视图和模板也相当容易，这对于典型的 [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) web 应用程序来说是完美的。虽然它确实有一些缺点，但对于大多数网络应用来说已经足够好了。

Flask 没有假设数据是如何存储的，但是有大量的库和扩展可以帮助你:

总之，如果您正在使用关系数据库，Django 会让您更容易上手，因为它有一个内置的 ORM 和迁移管理工具。但是，如果您正在使用非关系数据库，或者想要使用不同的 ORM，比如 SQLAlchemy，Django 几乎会在每一步都与您作对。另外，您很可能无法利用 Django 管理、模型表单或 [DRF 模型序列化程序](https://www.django-rest-framework.org/api-guide/serializers/#modelserializer)。

Flask 不碍事，让您可以自由挑选最适合您的应用程序的 ORM(或 ODM)。不过，自由是有代价的:学习曲线更高，出错的空间也更大，因为你要自己管理这些部分。

> 你自己做的越多，你犯的错误就越多，尤其是当事情扩大的时候。

### 作家（author 的简写）

既然大部分网络应用都需要认证(*你是谁？*)和授权(*你被允许做什么？*)，Django 提供了这一功能以及账户管理和会话支持(通过[用户模型](https://docs.djangoproject.com/en/4.1/ref/contrib/auth/))。Flask 支持基于 cookie 的会话，但是您必须求助于扩展网络来进行帐户管理、认证和授权。

### 管理

Django 附带了一个功能性的[管理面板](https://docs.djangoproject.com/en/4.1/ref/contrib/admin/)，这是一个 web 应用程序，它提供了一个用户界面来管理基于您的模型的数据。这是 Django 的另一个亮点。它允许您在构建应用程序时快速对模型执行 CRUD 操作，而无需编写任何额外的代码。再说一次，Flask 没有附带这样的东西，但是 [Flask-Admin](https://github.com/flask-admin/flask-admin) 扩展提供了[所有相同的功能以及更多的功能](https://flask-admin.readthedocs.io/en/latest/advanced/#migrating-from-django):

> Django 自动做很多事情。
> 
> Flask 哲学略有不同——显式比隐式好。如果某个东西应该初始化，那么它应该由开发人员来初始化。
> 
> Flask-Admin 遵循这个约定。作为一名开发人员，应该由您来告诉 Flask-Admin 应该显示什么以及如何显示。
> 
> 有时这将需要编写一些样板代码，但将来会有回报，尤其是如果您必须实现一些定制逻辑的话。

Flask-Admin 支持一定数量的数据库后端，比如 SQLAlchemy、Peewee、MongoEngine 等等。你也可以[添加你自己的后端](https://flask-admin.readthedocs.io/en/latest/adding_a_new_model_backend/)。它还可以与(或不与)流行的 Flask auth 扩展一起使用:

1.  [烧瓶-登录](https://flask-login.readthedocs.io/)和[烧瓶-负责人](https://pythonhosted.org/Flask-Principal/)
2.  [烧瓶-安全](https://flask-security-too.readthedocs.io/)

### 路由和视图

这两个框架都允许您将 URL 映射到视图，并支持基于函数和类的视图。

姜戈

当一个请求匹配一个 URL 模式时，保存 HTTP 请求信息的请求对象被传递给一个视图，然后这个视图被调用。任何时候你需要访问请求对象，你必须*显式地*传递它。

URL 和视图分别在单独的文件中定义- *urls.py* 和 *views.py* 。

1.  [路由](https://docs.djangoproject.com/en/4.1/topics/http/urls/)
2.  [视图](https://docs.djangoproject.com/en/4.1/topics/http/views/)
3.  [基于类的视图](https://docs.djangoproject.com/en/4.1/topics/class-based-views/)
4.  [请求上下文](https://docs.djangoproject.com/en/4.1/ref/request-response/)
5.  [Django 如何处理请求](https://docs.djangoproject.com/en/4.1/topics/http/urls/#how-django-processes-a-request)

**烧瓶**

在其核心，Flask 使用 [Werkzeug](https://werkzeug.palletsprojects.com) ，它提供 URL 路由和请求/响应处理。

请求对象在 Flask 中是*全局的*，所以您可以更容易地访问它(只要您导入它)。URL 通常与视图一起定义(通过一个[装饰器](https://flask.palletsprojects.com/patterns/viewdecorators/)，但是它们可以被[分离出来](https://flask.palletsprojects.com/patterns/lazyloading/#converting-to-centralized-url-map)到一个类似 Django 模式的集中位置。

1.  [路由](https://flask.palletsprojects.com/api/#url-route-registrations)
2.  [视图](https://flask.palletsprojects.com/views/#views)
3.  [基于类的视图](https://flask.palletsprojects.com/api/#class-based-views)
4.  [请求上下文](https://flask.palletsprojects.com/reqcontext/)

> 你注意到 Django 和 Flask 处理请求对象的不同了吗？一般来说，Flask 倾向于更明确地处理事情，但在这种情况下，情况正好相反:Django 强迫你明确地传递请求对象，而 Flask 的请求对象只是神奇地可用。这是 Flask 的难点之一，尤其是对于那些来自类似风格框架的新手来说，比如 [Express.js](https://expressjs.com/) 。

### 形式

表单是大多数 web 应用程序的另一个重要组成部分，与 Django 打包在一起。这包括输入处理、客户端和服务器端验证以及各种安全问题的处理，如跨站点请求伪造(CSRF)、跨站点脚本(XSS)和 SQL 注入。它们可以从数据模型中创建(通过[模型表单](https://docs.djangoproject.com/en/4.1/topics/forms/modelforms/))并与管理面板很好地集成。

Flask 默认不支持表单，但是强大的 [Flask-WTF](https://flask-wtf.readthedocs.io) 扩展集成了 Flask 和 [WTForms](https://wtforms.readthedocs.io) 。 [WTForms-Alchemy](https://wtforms-alchemy.readthedocs.io/en/latest/) 可以用来自动创建基于 SQLAlchemy 模型的表单，在表单和 ORM 之间架起一座桥梁，就像 Django 的 ModelForm 一样。

### 可重用组件

关于项目结构，随着你的应用程序变得越来越复杂，这两个框架都让你很容易通过将相关的文件组合在一起，展示相似的功能来分解它们。因此，例如，您可以将所有与用户相关的功能组合在一起，其中可以包括路由、视图、表单、模板和静态资产。

Django 有一个应用程序的概念，而 Flask 有 T2 的蓝图。

Django 应用程序比 Flask blueprints 更复杂，但是一旦设置好，它们更容易使用和重用。另外，由于 *urls.py* 、 *models.py* 和 *views.py* 约定一致的项目结构！-您可以相当容易地向 Django 项目添加新的开发人员。与此同时，蓝图更简单，更容易启动和运行。

### 模板和静态文件

模板引擎允许您从后端动态地将信息注入到 HTML 页面中。Flask 默认使用 Jinja2 ，而 Django 有自己的[模板引擎](https://docs.djangoproject.com/en/4.1/ref/templates/)。它们在语法和功能集方面[非常相似](http://jinja.pocoo.org/docs/switching/#django)。也可以用 Django 搭配 [Jinja2。](https://docs.djangoproject.com/en/4.1/topics/templates/#django.template.backends.jinja2.Jinja2)

两个框架都支持静态文件处理:

1.  姜戈
2.  [烧瓶](https://flask.palletsprojects.com/quickstart/#static-files)

Django 附带了一个方便的管理命令，用于收集所有静态文件，并将它们放在一个中心位置进行生产部署。

### 异步视图

随着 [Django 3.1](https://docs.djangoproject.com/en/4.1/releases/3.1/#asynchronous-views-and-middleware-support) 的引入，Django 支持[异步处理程序](https://docs.djangoproject.com/en/4.1/topics/async/)。使用关键字`async`可以使视图异步。异步支持也适用于中间件。如果需要在异步视图中进行同步调用，可以使用 [sync_to_async](https://docs.djangoproject.com/en/4.1/topics/async/#asgiref.sync.sync_to_async) 函数/装饰器。这可以用来与 Django 中还不支持异步的其他部分进行交互，比如 ORM 和缓存层。

异步 web 服务器，包括但不限于[达芙妮](https://docs.djangoproject.com/en/4.1/howto/deployment/asgi/daphne/)、 [Hypercorn](https://docs.djangoproject.com/en/4.1/howto/deployment/asgi/hypercorn/) 、[uvicon](https://docs.djangoproject.com/en/4.1/howto/deployment/asgi/uvicorn/)，应该与 Django 一起使用，以充分利用异步视图的能力。

[Flask 2.0](https://palletsprojects.com/blog/flask-2-0-released/) 增加了对[异步](https://flask.palletsprojects.com/en/2.2.x/async-await/)路由/视图、错误处理程序、请求函数之前和之后以及拆卸回调的内置支持！

> 关于 Django 和 Flask 中异步视图的更多信息，请分别查阅 Django 中的[异步视图和 Flask 2.0](/blog/django-async-views/) 中的[异步视图。](/blog/flask-async/)

### 测试

两个框架都有内置的测试支持。

对于单元测试，它们都利用了 Python 的 [unittest](https://docs.python.org/3/library/unittest.html) 框架。它们中的每一个都支持一个测试客户机，您可以向它发送请求，然后检查和验证响应的一部分。

更多信息，请分别参见[测试烧瓶应用](https://flask.palletsprojects.com/testing/)和[在 Django](https://docs.djangoproject.com/en/4.1/topics/testing/) 的测试。

在扩展方面，如果你喜欢 unittest 框架的工作方式，请查看[烧瓶测试](https://pythonhosted.org/Flask-Testing)。另一方面， [pytest-flask](https://github.com/pytest-dev/pytest-flask) 延长件为烧瓶增加了 [pytest](https://docs.pytest.org) 支架。对于 Django，请查看 [pytest-django](https://pytest-django.readthedocs.io) 。

### 其他功能

Django 还有几个没有提到的特性，但 Flask 没有:

## 安全性

如前所述，Django 内置了针对一些常见攻击媒介的保护，如 CSRF、XSS 和 SQL 注入。这些安全措施有助于防止代码中的漏洞。Django 开发团队还[主动披露](https://docs.djangoproject.com/en/dev/internals/security/)并快速修复已知的安全漏洞。另一方面，Flask 的代码库要小得多，因此暴露在攻击面前的空间也更小。然而，当你手工制作的应用程序代码出现安全漏洞时，你需要解决并修复它们。

归根结底，你的安全取决于你最薄弱的一环。因为 Flask 更依赖于第三方扩展，所以应用程序的安全性将取决于最不安全的扩展。这给开发团队带来了更大的压力，他们需要通过评估和监控第三方库和扩展来维护安全性。保持它们最新是最重要的(也是最困难的)事情，因为每个扩展都有自己的开发团队、文档和发布周期。在某些情况下，可能只有一两个开发人员维护一个特定的扩展。当评估一个扩展优于另一个扩展时，一定要回顾 GitHub 的问题，看看维护人员通常需要多长时间来应对关键问题。

这并不意味着 Django 天生就比 Flask 更安全；在应用程序的生命周期中，前期保护和维护更容易。

资源:

1.  [烧瓶安全注意事项](https://flask.palletsprojects.com/security/)
2.  [Django 的安全](https://docs.djangoproject.com/en/4.1/topics/security/)
3.  [保护烧瓶网络应用程序](https://smirnov-am.github.io/securing-flask-web-applications/)
4.  [保护您的 Django Web 应用免受安全威胁](https://web.archive.org/web/20221002220405/https://hashedin.com/blog/protect-your-django-web-application-from-security-threats/)

## 灵活性

从设计上来说，Flask 比 Django 灵活得多，而且它应该被[扩展](https://flask.palletsprojects.com/extensions/)。因此，Flask 通常需要更长的设置时间，因为您必须根据业务需求添加适当的扩展——例如 ORM、权限、认证等等。这种前期成本为不符合标准 Django 模型的应用程序带来了更大的灵活性。

不过，小心点。灵活性给了开发人员更多的自由和控制，但是这会减慢开发速度，特别是对于大型团队，因为需要做出更多的决策。

开发人员喜欢自由地做他们想做的任何事情来解决问题。由于 Flask 没有提供许多关于如何开发应用程序的约束或意见，开发者可以介绍他们自己的。结果是，两个功能上可以互换的 Flask 应用程序通常会有不同的结构。因此，你需要一个更成熟的团队，理解设计模式、可伸缩性和测试的重要性，来处理这样的灵活性。

## 教育

学习模式，而不是语言或框架。

不管你的最终目标是学习 Flask 还是 Django，先从 Flask 开始。这是学习 web 开发基础和最佳实践以及几乎所有框架都通用的 web 框架核心部分的一个很好的工具。

1.  Flask 比 Django 更轻，也更露骨。因此，如果您是 web 开发的新手，但不熟悉 Python，您会发现在 Flask 中开发要容易得多，因为这感觉就像您在用普通的 Python 定义请求处理程序和视图等等。
2.  姜戈开销很大。从项目结构到设置，再到安装一些你一无所知的螺母和螺栓，你会迷失方向，最终学到的更多的是 Django 本身，而不是实际的基础知识。

几乎所有情况下，都建议先学 Flask 再学 Django。你唯一应该偏离这一点的时候是当你只需要让一个应用程序快速运行来满足一些外部利益相关者的时候。一定要回到 Flask，在某个时候学习基础知识。

## 开放源码

Django 和 Flask 有强大的开源社区。

**截至 2023 年 1 月 30 日的 GitHub 统计:**

| 公制的 | 姜戈 | [烧瓶](https://github.com/pallets/flask) |
| --- | --- | --- |
| 第一次提交 | 2005 | 2010 |
| 贡献者 | 2,337 | 690 |
| 用户* | 1,138,608 | 1,290,243 |
| 观察者 | 2,264 | 2,149 |
| 明星 | 68,431 | 61,711 |

*依赖项被其他存储库使用的次数

更多信息，请查看 Open Hub 的 Django 和 Flask 的[开源比较。](https://www.openhub.net/p/_compare?project_0=Django&project_1=Flask)

**截至 2023 年 1 月 30 日堆栈溢出问题:**

--

我们在这里能得出什么结论？

1.  两个社区都非常活跃
2.  Django 更老，有更多的贡献者
3.  Flask 被更多的项目使用
4.  Django 上有更多的内容

要从开源的角度真正比较这些框架(或生态系统)，您必须考虑 Jinja2 和 Werkzeug 以及一些核心 Flask 库和扩展，如 SQLAlchemy / Flask-SQLAlchemy、Alembic / Flask-Alembic 和 WTForms / Flask-WTF。

由于核心 Flask 功能分散在多个项目中，社区很难创建和开发跨项目的必要协同来保持势头。例如，Flask 没有一个单一的、事实上的扩展来创建 RESTful APIs 截至 2023 年 1 月，有(可以说)四种流行的扩展:

1.  连接
2.  [烧瓶-RESTful](https://flask-restful.readthedocs.io)
3.  [烧瓶级](http://flask-classful.teracy.org/)
4.  [烧瓶-RESTX](https://flask-restx.readthedocs.io/)

更重要的是，为了找到这些扩展，你需要有相当扎实的信息检索技能。您必须过滤掉所有未维护的扩展以及引用它们的文章。事实上，RESTful APIs 有如此多不同的扩展项目，以至于开发自己的开源项目通常更容易。在这一点上，你可能会保持一段时间，但最终它会成为问题的一部分，而不是一个解决方案。

注意事项:

1.  这不是对 Flask 社区的抨击。这在整个开源中是一个问题，特别是在微型 web 框架中，您经常不得不拼凑由不同开发人员在不同发布周期维护的大量项目，这些项目具有不同的文档质量水平。如果您想看到这一点的极致，请前往 JavaScript 社区。
2.  Django 社区也不能幸免。这只是一个小问题，因为它几乎处理了构建和保护开箱即用的标准 web 应用程序所需的一切。

关于这篇综述的更多信息，请参见 Django vs Flask 的“开源动力”部分:实践者的视角:

> 由于没有统一战线，跨推广的协同努力的机会无法实现，从而产生了漏洞百出的推广。这就让开发人员去填补那些已经在工作的全包功能的空白，如果他们只是选择了一个不同的工具来完成这项工作。

## 雇用

尽管 Python 和 Django 都很受欢迎，但很难雇佣 Django 开发人员。他们很难找到和留住，因为他们的需求很高，而且他们通常更高端，所以他们可能相当昂贵。也没有很多新的、有抱负的 web 开发人员学习 Django，因为这个行业更关注较小的框架，而框架本身很难学习。

1.  **行业趋势**:由于微服务的兴起，有抱负的 web 开发人员通常会学习更小、更轻量级的框架。此外，由于客户端 JavaScript 框架 Angular、React 和 Vue 的流行，越来越多的 web 开发人员选择 JavaScript 而不是 Python 或 Ruby 作为他们的第一语言。
2.  难以学习:令人惊讶的是，缺乏适合初学者的 Django 教程。即使是 Django 文档，它非常全面，还有臭名昭著的[民意测验教程](https://docs.djangoproject.com/en/4.1/intro/tutorial01/)也不是为初学者设计的。

Flask 也很难租到，但它比 Django 容易，因为它是一个轻量级框架，抽象层更少。一个在不同语言的类似框架中有经验的优秀开发者，比如 [Express.js](https://expressjs.com/) 或 [Sinatra](http://sinatrarb.com/) ，可以很快熟悉 Flask 应用。当雇佣这样的开发人员时，把你的搜索重点放在那些理解设计模式和基本软件原则的人身上，而不是他们知道的语言或框架。

## 用例

当你决定一个框架时，一定要考虑到你的项目的个人需求。由于 Django 提供了许多额外的功能，您应该好好利用它们。如果你对姜戈处理事情的方式有强烈的不同意见，你可以选择弗拉斯克。如果您不想利用 Django 提供的结构和工具，也可以这么说。

让我们看一些例子。

### 数据库ˌ资料库

如果您的应用程序使用 SQLite、PostgreSQL、MySQL、MariaDB 或 Oracle，您应该仔细研究一下 Django。另一方面，如果您使用 NoSQL 或者根本没有数据库，那么 Flask 是一个可靠的选择。

### 项目规模和预期寿命

Flask 更适合于较小的、不太复杂的项目，这些项目有明确的范围和较短的预期生命周期。

因为 Django 强制使用一致的应用程序结构，而不管项目的大小，几乎所有的 Django 项目都有相似的结构。因此，Django 可以更好地处理更大的项目(更大的团队),这些项目具有更长的生命周期和更大的增长潜力，因为你很可能会不时地雇佣新的开发人员。

### 应用类型

您正在构建什么样的应用程序？

Django 擅长用服务器端模板创建全功能的 web 应用程序。如果您只是开发一个静态网站或 RESTful web 服务来支持您的 SPA 或移动应用程序，Flask 是一个可靠的选择。Django 结合 Django REST 框架在后一种情况下也能很好地工作。

### RESTful APIs

设计 RESTful API？

[Django REST 框架](https://www.django-rest-framework.org/) (DRF)，最流行的[第三方 Django 包](https://djangopackages.org/)之一，是一个用于通过 RESTful 接口公开 Django 模型的框架。它提供了您需要的一切(视图、序列化器、验证、授权)和更多(可浏览 API、版本控制、缓存)来快速轻松地构建 API。也就是说，请记住，与 Django ORM 一样，它也是为了与关系数据库相结合。

Flask 也有许多很好的扩展:

一定要看看[连接](https://connexion.readthedocs.io)，它将视图、序列化和验证功能组合到一个包中！

--

回顾一下[这个](https://softwareengineering.stackexchange.com/a/188358)堆栈交换答案，了解您在选择框架时可能需要考虑的许多其他需求。

## 表演

Flask 的性能稍好，因为它更小，层数更少。不过，这里的差别可以忽略不计，尤其是当你考虑 I/O 的时候。

## 结论

那么，您应该使用哪个框架呢？一如既往，视情况而定。选择使用一种框架或语言或工具，几乎完全取决于当前的环境和问题。

Django 功能齐全，因此您或您的团队需要做的决策更少。那样你可能会走得更快。但是，如果您不同意 Django 为您做出的选择之一，或者您有独特的应用程序需求，这限制了您可以利用的特性的数量，那么您可能希望使用 Flask。

总会有权衡和妥协。

考虑项目限制，如时间、知识和预算。你的应用有哪些关键特性？什么是你不能妥协的？你需要快速行动吗？你的应用需要很大的灵活性吗？回答这些问题时，尽量把你的意见放在一边。

最终，这两个框架都降低了构建 web 应用程序的门槛，使它们的开发变得更加容易和快捷。

我希望这有所帮助。要了解更多信息，请查看以下两个优秀资源:

1.  Django vs Flask:一个实践者的视角
2.  [Flask vs. Django:选择你的 Python Web 框架](https://kite.com/blog/python/flask-vs-django-python)