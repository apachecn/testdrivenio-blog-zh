# 用 FastAPI、MongoDB 和 Beanie 构建 CRUD 应用程序

> 原文：<https://testdriven.io/blog/fastapi-beanie/>

在本教程中，你将学习如何用 [FastAPI](https://fastapi.tiangolo.com/) 和 [MongoDB](https://www.mongodb.com/) 开发异步 API。我们将使用 [Beanie ODM](https://roman-right.github.io/beanie/) 库与 MongoDB 进行异步交互。

## 目标

本教程结束时，您将能够:

1.  解释什么是 Beanie ODM 以及为什么您可能想要使用它
2.  使用 Beanie ODM 与 MongoDB 异步交互
3.  用 Python 和 FastAPI 开发 RESTful API

## 为什么是 Beanie ODM？

Beanie 是 MongoDB 的异步对象文档映射器(ODM ),它支持开箱即用的数据和模式迁移。它使用[马达](https://motor.readthedocs.io)作为异步数据库引擎，使用[活塞](https://pydantic-docs.helpmanual.io/)。

虽然您可以简单地使用 Motor，但是 Beanie 提供了一个额外的抽象层，使得与 Mongo 数据库中的集合进行交互更加容易。

> 想只用电机？查看[用 FastAPI 和 MongoDB](/blog/fastapi-mongo) 构建 CRUD 应用程序。

## 初始设置

首先创建一个新文件夹来保存名为“fastapi-beanie”的项目:

```py
`$ mkdir fastapi-beanie
$ cd fastapi-beanie` 
```

接下来，创建并激活虚拟环境:

```py
`$ python3.10 -m venv venv
$ source venv/bin/activate
(venv)$ export PYTHONPATH=$PWD` 
```

> 随意把 venv 和 Pip 换成[诗歌](https://python-poetry.org)或 [Pipenv](https://github.com/pypa/pipenv) 。更多信息，请查看[现代 Python 环境](/blog/python-environments/)。

接下来，创建以下文件和文件夹:

```py
`├── app
│   ├── __init__.py
│   ├── main.py
│   └── server
│       ├── app.py
│       ├── database.py
│       ├── models
│       └── routes
└── requirements.txt` 
```

将以下依赖项添加到您的 *requirements.txt* 文件中:

```py
`beanie==1.11.0
fastapi==0.78.0
uvicorn==0.17.6` 
```

从终端安装依赖项:

```py
`(venv)$ pip install -r requirements.txt` 
```

在 *app/main.py* 文件中，定义运行应用程序的入口点:

```py
`import uvicorn

if __name__ == "__main__":
    uvicorn.run("server.app:app", host="0.0.0.0", port=8000, reload=True)` 
```

这里，我们指示文件在端口 8000 上运行一个[uvicon](https://www.uvicorn.org/)服务器，并在每次文件更改时重新加载。

在通过入口点文件启动服务器之前，在 *app/server/app.py* 中创建一个基本路由:

```py
`from fastapi import FastAPI

app = FastAPI()

@app.get("/", tags=["Root"])
async def read_root() -> dict:
    return {"message": "Welcome to your beanie powered app!"}` 
```

从控制台运行入口点文件:

```py
`(venv)$ python app/main.py` 
```

在浏览器中导航至 [http://localhost:8000](http://localhost:8000) 。您应该看到:

```py
`{ "message":  "Welcome to your beanie powered app!" }` 
```

## 我们在建造什么？

我们将构建一个产品评论应用程序，它允许我们执行以下操作:

*   创建评论
*   阅读评论
*   更新评论
*   删除评论

在开始编写路线之前，让我们使用 Beanie 来配置应用程序的数据库模型。

## 数据库模式

Beanie 允许您创建[文档](https://roman-right.github.io/beanie/api-documentation/document/)，这些文档可以用来与数据库中的集合进行交互。文档代表您的数据库模式。它们可以通过创建从 Beanie 继承`Document`类的子类来定义。`Document`类由 Pydantic 的`BaseModel`提供支持，这使得定义集合和数据库模式以及交互式 Swagger 文档页面中显示的示例数据变得容易。

示例:

```py
`from beanie import Document

class TestDrivenArticle(Document):
    title: str
    content: str
    date: datetime
    author: str` 
```

定义的文档表示文章将如何存储在数据库中。然而，它是一个普通的文档类，没有与之相关联的数据库集合。要关联一个集合，只需添加一个`Settings`类作为子类:

```py
`from beanie import Document

class TestDrivenArticle(Document):
    title: str
    content: str
    date: datetime
    author: str

    class Settings:
        name = "testdriven_collection"` 
```

现在我们已经知道了模式是如何创建的，我们将为我们的应用程序创建模式。在“app/server/models”文件夹中，创建一个名为 *product_review.py* 的新文件:

```py
`from datetime import datetime

from beanie import Document
from pydantic import BaseModel
from typing import Optional

class ProductReview(Document):
    name: str
    product: str
    rating: float
    review: str
    date: datetime = datetime.now()

    class Settings:
        name = "product_review"` 
```

由于`Document`类是由 Pydantic 支持的，我们可以定义示例模式数据，使开发人员更容易从交互式 Swagger 文档中使用 API。

像这样添加`Config`子类:

```py
`from datetime import datetime

from beanie import Document
from pydantic import BaseModel
from typing import Optional

class ProductReview(Document):
    name: str
    product: str
    rating: float
    review: str
    date: datetime = datetime.now()

    class Settings:
        name = "product_review"

    class Config:
        schema_extra = {
            "example": {
                "name": "Abdulazeez",
                "product": "TestDriven TDD Course",
                "rating": 4.9,
                "review": "Excellent course!",
                "date": datetime.now()
            }
        }` 
```

因此，在上面的代码块中，我们定义了一个名为`ProductReview`的 Beanie 文档，它表示产品评论将如何存储。我们还定义了集合`product_review`，数据将存储在其中。

我们将在路由中使用这个模式来实施正确的请求体。

最后，让我们定义更新产品评论的模式:

```py
`class UpdateProductReview(BaseModel):
    name: Optional[str]
    product: Optional[str]
    rating: Optional[float]
    review: Optional[str]
    date: Optional[datetime]

    class Config:
        schema_extra = {
            "example": {
                "name": "Abdulazeez Abdulazeez",
                "product": "TestDriven TDD Course",
                "rating": 5.0,
                "review": "Excellent course!",
                "date": datetime.now()
            }
        }` 
```

上面的`UpdateProductReview`类属于类型 [BaseModel](https://pydantic-docs.helpmanual.io/usage/models/) ，它允许我们只对请求体中的字段进行修改。

有了模式之后，让我们在继续编写路由之前设置 MongoDB 和我们的数据库。

## MongoDB

在这一节中，我们将连接 MongoDB 并配置我们的应用程序与之通信。

> 据[维基百科](https://en.wikipedia.org/wiki/MongoDB)介绍，MongoDB 是一个跨平台的面向文档的数据库程序。作为一个 NoSQL 数据库程序，MongoDB 使用带有可选模式的类似 JSON 的文档。

### MongoDB 设置

如果您的机器上没有安装 MongoDB，请参考文档中的[安装](https://docs.mongodb.com/manual/installation/)指南。安装完成后，继续按照指南运行 [mongod](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) 守护进程。一旦完成，您就可以通过使用`mongo` shell 命令连接到实例来验证 MongoDB 已经启动并正在运行:

作为参考，本教程使用 MongoDB 社区版 v5.0.7。

```py
`$ mongo --version
MongoDB shell version v5.0.7

Build Info: {
 "version": "5.0.7",
 "gitVersion": "b977129dc70eed766cbee7e412d901ee213acbda",
 "modules": [],
 "allocator": "system",
 "environment": {
 "distarch": "x86_64",
 "target_arch": "x86_64"
 }
}` 
```

### 设置数据库

在 *database.py* 中，添加以下内容:

```py
`from beanie import init_beanie
import motor.motor_asyncio

from app.server.models.product_review import ProductReview

async def init_db():
    client = motor.motor_asyncio.AsyncIOMotorClient(
        "mongodb://localhost:27017/productreviews"
    )

    await init_beanie(database=client.db_name, document_models=[ProductReview])` 
```

在上面的代码块中，我们导入了 [init_beanie](https://roman-right.github.io/beanie/tutorial/initialization/) 方法，该方法负责初始化由 [motor.motor_asyncio](https://motor.readthedocs.io/en/stable/api-asyncio/asyncio_motor_client.html#motor.motor_asyncio.AsyncIOMotorClient) 驱动的数据库引擎。`init_beanie`方法有两个参数:

1.  `database` -要使用的数据库的名称。
2.  `document_models`——定义的文档模型列表——在我们的例子中是`ProductReview`模型。

`init_db`函数将在应用程序启动事件中被调用。更新 *app.py* 以包含启动事件:

```py
`from fastapi import FastAPI

from app.server.database import init_db

app = FastAPI()

@app.on_event("startup")
async def start_db():
    await init_db()

@app.get("/", tags=["Root"])
async def read_root() -> dict:
    return {"message": "Welcome to your beanie powered app!"}` 
```

现在我们已经有了数据库配置，让我们来写路线。

## 路线

在本节中，我们将构建从应用程序对数据库执行 CRUD 操作的路线:

1.  事后审查
2.  获取单个评论和获取所有评论
3.  提交单个评论
4.  删除单个评论

在“routes”文件夹中，创建名为 *product_review.py* 的文件:

```py
`from beanie import PydanticObjectId
from fastapi import APIRouter, HTTPException
from typing import List

from app.server.models.product_review import ProductReview, UpdateProductReview

router = APIRouter()` 
```

在上面的代码块中，我们导入了`PydanticObjectId`，它将在检索单个请求时用于类型提示 ID 参数。我们还导入了负责处理路由操作的`APIRouter`类。我们还导入了之前定义的模型类。

Beanie 文档模型允许我们用更少的代码直接与数据库交互。例如，要检索数据库集合中的所有记录，我们所要做的就是:

```py
`data = await ProductReview.find_all().to_list()
return data # A list of all records in the collection.` 
```

在我们开始为 CRUD 操作编写 route 函数之前，让我们在 *app.py* 中注册路由:

```py
`from fastapi import FastAPI

from app.server.database import init_db
from app.server.routes.product_review import router as Router

app = FastAPI()
app.include_router(Router, tags=["Product Reviews"], prefix="/reviews")

@app.on_event("startup")
async def start_db():
    await init_db()

@app.get("/", tags=["Root"])
async def read_root() -> dict:
    return {"message": "Welcome to your beanie powered app!"}` 
```

### 创造

在 *routes/product_review.py* 中，添加以下内容:

```py
`@router.post("/", response_description="Review added to the database")
async def add_product_review(review: ProductReview) -> dict:
    await review.create()
    return {"message": "Review added successfully"}` 
```

这里，我们定义了 route 函数，它接受一个类型为`ProductReview`的参数。如前所述，文档类可以直接与数据库交互。

新记录是通过调用 [create()](https://roman-right.github.io/beanie/api-documentation/document/#documentcreate) 方法创建的。

上面的路由需要类似的有效负载，如下所示:

```py
`{ "name":  "Abdulazeez", "product":  "TestDriven TDD Course", "rating":  4.9, "review":  "Excellent course!", "date":  "2022-05-17T13:53:17.196135" }` 
```

测试路线:

```py
`$ curl -X 'POST' \
  'http://0.0.0.0:8000/reviews/' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
 "name": "Abdulazeez",
 "product": "TestDriven TDD Course",
 "rating": 4.9,
 "review": "Excellent course!",
 "date": "2022-05-17T13:53:17.196135"
}'` 
```

上面的请求应该会返回一条成功的消息:

```py
`{ "message":  "Review added successfully" }` 
```

### 阅读

接下来是使我们能够检索数据库中存在的单个评论和所有评论的路线:

```py
`@router.get("/{id}", response_description="Review record retrieved")
async def get_review_record(id: PydanticObjectId) -> ProductReview:
    review = await ProductReview.get(id)
    return review

@router.get("/", response_description="Review records retrieved")
async def get_reviews() -> List[ProductReview]:
    reviews = await ProductReview.find_all().to_list()
    return reviews` 
```

在上面的代码块中，我们定义了两个函数:

1.  在第一个函数中，该函数接受一个类型为`ObjectiD`的 ID，这是 MongoDB IDs 的默认编码。使用 [get()](https://roman-right.github.io/beanie/api-documentation/document/#documentget) 方法检索记录。
2.  第二，我们使用 [find_all()](https://roman-right.github.io/beanie/tutorial/finding-documents/#finding-documents) 方法检索所有的评论。追加了`to_list()`方法，因此结果以列表的形式返回。

> 另一种可以用来检索单个条目的方法是采用条件的 [find_one()](https://roman-right.github.io/beanie/tutorial/finding-documents/#finding-single-documents) 方法。例如:
> 
> ```py
> `# Return a record who has a rating of 4.0
> await ProductReview.find_one(ProductReview.rating == 4.0)` 
> ```

让我们测试检索所有记录的第一条路线:

```py
`$ curl -X 'GET' \
  'http://0.0.0.0:8000/reviews/' \
  -H 'accept: application/json'` 
```

回应:

```py
`[ { "_id":  "62839ad1d9a88a040663a734", "name":  "Abdulazeez", "product":  "TestDriven TDD Course", "rating":  4.9, "review":  "Excellent course!", "date":  "2022-05-17T13:53:17.196000" } ]` 
```

接下来，让我们测试检索与提供的 ID 匹配的单个记录的路径:

```py
`$ curl -X 'GET' \
  'http://0.0.0.0:8000/reviews/62839ad1d9a88a040663a734' \
  -H 'accept: application/json'` 
```

回应:

```py
`{ "_id":  "62839ad1d9a88a040663a734", "name":  "Abdulazeez", "product":  "TestDriven TDD Course", "rating":  4.9, "review":  "Excellent course!", "date":  "2022-05-17T13:53:17.196000" }` 
```

### 更新

接下来，我们来写一下更新复习记录的路线:

```py
`@router.put("/{id}", response_description="Review record updated")
async def update_student_data(id: PydanticObjectId, req: UpdateProductReview) -> ProductReview:
    req = {k: v for k, v in req.dict().items() if v is not None}
    update_query = {"$set": {
        field: value for field, value in req.items()
    }}

    review = await ProductReview.get(id)
    if not review:
        raise HTTPException(
            status_code=404,
            detail="Review record not found!"
        )

    await review.update(update_query)
    return review` 
```

在这个函数中，我们过滤掉了没有更新的字段，以防止用`None`覆盖现有字段。

要更新记录，需要更新查询。我们定义了一个更新查询，用请求体中传递的数据覆盖现有字段。然后我们检查记录是否存在。如果存在，它将被更新并返回更新后的记录，否则将引发 404 异常。

让我们测试一下路线:

```py
`$ curl -X 'PUT' \
  'http://0.0.0.0:8000/reviews/62839ad1d9a88a040663a734' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
 "name": "Abdulazeez Abdulazeez",
 "product": "TestDriven TDD Course",
 "rating": 5
}'` 
```

回应:

```py
`{ "_id":  "62839ad1d9a88a040663a734", "name":  "Abdulazeez Abdulazeez", "product":  "TestDriven TDD Course", "rating":  5.0, "review":  "Excellent course!", "date":  "2022-05-17T13:53:17.196000" }` 
```

### 删除

最后，让我们编写负责删除记录的路由:

```py
`@router.delete("/{id}", response_description="Review record deleted from the database")
async def delete_student_data(id: PydanticObjectId) -> dict:
    record = await ProductReview.get(id)

    if not record:
        raise HTTPException(
            status_code=404,
            detail="Review record not found!"
        )

    await record.delete()
    return {
        "message": "Record deleted successfully"
    }` 
```

因此，在删除记录之前，我们首先检查记录是否存在。通过调用 [delete()](https://roman-right.github.io/beanie/api-documentation/document/#documentdelete) 方法删除记录。

让我们测试一下路线:

```py
`$ curl -X 'DELETE' \
  'http://0.0.0.0:8000/reviews/62839ad1d9a88a040663a734' \
  -H 'accept: application/json'` 
```

回应:

```py
`{ "message":  "Record deleted successfully" }` 
```

我们已经成功构建了一个由 FastAPI、MongoDB 和 Beanie ODM 支持的 CRUD 应用程序。

## 结论

在本教程中，您学习了如何使用 FastAPI、MongoDB 和 Beanie ODM 创建 CRUD 应用程序。通过回顾本教程开头的目标来进行快速自检，您可以在 [GitHub](https://github.com/Youngestdev/fastapi-beanie) 上找到本教程中使用的代码。

想要更多吗？

1.  用 pytest 设置单元和集成测试。
2.  添加其他路线。
3.  为您的应用程序创建一个 GitHub repo，并使用 GitHub 操作配置 CI/CD。

> 查看[测试驱动的 FastAPI 开发和 Docker](/courses/tdd-fastapi/) 课程，了解有关测试和设置 FastAPI 应用的 CI/CD 的更多信息。

干杯！