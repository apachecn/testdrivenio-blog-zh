# 有效地使用 Django REST 框架序列化程序

> 原文：<https://testdriven.io/blog/drf-serializers/>

在本文中，我们将通过例子来看看如何更有效地使用 [Django REST 框架](https://www.django-rest-framework.org/) (DRF)序列化器。一路上，我们将深入一些高级概念，比如使用`source`关键字、传递上下文、验证数据等等。

> 本文假设您已经对 Django REST 框架有了相当的了解。

## 包括什么？

本文涵盖:

1.  在字段或对象级别验证数据
2.  自定义序列化和反序列化输出
3.  保存时传递附加数据
4.  将上下文传递给序列化程序
5.  重命名序列化程序输出字段
6.  将序列化程序函数响应附加到数据
7.  从一对一模型中获取数据
8.  将数据附加到序列化输出
9.  创建单独的读写序列化程序
10.  设置只读字段
11.  处理嵌套序列化

> 本文中介绍的概念相互之间没有联系。我建议把这篇文章作为一个整体来读，但是你也可以自由地钻研你特别感兴趣的概念。

## 自定义数据验证

DRF 在反序列化过程中强制执行数据验证，这就是为什么您需要在访问经过验证的数据之前调用`is_valid()`。如果数据无效，错误将被追加到序列化程序的`error`属性中，并抛出一个`ValidationError`。

有两种类型的自定义数据验证器:

1.  自定义字段
2.  对象级

让我们看一个例子。假设我们有一个`Movie`模型:

```
`from django.db import models

class Movie(models.Model):
    title = models.CharField(max_length=128)
    description = models.TextField(max_length=2048)
    release_date = models.DateField()
    rating = models.PositiveSmallIntegerField()

    us_gross = models.IntegerField(default=0)
    worldwide_gross = models.IntegerField(default=0)

    def __str__(self):
        return f'{self.title}'` 
```

我们的型号有`title`、`description`、`release_date`、`rating`、`us_gross`和`worldwide_gross`。

我们还有一个简单的`ModelSerializer`，它序列化所有的字段:

```
`from rest_framework import serializers
from examples.models import Movie

class MovieSerializer(serializers.ModelSerializer):
    class Meta:
        model = Movie
        fields = '__all__'` 
```

假设只有当这两个条件都成立时，模型才有效:

1.  `rating`介于 1 和 10 之间
2.  `us_gross`小于`worldwide_gross`

我们可以为此使用定制的数据验证器。

### 自定义字段验证

自定义字段验证允许我们验证特定的字段。我们可以通过向序列化程序添加`validate_<field_name>`方法来使用它，如下所示:

```
`from rest_framework import serializers
from examples.models import Movie

class MovieSerializer(serializers.ModelSerializer):
    class Meta:
        model = Movie
        fields = '__all__'

    def validate_rating(self, value):
        if value < 1 or value > 10:
            raise serializers.ValidationError('Rating has to be between 1 and 10.')
        return value` 
```

我们的`validate_rating`方法将确保评分始终保持在 1 到 10 之间。

### 对象级验证

有时，为了验证字段，您必须将它们相互比较。这时您应该使用对象级验证方法。

示例:

```
`from rest_framework import serializers
from examples.models import Movie

class MovieSerializer(serializers.ModelSerializer):
    class Meta:
        model = Movie
        fields = '__all__'

    def validate(self, data):
        if data['us_gross'] > data['worldwide_gross']:
            raise serializers.ValidationError('worldwide_gross cannot be bigger than us_gross')
        return data` 
```

`validate`方法将确保`us_gross`永远不会大于`worldwide_gross`。

> 您应该避免通过`self.initial_data`访问定制字段验证器中的附加字段。该字典包含原始数据，这意味着您的数据类型不一定匹配所需的数据类型。DRF 还会将验证错误附加到错误的字段中。

### 功能验证器

如果我们在多个序列化器中使用同一个验证器，我们可以创建一个函数验证器，而不是反复编写相同的代码。让我们编写一个验证器来检查数字是否在 1 到 10 之间:

```
`def is_rating(value):
    if value < 1:
        raise serializers.ValidationError('Value cannot be lower than 1.')
    elif value > 10:
        raise serializers.ValidationError('Value cannot be higher than 10')` 
```

我们现在可以像这样把它附加到我们的`MovieSerializer`中:

```
`from rest_framework import serializers
from examples.models import Movie

class MovieSerializer(serializers.ModelSerializer):
    rating = IntegerField(validators=[is_rating])
    ...` 
```

## 自定义输出

在`BaseSerializer`类中，我们可以覆盖的两个最有用的函数是`to_representation()`和`to_internal_value()`。通过重写它们，我们可以分别更改序列化和反序列化行为，以追加额外的数据、提取数据和处理关系。

1.  `to_representation()`允许我们改变序列化输出
2.  `to_internal_value()`允许我们更改反序列化输出

假设您有以下模型:

```
`from django.contrib.auth.models import User
from django.db import models

class Resource(models.Model):
    title = models.CharField(max_length=256)
    content = models.TextField()
    liked_by = models.ManyToManyField(to=User)

    def __str__(self):
        return f'{self.title}'` 
```

每个资源都有一个`title`、`content`和`liked_by`字段。`liked_by`代表喜欢该资源的用户。

我们的序列化程序是这样定义的:

```
`from rest_framework import serializers
from examples.models import Resource

class ResourceSerializer(serializers.ModelSerializer):
    class Meta:
        model = Resource
        fields = '__all__'` 
```

如果我们序列化一个资源并访问它的`data`属性，我们将得到以下输出:

```
`{ "id":  1, "title":  "C++ with examples", "content":  "This is the resource's content.", "liked_by":  [ 2, 3 ] }` 
```

### 至表示法()

现在，假设我们想给序列化数据添加一个总的赞数。实现这一点最简单的方法是在我们的序列化程序类中实现`to_representation`方法:

```
`from rest_framework import serializers
from examples.models import Resource

class ResourceSerializer(serializers.ModelSerializer):
    class Meta:
        model = Resource
        fields = '__all__'

    def to_representation(self, instance):
        representation = super().to_representation(instance)
        representation['likes'] = instance.liked_by.count()

        return representation` 
```

这段代码获取当前的表示，将`likes`附加到它上面，然后返回它。

如果我们序列化另一个资源，我们将得到以下结果:

```
`{ "id":  1, "title":  "C++ with examples", "content":  "This is the resource's content.", "liked_by":  [ 2, 3 ], "likes":  2 }` 
```

### 至内部值()

假设使用我们的 API 的服务在创建资源时向端点附加了不必要的数据:

```
`{ "info":  { "extra":  "data", ... }, "resource":  { "id":  1, "title":  "C++ with examples", "content":  "This is the resource's content.", "liked_by":  [ 2, 3 ], "likes":  2 } }` 
```

如果我们尝试序列化这些数据，我们的序列化程序将会失败，因为它将无法提取资源。

我们可以覆盖`to_internal_value()`来提取资源数据:

```
`from rest_framework import serializers
from examples.models import Resource

class ResourceSerializer(serializers.ModelSerializer):
    class Meta:
        model = Resource
        fields = '__all__'

    def to_internal_value(self, data):
        resource_data = data['resource']

        return super().to_internal_value(resource_data)` 
```

耶！我们的序列化程序现在可以正常工作了。

## 序列化程序保存

调用`save()`将创建一个新实例或更新一个现有实例，这取决于在实例化序列化程序类时是否传递了一个现有实例:

```
`# this creates a new instance
serializer = MySerializer(data=data)

# this updates an existing instance
serializer = MySerializer(instance, data=data)` 
```

### 将数据直接传递到保存

有时，您会希望在保存实例时传递额外的数据。这些附加数据可能包括当前用户、当前时间或请求数据等信息。

您可以通过在调用`save()`时包含额外的关键字参数来做到这一点。例如:

```
`serializer.save(owner=request.user)` 
```

> 请记住，传递给`save()`的值不会被验证。

## 序列化程序上下文

有些情况下，您需要向序列化程序传递额外的数据。您可以通过使用 serializer `context`属性来做到这一点。然后，您可以在序列化器(如`to_representation`)中或者在验证数据时使用这些数据。

您通过关键字`context`将数据作为字典传递:

```
`from rest_framework import serializers
from examples.models import Resource

resource = Resource.objects.get(id=1)
serializer = ResourceSerializer(resource, context={'key': 'value'})` 
```

然后，您可以从`self.context`字典中的序列化程序类中获取它，如下所示:

```
`from rest_framework import serializers
from examples.models import Resource

class ResourceSerializer(serializers.ModelSerializer):
    class Meta:
        model = Resource
        fields = '__all__'

    def to_representation(self, instance):
        representation = super().to_representation(instance)
        representation['key'] = self.context['key']

        return representation` 
```

我们的串行化器输出现在将包含带有`value`的`key`。

## 源关键字

DRF 序列化器附带了`source`关键字，它非常强大，可以在多种情况下使用。我们可以用它来:

1.  重命名序列化程序输出字段
2.  将序列化程序函数响应附加到数据
3.  从一对一模型中提取数据

假设您正在构建一个社交网络，每个用户都有自己的`UserProfile`，它与`User`模型有一对一的关系:

```
`from django.contrib.auth.models import User
from django.db import models

class UserProfile(models.Model):
    user = models.OneToOneField(to=User, on_delete=models.CASCADE)
    bio = models.TextField()
    birth_date = models.DateField()

    def __str__(self):
        return f'{self.user.username} profile'` 
```

我们使用一个`ModelSerializer`来序列化我们的用户:

```
`class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'is_staff', 'is_active']` 
```

