# 带有异步 SQLAlchemy、SQLModel 和 Alembic 的 FastAPI

> 原文：<https://testdriven.io/blog/fastapi-sqlmodel/>

本教程着眼于如何通过 SQLModel 和 FastAPI 异步地使用 SQLAlchemy。我们还将配置 Alembic 来处理数据库迁移。

> 本教程假设您有使用 Docker 处理 FastAPI 和 Postgres 的经验。需要帮助快速掌握 FastAPI、Postgres 和 Docker 吗？从以下资源开始:
> 
> 1.  [使用 FastAPI 和 Pytest 开发和测试异步 API](/blog/fastapi-crud/)
> 2.  [用 FastAPI 和 Docker 进行测试驱动开发](/courses/tdd-fastapi/)

## 项目设置

从从[fastapi-SQL model-alem BIC](https://github.com/testdrivenio/fastapi-sqlmodel-alembic)repo 克隆基础项目开始:

```
`$ git clone -b base https://github.com/testdrivenio/fastapi-sqlmodel-alembic
$ cd fastapi-sqlmodel-alembic` 
```

从项目根目录，创建映像并启动 Docker 容器:

```
`$ docker-compose up -d --build` 
```

构建完成后，导航到[http://localhost:8004/ping](http://localhost:8004/ping)。您应该看到:

在继续之前，快速浏览一下项目结构。

## SQLModel

接下来，让我们添加 [SQLModel](https://sqlmodel.tiangolo.com/) ，这是一个用于通过 Python 对象从 Python 代码与 SQL 数据库进行交互的库。基于 Python 类型注释，它本质上是一个位于 [pydantic](https://pydantic-docs.helpmanual.io/) 和 [SQLAlchemy](https://www.sqlalchemy.org/) 之上的包装器，使得两者都能轻松工作。

我们还需要[心理医生](https://www.psycopg.org/)。

将两个依赖项添加到 *project/requirements.txt* :

```
`fastapi==0.68.1
psycopg2-binary==2.9.1
sqlmodel==0.0.4
uvicorn==0.15.0` 
```

在“项目/app”中新建两个文件， *db.py* 和 *models.py* 。

*project/app/models.py* :

```
`from sqlmodel import SQLModel, Field

class SongBase(SQLModel):
    name: str
    artist: str

class Song(SongBase, table=True):
    id: int = Field(default=None, primary_key=True)

class SongCreate(SongBase):
    pass` 
```

这里，我们定义了三个模型:

1.  `SongBase`是其他人继承的基础模型。它有两个属性，`name`和`artist`，都是字符串。这是一个纯数据模型，因为它缺少`table=True`，这意味着它只能用作 pydantic 模型。
2.  与此同时，`Song`向基本模型添加了一个`id`属性。这是一个表格模型，所以它是一个 pydantic 和 SQLAlchemy 模型。它代表一个数据库表。
3.  `SongCreate`是一个纯数据的 pydantic 模型，将用于创建新的歌曲实例。

*项目/app/db.py* :

```
`import os

from sqlmodel import create_engine, SQLModel, Session

DATABASE_URL = os.environ.get("DATABASE_URL")

engine = create_engine(DATABASE_URL, echo=True)

def init_db():
    SQLModel.metadata.create_all(engine)

def get_session():
    with Session(engine) as session:
        yield session` 
```

在此，我们:

1.  使用 SQLModel 中的`create_engine`初始化新的 SQLAlchemy [引擎](https://docs.sqlalchemy.org/en/14/core/engines.html)。SQLModel 的`create_engine`和 SQLAlchemy 的版本之间的主要区别在于，SQLModel 版本添加了类型注释(用于编辑器支持)并启用了[SQLAlchemy“2.0”风格的引擎和连接](https://docs.sqlalchemy.org/en/14/core/future.html)。此外，我们传入了`echo=True`,这样我们可以在终端中看到生成的 SQL 查询。出于调试目的，在开发模式下启用这一点总是好的。
2.  创建了 SQLAlchemy [会话](https://docs.sqlalchemy.org/en/14/orm/session.html)。

接下来，在 *project/app/main.py* 中，让我们使用 [startup](https://fastapi.tiangolo.com/advanced/events/#startup-event) 事件在启动时创建表:

```
`from fastapi import FastAPI

from app.db import init_db
from app.models import Song

app = FastAPI()

@app.on_event("startup")
def on_startup():
    init_db()

@app.get("/ping")
async def pong():
    return {"ping": "pong!"}` 
```

值得注意的是`from app.models import Song`是必填项。没有它，就不会创建歌单。

要进行测试，请关闭旧容器和卷，重建映像，并启动新容器:

```
`$ docker-compose down -v
$ docker-compose up -d --build` 
```

通过`docker-compose logs web`打开容器日志。您应该看到:

```
`web_1  | CREATE TABLE song (
web_1  |    name VARCHAR NOT NULL,
web_1  |    artist VARCHAR NOT NULL,
web_1  |    id SERIAL,
web_1  |    PRIMARY KEY (id)
web_1  | )` 
```

打开 psql:

```
`$ docker-compose exec db psql --username=postgres --dbname=foo

psql (13.4 (Debian 13.4-1.pgdg100+1))
Type "help" for help.

foo=# \dt

        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | song | table | postgres
(1 row)

foo=# \q` 
```

有了这个表，让我们给 *project/app/main.py* 添加一些新的路由:

```
`from fastapi import Depends, FastAPI
from sqlalchemy import select
from sqlmodel import Session

from app.db import get_session, init_db
from app.models import Song, SongCreate

app = FastAPI()

@app.on_event("startup")
def on_startup():
    init_db()

@app.get("/ping")
async def pong():
    return {"ping": "pong!"}

@app.get("/songs", response_model=list[Song])
def get_songs(session: Session = Depends(get_session)):
    result = session.execute(select(Song))
    songs = result.scalars().all()
    return [Song(name=song.name, artist=song.artist, id=song.id) for song in songs]

@app.post("/songs")
def add_song(song: SongCreate, session: Session = Depends(get_session)):
    song = Song(name=song.name, artist=song.artist)
    session.add(song)
    session.commit()
    session.refresh(song)
    return song` 
```

添加歌曲:

```
`$ curl -d '{"name":"Midnight Fit", "artist":"Mogwai"}' -H "Content-Type: application/json" -X POST http://localhost:8004/songs

{
  "id": 1,
  "name": "Midnight Fit",
  "artist": "Mogwai"
}` 
```

在浏览器中，导航到[http://localhost:8004/songs](http://localhost:8004/songs)。您应该看到:

```
`{ "id":  1, "name":  "Midnight Fit", "artist":  "Mogwai" }` 
```

## 异步 SQLModel

接下来，让我们为 SQLModel 添加异步支持。

首先，把容器和体积拿下来:

更新 *docker-compose.yml* 中的数据库 URI，在`+asyncpg`中增加:

接下来，用 [asyncpg](https://magicstack.github.io/asyncpg/) 替换 Psycopg:

```
`asyncpg==0.24.0
fastapi==0.68.1
sqlmodel==0.0.4
uvicorn==0.15.0` 
```

更新 *project/app/db.py* :使用 SQLAlchemy 引擎和会话的异步风格:

```
`import os

from sqlmodel import SQLModel

from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = os.environ.get("DATABASE_URL")

engine = create_async_engine(DATABASE_URL, echo=True, future=True)

async def init_db():
    async with engine.begin() as conn:
        # await conn.run_sync(SQLModel.metadata.drop_all)
        await conn.run_sync(SQLModel.metadata.create_all)

async def get_session() -> AsyncSession:
    async_session = sessionmaker(
        engine, class_=AsyncSession, expire_on_commit=False
    )
    async with async_session() as session:
        yield session` 
```

注意事项:

1.  我们使用了 SQLAlchemy 构造——例如， [create_async_engine](https://docs.sqlalchemy.org/en/14/orm/extensions/asyncio.html#sqlalchemy.ext.asyncio.create_async_engine) 和[async session](https://docs.sqlalchemy.org/en/14/orm/extensions/asyncio.html#sqlalchemy.ext.asyncio.AsyncSession)——因为在撰写本文时，SQLModel 还没有它们的包装器。
2.  我们通过传入`expire_on_commit=False`来禁用[提交时过期](https://docs.sqlalchemy.org/en/14/orm/extensions/asyncio.html#preventing-implicit-io-when-using-asyncsession)行为。
3.  `metadata.create_all`不异步执行，所以我们使用 [run_sync](https://docs.sqlalchemy.org/en/14/orm/extensions/asyncio.html#sqlalchemy.ext.asyncio.AsyncSession.run_sync) 在异步函数中同步执行。

将`on_startup`变成 *project/app/main.py* 中的异步函数:

```
`@app.on_event("startup")
async def on_startup():
    await init_db()` 
```

就是这样。重建图像并旋转容器:

```
`$ docker-compose up -d --build` 
```

确保已经创建了表。

最后，更新 *project/app/main.py* 中的路由处理程序以使用异步执行:

```
`from fastapi import Depends, FastAPI
from sqlalchemy.future import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.db import get_session, init_db
from app.models import Song, SongCreate

app = FastAPI()

@app.on_event("startup")
async def on_startup():
    await init_db()

@app.get("/ping")
async def pong():
    return {"ping": "pong!"}

@app.get("/songs", response_model=list[Song])
async def get_songs(session: AsyncSession = Depends(get_session)):
    result = await session.execute(select(Song))
    songs = result.scalars().all()
    return [Song(name=song.name, artist=song.artist, id=song.id) for song in songs]

@app.post("/songs")
async def add_song(song: SongCreate, session: AsyncSession = Depends(get_session)):
    song = Song(name=song.name, artist=song.artist)
    session.add(song)
    await session.commit()
    await session.refresh(song)
    return song` 
```

添加一首新歌，并确保[http://localhost:8004/songs](http://localhost:8004/songs)按预期工作。

## 蒸馏器

最后，让我们添加 [Alembic](https://alembic.sqlalchemy.org/) 来正确处理数据库模式变化。

将其添加到需求文件中:

```
`alembic==1.7.1
asyncpg==0.24.0
fastapi==0.68.1
sqlmodel==0.0.4
uvicorn==0.15.0` 
```

从 *project/app/main.py* 中移除启动事件，因为我们不再需要在启动时创建的表:

```
`@app.on_event("startup")
async def on_startup():
    await init_db()` 
```

同样，关闭现有的容器和卷:

向上旋转容器:

```
`$ docker-compose up -d --build` 
```

在构建新图像时，快速浏览一下使用带有 Alembic 的 Asyncio】。

一旦容器备份完毕，用[异步](https://github.com/sqlalchemy/alembic/tree/rel_1_7_1/alembic/templates/async)模板初始化 Alembic:

```
`$ docker-compose exec web alembic init -t async migrations` 
```

在生成的“项目/迁移”文件夹中，将 SQLModel 导入到 *script.py.mako* 中，这是一个[樱井真子](https://www.makotemplates.org/)模板文件:

```
`"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}

"""
from alembic import op
import sqlalchemy as sa
import sqlmodel             # NEW
${imports if imports else ""}

# revision identifiers, used by Alembic.
revision = ${repr(up_revision)}
down_revision = ${repr(down_revision)}
branch_labels = ${repr(branch_labels)}
depends_on = ${repr(depends_on)}

def upgrade():
    ${upgrades if upgrades else "pass"}

def downgrade():
    ${downgrades if downgrades else "pass"}` 
```

现在，当生成新的迁移文件时，它将包含`import sqlmodel`。

接下来，我们需要更新*project/migrations/env . py*的顶部，如下所示:

```
`import asyncio
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import AsyncEngine
from sqlmodel import SQLModel                       # NEW

from alembic import context

from app.models import Song                         # NEW

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

# Interpret the config file for Python logging.
# This line sets up loggers basically.
fileConfig(config.config_file_name)

# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata
target_metadata = SQLModel.metadata             # UPDATED

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.

...` 
```

这里，我们导入了 SQLModel 和我们的歌曲模型。然后，我们将`target_metadata`设置为模型的[元数据](https://docs.sqlalchemy.org/en/14/core/metadata.html#sqlalchemy.schema.MetaData)、`SQLModel.metadata`。关于`target_metadata`论点的更多信息，请查看官方 Alembic 文档中的[自动生成迁移](https://alembic.sqlalchemy.org/en/latest/autogenerate.html)。

更新*项目/alembic.ini* 中的`sqlalchemy.url`:

```
`sqlalchemy.url  =  postgresql+asyncpg://postgres:postgres@db:5432/foo` 
```

要生成第一个迁移文件，请运行:

```
`$ docker-compose exec web alembic revision --autogenerate -m "init"` 
```

如果一切顺利，您应该会在“项目/迁移/版本”中看到一个新的迁移文件，如下所示:

```
`"""init

Revision ID: f9c634db477d
Revises:
Create Date: 2021-09-10 00:24:32.718895

"""
from alembic import op
import sqlalchemy as sa
import sqlmodel

# revision identifiers, used by Alembic.
revision = 'f9c634db477d'
down_revision = None
branch_labels = None
depends_on = None

def upgrade():
    # ### commands auto generated by Alembic - please adjust! ###
    op.create_table('song',
    sa.Column('name', sqlmodel.sql.sqltypes.AutoString(), nullable=False),
    sa.Column('artist', sqlmodel.sql.sqltypes.AutoString(), nullable=False),
    sa.Column('id', sa.Integer(), nullable=True),
    sa.PrimaryKeyConstraint('id')
    )
    op.create_index(op.f('ix_song_artist'), 'song', ['artist'], unique=False)
    op.create_index(op.f('ix_song_id'), 'song', ['id'], unique=False)
    op.create_index(op.f('ix_song_name'), 'song', ['name'], unique=False)
    # ### end Alembic commands ###

def downgrade():
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_index(op.f('ix_song_name'), table_name='song')
    op.drop_index(op.f('ix_song_id'), table_name='song')
    op.drop_index(op.f('ix_song_artist'), table_name='song')
    op.drop_table('song')
    # ### end Alembic commands ###` 
```

应用迁移:

```
`$ docker-compose exec web alembic upgrade head` 
```

确保您可以添加歌曲。

让我们快速测试一个模式变更。更新*项目/app/models.py* 中的`SongBase`模型:

```
`class SongBase(SQLModel):
    name: str
    artist: str
    year: Optional[int] = None` 
```

不要忘记重要的一点:

```
`from typing import Optional` 
```

创建新的迁移文件:

```
`$ docker-compose exec web alembic revision --autogenerate -m "add year"` 
```

从自动生成的迁移文件中更新`upgrade`和`downgrade`函数，如下所示:

```
`def upgrade():
    # ### commands auto generated by Alembic - please adjust! ###
    op.add_column('song', sa.Column('year', sa.Integer(), nullable=True))
    op.create_index(op.f('ix_song_year'), 'song', ['year'], unique=False)
    # ### end Alembic commands ###

def downgrade():
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_index(op.f('ix_song_year'), table_name='song')
    op.drop_column('song', 'year')
    # ### end Alembic commands ###` 
```

应用迁移:

```
`$ docker-compose exec web alembic upgrade head` 
```

更新路线处理程序:

```
`@app.get("/songs", response_model=list[Song])
async def get_songs(session: AsyncSession = Depends(get_session)):
    result = await session.execute(select(Song))
    songs = result.scalars().all()
    return [Song(name=song.name, artist=song.artist, year=song.year, id=song.id) for song in songs]

@app.post("/songs")
async def add_song(song: SongCreate, session: AsyncSession = Depends(get_session)):
    song = Song(name=song.name, artist=song.artist, year=song.year)
    session.add(song)
    await session.commit()
    await session.refresh(song)
    return song` 
```

测试:

```
`$ curl -d '{"name":"Midnight Fit", "artist":"Mogwai", "year":"2021"}' -H "Content-Type: application/json" -X POST http://localhost:8004/songs` 
```

## 结论

在本教程中，我们介绍了如何配置 SQLAlchemy、SQLModel 和 Alembic 来异步使用 FastAPI。

如果你正在寻找更多的挑战，请查看我们所有的 FastAPI [教程](/blog/topics/fastapi/)和[课程](/courses/topics/fastapi/)。

您可以在[fastapi-SQL model-alem BIC](https://github.com/testdrivenio/fastapi-sqlmodel-alembic)repo 中找到源代码。干杯！