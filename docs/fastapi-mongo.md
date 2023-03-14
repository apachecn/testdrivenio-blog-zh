# 用 FastAPI 和 MongoDB 构建 CRUD 应用程序

> 原文：<https://testdriven.io/blog/fastapi-mongo/>

在本教程中，你将学习如何用 [FastAPI](https://fastapi.tiangolo.com/) 和 [MongoDB](https://www.mongodb.com/) 开发异步 API。我们将使用[马达](https://motor.readthedocs.io)包与 MongoDB 进行异步交互。

## 目标

本教程结束时，您将能够:

1.  用 Python 和 FastAPI 开发 RESTful API
2.  与 MongoDB 异步交互
3.  使用 MongoDB Atlas 在云中运行 MongoDB
4.  将 FastAPI 应用程序部署到 Heroku

## 初始设置

首先创建一个新文件夹来保存名为“fastapi-mongo”的项目:

```py
`$ mkdir fastapi-mongo
$ cd fastapi-mongo` 
```

接下来，创建并激活虚拟环境:

```py
`$ python3.9 -m venv venv
$ source venv/bin/activate
$ export PYTHONPATH=$PWD` 
```

> 你可以随意把 virtualenv 和 Pip 换成诗歌[或](https://python-poetry.org) [Pipenv](https://github.com/pypa/pipenv) 。更多信息，请查看[现代 Python 环境](/blog/python-environments/)。

接下来，创建以下文件和文件夹:

```py
`├── app
│   ├── __init__.py
│   ├── main.py
│   └── server
│       ├── app.py
│       ├── database.py
│       ├── models
│       └── routes
└── requirements.txt` 
```

将以下依赖项添加到您的 *requirements.txt* 文件中:

```py
`fastapi==0.73.0
uvicorn==0.17.4` 
```

安装它们:

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
async def read_root():
    return {"message": "Welcome to this fantastic app!"}` 
```

> [标签](https://fastapi.tiangolo.com/tutorial/path-operation-configuration/#tags)是用于对路线进行分组的标识符。具有相同标签的路线被分组到 API 文档的一个部分中。

从控制台运行入口点文件:

```py
`(venv)$ python app/main.py` 
```

在浏览器中导航至 [http://localhost:8000](http://localhost:8000) 。您应该看到:

```py
`{ "message":  "Welcome to this fantastic app!" }` 
```

也可以在[http://localhost:8000/docs](http://localhost:8000/docs)查看交互 API 文档:

![fastapi swagger ui](img/8b8b9e3b82a2de35a880cf835ef5b0a4.png)

## 路线

我们将构建一个简单的应用程序，通过以下 CRUD 途径存储学生数据:

![crud routes](img/0623ce8567cc2fd4236bd11aa3b23360.png)

在我们开始编写路由之前，让我们首先定义相关的模式并配置 MongoDB。

## (计划或理论的)纲要

让我们定义我们的数据将基于的[模式](https://pydantic-docs.helpmanual.io/usage/schema/)，它将表示数据如何存储在 MongoDB 数据库中。

> Pydantic 模式用于验证数据以及序列化(JSON -> Python)和反序列化(Python -> JSON)。换句话说，它不充当 Mongo [模式验证器](https://docs.mongodb.com/manual/core/schema-validation/)。

在“app/server/models”文件夹中，创建一个名为 *student.py* 的新文件:

```py
`from typing import Optional

from pydantic import BaseModel, EmailStr, Field

class StudentSchema(BaseModel):
    fullname: str = Field(...)
    email: EmailStr = Field(...)
    course_of_study: str = Field(...)
    year: int = Field(..., gt=0, lt=9)
    gpa: float = Field(..., le=4.0)

    class Config:
        schema_extra = {
            "example": {
                "fullname": "John Doe",
                "email": "[[email protected]](/cdn-cgi/l/email-protection)",
                "course_of_study": "Water resources engineering",
                "year": 2,
                "gpa": "3.0",
            }
        }

class UpdateStudentModel(BaseModel):
    fullname: Optional[str]
    email: Optional[EmailStr]
    course_of_study: Optional[str]
    year: Optional[int]
    gpa: Optional[float]

    class Config:
        schema_extra = {
            "example": {
                "fullname": "John Doe",
                "email": "[[email protected]](/cdn-cgi/l/email-protection)x.edu.ng",
                "course_of_study": "Water resources and environmental engineering",
                "year": 4,
                "gpa": "4.0",
            }
        }

def ResponseModel(data, message):
    return {
        "data": [data],
        "code": 200,
        "message": message,
    }

def ErrorResponseModel(error, code, message):
    return {"error": error, "code": code, "message": message}` 
```

在上面的代码中，我们定义了一个名为`StudentSchema`的 Pydantic [模式](https://pydantic-docs.helpmanual.io/usage/schema/)，它表示学生数据将如何存储在 MongoDB 数据库中。

在 Pydantic 中，[省略号](https://pydantic-docs.helpmanual.io/usage/models/#required-fields)，`...`表示某个字段是必填的。它可以被替换为`None`或默认值。在`StudentSchema`中，每个字段都有一个省略号，因为每个字段都很重要，如果没有设置值，程序就不能继续运行。

在`StudentSchema`的`gpa`和`year`字段中，我们添加了[验证器](https://pydantic-docs.helpmanual.io/usage/schema/#field-customization) `gt`、`lt`和`le`:

1.  `year`字段中的`gt`和`lt`保证传递的值大于 *0* 小于 *9* 。因此，诸如 *0* 、 *10* 、 *11* 的值将导致错误。
2.  `gpa`字段中的`le`验证器确保传递的值小于或等于 *4.0* 。

这个模式将帮助用户以适当的形式向 API 发送 HTTP 请求——即，要发送的数据类型和发送方式。

> FastAPI 使用 Pyantic 模式结合 [Json 模式](https://json-schema.org)自动记录数据模型。 [Swagger UI](https://swagger.io/tools/swagger-ui/) 然后从生成的数据模型中渲染数据。你可以在这里阅读更多关于 FastAPI 如何生成 API 文档[。](https://fastapi.tiangolo.com/features/#automatic-docs)

由于我们使用了`EmailStr`，我们需要安装[电子邮件验证器](https://github.com/JoshData/python-email-validator)。

将其添加到需求文件中:

安装:

```py
`(venv)$ pip install -r requirements.txt` 
```

有了模式之后，让我们在为 API 编写路由之前设置 MongoDB。

## MongoDB

在这一节中，我们将连接 MongoDB 并配置我们的应用程序与之通信。

> 据[维基百科](https://en.wikipedia.org/wiki/MongoDB)介绍，MongoDB 是一个跨平台的面向文档的数据库程序。作为一个 NoSQL 数据库程序，MongoDB 使用带有可选模式的类似 JSON 的文档。

### MongoDB 设置

如果您的机器上没有安装 MongoDB，请参考文档中的[安装](https://docs.mongodb.com/manual/installation/)指南。安装完成后，继续按照指南运行 [mongod](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) 守护进程。一旦完成，您就可以通过使用`mongo` shell 命令连接到实例来验证 MongoDB 已经启动并正在运行:

作为参考，本教程使用 MongoDB 社区版 v5.0.6。

```py
`$ mongo --version

MongoDB shell version v5.0.6

Build Info: {
    "version": "5.0.6",
    "gitVersion": "212a8dbb47f07427dae194a9c75baec1d81d9259",
    "modules": [],
    "allocator": "system",
    "environment": {
        "distarch": "x86_64",
        "target_arch": "x86_64"
    }
}` 
```

### 电机设置

接下来，我们将配置异步 MongoDB 驱动程序 [Motor](https://motor.readthedocs.io/) 来与数据库交互。

首先将依赖项添加到需求文件中:

安装:

```py
`(venv)$ pip install -r requirements.txt` 
```

回到应用程序，将数据库连接信息添加到 *app/server/database.py* :

```py
`import motor.motor_asyncio

MONGO_DETAILS = "mongodb://localhost:27017"

client = motor.motor_asyncio.AsyncIOMotorClient(MONGO_DETAILS)

database = client.students

student_collection = database.get_collection("students_collection")` 
```

在上面的代码中，我们导入了 Motor，定义了连接细节，并通过 [AsyncIOMotorClient](https://motor.readthedocs.io/en/stable/api-asyncio/asyncio_motor_client.html#motor.motor_asyncio.AsyncIOMotorClient) 创建了一个客户端。

然后我们引用了一个名为`students`的数据库和一个名为`students_collection`的集合(类似于关系数据库中的一个表)。因为这些只是引用而不是实际的 I/O，所以也不需要`await`表达式。当进行第一次 I/O 操作时，如果数据库和集合尚不存在，则将创建它们。

接下来，创建一个快速助手函数，用于将数据库查询的结果解析成 Python dict。

将此内容也添加到 *database.py* 文件中:

```py
`import motor.motor_asyncio

MONGO_DETAILS = "mongodb://localhost:27017"

client = motor.motor_asyncio.AsyncIOMotorClient(MONGO_DETAILS)

database = client.students

student_collection = database.get_collection("students_collection")

# helpers

def student_helper(student) -> dict:
    return {
        "id": str(student["_id"]),
        "fullname": student["fullname"],
        "email": student["email"],
        "course_of_study": student["course_of_study"],
        "year": student["year"],
        "GPA": student["gpa"],
    }` 
```

接下来，让我们编写 CRUD 数据库操作。

### 数据库 CRUD 操作

首先从 *database.py* 文件顶部的 [bson](https://github.com/py-bson/bson) 包中导入`ObjectId`方法:

```py
`from bson.objectid import ObjectId` 
```

> bson 作为电机的附属设备安装。

接下来，为 CRUD 操作添加以下每个函数:

```py
`# Retrieve all students present in the database
async def retrieve_students():
    students = []
    async for student in student_collection.find():
        students.append(student_helper(student))
    return students

# Add a new student into to the database
async def add_student(student_data: dict) -> dict:
    student = await student_collection.insert_one(student_data)
    new_student = await student_collection.find_one({"_id": student.inserted_id})
    return student_helper(new_student)

# Retrieve a student with a matching ID
async def retrieve_student(id: str) -> dict:
    student = await student_collection.find_one({"_id": ObjectId(id)})
    if student:
        return student_helper(student)

# Update a student with a matching ID
async def update_student(id: str, data: dict):
    # Return false if an empty request body is sent.
    if len(data) < 1:
        return False
    student = await student_collection.find_one({"_id": ObjectId(id)})
    if student:
        updated_student = await student_collection.update_one(
            {"_id": ObjectId(id)}, {"$set": data}
        )
        if updated_student:
            return True
        return False

# Delete a student from the database
async def delete_student(id: str):
    student = await student_collection.find_one({"_id": ObjectId(id)})
    if student:
        await student_collection.delete_one({"_id": ObjectId(id)})
        return True` 
```

在上面的代码中，我们定义了异步操作来通过 motor 创建、读取、更新和删除数据库中的学生数据。

在更新和删除操作中，在数据库中搜索学生以决定是否执行操作。返回值指导如何向用户发送响应，这将在下一节中讨论。

## CRUD 路线

在本节中，我们将添加路由来补充数据库文件中的数据库操作。

在“routes”文件夹中，创建一个名为 *student.py* 的新文件，并向其中添加以下内容:

```py
`from fastapi import APIRouter, Body
from fastapi.encoders import jsonable_encoder

from app.server.database import (
    add_student,
    delete_student,
    retrieve_student,
    retrieve_students,
    update_student,
)
from app.server.models.student import (
    ErrorResponseModel,
    ResponseModel,
    StudentSchema,
    UpdateStudentModel,
)

router = APIRouter()` 
```

我们将使用 FastAPI 的 [JSON 兼容编码器](https://fastapi.tiangolo.com/tutorial/encoder/)将我们的模型转换成 JSON 兼容的格式。

接下来，在 *app/server/app.py* 中连线学生路线:

```py
`from fastapi import FastAPI

from app.server.routes.student import router as StudentRouter

app = FastAPI()

app.include_router(StudentRouter, tags=["Student"], prefix="/student")

@app.get("/", tags=["Root"])
async def read_root():
    return {"message": "Welcome to this fantastic app!"}` 
```

### 创造

回到 routes 文件，添加以下用于创建新学生的处理程序:

```py
`@router.post("/", response_description="Student data added into the database")
async def add_student_data(student: StudentSchema = Body(...)):
    student = jsonable_encoder(student)
    new_student = await add_student(student)
    return ResponseModel(new_student, "Student added successfully.")` 
```

因此，该路由期望一个匹配`StudentSchema`格式的有效载荷。示例:

```py
`{ "fullname":  "John Doe", "email":  "[[email protected]](/cdn-cgi/l/email-protection)", "course_of_study":  "Water resources engineering", "year":  2, "gpa":  "3.0", }` 
```

启动 Uvicorn 服务器:

```py
`(venv)$ python app/main.py` 
```

并在[http://localhost:8000/docs](http://localhost:8000/docs)刷新交互 API 文档页面查看新路由:

![swagger ui](img/595285c47d5d995cabb79c3b89ca11b4.png)

也测试一下:

![swagger ui](img/27f1f95bd599e94817d95096c5067f13.png)

因此，当一个请求被发送到端点时，在调用`add_student`数据库方法和将响应存储在`new_student`变量之前，它将 JSON 编码的请求体存储在变量`student`中。然后通过`ResponseModel`返回来自数据库的响应。

也测试一下验证器:

1.  年份必须大于 0 小于 10
2.  GPA 必须小于或等于 4.0

![swagger ui](img/332ca45bebf9c0d59cace11b014d8785.png)

### 阅读

向右移动，添加以下路线来检索所有学生和单个学生:

```py
`@router.get("/", response_description="Students retrieved")
async def get_students():
    students = await retrieve_students()
    if students:
        return ResponseModel(students, "Students data retrieved successfully")
    return ResponseModel(students, "Empty list returned")

@router.get("/{id}", response_description="Student data retrieved")
async def get_student_data(id):
    student = await retrieve_student(id)
    if student:
        return ResponseModel(student, "Student data retrieved successfully")
    return ErrorResponseModel("An error occurred.", 404, "Student doesn't exist.")` 
```

![swagger ui](img/aed73117f594ea95c1ddd9d197db52a3.png)

> 如果您没有为 id 传递一个有效的 ObjectId(例如，`1`)来检索单个学生路由，会发生什么情况？如何在应用程序中更好地处理这个问题？

当实现删除操作时，您将有机会测试空数据库的响应。

### 更新

接下来，编写更新学生数据的单独路径:

```py
`@router.put("/{id}")
async def update_student_data(id: str, req: UpdateStudentModel = Body(...)):
    req = {k: v for k, v in req.dict().items() if v is not None}
    updated_student = await update_student(id, req)
    if updated_student:
        return ResponseModel(
            "Student with ID: {} name update is successful".format(id),
            "Student name updated successfully",
        )
    return ErrorResponseModel(
        "An error occurred",
        404,
        "There was an error updating the student data.",
    )` 
```

![swagger ui](img/a0f5128b1471c52a62a59eeb53e2416d.png)

### 删除

最后，添加删除路径:

```py
`@router.delete("/{id}", response_description="Student data deleted from the database")
async def delete_student_data(id: str):
    deleted_student = await delete_student(id)
    if deleted_student:
        return ResponseModel(
            "Student with ID: {} removed".format(id), "Student deleted successfully"
        )
    return ErrorResponseModel(
        "An error occurred", 404, "Student with id {0} doesn't exist".format(id)
    )` 
```

检索您之前创建的用户的 ID，并测试删除路由:

![swagger ui](img/c7ebc0e9f62df32fb6c252dad94af0b7.png)

移除任何剩余的学生，并再次测试读取路线，确保响应适用于空数据库。

## 部署

在这一节中，我们将把应用程序部署到 Heroku，并为 MongoDB 配置一个云数据库。

### 蒙戈布地图集

在部署之前，我们需要设置 [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) ，这是一个云数据库服务，用于 MongoDB 托管我们的数据库。

遵循[入门](https://docs.atlas.mongodb.com/getting-started/)指南，您将创建一个帐户，部署一个免费层集群，设置一个用户，并将一个 IP 地址列入白名单。

> 出于测试目的，将`0.0.0.0/0`用于白名单中的 IP，以允许来自任何地方的访问。对于生产应用程序，您需要限制对静态 IP 的访问。

完成后，通过单击“Connect”按钮从集群中获取数据库连接信息:

![mongodb atlas](img/043b374d63cb1ccab6e54f2d584caab6.png)

单击第二个选项“连接到您的应用程序”:

![mongodb atlas](img/988b8f8d034cae48c64f9ae6413ccde6.png)

复制连接 URL，确保更新密码。将默认数据库也设置为“学生”。它将类似于:

```py
`mongodb+srv://foobar:foobar@cluster0.0reol.mongodb.net/students?retryWrites=true&w=majority` 
```

我们将定义它有一个环境变量，而不是在我们的应用程序中硬编码这个值。创建一个名为*的新文件。项目根中的 env* 以及到它的连接信息:

```py
`MONGO_DETAILS=your_connection_url` 
```

> 确保用复制的 URL 替换`your_connection_url`。

接下来，为了简化应用程序中环境变量的管理，让我们安装 [Python Decouple](https://github.com/henriquebastos/python-decouple/) 包。将它添加到您的需求文件中，如下所示:

安装:

```py
`(venv)$ pip install -r requirements.txt` 
```

在 *app/server/database.py* 文件中，导入库:

```py
`from decouple import config` 
```

导入的`config`方法在根目录中扫描一个*。env* 文件并读取传递给它的内容。因此，在我们的例子中，它将读取`MONGO_DETAILS`变量。

接下来，将`MONGO_DETAILS`变量改为:

```py
`MONGO_DETAILS = config("MONGO_DETAILS")  # read environment variable` 
```

### 本地测试

在部署之前，让我们使用云数据库在本地测试应用程序，以确保连接配置正确。重启您的 Uvicorn 服务器，并从位于[http://localhost:8000/docs](http://localhost:8000/docs)的交互文档中测试每条路由。

您应该能够在 Atlas 仪表盘上看到数据:

![mongodb atlas](img/04467e2161ebb5196cf5f53b37c13fe5.png)

### 部署到 Heroku

最后，让我们将应用程序部署到 [Heroku](https://heroku.com) 。

> Heroku 是一个云平台即服务(PaaS ),用于部署和扩展应用程序。

如有必要，[注册](https://signup.heroku.com/)一个 Heroku 账号，安装 [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli) 。

在继续之前，创建一个*。gitignore 项目中的*文件，防止签入“venv”文件夹和*。要 git 的 env* 文件:

添加以下内容:

接下来，将一个 *Procfile* 添加到您的项目的根目录:

```py
`web:  uvicorn  app.server.app:app  --host  0.0.0.0  --port=$PORT` 
```

注意事项:

1.  Procfile 是一个文本文件，放在项目的根目录下，指导 Heroku 如何运行你的应用程序。因为我们提供的是 web 应用程序，所以我们定义了流程类型`web`以及提供 Uvicorn 的命令。
2.  Heroku 为您的应用程序动态公开一个在部署时运行的端口，这是通过`$PORT`环境变量公开的。

您的项目现在应该有以下文件和文件夹:

```py
`├── .env
├── .gitignore
├── Procfile
├── app
│   ├── __init__.py
│   ├── main.py
│   └── server
│       ├── app.py
│       ├── database.py
│       ├── models
│       │   └── student.py
│       └── routes
│           └── student.py
└── requirements.txt` 
```

在项目的根目录中，初始化一个新的 git 存储库:

```py
`(venv)$ git init
(venv)$ git add .
(venv)$ git commit -m "My fastapi and mongo application"` 
```

现在，我们可以[在 Heroku 上创建](https://devcenter.heroku.com/articles/git#for-a-new-heroku-app)一个新应用程序:

除了创建一个新的应用程序，这个命令还会在 Heroku 上创建一个远程 git 存储库，以便我们将应用程序推送到该存储库进行部署。然后，它会自动在本地存储库上将其设置为远程。

> 您可以通过运行`git remote -v`来验证遥控器是否已设置。

记下应用程序的 URL。

既然我们没有加上*。env* 文件到 git，我们需要在 Heroku 环境中设置环境变量:

```py
`(venv)$ heroku config:set MONGO_DETAILS="your_mongo_connection_url"` 
```

> 同样，确保用真实的连接 URL 替换`your_connection_url`。

将您的代码推送到 Heroku，并确保至少有一个应用程序实例正在运行:

```py
`(venv)$ git push heroku master
(venv)$ heroku ps:scale web=1` 
```

运行`heroku open`在默认浏览器中打开你的应用。

您已经成功地将您的应用程序部署到 Heroku。测试一下。

## 结论

在本教程中，您学习了如何使用 FastAPI 和 MongoDB 创建一个 CRUD 应用程序，并将其部署到 Heroku。通过回顾本教程开头的目标，快速进行自我检查。你可以在 [GitHub](https://github.com/Youngestdev/async-fastapi-mongo) 上找到本教程使用的代码。

想要更多吗？

1.  用 pytest 设置单元和集成测试。
2.  添加其他路线。
3.  为您的应用程序创建一个 GitHub repo，并使用 GitHub 操作配置 CI/CD。
4.  使用 [Fixie Socks](https://elements.heroku.com/addons/fixie-socks) 在 Heroku 上配置一个静态 IP，并限制对 MongoDB Atlas 数据库的访问。
5.  回顾一下[用 FastAPI、MongoDB 和 Beanie](/blog/fastapi-beanie/) 构建 CRUD 应用教程，看看如何利用 [Beanie ODM](https://roman-right.github.io/beanie/) ，它在 Motor 上提供了一个额外的抽象层，使得与 Mongo 数据库中的集合进行交互变得更加容易。

> 查看[测试驱动的 FastAPI 开发和 Docker](/courses/tdd-fastapi/) 课程，了解有关测试和设置 FastAPI 应用的 CI/CD 的更多信息。

干杯！