让我们序列化一个用户:

```
`{ "id":  1, "username":  "admin", "email":  "[[email protected]](/cdn-cgi/l/email-protection)", "is_staff":  true, "is_active":  true }` 
```

### 重命名序列化程序输出字段

要重命名序列化程序输出字段，我们需要向序列化程序添加一个新字段，并将其传递给`fields`属性。

```
`class UserSerializer(serializers.ModelSerializer):
    active = serializers.BooleanField(source='is_active')

    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'is_staff', 'active']` 
```

我们的活动字段现在将被命名为`active`而不是`is_active`。

### 将序列化程序函数响应附加到数据

我们可以使用`source`添加一个等于函数返回的字段。

```
`class UserSerializer(serializers.ModelSerializer):
    full_name = serializers.CharField(source='get_full_name')

    class Meta:
        model = User
        fields = ['id', 'username', 'full_name', 'email', 'is_staff', 'active']` 
```

> `get_full_name()`是 Django 用户模型中的一个方法，它连接了`user.first_name`和`user.last_name`。

我们的响应现在将包含`full_name`。

### 从一对一模型追加数据

现在让我们假设我们也想在`UserSerializer`中包含用户的`bio`和`birth_date`。我们可以通过使用 source 关键字向序列化程序添加额外的字段来做到这一点。

