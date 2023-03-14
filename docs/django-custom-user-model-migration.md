# 在 Django 项目中期迁移到自定义用户模型

> 原文：<https://testdriven.io/blog/django-custom-user-model-migration/>

本文着眼于如何在 Django 项目中期迁移到定制用户模型。

## 自定义用户模型

Django 的默认[用户模型](https://docs.djangoproject.com/en/4.1/ref/contrib/auth/#user-model)带有相对较少的字段。因为这些字段对于所有用例来说都是不够的，所以很多 Django 项目转向定制用户模型。

在迁移数据库之前，切换到定制用户模型很容易，但是之后会变得更加困难，因为这会影响外键、多对多关系和迁移等等。

为了避免经历这个繁琐的迁移过程，Django 的官方文档[强烈建议您在项目](https://docs.djangoproject.com/en/4.1/topics/auth/customizing/#using-a-custom-user-model-when-starting-a-project)开始时建立一个定制的用户模型，即使默认模型已经足够了。

直到今天，仍然没有在项目中期迁移到定制用户模型的官方方法。Django 社区仍然在讨论什么是最好的迁移方式。

在本文中，我们将研究一种在项目中期迁移到定制用户模型的相对简单的方法。我们将要使用的迁移过程不像互联网上的其他迁移过程那样具有破坏性，并且不需要任何原始 SQL 执行或手动修改迁移。

> 有关在项目开始时创建定制用户模型的更多信息，请查看 Django 文章中的[创建定制用户模型。](/blog/django-custom-user-model/)

## 虚拟项目

在项目中期迁移到定制用户模型是一个潜在的破坏性行为。因此，我准备了一个虚拟项目，您可以在进入实际代码库之前使用它来测试迁移过程。

> 如果您想使用自己的代码库，可以跳过这一部分。

我们将要使用的虚拟项目叫做 [django-custom-user](https://github.com/duplxey/django-custom-user/tree/base) 。这是一个简单的利用用户模型的 todo 应用程序。

克隆下来:

```py
`$ git clone --single-branch --branch base [[email protected]](/cdn-cgi/l/email-protection):duplxey/django-custom-user.git
$ cd django-custom-user` 
```

创建新的虚拟环境并激活它:

```py
`$ python3 -m venv venv && source venv/bin/activate` 
```

安装要求:

```py
`(venv)$ pip install -r requirements.txt` 
```

旋转 Postgres Docker 容器:

```py
`$ docker run --name django-todo-postgres -p 5432:5432 \
    -e POSTGRES_USER=django-todo -e POSTGRES_PASSWORD=complexpassword123 \
    -e POSTGRES_DB=django-todo -d postgres` 
```

> 或者，如果你愿意，你可以在 Docker 之外安装并运行 Postgres。只要确保转到 *core/settings.py* 并相应地更改`DATABASES`凭证。

迁移数据库:

```py
`(venv)$ python manage.py migrate` 
```

装载夹具:

```py
`(venv)$ python manage.py loaddata fixtures/auth.json --app auth
(venv)$ python manage.py loaddata fixtures/todo.json --app todo` 
```

这两个设备向数据库添加了一些用户、组和任务，并创建了一个具有以下凭证的超级用户:

```py
`username:  admin password:  password` 
```

接下来，运行服务器:

```py
`(venv)$ python manage.py runserver` 
```

最后，导航到位于[http://localhost:8000/admin](http://localhost:8000/admin)的管理面板，以超级用户身份登录，并确保数据已经成功加载。

## 迁移过程

我们将要使用的迁移过程假设:

1.  您的项目还没有自定义用户模型。
2.  您已经创建并迁移了数据库。
3.  没有挂起的迁移，所有现有迁移都已应用。
4.  你不想丢失任何数据。

> 如果您仍处于开发阶段，并且数据库中的数据不重要，则不必遵循这些步骤。要迁移到定制用户模型，您可以简单地擦除数据库，删除所有迁移文件，然后按照这里的步骤[执行](/blog/django-custom-user-model/)。

在继续之前，请完全备份您的数据库(和代码库)。在转移到生产环境之前，您还应该在分段分支/环境中尝试这些步骤。

### 迁移步骤

1.  将`AUTH_USER_MODEL`指向 *settings.py* 中默认的 Django 用户。
2.  相应地用`AUTH_USER_MODEL`或`get_user_model()`替换所有的`User`参考。
3.  启动一个新的 Django app，在 *settings.py* 中注册。
4.  在新创建的应用程序中创建一个空迁移。
5.  迁移数据库，以便应用空迁移。
6.  删除空的迁移文件。
7.  在新创建的应用程序中创建自定义用户模型。
8.  将`DJANGO_USER_MODEL`指向自定义用户。
9.  运行 makemigrations。

我们开始吧！

#### 第一步

要迁移到定制用户模型，我们首先需要去掉所有直接的`User`引用。为此，首先在*设置. py* 中添加一个名为`AUTH_USER_MODEL`的新属性，如下所示:

```py
`# core/settings.py

AUTH_USER_MODEL = 'auth.User'` 
```

这个属性告诉 Django 使用什么用户模型。因为我们还没有定制的用户模型，所以我们将它指向默认的 Django 用户模型。

#### 第二步

接下来，检查您的整个代码库，确保相应地用 [AUTH_USER_MODEL](https://docs.djangoproject.com/en/4.1/topics/auth/customizing/#substituting-a-custom-user-model) 或 [get_user_model()](https://docs.djangoproject.com/en/4.1/topics/auth/customizing/#django.contrib.auth.get_user_model) 替换所有的`User`引用:

```py
`# todo/models.py

class UserTask(GenericTask):
    user = models.ForeignKey(
        to=AUTH_USER_MODEL,
        on_delete=models.CASCADE
    )

    def __str__(self):
        return f'UserTask {self.id}'

class GroupTask(GenericTask):
    users = models.ManyToManyField(
        to=AUTH_USER_MODEL
    )

    def __str__(self):
        return f'GroupTask {self.id}'` 
```

不要忘记在文件顶部导入`AUTH_USER_MODEL`:

```py
`from core.settings import AUTH_USER_MODEL` 
```

> 还要确保你使用的所有第三方应用程序/包都是这样做的。如果他们中的任何一个直接引用了`User`模型，事情可能会被打破。你不必担心这个，因为大多数利用`User`模型的流行包并不直接引用它。

#### 第三步

接下来，我们需要启动一个新的 Django 应用程序，它将托管自定义用户模型。

我称它为*用户*，但是你可以选择一个不同的名字:

```py
`(venv)$ python manage.py startapp users` 
```

> 如果你愿意，你可以重用一个已经存在的应用程序，但是你需要确保这个应用程序中还没有迁移；否则，由于 Django 的限制，迁移过程将无法进行。

在 *settings.py* 中注册 app:

```py
`# core/settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'todo.apps.TodoConfig',
    'users.apps.UsersConfig',  # new
]` 
```

#### 第四步

接下来，我们需要欺骗 Django，让他认为*用户*应用程序负责`auth_user`表。这通常可以用 migrate 命令和`--fake`标志来完成，但在这种情况下不行，因为我们会遇到`InconsistentMigrationHistory`，因为大多数迁移都依赖于`auth`迁移。

无论如何，要绕过这一点，我们可以使用一个 hacky 的工作区。首先，我们将创建一个空迁移，应用它以便保存到`django_migrations`表中，然后与实际的`auth_user`迁移交换。

在*用户*应用程序中创建一个空迁移:

```py
`(venv)$ python manage.py makemigrations --empty users

Migrations for 'users':
  users\migrations\0001_initial.py` 
```

这将创建一个名为*users/migrations/0001 _ initial . py*的空迁移。

#### 第五步

迁移数据库，以便将空迁移添加到`django_migrations`表中:

```py
`(venv)$ python manage.py migrate

Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions, todo, users
Running migrations:
  Applying users.0001_initial... OK` 
```

#### 第六步

现在，删除空的迁移文件:

```py
`(venv)$ rm users/migrations/0001_initial.py` 
```

#### 第七步

转到 *users/models.py* ，定义自定义`User`模型，如下所示:

```py
`# users/models.py

from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    class Meta:
        db_table = 'auth_user'` 
```

暂时不要添加任何自定义字段。这个模型必须是 Django 默认用户模型的直接副本，因为我们将使用它来创建初始的`auth_user`表迁移。

此外，确保将其命名为`User`，否则您可能会因为[内容类型](https://docs.djangoproject.com/en/4.1/ref/contrib/contenttypes/)而遇到问题。您稍后可以更改模型的名称。

#### 第八步

导航到您的 *settings.py* ，将`AUTH_USER_MODEL`指向刚刚创建的定制用户模型:

```py
`# core/settings.py

AUTH_USER_MODEL = 'users.User'` 
```

> 如果你的应用不叫 *users* 一定要换一个。

#### 第九步

运行`makemigrations`生成初始`auth_user`迁移:

```py
`(venv)$ python manage.py makemigrations

Migrations for 'users':
  users\migrations\0001_initial.py
    - Create model User` 
```

就是这样！当您第一次通过 Django 的`auth`应用程序运行 migrate 时，已经应用了生成的迁移，所以再次运行 migrate 不会做任何事情。

## 添加新字段

一旦建立了自定义用户模型，添加新字段就很容易了。

例如，要添加一个`phone`和`address`字段，请将以下内容添加到自定义用户模型中:

```py
`# users/models.py

class User(AbstractUser):
    phone = models.CharField(max_length=32, blank=True, null=True)    # new
    address = models.CharField(max_length=64, blank=True, null=True)  # new

    class Meta:
        db_table = 'auth_user'` 
```

不要忘记在文件顶部导入`models`:

```py
`from django.db import models` 
```

接下来，进行迁移并迁移:

```py
`(venv)$ python manage.py makemigrations
(venv)$ python manage.py migrate` 
```

要确保这些字段已经在数据库中得到反映，请将它们放入 Docker 容器:

```py
`$ docker exec -it django-todo-postgres bash` 
```

通过`psql`连接到数据库:

```py
`[[email protected]](/cdn-cgi/l/email-protection):/# psql -U django-todo

psql (14.5 (Debian 14.5-1.pgdg110+1))
Type "help" for help.` 
```

并检查`auth_user`表:

```py
`django-todo=# \d+ auth_user

                                                                Table "public.auth_user"
    Column    |           Type           | Collation | Nullable |             Default              | Storage  | Compression | Stats target | Description
--------------+--------------------------+-----------+----------+----------------------------------+----------+-------------+--------------+-------------
 id           | integer                  |           | not null | generated by default as identity | plain    |             |              |
 password     | character varying(128)   |           | not null |                                  | extended |             |              |
 last_login   | timestamp with time zone |           |          |                                  | plain    |             |              |
 is_superuser | boolean                  |           | not null |                                  | plain    |             |              |
 username     | character varying(150)   |           | not null |                                  | extended |             |              |
 first_name   | character varying(150)   |           | not null |                                  | extended |             |              |
 last_name    | character varying(150)   |           | not null |                                  | extended |             |              |
 email        | character varying(254)   |           | not null |                                  | extended |             |              |
 is_staff     | boolean                  |           | not null |                                  | plain    |             |              |
 is_active    | boolean                  |           | not null |                                  | plain    |             |              |
 date_joined  | timestamp with time zone |           | not null |                                  | plain    |             |              |
 phone        | character varying(32)    |           |          |                                  | extended |             |              |
 address      | character varying(64)    |           |          |                                  | extended |             |              |` 
```

您可以看到已经添加了名为`phone`和`address`的新字段。

## Django 管理

要在 Django 管理面板中显示自定义用户模型，首先需要创建一个从`UserAdmin`继承的新类，然后注册它。接下来，在字段集中包含`phone`和`address`。

最终的*用户/管理员副本*应该如下所示:

```py
`# users/admin.py

from django.contrib import admin
from django.contrib.auth.admin import UserAdmin

from users.models import User

class CustomUserAdmin(UserAdmin):
    fieldsets = UserAdmin.fieldsets + (
        ('Additional info', {'fields': ('phone', 'address')}),
    )

admin.site.register(User, CustomUserAdmin)` 
```

再次运行服务器，登录，并导航到一个随机用户。向下滚动到底部，您应该会看到一个包含新字段的新部分。

> 如果您希望进一步定制 Django 管理，请从官方文档中查看 Django 管理站点。

## 重命名用户表/模型

此时，您可以像平常一样重命名用户模型和表。

要重命名用户模型，只需更改类名，要重命名表，只需更改`db_table`属性:

```py
`# users/models.py

class User(AbstractUser):  # <-- you can change me
    phone = models.CharField(max_length=32, blank=True, null=True)
    address = models.CharField(max_length=64, blank=True, null=True)

    class Meta:
        db_table = 'auth_user'  # <-- you can change me` 
```

如果您删除了`db_table`属性，表名将返回到`<app_name>_<model_name>`。

完成更改后，运行:

```py
`(venv)$ python manage.py makemigrations
(venv)$ python manage.py migrate` 
```

我一般不建议重命名任何东西，因为你的数据库结构会变得不一致。有些表格会有前缀`users_`，而有些表格会有前缀`auth_`。但另一方面，你可以争辩说`User`模式现在是*用户*应用的一部分，所以它不应该有`auth_`前缀。

如果您决定重命名该表，最终的数据库结构将如下所示:

```py
`django-todo=# \dt

                     List of relations
 Schema |            Name             | Type  |    Owner
--------+-----------------------------+-------+-------------
 public | auth_group                  | table | django-todo
 public | auth_group_permissions      | table | django-todo
 public | auth_permission             | table | django-todo
 public | django_admin_log            | table | django-todo
 public | django_content_type         | table | django-todo
 public | django_migrations           | table | django-todo
 public | django_session              | table | django-todo
 public | todo_task                   | table | django-todo
 public | todo_task_categories        | table | django-todo
 public | todo_taskcategory           | table | django-todo
 public | users_user                  | table | django-todo
 public | users_user_groups           | table | django-todo
 public | users_user_user_permissions | table | django-todo` 
```

## 结论

尽管在项目中期迁移到自定义用户模型的问题已经存在了很长时间，但是仍然没有官方的解决方案。

不幸的是，许多 Django 开发人员不得不经历这个迁移过程，因为 Django 文档没有充分强调您应该在项目开始时创建一个定制的用户模型。也许他们甚至可以把它包括在教程里？

希望我在文章中介绍的迁移过程对您没有任何问题。万一有些事情对你不起作用，或者你认为有些事情可以改进，我希望听到你的反馈。

您可以从 [django-custom-user](https://github.com/duplxey/django-custom-user) repo 中获得最终的源代码。