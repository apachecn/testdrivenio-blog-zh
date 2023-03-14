# Django 和 Aloe 的行为驱动开发

> 原文：<https://testdriven.io/blog/behavior-driven-development-with-django-and-aloe/>

假设你是一名 Django 开发人员，正在为一家精益创业公司构建一个社交网络。CEO 向你的团队施压，要求获得 MVP。工程师们已经同意使用[行为驱动开发](https://en.wikipedia.org/wiki/Behavior-driven_development) (BDD)来构建产品，以交付快速高效的结果。产品负责人给你第一个特性请求，并遵循所有好的编程方法的实践，你通过编写测试开始 BDD 过程。接下来，您编写一些功能代码以通过测试，然后考虑您的设计。最后一步要求您分析特征本身。它属于你的应用吗？

我们不能回答你这个问题，但是我们可以教你什么时候问这个问题。**在下面的教程中，我们通过使用 Django 和[Aloe](https://aloe.readthedocs.io/)编程一个示例特性，带你走过 BDD 开发周期。继续学习如何使用 BDD 过程来帮助快速捕捉和修复糟糕的设计，同时编写稳定的应用程序。**

## 目标

完成本教程后，您应该能够:

1.  描述并实践行为驱动开发(BDD)
2.  解释如何在新项目中实施 BDD
3.  使用芦荟测试您的 Django 应用程序

## 项目设置

当你阅读这篇文章时，你想要建立这个项目吗？

开始于:

1.  添加项目目录。
2.  创建和激活虚拟环境。

然后，安装以下依赖项并启动一个新的 Django 项目:

```py
`(venv)$ pip install \
        django==3.2.4 \
        djangorestframework==3.12.4 \
        aloe_django==0.2.0
(venv)$ django-admin startproject example_bdd .
(venv)$ python manage.py startapp example` 
```

> 如果在尝试安装`aloe_django`时出现此错误，您可能需要手动安装 [setuptools-scm](https://github.com/pypa/setuptools_scm) ( `pip install setuptools-scm`):
> 
> ```py
> distutils.errors.DistutilsError:
> Could not find suitable distribution for Requirement.parse('setuptools_scm') 
> ```

更新 *settings.py* 中的`INSTALLED_APPS`列表:

```py
`INSTALLED_APPS = [

    ...

    'aloe_django',
    'rest_framework',
    'example',
]` 
```

只是在找代码吗？从[回购](https://github.com/testdrivenio/django-aloe-bdd)中抢过来。

## BDD 概述

行为驱动开发是一种测试代码的方式，它挑战你不断地重新审视你的设计。当你写一个测试时，你要回答这个问题*我的代码做了我期望它做的事情吗？*通过断言。失败的测试暴露了代码中的错误。使用 BDD，你分析一个特性:*用户体验是我期望的那样吗？*没有什么比失败的测试更能暴露糟糕的特性了，但是带来糟糕体验的后果却是实实在在的。

将 BDD 作为测试开发周期的一部分来执行。用测试画出一个特性的功能边界。创建在细节中着色的代码。退一步考虑你的设计。然后再重新做一遍，直到画面完整。

> 查看下面的[帖子](https://behave.readthedocs.io/en/latest/philosophy.html)以获得对 BDD 更深入的解释。

## 您的第一个功能请求

"用户应该能够登录应用程序，看到他们的朋友名单."

这就是你的产品经理开始谈论这款应用的第一个功能的方式。虽然不多，但是你可以用它来写一个测试。她实际上要求两项功能:

1.  用户认证
2.  在用户之间形成关系的能力。

这里有一个经验法则:像对待灯塔一样对待一个连词，警告你不要试图一次测试太多东西。如果你在测试语句中看到“and”或“or ”,你应该把测试分成更小的部分。

记住这一点，利用特性请求的前半部分，编写一个测试场景:*用户可以登录应用程序*。为了支持用户身份验证，您的应用程序必须存储用户凭据，并为用户提供使用这些凭据访问其数据的方法。以下是你如何将这些标准转化为芦荟*。特征*文件。

**例子/特色/友谊.特色**

```py
`Feature: Friendships

  Scenario: A user can log into the app

    Given I empty the "User" table

    And I create the following users:
      | id | email             | username | password  |
      | 1  | [[email protected]](/cdn-cgi/l/email-protection) | Annie    | pAssw0rd! |

    When I log in with username "Annie" and password "pAssw0rd!"

    Then I am logged in` 
```

一个芦荟测试用例被称为一个*特性*。你使用两个文件编程特征:一个*特征*文件和一个*步骤*文件。

1.  *特性*文件由用简单英语编写的语句组成，这些语句描述了如何配置、执行和确认测试结果。使用`Feature`关键字来标记特性，使用`Scenario`关键字来定义您计划测试的用户情景。在上面的示例中，该场景定义了一系列步骤，解释如何填充*用户*数据库表，让用户登录应用程序，并验证登录。所有的步骤语句都必须以四个关键字之一开头:`Given`、`When`、`Then`或`And`。
2.  *步骤*文件包含使用正则表达式映射到*特征*文件步骤的 Python 函数。

> 您可能需要添加一个 *__init__。py* 文件到“特征”目录，以便解释器正确加载*友谊 _ 步骤. py* 文件。

运行`python manage.py harvest`并查看以下输出。

```py
`nosetests --verbosity=1
Creating test database for alias 'default'...
E
======================================================================
ERROR: A user can log into the app (example.features.friendships: Friendships)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "django-aloe-bdd/venv/lib/python3.9/site-packages/aloe/registry.py", line 151, in wrapped
    return function(*args, **kwargs)
  File "django-aloe-bdd/example/features/friendships.feature", line 5, in A user can log into the app
    Given I empty the "User" table
  File "django-aloe-bdd/venv/lib/python3.9/site-packages/aloe/registry.py", line 151, in wrapped
    return function(*args, **kwargs)
  File "django-aloe-bdd/venv/lib/python3.9/site-packages/aloe/exceptions.py", line 44, in undefined_step
    raise NoDefinitionFound(step)
aloe.exceptions.NoDefinitionFound: The step r"Given I empty the "User" table" is not defined
-------------------- >> begin captured logging << --------------------
asyncio: DEBUG: Using selector: KqueueSelector
--------------------- >> end captured logging << ---------------------

----------------------------------------------------------------------
Ran 1 test in 0.506s

FAILED (errors=1)
Destroying test database for alias 'default'...` 
```

测试失败是因为您没有将 step 语句映射到 Python 函数。在以下文件中执行此操作。

**示例/功能/友谊 _ 步骤. py**

```py
`from aloe import before, step, world
from aloe.tools import guess_types
from aloe_django.steps.models import get_model
from django.contrib.auth.models import User
from rest_framework.test import APIClient

@before.each_feature
def before_each_feature(feature):
    world.client = APIClient()

@step('I empty the "([^"]+)" table')
def step_empty_table(self, model_name):
    get_model(model_name).objects.all().delete()

@step('I create the following users:')
def step_create_users(self):
    for user in guess_types(self.hashes):
        User.objects.create_user(**user)

@step('I log in with username "([^"]+)" and password "([^"]+)"')
def step_log_in(self, username, password):
    world.is_logged_in = world.client.login(username=username, password=password)

@step('I am logged in')
def step_confirm_log_in(self):
    assert world.is_logged_in` 
```

每个语句都通过一个`@step()`装饰器映射到一个 Python 函数。例如，`Given I empty the "User" table`将触发`step_empty_table()`功能运行。在这种情况下，字符串`"User"`将被捕获并作为`model_name`参数传递给函数。Aloe API 包括一个名为`world`的特殊全局变量，可以用来在测试步骤之间存储和检索数据。注意`world.is_logged_in`变量是如何在`step_log_in()`中创建，然后在`step_confirm_log_in()`中访问的。Aloe 还定义了一个特殊的`@before` decorator 来在测试运行之前执行函数。

最后一件事:考虑以下语句的结构:

```py
`And I create the following users:
  | id | email             | username | password  |
  | 1  | [[email protected]](/cdn-cgi/l/email-protection) | Annie    | pAssw0rd! |` 
```

有了 Aloe，您可以使用表格结构来表示字典列表。然后，您可以使用`self.hashes`来访问数据。在`guess_types()`函数中包装`self.hashes`会返回一个列表，其中包含正确输入的字典值。在这个例子中，`guess_types(self.hashes)`返回这个代码。

```py
`[{'id': 1, 'email': '[[email protected]](/cdn-cgi/l/email-protection)', 'username': 'Annie', 'password': 'pAssw0rd!'}]` 
```

用下面的命令运行 Aloe 测试套件，看到所有测试都通过了。

```py
`(venv)$ python manage.py harvest` 
```

```py
`nosetests --verbosity=1
Creating test database for alias 'default'... . ---------------------------------------------------------------------- Ran 1 test in 0.512s

OK
Destroying test database for alias 'default'...` 
```

为功能请求的第二部分编写一个测试场景:*用户可以看到朋友列表*。

**例子/特色/友谊.特色**

```py
`Scenario: A user can see a list of friends

  Given I empty the "Friendship" table

  When I get a list of friends

  Then I see the following response data:
    | id | email | username |` 
```

在运行 Aloe 测试套件之前，修改第一个场景，使用关键字`Background`代替`Scenario`。背景是一种特殊类型的场景，在由*特征*文件中的`Scenario`定义的每个程序块之前运行一次。每个场景都需要从头开始，每次运行时使用`Background`都会刷新数据。

**例子/特色/友谊.特色**

```py
`Feature: Friendships

  Background: Set up common data

    Given I empty the "User" table

    And I create the following users:
      | id | email             | username | password  |
      | 1  | [[email protected]](/cdn-cgi/l/email-protection) | Annie    | pAssw0rd! |
      | 2  | [[email protected]](/cdn-cgi/l/email-protection) | Brian    | pAssw0rd! |
      | 3  | [[email protected]](/cdn-cgi/l/email-protection) | Casey    | pAssw0rd! |

    When I log in with username "Annie" and password "pAssw0rd!"

    Then I am logged in

  Scenario: A user can see a list of friends

    Given I empty the "Friendship" table

    And I create the following friendships:
      | id | user1 | user2 |
      | 1  | 1     | 2     |

    # Annie and Brian are now friends.

    When I get a list of friends

    Then I see the following response data:
      | id | email             | username |
      | 2  | [[email protected]](/cdn-cgi/l/email-protection) | Brian    |` 
```

现在您正在处理多个用户之间的友谊，开始时向数据库添加几个新的用户记录。新的场景从“Friendship”表中清除所有条目，并创建一条新记录来定义 Annie 和 Brian 之间的友谊。然后，它调用一个 API 来检索 Annie 的朋友列表，并确认响应数据包括 Brian。

第一步是创建一个`Friendship`模型。很简单:它只是将两个用户链接在一起。

**example/models.py**

```py
`from django.conf import settings
from django.db import models

class Friendship(models.Model):
    user1 = models.ForeignKey(
      settings.AUTH_USER_MODEL,
      on_delete=models.CASCADE,
      related_name='user1_friendships'
    )
    user2 = models.ForeignKey(
      settings.AUTH_USER_MODEL,
      on_delete=models.CASCADE,
      related_name='user2_friendships'
    )` 
```

进行迁移并运行它。

```py
`(venv)$ python manage.py makemigrations
(venv)$ python manage.py migrate` 
```

接下来，为`I create the following friendships:`语句创建一个新的测试步骤。

**示例/功能/友谊 _ 步骤. py**

```py
`@step('I create the following friendships:')
def step_create_friendships(self):
    Friendship.objects.bulk_create([
        Friendship(
            id=data['id'],
            user1=User.objects.get(id=data['user1']),
            user2=User.objects.get(id=data['user2'])
        ) for data in guess_types(self.hashes)
    ])` 
```

将`Friendship`模型导入添加到文件中。

```py
`from ..models import Friendship` 
```

创建一个 API 来获取登录用户的朋友列表。创建一个序列化程序来处理`User`资源的表示。

**example/serializer . py**

```py
`from django.contrib.auth.models import User
from rest_framework import serializers

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('id', 'email', 'username',)
        read_only_fields = fields` 
```

创建一个管理器来处理您的`Friendship`模型的表级功能。

**example/models.py**

```py
`# New import!
from django.db.models import Q

class FriendshipManager(models.Manager):
    def friends(self, user):
        """Get all users that are friends with the specified user."""
        # Get all friendships that involve the specified user.
        friendships = self.get_queryset().select_related(
            'user1', 'user2'
        ).filter(
            Q(user1=user) |
            Q(user2=user)
        )

        def other_user(friendship):
            if friendship.user1 == user:
                return friendship.user2
            return friendship.user1

        return map(other_user, friendships)` 
```

`friends()`函数检索指定用户与其他用户共享的所有友谊。然后它返回其他用户的列表。将`objects = FriendshipManager()`添加到`Friendship`模型中。

创建一个简单的`ListAPIView`来返回您的`User`资源的 JSON 序列化列表。

**example/views.py**

```py
`from rest_framework.generics import ListAPIView

from .models import Friendship
from .serializers import UserSerializer

class FriendsView(ListAPIView):
    serializer_class = UserSerializer

    def get_queryset(self):
        return Friendship.objects.friends(self.request.user)` 
```

最后，添加一个 URL 路径。

**example_bdd/urls.py**

```py
`from django.urls import path

from example.views import FriendsView

urlpatterns = [
    path('friends/', FriendsView.as_view(), name='friends'),
]` 
```

创建剩余的 Python 步骤函数:一个调用新的 API，另一个通用函数确认响应负载数据。(我们可以重用这个函数来检查任何有效载荷。)

**示例/功能/友谊 _ 步骤. py**

```py
`@step('I get a list of friends')
def step_get_friends(self):
    world.response = world.client.get('/friends/')

@step('I see the following response data:')
def step_confirm_response_data(self):
    response = world.response.json()
    if isinstance(response, list):
        assert guess_types(self.hashes) == response
    else:
        assert guess_types(self.hashes)[0] == response` 
```

运行测试并观察它们是否通过。

```py
`(venv)$ python manage.py harvest` 
```

想想另一个测试场景。没有朋友的用户在调用 API 时应该会看到一个空列表。

**例子/特色/友谊.特色**

```py
`Scenario: A user with no friends sees an empty list

  Given I empty the "Friendship" table

  # Annie has no friends.

  When I get a list of friends

  Then I see the following response data:
    | id | email | username |` 
```

不需要新的 Python 函数。您可以重复使用所有步骤！测试无需任何干预即可通过。

您还需要最后一项功能来实现这一特性。用户可以获得他们的朋友列表，但是他们如何结交新朋友呢？这里有一个新的场景:“一个用户应该能够将另一个用户添加为好友。”用户应该能够调用 API 来与另一个用户建立友谊。您知道，如果在数据库中创建了记录，API 就会工作。

**例子/特色/友谊.特色**

```py
`Scenario: A user can add a friend

  Given I empty the "Friendship" table

  When I add the following friendship:
    | user1 | user2 |
    | 1     | 2     |

  Then I see the following rows in the "Friendship" table:
    | user1 | user2 |
    | 1     | 2     |` 
```

创建新的阶跃函数。

**示例/功能/友谊 _ 步骤. py**

```py
`@step('I add the following friendship:')
def step_add_friendship(self):
    world.response = world.client.post('/friendships/', data=guess_types(self.hashes[0]))

@step('I see the following rows in the "([^"]+)" table:')
def step_confirm_table(self, model_name):
    model_class = get_model(model_name)
    for data in guess_types(self.hashes):
        has_row = model_class.objects.filter(**data).exists()
        assert has_row` 
```

扩展管理器并做一些重构。

**example/models.py**

```py
`class FriendshipManager(models.Manager):
    def friendships(self, user):
        """Get all friendships that involve the specified user."""
        return self.get_queryset().select_related(
            'user1', 'user2'
        ).filter(
            Q(user1=user) |
            Q(user2=user)
        )

    def friends(self, user):
        """Get all users that are friends with the specified user."""
        friendships = self.friendships(user)

        def other_user(friendship):
            if friendship.user1 == user:
                return friendship.user2
            return friendship.user1

        return map(other_user, friendships)` 
```

添加一个新的序列化程序来呈现`Friendship`资源。

**example/serializer . py**

```py
`class FriendshipSerializer(serializers.ModelSerializer):
    class Meta:
        model = Friendship
        fields = ('id', 'user1', 'user2',)
        read_only_fields = ('id',)` 
```

添加新视图。

**example/views.py**

```py
`class FriendshipsView(ModelViewSet):
    serializer_class = FriendshipSerializer

    def get_queryset(self):
        return Friendship.objects.friendships(self.request.user)` 
```

添加新的 URL。

**example/urls.py**

```py
`path('friendships/', FriendshipsView.as_view({'post': 'create'})),` 
```

您的代码工作，测试通过！

## 分析特征

既然您已经成功地对您的特性进行了编程和测试，那么是时候对其进行分析了。当一个用户添加另一个用户时，两个用户成为朋友。这不是理想的行为。也许另一个用户不想成为朋友-他们没有发言权吗？一个用户应该*请求*与另一个用户建立友谊，而另一个用户应该能够接受或拒绝这个友谊。

修改用户添加另一个用户为好友的场景:“一个用户应该能够请求与另一个用户成为朋友。”

用这个替换`Scenario: A user can add a friend`。

**例子/特色/友谊.特色**

```py
`Scenario: A user can request a friendship with another user

  Given I empty the "Friendship" table

  When I request the following friendship:
    | user1 | user2 |
    | 1     | 2     |

  Then I see the following response data:
    | id | user1 | user2 | status  |
    | 3  | 1     | 2     | PENDING |` 
```

重构您的测试步骤以使用新的 API，`/friendship-requests/`。

**示例/功能/友谊 _ 步骤. py**

```py
`@step('I request the following friendship:')
def step_request_friendship(self):
    world.response = world.client.post('/friendship-requests/', data=guess_types(self.hashes[0]))` 
```

首先向`Friendship`模型添加一个新的`status`字段。

**example/models.py**

```py
`class Friendship(models.Model):
    PENDING = 'PENDING'
    ACCEPTED = 'ACCEPTED'
    REJECTED = 'REJECTED'
    STATUSES = (
      (PENDING, PENDING),
      (ACCEPTED, ACCEPTED),
      (REJECTED, REJECTED),
    )
    objects = FriendshipManager()
    user1 = models.ForeignKey(
      settings.AUTH_USER_MODEL,
      on_delete=models.CASCADE,
      related_name='user1_friendships'
    )
    user2 = models.ForeignKey(
      settings.AUTH_USER_MODEL,
      on_delete=models.CASCADE,
      related_name='user2_friendships'
    )
    status = models.CharField(max_length=8, choices=STATUSES, default=PENDING)` 
```

友谊可以是`ACCEPTED`或`REJECTED`。如果其他用户没有采取行动，那么默认状态是`PENDING`。

进行迁移并迁移数据库。

```py
`(venv)$ python manage.py makemigrations
(venv)$ python manage.py migrate` 
```

将`FriendshipsView`重命名为`FriendshipRequestsView`。

**example/views.py**

```py
`class FriendshipRequestsView(ModelViewSet):
    serializer_class = FriendshipSerializer

    def get_queryset(self):
        return Friendship.objects.friendships(self.request.user)` 
```

用新的 URL 路径替换旧的路径。

**example/urls.py**

```py
`path('friendship-requests/', FriendshipRequestsView.as_view({'post': 'create'}))` 
```

添加新的测试场景来测试接受和拒绝操作。

**例子/特色/友谊.特色**

```py
`Scenario: A user can accept a friendship request

  Given I empty the "Friendship" table

  And I create the following friendships:
    | id | user1 | user2 | status  |
    | 1  | 2     | 1     | PENDING |

  When I accept the friendship request with ID "1"

  Then I see the following response data:
    | id | user1 | user2 | status   |
    | 1  | 2     | 1     | ACCEPTED |

Scenario: A user can reject a friendship request

  Given I empty the "Friendship" table

  And I create the following friendships:
    | id | user1 | user2 | status  |
    | 1  | 2     | 1     | PENDING |

  When I reject the friendship request with ID "1"

  Then I see the following response data:
    | id | user1 | user2 | status   |
    | 1  | 2     | 1     | REJECTED |` 
```

添加新的测试步骤。

**示例/功能/友谊 _ 步骤. py**

```py
`@step('I accept the friendship request with ID "([^"]+)"')
def step_accept_friendship_request(self, pk):
    world.response = world.client.put(f'/friendship-requests/{pk}/', data={
      'status': Friendship.ACCEPTED
    })

@step('I reject the friendship request with ID "([^"]+)"')
def step_reject_friendship_request(self, pk):
    world.response = world.client.put(f'/friendship-requests/{pk}/', data={
      'status': Friendship.REJECTED
    })` 
```

再添加一个 URL 路径。用户需要针对他们想要接受或拒绝的特定友谊。

**example/urls.py**

```py
`path('friendship-requests/<int:pk>/', FriendshipRequestsView.as_view({'put': 'partial_update'}))` 
```

更新`Scenario: A user can see a list of friends`以包含新的`status`字段。

**例子/特色/友谊.特色**

```py
`Scenario: A user can see a list of friends

  Given I empty the "Friendship" table

  And I create the following friendships:
    | id | user1 | user2 | status   |
    | 1  | 1     | 2     | ACCEPTED |

  # Annie and Brian are now friends.

  When I get a list of friends

  Then I see the following response data:
    | id | email             | username |
    | 2  | [[email protected]](/cdn-cgi/l/email-protection) | Brian    |` 
```

在`Scenario: A user can see a list of friends`之后再添加一个场景来测试状态过滤。用户的朋友包括已经接受用户的友谊请求的人。没有采取行动或拒绝请求的人不予考虑。

**例子/特色/友谊.特色**

```py
`Scenario: A user with no accepted friendship requests sees an empty list

  Given I empty the "Friendship" table

  And I create the following friendships:
    | id | user1 | user2 | status   |
    | 1  | 1     | 2     | PENDING  |
    | 2  | 1     | 3     | REJECTED |

  When I get a list of friends

  Then I see the following response data:
    | id | email | username |` 
```

编辑`step_create_friendships()`函数，在`Friendship`模型上实现`status`字段。

**示例/功能/友谊 _ 步骤. py**

```py
`@step('I create the following friendships:')
def step_create_friendships(self):
    Friendship.objects.bulk_create([
        Friendship(
            id=data['id'],
            user1=User.objects.get(id=data['user1']),
            user2=User.objects.get(id=data['user2']),
            status=data['status']
        ) for data in guess_types(self.hashes)
    ])` 
```

并编辑`FriendshipSerializer`序列化器来实现`Friendship`模型上的状态字段。

**example/serializer . py**

```py
`class FriendshipSerializer(serializers.ModelSerializer):
    class Meta:
        model = Friendship
        fields = ('id', 'user1', 'user2', 'status',)
        read_only_fields = ('id',)` 
```

通过调整管理器上的`friends()`方法完成过滤。

**example/models.py**

```py
`def friends(self, user):
    """Get all users that are friends with the specified user."""
    friendships = self.friendships(user).filter(status=Friendship.ACCEPTED)

    def other_user(friendship):
        if friendship.user1 == user:
            return friendship.user2
        return friendship.user1

    return map(other_user, friendships)` 
```

功能完成！

## 结论

如果你从这篇文章中得到了什么，我希望是这个:行为驱动的开发不仅是关于编写、测试和设计代码，也是关于特性分析。没有这个关键的步骤，你就不是在创造软件，你只是在编程。BDD 不是生产软件的唯一方法，但它是一个很好的方法。如果你正在用 Django 项目实践 BDD，试试芦荟吧。

从[回购](https://github.com/testdrivenio/django-aloe-bdd)中抓取代码。