让我们修改序列化程序类:

```
`class UserSerializer(serializers.ModelSerializer):
    bio = serializers.CharField(source='userprofile.bio')
    birth_date = serializers.DateField(source='userprofile.birth_date')

    class Meta:
        model = User
        fields = [
            'id', 'username', 'email', 'is_staff',
            'is_active', 'bio', 'birth_date'
        ]  # note we also added the new fields here` 
```

我们可以访问`userprofile.<field_name>`，因为它与我们的用户是一对一的关系。

这是我们最终的 JSON 回应:

```
`{ "id":  1, "username":  "admin", "email":  "", "is_staff":  true, "is_active":  true, "bio":  "This is my bio.", "birth_date":  "1995-04-27" }` 
```

## SerializerMethodField

`SerializerMethodField`是一个只读字段，它通过调用它所附加到的序列化程序类上的方法来获取其值。它可用于将任何类型的数据附加到对象的序列化表示中。

`SerializerMethodField`通过调用`get_<field_name>`获取其数据。

如果我们想给我们的`User`序列化器添加一个`full_name`属性，我们可以这样实现:

```
`from django.contrib.auth.models import User
from rest_framework import serializers

class UserSerializer(serializers.ModelSerializer):
    full_name = serializers.SerializerMethodField()

    class Meta:
        model = User
        fields = '__all__'

    def get_full_name(self, obj):
        return f'{obj.first_name}  {obj.last_name}'` 
```

这段代码创建了一个用户序列化器，它也包含了`get_full_name()`函数的结果`full_name`。

## 不同的读写序列化程序

如果您的序列化程序包含大量的嵌套数据，这对于写操作不是必需的，您可以通过创建单独的读和写序列化程序来提高 API 性能。

您可以这样做，在您的`ViewSet`中覆盖`get_serializer_class()`方法，如下所示:

```
`from rest_framework import viewsets

from .models import MyModel
from .serializers import MyModelWriteSerializer, MyModelReadSerializer

class MyViewSet(viewsets.ModelViewSet):
    queryset = MyModel.objects.all()

    def get_serializer_class(self):
        if self.action in ["create", "update", "partial_update", "destroy"]:
            return MyModelWriteSerializer

        return MyModelReadSerializer` 
```

这段代码检查使用了什么 REST 操作，并为写操作返回`MyModelWriteSerializer`,为读操作返回`MyModelReadSerializer`。

## 只读字段

序列化器字段带有`read_only`选项。通过将它设置为`True`，DRF 在 API 输出中包含该字段，但是在创建和更新操作中忽略它:

```
`from rest_framework import serializers

class AccountSerializer(serializers.Serializer):
    id = IntegerField(label='ID', read_only=True)
    username = CharField(max_length=32, required=True)` 
```

> 设置`id`、`create_date`等字段。只读将会在写入操作时提高性能。

如果您想将多个字段设置为`read_only`，您可以使用`Meta`中的`read_only_fields`来指定它们，如下所示:

```
`from rest_framework import serializers

class AccountSerializer(serializers.Serializer):
    id = IntegerField(label='ID')
    username = CharField(max_length=32, required=True)

    class Meta:
        read_only_fields = ['id', 'username']` 
