# Django 的权限

> 原文：<https://testdriven.io/blog/django-permissions/>

Django 自带强大的[权限](https://docs.djangoproject.com/en/4.0/topics/auth/default/#permissions-and-authorization)系统。

在本文中，我们将了解如何为用户和组分配权限，以便授权他们执行特定的操作。

## 目标

完成本文后，您将能够:

1.  解释 Django 的权限和组是如何工作的
2.  利用 Django 内置许可系统的力量

## 身份验证与授权

这篇文章是关于授权的。

*   **认证**是确认用户是否有权访问系统的过程。通常，用户名/电子邮件和密码用于验证用户。
*   **授权:**与“认证”用户在系统中可以做什么有关。

换句话说，认证回答了“你是谁？”而授权回答‘你能做什么？’。

## 用户级权限

当`django.contrib.auth`被添加到 *settings.py* 文件中的`INSTALLED_APPS`设置中时，Django [自动](https://docs.djangoproject.com/en/4.0/topics/auth/default/#default-permissions)为创建的每个 Django 模型创建`add`、`change`、`delete`和`view`权限。

Django 中的权限遵循以下命名顺序:

`{app}.{action}_{model_name}`

注意事项:

*   `app`是相关模型所在的 Django 应用的名称
*   `action`:是`add`、`change`、`delete`还是`view`
*   `model_name`:小写的型号名称

让我们假设我们在一个名为“博客”的应用程序中有以下模型:

```
`from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=400)
    body = models.TextField()` 
```

默认情况下，Django 将创建以下权限:

1.  `blog.add_post`
2.  `blog.change_post`
3.  `blog.delete_post`
4.  `blog.view_post`

然后，您可以检查用户(通过 Django 用户对象)是否拥有使用`has_perm()`方法的权限:

```
`from django.contrib.auth import get_user_model
from django.contrib.auth.models import User, Permission
from django.contrib.contenttypes.models import ContentType

from blog.models import Post

content_type = ContentType.objects.get_for_model(Post)
post_permission = Permission.objects.filter(content_type=content_type)
print([perm.codename for perm in post_permission])
# => ['add_post', 'change_post', 'delete_post', 'view_post']

user = User.objects.create_user(username="test", password="test", email="[[email protected]](/cdn-cgi/l/email-protection)")

# Check if the user has permissions already
print(user.has_perm("blog.view_post"))
# => False

# To add permissions
for perm in post_permission:
    user.user_permissions.add(perm)

print(user.has_perm("blog.view_post"))
# => False
# Why? This is because Django's permissions do not take
# effect until you allocate a new instance of the user.

user = get_user_model().objects.get(email="[[email protected]](/cdn-cgi/l/email-protection)")
print(user.has_perm("blog.view_post"))
# => True` 
```

超级用户将始终拥有设置为`True`的权限，即使该权限不存在:

```
`from django.contrib.auth.models import User

superuser = User.objects.create_superuser(
    username="super", password="test", email="[[email protected]](/cdn-cgi/l/email-protection)"
)

# Output will be true
print(superuser.has_perm("blog.view_post"))

# Output will be true even if the permission does not exists
print(superuser.has_perm("foo.add_bar"))` 
```

> 超级用户是 Django 中拥有系统中所有权限的用户类型。无论是自定义权限还是 Django 创建的权限，超级用户都可以访问每一个权限。
> 
> staff 用户就像系统中的任何其他用户一样，但是具有能够访问 Django 管理界面的额外优势。Django 管理界面只对超级用户和职员用户开放。

## 组级权限

每次都必须给用户分配权限，这很繁琐，而且不可伸缩。在某些情况下，您可能希望向一组用户添加新的权限。这里是 Django 组织发挥作用的地方。

什么是团体？

*   英语定义:组是被分类在一起的一组对象。
*   Django 定义:组模型是一种对用户进行分类的通用方法，因此您可以对这些用户应用权限或其他标签。用户可以属于任意数量的组。

使用 Django，您可以创建组来对用户进行分类，并为每个组分配权限，因此在创建用户时，您可以将用户分配到一个组，反过来，用户拥有该组的所有权限。

要创建一个组，您需要来自`django.contrib.auth.models`的`Group`模型。

让我们为以下角色创建组:

*   `Author`:可以查看和添加帖子
*   `Editor`:可以查看、添加、编辑帖子
*   `Publisher`:可以查看、添加、编辑、删除帖子

代码:

```
`from django.contrib.auth.models import Group, User, Permission
from django.contrib.contenttypes.models import ContentType
from django.shortcuts import get_object_or_404

from blog.models import Post

author_group, created = Group.objects.get_or_create(name="Author")
editor_group, created = Group.objects.get_or_create(name="Editor")
publisher_group, created = Group.objects.get_or_create(name="Publisher")

content_type = ContentType.objects.get_for_model(Post)
post_permission = Permission.objects.filter(content_type=content_type)
print([perm.codename for perm in post_permission])
# => ['add_post', 'change_post', 'delete_post', 'view_post']

for perm in post_permission:
    if perm.codename == "delete_post":
        publisher_group.permissions.add(perm)

    elif perm.codename == "change_post":
        editor_group.permissions.add(perm)
        publisher_group.permissions.add(perm)
    else:
        author_group.permissions.add(perm)
        editor_group.permissions.add(perm)
        publisher_group.permissions.add(perm)

user = User.objects.get(username="test")
user.groups.add(author_group)  # Add the user to the Author group

user = get_object_or_404(User, pk=user.id)

print(user.has_perm("blog.delete_post")) # => False
print(user.has_perm("blog.change_post")) # => False
print(user.has_perm("blog.view_post")) # => True
print(user.has_perm("blog.add_post")) # => True` 
```

## 强制权限

除了 Django Admin 之外，权限通常是在视图层执行的，因为用户是从请求对象获得的。

要在基于类的视图中实施权限，您可以使用来自`django.contrib.auth.mixins`的[permissionrequiredminix](https://docs.djangoproject.com/en/4.0/topics/auth/default/#django.contrib.auth.mixins.PermissionRequiredMixin)，如下所示:

```
`from django.contrib.auth.mixins import PermissionRequiredMixin
from django.views.generic import ListView

from blog.models import Post

class PostListView(PermissionRequiredMixin, ListView):
    permission_required = "blog.view_post"
    template_name = "post.html"
    model = Post` 
```

`permission_required`可以是单一权限，也可以是可重复的权限。如果使用 iterable，用户必须拥有所有权限才能访问视图:

```
`from django.contrib.auth.mixins import PermissionRequiredMixin
from django.views.generic import ListView

from blog.models import Post

class PostListView(PermissionRequiredMixin, ListView):
    permission_required = ("blog.view_post", "blog.add_post")
    template_name = "post.html"
    model = Post` 
```

对于基于函数的视图，使用`permission_required`装饰器:

```
`from django.contrib.auth.decorators import permission_required

@permission_required("blog.view_post")
def post_list_view(request):
    return HttpResponse()` 
```

您还可以在 Django 模板中检查权限。使用 Django 的 auth context 处理器，当你渲染你的模板时，一个 [perms](https://docs.djangoproject.com/en/4.0/topics/auth/default/#permissions) 变量是默认可用的。`perms`变量实际上包含了 Django 应用程序中的所有权限。

例如:

```
`{% if perms.blog.view_post %}
  {# Your content here #}
{% endif %}` 
```

## 模型级权限

您还可以通过 [model Meta](https://docs.djangoproject.com/en/4.0/ref/models/options/) 选项向 Django 模型添加定制权限。

让我们给`Post`模型添加一个`is_published`标志:

```
`from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=400)
    body = models.TextField()
    is_published = models.Boolean(default=False)` 
```

接下来，我们将设置一个名为`set_published_status`的自定义权限:

```
`from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=400)
    body = models.TextField()
    is_published = models.Boolean(default=False)

    class Meta:
        permissions = [
            (
                "set_published_status",
                "Can set the status of the post to either publish or not"
            )
        ]` 
```

为了实施这个权限，我们可以在视图中使用 Django 提供的 mixin，这给了我们显式检查用户是否拥有所需权限的灵活性。

下面是一个基于类的视图，它检查用户是否有权限设置帖子的发布状态:

```
`from django.contrib.auth.mixins import UserPassesTestMixin
from django.shortcuts import render
from django.views.generic import View

from blog.models import Post

class PostListView(UserPassesTestMixin, View):
    template_name = "post_details.html"

    def test_func(self):
        return self.request.user.has_perm("blog.set_published_status")

    def post(self, request, *args, **kwargs):
        post_id = request.POST.get('post_id')
        published_status = request.POST.get('published_status')

        if post_id:
            post = Post.objects.get(pk=post_id)
            post.is_published = bool(published_status)
            post.save()

        return render(request, self.template_name)` 
```

因此，使用`UserPassesTestMixin`，您需要覆盖该类的`test_func`方法并添加您自己的测试。请注意，该方法的返回值必须始终是布尔值。

## 对象级权限

如果您使用的是 Django REST 框架，它已经将[对象级权限](https://www.django-rest-framework.org/api-guide/permissions/#object-level-permissions)内置到基本权限类中。`BasePermission`有`has_permission`，主要用于列表视图，还有`has_object_permission`，用于检查用户是否有权限访问单个模型实例。

> 有关 Django REST 框架中权限的更多信息，请查看 Django REST 框架中的[权限。](/blog/drf-permissions/)

如果您没有使用 Django REST 框架，要实现对象级权限，您可以使用第三方，比如:

> 更多与权限相关的包，请查看 [Django 包](https://djangopackages.org/grids/g/perms/)。

## 结论

在本文中，您已经学习了如何向 Django 模型添加权限并检查权限。如果您有一定数量的用户类型，您可以将每个用户类型创建为一个组，并向该组授予必要的权限。然后，对于添加到系统和所需组中的每个用户，权限会自动授予每个用户。