# 将 Masonite ORM 与 FastAPI 集成

> 原文：<https://testdriven.io/blog/masonite-orm-fastapi/>

在本教程中，你将学习如何使用 [Masonite ORM](https://orm.masoniteproject.com/) 和 [FastAPI](https://fastapi.tiangolo.com/) 。

## 目标

学完本教程后，您应该能够:

1.  将 Masonite ORM 与 FastAPI 集成
2.  使用 Masonite ORM 与 Postgres、MySQL 和 SQLite 进行交互
3.  用 Masonite ORM 声明数据库应用程序中的关系
4.  用 pytest 测试 FastAPI 应用程序

## 为什么使用 Masonite ORM

Masonite ORM 是一个干净的、易于使用的对象关系映射库，它是为 [Masonite](https://docs.masoniteproject.com/) web 框架构建的。Masonite ORM 建立在[演说家 ORM](https://orator-orm.com/) 的基础上，这是一个活跃的记录 ORM，它在很大程度上受到了 Laravel 的[雄辩 ORM](https://laravel.com/docs/eloquent) 的启发。

Masonite ORM 被开发出来作为 astorator ORM 的[替代品](https://github.com/sdispater/orator/issues/369#issuecomment-633591842),因为 astorator 不再接收更新和错误修复。

尽管 Masonite ORM 是为 Masonite web 项目设计的，但是您也可以将 mason ite ORM 用于其他 Python web 框架或项目。

## FastAPI

FastAPI 是一个现代的、高性能的、内置电池的 Python web 框架，非常适合构建 RESTful APIs。它可以处理同步和异步请求，并内置了对数据验证、JSON 序列化、身份验证和授权以及 OpenAPI 的支持。

有关 FastAPI 的更多信息，请查看我们的 [FastAPI 摘要页面](https://testdriven.io/blog/topics/fastapi/)。

## 我们正在建造的东西

我们将使用以下模型构建一个简单的博客应用程序:

1.  用户
2.  邮件
3.  评论

用户将与帖子有一对多的关系，而帖子也将与评论有一对多的关系。

API 端点:

1.  `/api/v1/users` -获取所有用户的详细信息
2.  `/api/v1/users/<user_id>` -获取单个用户的详细信息
3.  `/api/v1/posts` -获取所有帖子
4.  `/api/v1/posts/<post_id>` -获得一个帖子
5.  从一篇文章中获取所有评论

## 项目设置

创建一个目录来保存名为“fastapi-masonite”的项目:

```py
`$ mkdir fastapi-masonite
$ cd fastapi-masonite` 
```

创建虚拟环境并激活它:

```py
`$ python3.10 -m venv .env
$ source .env/bin/activate

(.env)$` 
```

> 你可以随意把 virtualenv 和 Pip 换成诗歌[或](https://python-poetry.org) [Pipenv](https://github.com/pypa/pipenv) 。更多信息，请查看[现代 Python 环境](/blog/python-environments/)。

创建一个 *requirements.txt* 文件，并向其中添加以下需求:

```py
`fastapi==0.89.1
uvicorn==0.20.0` 
```

[uvicon](http://www.uvicorn.org/)是一个 [ASGI](https://asgi.readthedocs.io/en/latest/) (异步服务器网关接口)兼容服务器，将用于启动 FastAPI。

安装要求:

```py
`(.env)$ pip install -r requirements.txt` 
```

在项目的根文件夹中创建一个 *main.py* 文件，并添加以下几行:

```py
`from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def say_hello():
    return {"msg": "Hello World"}` 
```

使用以下命令运行 FastAPI 服务器:

```py
`(.env)$ uvicorn main:app --reload` 
```

打开您选择的网络浏览器并导航至 [http://127.0.0.1:8000](http://127.0.0.1:8000) 。您应该会看到以下 JSON 响应:

## Masonite ORM

将以下需求添加到 *requirements.txt* 文件中:

```py
`masonite-orm==2.18.6
psycopg2-binary==2.9.5` 
```

安装新的依赖项:

```py
`(.env)$ pip install -r requirements.txt` 
```

创建以下文件夹:

```py
`models
databases/migrations
config` 
```

“models”文件夹将包含我们的模型文件，“databases/migrations”文件夹将包含我们的迁移文件，“config”文件夹将保存我们的 Masonite 数据库配置文件。

### 数据库配置

在“config”文件夹中，创建一个 *database.py* 文件。Masonite ORM 需要这个文件，因为这是我们声明数据库配置的地方。

> 欲了解更多信息，请访问[文档](https://orm.masoniteproject.com/installation#configuration)。

在 *database.py* 文件中，我们需要添加`DATABASE`变量和一些连接信息，从`masonite-orm.connections`导入`ConnectionResolver`，并注册连接细节:

```py
`# config/database.py

from masoniteorm.connections import ConnectionResolver

DATABASES = {
  "default": "postgres",
  "mysql": {
    "host": "127.0.0.1",
    "driver": "mysql",
    "database": "masonite",
    "user": "root",
    "password": "",
    "port": 3306,
    "log_queries": False,
    "options": {
      #
    }
  },
  "postgres": {
    "host": "127.0.0.1",
    "driver": "postgres",
    "database": "test",
    "user": "test",
    "password": "test",
    "port": 5432,
    "log_queries": False,
    "options": {
      #
    }
  },
  "sqlite": {
    "driver": "sqlite",
    "database": "db.sqlite3",
  }
}

DB = ConnectionResolver().set_connection_details(DATABASES)` 
```

这里，我们定义了三种不同的数据库设置:

1.  关系型数据库
2.  Postgres
3.  SQLite

我们将默认连接设置为 Postgres。

> 注意:确保您已经启动并运行了 Postgres 数据库。如果要使用 MySQL，将默认连接改为`mysql`。

### Masonite 型号

要创建一个新的样板文件 [Masonite 模型](https://orm.masoniteproject.com/models#creating-a-model)，从终端的项目根文件夹中运行下面的`masonite-orm`命令:

```py
`(.env)$ masonite-orm model User --directory models` 
```

您应该会看到一条成功消息:

```py
`Model created: models/User.py` 
```

因此，该命令应该在“models”目录中创建一个包含以下内容的 *User.py* 文件:

```py
`""" User Model """

from masoniteorm.models import Model

class User(Model):
    """User Model"""

    pass` 
```

> 如果您收到一个`FileNotFoundError`，检查以确保“模型”文件夹存在。

对帖子和评论模型运行相同的命令:

```py
`(.env)$ masonite-orm model Post --directory models
> Model created: models/Post.py

(.env)$ masonite-orm model Comment --directory models
> Model created: models/Comment.py` 
```

接下来，我们可以创建初始迁移:

```py
`(.env)$ masonite-orm migration migration_for_user_table --create users` 
```

我们添加了`--create`标志来告诉 Masonite 将要创建的迁移文件是针对我们的`users`表的，并且应该在迁移运行时创建数据库表。

在“数据库/迁移”文件夹中，应该已经创建了一个新文件:

`<timestamp>_migration_for_user_table.py`

内容:

```py
`"""MigrationForUserTable Migration."""

from masoniteorm.migrations import Migration

class MigrationForUserTable(Migration):
    def up(self):
        """
 Run the migrations.
 """
        with self.schema.create("users") as table:
            table.increments("id")

            table.timestamps()

    def down(self):
        """
 Revert the migrations.
 """
        self.schema.drop("users")` 
```

创建剩余的迁移文件:

```py
`(.env)$ masonite-orm migration migration_for_post_table --create posts
> Migration file created: databases/migrations/2022_05_04_084820_migration_for_post_table.py

(.env)$ masonite-orm migration migration_for_comment_table --create comments
> Migration file created: databases/migrations/2022_05_04_084833_migration_for_comment_table.py` 
```

接下来，让我们填充每个数据库表的字段。

### 数据库表

`users`表应该有以下字段:

1.  名字
2.  电子邮件(唯一)
3.  地址(可选)
4.  电话号码(可选)
5.  性别(可选)

将与用户模型相关联的迁移文件更改为:

```py
`"""MigrationForUserTable Migration."""

from masoniteorm.migrations import Migration

class MigrationForUserTable(Migration):
    def up(self):
        """
 Run the migrations.
 """
        with self.schema.create("users") as table:
            table.increments("id")
            table.string("name")
            table.string("email").unique()
            table.text("address").nullable()
            table.string("phone_number", 11).nullable()
            table.enum("sex", ["male", "female"]).nullable()
            table.timestamps()

    def down(self):
        """
 Revert the migrations.
 """
        self.schema.drop("users")` 
```

> 有关表方法和列类型的更多信息，请查看文档中的[模式&迁移](https://orm.masoniteproject.com/schema-and-migrations)。

接下来，更新帖子和评论模型的字段，注意这些字段。

帖子:

```py
`"""MigrationForPostTable Migration."""

from masoniteorm.migrations import Migration

class MigrationForPostTable(Migration):
    def up(self):
        """
 Run the migrations.
 """
        with self.schema.create("posts") as table:
            table.increments("id")
            table.integer("user_id").unsigned()
            table.foreign("user_id").references("id").on("users")
            table.string("title")
            table.text("body")
            table.timestamps()

    def down(self):
        """
 Revert the migrations.
 """
        self.schema.drop("posts")` 
```

评论:

```py
`"""MigrationForCommentTable Migration."""

from masoniteorm.migrations import Migration

class MigrationForCommentTable(Migration):
    def up(self):
        """
 Run the migrations.
 """
        with self.schema.create("comments") as table:
            table.increments("id")
            table.integer("user_id").unsigned().nullable()
            table.foreign("user_id").references("id").on("users")
            table.integer("post_id").unsigned().nullable()
            table.foreign("post_id").references("id").on("posts")
            table.text("body")
            table.timestamps()

    def down(self):
        """
 Revert the migrations.
 """
        self.schema.drop("comments")` 
```

注意到:

```py
`table.integer("user_id").unsigned()
table.foreign("user_id").references("id").on("users")` 
```

上面几行创建了一个从`posts` / `comments`表到`users`表的外键。`user_id`列引用`users`表上的`id`列

要应用迁移，请在终端中运行以下命令:

```py
`(.env)$ masonite-orm migrate` 
```

您应该会看到关于每个迁移的成功消息:

```py
`Migrating: 2022_05_04_084807_migration_for_user_table
Migrated: 2022_05_04_084807_migration_for_user_table (0.08s)
Migrating: 2022_05_04_084820_migration_for_post_table
Migrated: 2022_05_04_084820_migration_for_post_table (0.04s)
Migrating: 2022_05_04_084833_migration_for_comment_table
Migrated: 2022_05_04_084833_migration_for_comment_table (0.02s)` 
```

到目前为止，我们已经在表中添加并引用了外键，这些外键是在数据库中创建的。但是，我们仍然需要告诉 Masonite 每个模型之间的关系类型。

### 表关系

为了定义一对多的关系，我们需要从*模型/User.py* 中的`masoniteorm.relationships`导入`has_many`，并将其作为装饰者添加到我们的函数中:

```py
`# models/User.py

from masoniteorm.models import Model
from masoniteorm.relationships import has_many

class User(Model):
    """User Model"""

    @has_many("id", "user_id")
    def posts(self):
        from .Post import Post

        return Post

    @has_many("id", "user_id")
    def comments(self):
        from .Comment import Comment

        return Comment` 
```

请注意，`has_many`有两个参数:

1.  将在另一个表中引用的主表上的主键列的名称
2.  将作为外键引用的列的名称

在`users`表中，`id`是主键列，而`user_id`是引用`users`表记录的`posts`表中的列。

对*型号/Post.py* 进行同样的操作:

```py
`# models/Post.py

from masoniteorm.models import Model
from masoniteorm.relationships import has_many

class Post(Model):
    """Post Model"""

    @has_many("id", "post_id")
    def comments(self):
        from .Comment import Comment

        return Comment` 
```

配置好数据库后，让我们用 FastAPI 连接我们的 API。

## FastAPI RESTful API

### 迂腐的

FastAPI 严重依赖 [Pydantic](https://pydantic-docs.helpmanual.io/) 来操作(读取和返回)数据。

在根文件夹中，创建一个名为 *schema.py* 的新 Python 文件:

```py
`# schema.py

from pydantic import BaseModel

from typing import Optional

class UserBase(BaseModel):
    name: str
    email: str
    address: Optional[str] = None
    phone_number: Optional[str] = None
    sex: Optional[str] = None

class UserCreate(UserBase):
    email: str

class UserResult(UserBase):
    id: int

    class Config:
        orm_mode = True` 
```

这里，我们为用户对象定义了一个基本模型，然后添加了两个 Pydantic 模型，一个用于读取数据，另一个用于从 API 返回数据。我们对可空值使用了`Optional`类型。

> 你可以在这里阅读更多关于 Pydantic 模型[。](https://pydantic-docs.helpmanual.io/usage/models/)

在`UserResult` Pydantic 类中，我们增加了一个`Config`类，并将`orm_mode`设置为`True`。这告诉 Pydantic 不仅要将数据作为 dict 读取，还要作为具有属性的对象读取。因此，您可以选择:

```py
`user_id = user["id"]   # as a dict

user_id = user.id  # as an attribute` 
```

> [更多关于 Pydantic ORM 模型的信息](https://pydantic-docs.helpmanual.io/usage/models/#orm-mode-aka-arbitrary-class-instances)。

接下来，为帖子和评论对象添加模型:

```py
`# schema.py

from pydantic import BaseModel

from typing import Optional

class UserBase(BaseModel):
    name: str
    email: str
    address: Optional[str] = None
    phone_number: Optional[str] = None
    sex: Optional[str] = None

class UserCreate(UserBase):
    email: str

class UserResult(UserBase):
    id: int

    class Config:
        orm_mode = True

class PostBase(BaseModel):
    user_id: int
    title: str
    body: str

class PostCreate(PostBase):
    pass

class PostResult(PostBase):
    id: int

    class Config:
        orm_mode = True

class CommentBase(BaseModel):
    user_id: int
    body: str

class CommentCreate(CommentBase):
    pass

class CommentResult(CommentBase):
    id: int
    post_id: int

    class Config:
        orm_mode = True` 
```

### API 端点

现在，让我们添加 API 端点。在 *main.py* 文件中，导入 Pydantic 模式和 Masonite 模型:

```py
`import schema

from models.Post import Post
from models.User import User
from models.Comment import Comment` 
```

要获取所有用户，可以使用 Masonite 的[。集合实例上用户模型的所有方法调用](https://orm.masoniteproject.com/models#selecting)返回:

```py
`@app.get("/api/v1/users", response_model=List[schema.UserResult])
def get_all_users():
    users = User.all()
    return users.all()` 
```

确保导入`typing.List`:

要添加用户，请添加以下发布端点:

```py
`@app.post("/api/v1/users", response_model=schema.UserResult)
def add_user(user_data: schema.UserCreate):
    user = User.where("email", user_data.email).get()
    if user:
        raise HTTPException(status_code=400, detail="User already exists")
    user = User()
    user.email = user_data.email
    user.name = user_data.name
    user.address = user_data.address
    user.sex = user_data.sex
    user.phone_number = user_data.phone_number

    user.save() # saves user details to the database.
    return user` 
```

导入[http 异常](https://fastapi.tiangolo.com/tutorial/handling-errors/#import-httpexception):

```py
`from fastapi import FastAPI, HTTPException` 
```

检索单个用户:

```py
`@app.get("/api/v1/users/{user_id}", response_model=schema.UserResult)
def get_single_user(user_id: int):
    user = User.find(user_id)
    return user` 
```

发布端点:

```py
`@app.get("/api/v1/posts", response_model=List[schema.PostResult])
def get_all_posts():
    all_posts = Post.all()
    return all_posts.all()

@app.get("/api/v1/posts/{post_id}", response_model=schema.PostResult)
def get_single_post(post_id: int):
    post = Post.find(post_id)
    return post

@app.post("/api/v1/posts", response_model=schema.PostResult)
def add_new_post(post_data: schema.PostCreate):
    user = User.find(post_data.user_id)
    if not user:
        raise HTTPException(status_code=400, detail="User not found")
    post = Post()
    post.title = post_data.title
    post.body = post_data.body
    post.user_id = post_data.user_id
    post.save()

    user.attach("posts", post)

    return post` 
```

我们将来自 API 的数据保存到数据库上的`posts`表中，然后为了将帖子链接到用户，我们附加了它，以便当我们调用`user.posts()`时，我们可以获得用户的所有帖子。

注释端点:

```py
`@app.post("/api/v1/{post_id}/comments", response_model=schema.CommentResult)
def add_new_comment(post_id: int, comment_data: schema.CommentCreate):
    post = Post.find(post_id)
    if not post:
        raise HTTPException(status_code=400, detail="Post not found")
    user = User.find(comment_data.user_id)
    if not user:
        raise HTTPException(status_code=400, detail="User not found")

    comment = Comment()
    comment.body = comment_data.body
    comment.user_id = comment_data.user_id
    comment.post_id = post_id

    comment.save()

    user.attach("comments", comment)
    post.attach("comments", comment)

    return comment

@app.get("/api/v1/posts/{post_id}/comments", response_model=List[schema.CommentResult])
def get_post_comments(post_id):
    post = Post.find(post_id)
    return post.comments.all()

@app.get("/api/v1/users/{user_id}/comments", response_model=List[schema.CommentResult])
def get_user_comments(user_id):
    user = User.find(user_id)
    return user.comments.all()` 
```

如果 FastAPI 服务器尚未运行，请启动它:

```py
`(.env)$ uvicorn main:app --reload` 
```

导航到[http://localhost:8000/docs](http://localhost:8000/docs)查看所有端点的 Swagger/OpenAPI 文档。测试每个端点以查看响应。

## 试验

因为我们是好公民，我们会增加一些测试。

### 固定装置

让我们为上面的代码编写测试。因为我们将使用 [pytest](https://docs.pytest.org/) ，继续将依赖项添加到 *requirements.txt* 文件中:

我们还需要 [HTTPX](https://www.python-httpx.org/) 库，因为 FastAPI 的 [TestClient](https://fastapi.tiangolo.com/tutorial/testing/) 基于它。也将其添加到需求文件中:

安装:

```py
`(.env)$ pip install -r requirements.txt` 
```

接下来，让我们为测试创建一个单独的配置文件，这样我们就不会覆盖主开发数据库中的数据。在“config”文件夹中，创建一个名为 *test_config.py* 的新文件:

```py
`# config/test_config.py

from masoniteorm.connections import ConnectionResolver

DATABASES = {
  "default": "sqlite",
  "sqlite": {
    "driver": "sqlite",
    "database": "db.sqlite3",
  }
}

DB = ConnectionResolver().set_connection_details(DATABASES)` 
```

注意，它类似于我们在 *config/database.py* 文件中的内容。唯一的区别是我们将`default`设置为`sqlite`,因为我们想要使用 SQLite 进行测试。

为了将我们的测试套件设置为总是使用我们在 *test_config.py* 配置中的配置，而不是默认的 *database.py* 文件，我们可以使用 pytest 的 [autouse](https://docs.pytest.org/en/latest/fixture.html#autouse-fixtures-fixtures-you-don-t-have-to-request) fixture。

创建一个名为“tests”的新文件夹，并在这个新文件夹中创建一个 *conftest.py* 文件:

```py
`import pytest
from masoniteorm.migrations import Migration

@pytest.fixture(autouse=True)
def setup_database():
    config_path = "config/test_config.py"

    migrator = Migration(config_path=config_path)
    migrator.create_table_if_not_exists()

    migrator.refresh()` 
```

这里，我们将 Masonite 的迁移配置路径设置为 *config/test_config.py* 文件，创建了`migration`表(如果之前还没有创建的话),然后刷新所有的迁移。因此，每个测试都将从数据库的一个干净副本开始。

现在，让我们为用户、帖子和评论定义一些装置:

```py
`import pytest
from masoniteorm.migrations import Migration

from models.Comment import Comment
from models.Post import Post
from models.User import User

@pytest.fixture(autouse=True)
def setup_database():
    config_path = "config/test_config.py"

    migrator = Migration(config_path=config_path)
    migrator.create_table_if_not_exists()

    migrator.refresh()

@pytest.fixture(scope="function")
def user():
    user = User()
    user.name = "John Doe"
    user.address = "United States of Nigeria"
    user.phone_number = 123456789
    user.sex = "male"
    user.email = "[[email protected]](/cdn-cgi/l/email-protection)"
    user.save()

    return user

@pytest.fixture(scope="function")
def post(user):
    post = Post()
    post.title = "Test Title"
    post.body = "this is the post body and can be as long as possible"
    post.user_id = user.id
    post.save()

    user.attach("posts", post)
    return post

@pytest.fixture(scope="function")
def comment(user, post):
    comment = Comment()
    comment.body = "This is a comment body"
    comment.user_id = user.id
    comment.post_id = post.id

    comment.save()

    user.attach("comments", comment)
    post.attach("comments", comment)

    return comment` 
```

至此，我们现在可以开始编写一些测试了。

### 测试规格

在“tests”文件夹中创建一个名为 *test_views.py* 的新测试文件。

首先添加以下代码来实例化一个 [TestClient](https://fastapi.tiangolo.com/tutorial/testing/) :

```py
`from fastapi.testclient import TestClient

from main import app  # => FastAPI app created in our main.py file

client = TestClient(app)` 
```

现在，我们将测试添加到:

1.  保存用户
2.  获取所有用户
3.  使用用户 ID 获取单个用户

代码:

```py
`from fastapi.testclient import TestClient

from main import app  # => FastAPI app created in our main.py file
from models.User import User

client = TestClient(app)

def test_create_new_user():
    assert len(User.all()) == 0 # Asserting that there's no user in the database

    payload = {
        "name": "My name",
        "email": "[[email protected]](/cdn-cgi/l/email-protection)",
        "address": "My full Address",
        "sex": "male",
        "phone_number": 123456789
    }

    response = client.post("/api/v1/users", json=payload)
    assert response.status_code == 200

    assert len(User.all()) == 1

def test_get_all_user_details(user):
    response = client.get("/api/v1/users")
    assert response.status_code == 200

    result = response.json()
    assert type(result) is list

    assert len(result) == 1
    assert result[0]["name"] == user.name
    assert result[0]["email"] == user.email
    assert result[0]["id"] == user.id

# Test to get a single user
def test_get_single_user(user):
    response = client.get(f"/api/v1/users/{user.id}")
    assert response.status_code == 200

    result = response.json()
    assert type(result) is dict

    assert result["name"] == user.name
    assert result["email"] == user.email
    assert result["id"] == user.id` 
```

运行测试以确保它们通过:

尝试为帖子和评论视图编写测试。

## 结论

在本教程中，我们介绍了如何将 Masonite ORM 与 FastAPI 一起使用。Masonite ORM 是一个相对较新的 ORM 库，有一个活跃的社区。如果您有使用 astorar ORM(或者任何其他基于 Python 的 ORM)的经验，Masonite ORM 应该很容易使用。