```

## 嵌套序列化程序

用`ModelSerializer`处理嵌套序列化有两种不同的方式:

1.  明确定义
2.  使用`depth`字段

### 明确定义

显式定义的工作方式是将一个外部的`Serializer`作为一个字段传递给我们的主序列化程序。

让我们看一个例子。我们有一个`Comment`，它是这样定义的:

```
`from django.contrib.auth.models import User
from django.db import models

class Comment(models.Model):
    author = models.ForeignKey(to=User, on_delete=models.CASCADE)
    datetime = models.DateTimeField(auto_now_add=True)
    content = models.TextField()` 
```

假设您有以下序列化程序:

```
`from rest_framework import serializers

class CommentSerializer(serializers.ModelSerializer):
    author = UserSerializer()

    class Meta:
        model = Comment
        fields = '__all__'` 
```

如果我们序列化一个`Comment`，你会得到如下输出:

```
`{ "id":  1, "datetime":  "2021-03-19T21:51:44.775609Z", "content":  "This is an interesting message.", "author":  1 }` 
```

如果我们还想序列化用户(而不是只显示他们的 ID)，我们可以向我们的`Comment`添加一个`author`序列化器字段:

```
`from rest_framework import serializers

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username']

class CommentSerializer(serializers.ModelSerializer):
    author = UserSerializer()

    class Meta:
        model = Comment
        fields = '__all__'` 
```

再次连载，你会得到这个:

```
`{ "id":  1, "author":  { "id":  1, "username":  "admin" }, "datetime":  "2021-03-19T21:51:44.775609Z", "content":  "This is an interesting message." }` 
```

### 使用深度场

谈到嵌套序列化，`depth`字段是最强大的特性之一。假设我们有三个模型- `ModelA`、`ModelB`和`ModelC`。`ModelA`取决于`ModelB`，而`ModelB`取决于`ModelC`。它们是这样定义的:

```
`from django.db import models

class ModelC(models.Model):
    content = models.CharField(max_length=128)

class ModelB(models.Model):
    model_c = models.ForeignKey(to=ModelC, on_delete=models.CASCADE)
    content = models.CharField(max_length=128)

class ModelA(models.Model):
    model_b = models.ForeignKey(to=ModelB, on_delete=models.CASCADE)
    content = models.CharField(max_length=128)` 
```

我们的`ModelA`序列化器是顶级对象，看起来像这样:

```
`from rest_framework import serializers

class ModelASerializer(serializers.ModelSerializer):
    class Meta:
        model = ModelA
        fields = '__all__'` 
```

如果我们序列化一个示例对象，我们将得到以下输出:

```
`{ "id":  1, "content":  "A content", "model_b":  1 }` 
```

现在假设我们也想在序列化`ModelA`时包含`ModelB`的内容。我们可以将显式定义添加到我们的`ModelASerializer`中，或者使用`depth`字段。

当我们在序列化器中将`depth`改为`1`时，如下所示:

```
`from rest_framework import serializers

class ModelASerializer(serializers.ModelSerializer):
    class Meta:
        model = ModelA
        fields = '__all__'
        depth = 1` 
```

输出更改为以下内容:

```
`{ "id":  1, "content":  "A content", "model_b":  { "id":  1, "content":  "B content", "model_c":  1 } }` 
```

如果我们将其更改为`2`，我们的序列化程序将更深入地序列化:

```
`{ "id":  1, "content":  "A content", "model_b":  { "id":  1, "content":  "B content", "model_c":  { "id":  1, "content":  "C content" } } }` 
```

> 缺点是你无法控制孩子的序列化。换句话说，使用`depth`将包括子节点上的所有字段。

## 结论

在本文中，您了解了许多更有效地使用 DRF 序列化程序的技巧和诀窍。

总结我们具体涉及的内容:

| 概念 | 方法 |
| --- | --- |
| 在字段或对象级别验证数据 | `validate_<field_name>`或`validate` |
| 自定义序列化和反序列化输出 | `to_representation`和`to_internal_value` |
| 保存时传递附加数据 | `serializer.save(additional=data)` |
| 将上下文传递给序列化程序 | `SampleSerializer(resource, context={'key': 'value'})` |
| 重命名序列化程序输出字段 | `source`关键字 |
| 将序列化程序函数响应附加到数据 | `source`关键字 |
| 从一对一模型中获取数据 | `source`关键字 |
| 将数据附加到序列化输出 | `SerializerMethodField` |
| 创建单独的读写序列化程序 | `get_serializer_class()` |
| 设置只读字段 | `read_only_fields` |
| 处理嵌套序列化 | `depth`字段 |