Стек: FastAPI, SQLAlchemy

Я подумал, что было бы неплохо создать небольшой блог на FastAPI. В качестве СУБД буду использовать PostgreSQL.
Для начала я создал структуру проекта:
```Bash
.
├── __init__.py
├── app
│   ├── __init__.py
│   ├── config.py
│   ├── database
│   │   ├── __init__.py
│   │   └── models.py
│   ├── homepage
│   │   ├── __init__.py
│   │   └── homepage.py
│   └── post
│       ├── __init__.py
│       └── post.py
└── main.py
```
Ее я подсмотрел у чувака, который начал делать свой Pinterest: https://github.com/shutsuensha/shutsuensha.ru/tree/main. У него огромный проект - я пока не собираюсь делать именно такой. Мне будет достаточно, если удастся реализовать создание постов + какие-нибудь комментарии, авторизацию и лайки.
# глава 1 посты
Я решил начать с постов. Для них создана отдельная папка - `post`. В ней будут храниться сами функции постов + схемы на Pydantic. Прежде чем писать функцию, которая будет отдавать все посты или добавлять новые посты, нужно сначала сделать структуру БД.
## структура БД
Структура БД у меня будет храниться в папке `database` в `models.py`. Сейчас модели выглядят так:
```Python
from sqlalchemy import BigInteger, String, Text, CheckConstraint
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Post(Base):
    __tablename__ = "post"
    post_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    title: Mapped[str] = mapped_column(String(150))
    content: Mapped[str] = mapped_column(Text())

    __table_args__ = (
        CheckConstraint("length(title) > 0", name="chk_length_title"),
        CheckConstraint("length(content) > 0", name="chk_length_content")
    )
```
Для таблицы `Post` пока что сделаны три простых поля - `post_id, title, content`:
1) `post_id` - `bigint` + `primary_key`. Такие настройки говорят ORM, что нужно прописать автоматическую генерацию ключей по полю + поставить в качестве типа поля `bigint`.
2) `title` - `String(150)` определяет, что в колонке в качестве типа должен стоять `varchar(150)`.
3) `content` - `Text()` определяет, что в колонке в качестве типа должен стоять `text`.
В константе `__table_args__` прописаны проверки для двух колонок - `title` и `content`. Проверки проверят, что во входящих значениях по этим полям больше нуля символов.
## а как создать БД?
Чтобы создать БД в SQLAlchemy нужно сначала создать движок. Грубо говоря - это ссылка на БД в формате строки. Она выглядит примерно как-то так:
```Python
from sqlalchemy import create_engine
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost:5432/mydatabase")
```
Компоненты ссылки:
1) `postgresql` - наименование СУБД.
2) `psycopg2` - движок, который будет использовать SQLAlchemy для подключения к БД.
3) `scott` - имя пользователя.
4) `tiger` - пароль для пользователя.
5) `localhost` - IP адрес хоста.
6) `5432` - порт.
7)  `mydatabase` - название БД.
Это все можно было бы закинуть в один файл с моделями, однако мне не очень хотелось превращать проект в помойку, поэтому я сделал отдельный файлик для запуска движка + запуска процесса создания таблиц.
```
.
├── __init__.py
├── app
│   ├── __init__.py
│   ├── config.py
│   ├── database
│   │   ├── __init__.py
│   │   ├── database.py  # новый файл здесь
│   │   └── models.py
│   ├── homepage
│   │   ├── __init__.py
│   │   └── homepage.py
│   └── post
│       ├── __init__.py
│       └── post.py
└── main.py
```
Туда я сразу закинул создание движка и создание сессий:
```Python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker


engine = create_engine()
session = sessionmaker(bind=engine, expire_on_commit=False)
```
Что такое `session`?
### сессии в SQLAlchemy
Сессия - это такая фича ORM, которая позволяет отправлять запросы в БД не по одному, а сразу скопом, отдавая на откуп ORM составление порядка и группировку запросов. Это означает, что ORM пойдет разбираться со всеми запросами, которые необходимо сделать к БД только когда мы обратимся к объекту сессии с просьбой закоммитить изменения в БД.
Сессия:
1) уберет лишние тысячу обращений в БД
2) отсортирует и сгруппирует запросы в нужном порядке
3) примет изменения в БД
Такой подход называется `Unit of Work`.
Еще одна позитивная сторона сессий - `Identity Map`. Карта идентичности гарантирует, что сессия будет наблюдать за данными, которые мы запрашиваем в код приложения из БД, и не позволит одним и тем же данным дублироваться в коде, забивая память.
### гениальный settings.py
Наверное, было заметно, что в последнем экземпляре кода не заполнены скобки у `create_engine()`. Я оставил их пустыми намеренно, потому что увидел реальную техномагию, которую очень хочу применять теперь всегда.
Когда я писал проекты раньше, то довольно часто делал файл `settings.py` или `constants.py`, где размещал все нужные приложению настройки и константы. Мне всегда казалось, что такой подход немого грубоват, но другого я не знал + всегда были большие проблемы, с которыми нужно было разобраться.
Однако я нашел парня, который сделал реальную бомбу в своем проекте - файл `config.py`: https://github.com/shutsuensha/shutsuensha.ru/blob/main/app/config.py. Кратко - он делает класс `Settings`, который унаследован от класса `BaseSettings` из Pydantic. Туда он заливает все основные константы по проекту. Но самое крутое, что все "секретные" константы берутся напрямую из файла `.env` с помощью интерфейса Pydantic.
Так как мне нужно так же хранить где-то свои секреты, я решил повторить успех и сделал такую же структуру в своем проекте:
```Python
from dotenv import load_dotenv
from pydantic-settings import BaseSettings, SettingsConfigDict


load_dotenv()

class Settings(BaseSettings):

    POSTGRES_DB_HOST: str
    POSTGRES_DB_NAME: str
    POSTGRES_DB_USER: str
    POSTGRES_DB_PASS: str
    POSTGRES_DB_PORT: int

    @property
    def POSTGRES_URL(self):
        return f"postgresql+psycopg://{self.POSTGRES_DB_USER}:{self.POSTGRES_DB_PASS}@{self.POSTGRES_DB_HOST}:{self.POSTGRES_DB_PORT}/{self.POSTGRES_DB_NAME}"

    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

settings: Settings = Settings()
```
Здесь есть сразу несколько интересных моментов:
1) библиотека  `dotenv` устанавливается вместе с пакетом `pydantic-settings`.
2) пакет `pydantic-settings` - это независимое DLC для Pydantic, которое можно использовать, чтобы писать красивые настройки проекта.
3) `dotenv` и `pydantic-settings` взаимоинтегрированы поэтому внутри класса `Settings` можно использовать класс `SettingsConfigDict`, чтобы подтянуть настройки из существующего `.env` файла. Эта механика прописана в доках: https://docs.pydantic.dev/latest/concepts/pydantic_settings/#dotenv-env-support.
4) использование свойства (`@property`) для того, чтобы вернуть сформированную строку через интерфейс точечной нотации из класса.
Теперь я могу создать тестовую базу через `psql`:
```SQL
postgres=# create user blog_admin with password 'mycoolp@ssword';
CREATE ROLE
postgres=# create database blog
postgres-# with
postgres-# encoding="UTF8"
postgres-# LC_COLLATE="ru_RU.UTF8"
postgres-# LC_CTYPE="ru_RU.UTF8"
postgres-# template=template0
postgres-# owner blog_admin;
CREATE DATABASE
```
И нужно открыть доступ к пользователю `blog_admin` в `pg_hba.conf`:
```Bash
host    all       blog_admin      0.0.0.0/0         md5
```
Теперь иду заполнять `.env`. Он должен лежать рядом с нашим файлом конфигурации `config.py` и лучше сразу его добавить в исключения `.gitignore`:
```
POSTGRES_DB_HOST="192.168.64.1"
POSTGRES_DB_NAME="blog"
POSTGRES_DB_USER="blog_admin"
POSTGRES_DB_PASS="mycoolp@ssword"
POSTGRES_DB_PORT=5432
```
### create_engine и создать БД
Теперь файл `database.py` выглядит так:
```Python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.config import settings


engine = create_engine(settings.POSTGRES_URL)
session = sessionmaker(bind=engine, expire_on_commit=False)
```
Все это очень круто и красиво, но я все еще не нашел нормального метода, который запускал бы коннект с БД всякий раз, когда я активирую сервер.
Для начала я добавил две функции - одна создает таблицы и базу данных (`create_db_and_tables()`) (хотя база данных у меня уже есть ), а другая выдает сессии для выполнения запросов (`get_session()`) + добавил специальную аннотированную константу - `SessionDep`:
```Python
def create_db_and_tables():
    Base.metadata.create_all(engine)


def get_session():
    with session_sync() as session:
        yield session


SessionDep = Annotated[session_sync, Depends(get_session)] 
```
`SessionDep` в будущем будет использоваться для обращений к БД из функций приложений.
`create_db_and_tables()` нужно еще откуда-то вызвать. Доки по FastAPI подсказывают, что настроить вызов такой функции можно специальным образом - при запуске сервера. Это я пропишу в `main.py`:
```Python
@app.on_event("startup")
def on_startup():
    database.create_db_and_tables()
```
Однако в доках такой метод называют устаревшим. Вместо него предлагается использовать `lifespan`. Грубо говоря он выполняет ту же функцию, что и `on_startup` - перед тем как приложение начнет получать реквесты будут сделаны некие действия. В моем случае это будет удаление БД и создание новой БД:
```Python
from contextlib import asynccontextmanager

from app.database import database


@asynccontextmanager
async def lifespan(app: FastAPI):
    database.delete_db_and_tables()
    database.create_db_and_tables()
    yield


app = FastAPI(lifespan=lifespan)
```
Теперь все готово. Можно запустить сервер:
```Bash
uvicorn main:app
```
Когда сервер будет запущен, в базе данных должна создаться таблица `post`. Я захотел это проверить, поэтому подключился к БД через `psql` и вывел список отношений:
```SQL
blog=# \d
                 List of relations
 Schema |       Name       |   Type   |   Owner
--------+------------------+----------+------------
 public | post             | table    | blog_admin
 public | post_post_id_seq | sequence | blog_admin
(2 rows)
```
Таблица создана. Ура-ура.
### проблемы
Моя первая проблема - неверная формулировка пароля для БД. Изначально я поставил пароль `mycoolp@ssword`, но не подумал о том, что строка, в которую помещается этот пароль вообще-то итак содержит `@`, но в качестве служебного символа:
```
"postgresql+psycopg2://scott:tiger@localhost:5432/mydatabase"
```
И это вызвало коллизию. Скорее всего эту строку можно как-то экранировать, но я не хотел этим заниматься на первых этапах, поэтому просто сменил пароль на БД.
Вторая проблема - пакет `psycopg2`. Это драйвер, который мы используем в SQLAlchemy и который по всей видимости нужно устанавливать отдельно. Однако его установка сама по себе может стать проблемой. Поэтому я воспользовался советом в сети - установить пакет `psycopg2-binary` вместо `psycopg2` и все сработало.
## а как создать асинхронную БД?
Этот вопрос я задал потому, что во всех гайдах, которые я посмотрел на данный момент гайдеры вообще не вдаются в различия между синхронной и асинхронной версиями и сразу лепят `async` везде, где его можно влепить.
Такую же привычку унаследовал и коннект к БД. Чтобы приконнектиться к БД можно использовать асинхронный движок `sqlalchemy` + можно настроить асинхронное создание таблиц и асинхронную выдачу сессий.
Теперь мой файл `database.py` выглядит так:
```Python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

from app.config import settings
from .models import Base


engine = create_async_engine(settings.POSTGRES_URL)

async_session_maker = async_sessionmaker(bind=engine, expire_on_commit=False)


async def get_session():
	"""
	Функция выдает сессии.
	"""
    async with new_session() as session:
        yield session


async def setup_db():
	"""
	Функция удаляет БД и создает снова новую.
	"""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)


SessionDep = Annotated[AsyncSession, Depends(get_session)]
```
Где я теперь использую `setup_db`? Все в том же `lifespan`. Функцию `lifespan` я разместил в файле `main.py`:
```Python
from fastapi import FastAPI
from contextlib import asynccontextmanager

from app.database import database

@asynccontextmanager
async def lifespan(app: FastAPI):
    await database.setup_db()
    yield


app = FastAPI(lifespan=lifespan)
```
Здесь опять понадобился асинхронный менеджер контекста + я добавил `await` к `database.setup_db()`.
Обязательно нужно поменять движок, который использует SQLAlchemy для получения доступа к БД. Строка, где он у меня упоминается находится в файле `config.py` в одном из свойств класса `Settings`:
```Python
    @property
    def POSTGRES_URL(self):
        return f"postgresql+asyncpg://{self.POSTGRES_DB_USER}:{self.POSTGRES_DB_PASS}@{self.POSTGRES_DB_HOST}:{self.POSTGRES_DB_PORT}/{self.POSTGRES_DB_NAME}"
```
Я заменил `psycopg2` на `asyncpg`.
## написать post-метод
Теперь вопрос: а как я могу отправить post-метод в эту таблицу, чтобы начать писать в нее данные? Изначально я создал модель `Post` в файлике `database/models.py`:
```Python 
from sqlalchemy import BigInteger, String, Text, CheckConstraint
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Post(Base):
    __tablename__ = "post"
    post_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    title: Mapped[str] = mapped_column(String(150))
    content: Mapped[str] = mapped_column(Text())

    __table_args__ = (
        CheckConstraint("length(title) > 0", name="chk_length_title"),
        CheckConstraint("length(content) > 0", name="chk_length_content")
    )
```
Эта крутая модель, как я говорил должна создать таблицу в моей БД с такими полями, которые я описал. Кстати, SQLAlchemy сама конвертирует типы питоновских данных в типы данных SQL, но делает она это в пределах разумного, то есть, если  я пишу `Mapped[int]` то полю в таблице будет задан тип `Integer` - для всех остальных integer-ов в SQL мне нужно дать пояснение, поэтому я написал `BigInteger`.
Теперь мне нужно написать сам запрос, который будет заполнять эти поля:
```Python
from fastapi import APIRouter

from .schemas import PostCreate
from ..database import database, models


router = APIRouter()


@router.post("/p")
async def add_post(post: PostCreate, session: database.SessionDep):
    new_post = models.Post(
        title=post.title,
        content=post.content
    )
    session.add(new_post)
    await session.commit()
```
Здесь написана асинхронная функция `add_post`. На вход она ожидает две сущности:
1) `PostCreate` - схема.
2) `SessionDep` - штука для зависимостей.
Внутри функции происходят простые вещи:
3) `new_post` - берет модель для БД `Post` и передает в нее данные, которые были получены от пользователя и трансформированы в словарь автоматически в переменной `post`.
4) `session.add(new_post)` обращаемся к сессии с просьбой добавить новые данные в БД.
5) `await.commit()` - применяем данные к БД.
### что такое схема?
Схема - это сущность Pydantic, которую FastAPI использует для валидации и форматирования входящих данных. 
Например, у нас есть задача принять от пользователя данные и каким-то образом их верифицирвать. Можно было бы придумать некий скрипт, написанный вручную, который с помощью строковых методов или чего-то такого проверяет пошагово каждый экземпляр данных. А можно поступить проще и написать класс Pydantic:
```Python
from pydantic import BaseModel


class PostCreate(BaseModel):
	title: str
	content: str
```
Готово. Теперь можно использовать этот класс в качестве аннотации к функции FastAPI, чтобы FastAPI автоматически верифицировал данные в теле входящего запроса + приводил бы их к нужным типам.
### что такое зависимость?
Когда мы обсуждаем зависимость в контексте `SessionDep`, которую я передал в параметры функции, нужно сначала вспомнить какое значение имеет эта переменная. Ее я прописал в файле `database.py`:
```Python
SessionDep = Annotated[AsyncSession, Depends(get_session)]
```
В данном случае мы как бы запрашиваем объект типа `AsyncSession` от функции `get_session`. Значение буквально говорит: переменная зависит от получения объекта `AsyncSession` из функции `get_session`. Сама функция `get_session` написана так:
```Python
async def get_session():
    async with async_session_maker() as session:
        yield session
```
Когда я пишу запрос к функции с параметром `session: SessionDep`, SQLAlchemy `get_session`сразу создает и отдает открывает сессию, с которой можно работать внутри функции.
### пишу post-запрос
Чтобы написать POST запрос я использую `curl`:
```Bash
curl -X POST -H "Content-Type: application/json" -d '{"title": "Мой первый пост!", "content": "Ура-ура!"}' http://127.0.0.1:8000/p
```
Параметры:
1) `-X` - параметр, который позволяет указать тип запроса.
2) `-H` - описываем дополнительный заголовок запроса - тип отправляемых данных.
3) `-d` - сами данные в нужном типе (в моем случае JSON).
В самом конце стоит ссылка на эндпоинт. После отправки запроса на сервер должен придти отчет:
```Bash
INFO:     127.0.0.1:55136 - "POST /p HTTP/1.1" 200 OK
```
## получаю посты
Чтобы получить посты нужно написать GET запрос. Я написал его в том же файле `post.py` и изначально он выглядел так:
```Python
@router.get("/p")
async def get_posts(session: database.SessionDep):
    result = await session.execute(text("""
        select post_id, title, content
        from post
    """))
    return result.scalars.all()
```
Однако такой запрос не вернет ожидаемых словарей с полями - он вернет список с идентификаторами записей. Я сначала удивился, потому что своровал этот способ у парня, который сказал, что именно так мы сможем достать записи из БД. На самом деле так мы достаем только их номера.
Как же реально достать записи из БД? Для этого нужно опять воспользоваться схемой Pydantic. Я написал вторую схему в файле `schemas.py`:
```Python
from pydantic import BaseModel


class PostCreateSchema(BaseModel):
    title: str
    content: str

# вторая схема
class PostSchema(PostCreate):
    post_id: int
```
Так как у нас в таблице всего три поля, а для создания поста пользователю не нужно указывать `post_id` (потому что он указывается СУБД автоматически), я решил, что можно унаследовать схему поста от схемы создания поста и добавить всего одно поле - `post_id`.
Теперь нужно импортировать схему в файл с функцией и упомянуть ее в декораторе:
```Python
@router.get("/p", response_model=list[PostSchema])
async def get_posts(session: SessionDep):
	result = await session.execute(text("""
		select post_id, title, content
		from post
	"""))
	return result.all()
```
Теперь FastAPI поймет к каким данным нужно преобразовать полученные из SQLAlchemy объекты (мы получаем итератор в результате работы `execute()`) и вернет красивый словарь:
```
[{"title":"Мой первый пост!","content":"Ура-ура!","post_id":1}]
```
### ORM и raw запросы
Я послушал штук 200 видео про FastAPI и все гайдеры наперебой избегают использования сырых SQL запросов в коде. У этого может быть несколько причин:
1) использование ORM запроса благоприятно влияет на какие-то фишки внутри SQLAlchemy.
2) гайдеры не хотят напрягать новичков лишним синтаксисом.
3) использование ORM - стандарт для современного разработчика.
Аргументом в пользу ORM часто выступает фраза "так проще, потому что мы пишем SQL запросы на Python", но т.к. я знаю SQL лучше, чем я знаю синтаксис SQLAlchemy ORM мне проще писать сырой SQL запрос и проще понять его в коде.
Поэтому в части получения постов я написал именно сырой SQL запрос:
```Python
@router.get("/p", response_model=list[PostSchema])
async def get_posts(session: database.SessionDep):
    result = await session.execute(text("""
        select post_id, title, content
        from post
    """))  # вот тут мой запрос.
    return result.all()
```
А как написать такой же запрос на ORM? Здесь есть свои сложности.
Чтобы написать такой же запрос нужно обратиться к модели, которую я написал ранее в `models.py` и импортировать из SQLAlchemy конструкцию `select`, которая является калькой на `select` из SQL:
```Python
from sqlalchemy import select


@app.router("/p", response_model=list[PostSchema])
async def get_posts(session: database.SessionDep):
	query = select(models.Post)
	result = session.execute(query)
	return result.all()
```
Если написать запрос так, то он вернет ошибку валидации:
```Python
fastapi.exceptions.ResponseValidationError: 3 validation errors:
  {'type': 'missing', 'loc': ('response', 0, 'title'), 'msg': 'Field required', 'input': (<app.database.models.Post object at 0x78f46526fb10>,)}
  {'type': 'missing', 'loc': ('response', 0, 'content'), 'msg': 'Field required', 'input': (<app.database.models.Post object at 0x78f46526fb10>,)}
  {'type': 'missing', 'loc': ('response', 0, 'post_id'), 'msg': 'Field required', 'input': (<app.database.models.Post object at 0x78f46526fb10>,)}
```
Из ошибки видно, что Pydantic модель, которая должна проверять данные из БД, не получает сами значения из полей БД. Вместо них она получает какой-то объект класса `Post` - `(<app.database.models.Post object at 0x78f46526fb10>,)`. То есть она буквально получает необработанный объект модели, который нужно по идее сначала распаковать и найти в нем нужные данные.
Чтобы это сделать я добавлю функцию `scalars()`:
```Python
return result.scalars().all()
```
Смысл ее работы в названии: скаляр - это одномерный массив. В контексте нашей функции к такому массиву будут преобразованы полученные из БД значения. Результат будет такой же, какой был при использовании SQL запроса.
Так же, `scalars()` можно применить вместо `execute()`, если мы ожидаем получить именно простые значения, а не объекты модели:
```Python
result = session.scalars(query)
return result.all()
```
### чтобы получить один пост
Получение одного поста я планирую сделать по индексу - запрос должен содержать параметр пути, в котором должен быть передан индекс. То есть запрос будет выглядеть примерно так:
```
http://127.0.0.1:8000/p/{index}
```
Здесь возникает несколько нюансов:
1) я хочу, чтобы в мою функцию попали именно числа, а не любой ввод.
2) данные придется достать из БД по индексу.
Чтобы решить первую проблему нужно просто добавить проверку типов и конвертер пути к функции. Конвертер будет выглядеть так:
```Python
@router.get("/p/{index:int}")
```
Запись `:int` указывает, что этот шаблон пути подойдет только в том случае, если эту часть пути в пользовательском запросе можно преобразовать к числу. Дополнительно можно добавить типизацию внутрь функции:
```Python
@router.get("/p/{index:int}")
async def get_post(index: int, session: database.SessionDep):
```
Конвертер и типизация будут проверять входящий параметр на соответствие числовому типу.
Как написать запрос на получение поста с конкретным идентификатором? В SQL такой запрос выглядел бы так:
```SQL
select post_id, title, content from post
where post_id = index;
```
Но я хочу написать такой же, но на ORM SQLAlchemy. Синтаксис ORM предлагает примерно схожую концепцию:
```Python
@router.get("/p/{index:int}")
async def get_post(index: int, session: database.SessionDep):
	query = select(models.Post).where(models.Post.post_id == index)
	result = await session.scalar(query)
	return result
```
# глава 2 разнообразие БД
Я пока создал только одну таблицу в БД, но сразу понятно, что так сервисы с постами не работают. К каждому посту должен быть прикреплен его автор. Это означает, что помимо таблицы с постами должна быть реализована так же и таблица с пользователями.
Она будет выглядеть проще, чем модель постов, но это только пока:
```Python
class User(Base):
	__tablename__ = "users"

	user_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
	username: Mapped[str] = mapped_column(String(200), unique=True)

	__table_args__ = (
		CheckConstraint("length(username) > 0", name="chk_length_username"),
	)
```
Стоит дать пару комментариев по поводу именования таблиц. Таблицу `post` я назвал именно `post` изначально, а для таблицы `users` я выдал название во множественном числе. Почему? Потому что таблицу `user` PostgreSQL не даст создать в БД. Теперь, после того как я назвал во множественном числе таблицу пользователей, я должен назвать во множественном числе и таблицу постов, потому что название таблиц должны быть согласованными по синтаксису - так будет проще обращаться к таблицам, когда их станет очень много.
Далее, я поставил после `CheckConstraint` запятую - это способ создать кортеж в Python. `__table_args__` требует на вход именно последовательность, поэтому если не указывать запятую после `CheckConstraint` SQLAlchemy просигналит об ошибке. Так происходит, потому что Python автоматически не учитывает скобки, если в них упомянут только один элемент.
## как связать БД?
Теперь возникает вопрос: а как указать, что поле из таблицы постов теперь будет ссылаться на пользователя из таблицы пользователей? В SQL такая механика реализовывается через конструкцию `references` при создании БД. В этом случае таблица постов создавалась бы так:
```SQL
create table posts(
	post_id bigint generated always as identity primary key,
	title varchar(200) not null check(length(title) > 0),
	content text not null check(length(content) > 0),
	user_id bigint not null references users(user_id)
)
```
Однако как реализовать то же самое, но на языке моделей SQLAlchemy? Я написал так:
```Python
class Post(Base):
    __tablename__ = "posts"
    
    post_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    title: Mapped[str] = mapped_column(String(150))
    content: Mapped[str] = mapped_column(Text())

    user_id: Mapped[int] = mapped_column(
            BigInteger,
            ForeignKey("users.user_id", ondelete="cascade")
    )

    __table_args__ = (
        CheckConstraint("length(title) > 0", name="chk_length_title"),
        CheckConstraint("length(content) > 0", name="chk_length_content")
    )
```
Объяснение по `user_id`:
1) `ForeignKey` - название отражает термин "внешний ключ".
2) Первым аргументом я передал поле первичного ключа из таблицы `users`.
3) Вторым аргументом я передал настройку `ondelete="cascade"`. Она означает, что при удалении пользователя так же будут удалены все связанные с ним данные, т.е. в данном случае его посты.
4) Тип данных ключа `BigInteger` должен совпадать с типом данных в таблице `users`.
## обновление модели Post
Для базы данных по постам я дописал еще поле - поле для фиксации времени создания поста:
```Python
class Post(Base):
    __tablename__ = "posts"
    
    post_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    title: Mapped[str] = mapped_column(String(150))
    content: Mapped[str] = mapped_column(Text())
    created_at: Mapped[datetime] = mapped_column(
        TIMESTAMP(timezone=True), 
        default=lambda: datetime.now(timezone.utc)
    )  # здесь

    user_id: Mapped[int] = mapped_column(
            BigInteger,
            ForeignKey("users.user_id", ondelete="cascade")
    )

    __table_args__ = (
        CheckConstraint("length(title) > 0", name="chk_length_title"),
        CheckConstraint("length(content) > 0", name="chk_length_content")
    )
```
Объяснение по `created_at`:
1) `TIMESTAMP` - сущность из SQLAlchemy, которая определяет тип поля.
2) `default` - параметр, который будет устанавливать для поста значение по умолчанию, если такое значение не было передано вместе с запросом.
3) Внутри `default` лежит анонимная `lambda`, которая будет брать время из функции `now()` модуля `datetime`.
Остается только одна проблема - часовой пояс. И я так и не нашел как можно вставить нормальную поправку на время, потому что все мои попытки дополнить конструкцию `datetime.now()` нормальной таймзоной в итоге превратились все в то же дефолтное время UTC. Стоит прочитать заметку: https://stackoverflow.com/a/462028/30499559
После установки времени я пошел дополнять схемы для `post`:
```Python
from datetime import datetime

from pydantic import BaseModel


class PostSchema(BaseModel):
    post_id: int
    user_id: int  # поле для юзера
    created_at: datetime  # поле для времени
    title: str
    content: str

    class Config:
        from_attributes = True


class PostCreateSchema(BaseModel):
    user_id: int  # поле для юзера
    title: str
    content: str
```
И немного исправил функцию, которая должна добавлять посты в БД:
```Python
@router.post("/p")
async def add_post(post: PostCreateSchema, session: database.SessionDep):
    new_post = models.Post(
        title=post.title,
        content=post.content,
        user_id=post.user_id  # новое поле
    )
    session.add(new_post)
    await session.commit()
```
## приложение user
Для приложения `user` я сделал отдельную папку. Структура всего проекта теперь выглядит так:
```
.
├── app
│   ├── config.py
│   ├── database
│   │   ├── database.py
│   │   ├── __init__.py
│   │   ├── models.py
│   ├── homepage
│   │   ├── homepage.py
│   │   ├── __init__.py
│   ├── __init__.py
│   ├── post
│   │   ├── __init__.py
│   │   ├── post.py
│   │   └── schemas.py
│   └── user  # новое приложение
│       ├── __init__.py
│       ├── schemas.py
│       └── user.py
├── __init__.py
├── main.py
└── requirements.txt
```
Внутри - те же файлы, которые я делал для `post`. В `schemas.py` сохранены схемы для создания пользователя и выдачи информации по пользователю:
```Python
from pydantic import BaseModel


class UserSchema(BaseModel):
    user_id: int
    username: str

    class Config:
       from_attributes = True


class UserCreateSchema(BaseModel):
    username: str
```
Кстати, `from_attributes` - это настройка, которая обеспечивает интеграцию схем Pydantic и моделей ORM. Она необходима, потому что Pydantic изначально не может производить валидацию экземпляров класса, т.е. он не может взять экземпляр и проверить его атрибуты. Так происходит в стандартном варианте, потому что Pydantic ждет на вход либо словарь, либо JSON объект.
Далее, я сделал функцию для создания пользователя в `user.py`:
```Python
@router.post("/user")
async def add_user(user: UserCreateSchema, session: database.SessionDep):
    new_user = models.User(
        username = user.username
    )
    session.add(new_user)
    await session.commit()
```
## миграции
Я должен был об этом задуматься раньше, но мне так нравилось писать простое приложение с простыми SQL запросами, с использованием `psql`, что я совсем забыл обо всем остальном.
Когда приложение масштабируется и в него требуется вносить правки, зачастую эти самые правки касаются состояния БД. Например, внезапно на ресурс пришла куча пользователей и СУБД начала захлебываться от количества запросов. Причины могут быть разные. Одна из них - плохая инфраструктура БД.
Например, у нас есть одна масштабная таблица для информации по всем пользователям. В этой таблице находятся сущности, которые мы постоянно запрашиваем из БД, например, имя пользователя и его почта, а еще сущности, которые нам практически не нужны при работе пользователя с сервисом, например, его место рождения, дата свадьбы и т.д. Такую таблицу стоит перестроить (разбить на таблицы поменьше и соединить их связью один к одному) и неплохо было бы протестировать новую постройку.
В данный момент мои таблицы удаляются и снова создаются при запуске сервера. Такой подход хорош для быстрой практики, однако на реальном сервисе - это отстой. Неплохо было бы иметь такой инструмент, который может проанализировать наши модели, составить по этим моделям запросы к БД, сделать эти запросы, а еще сохранить предыдущее состояние БД, чтобы к нему можно было откатиться если что. 
Именно такую функцию выполняет встроенный в Django инструмент для миграций. В FastAPI такого встроенного инструмента нет + у меня есть внешний ORM от SQLAlchemy, который было бы неплохо еще и интегрировать с новым инструментом. К счастью мудрые белые люди сделали Alembic.
### что такое Alembic?
Alembic - это универсальный инструмент для выполнения миграций. В идеале он должен посмотреть в наши модели, проанализировать изменения, которые мы внесли в них, затем создать миграционные файлы (буквально запросы SQL) и применить их на нашу БД. Формально - это именно то, что сейчас выполняет моя функция на бесконечное удаление и создание таблиц в БД, но гораздо лучше, потому что выполнение миграций - это контролируемый процесс, который внесет обновления в БД, только после соответствующей команды.
### что такое миграция?
Миграция - это буквально файл, который содержит информацию об изменениях, которые нужно внести в БД на основании анализа моделей SQLAlchemy. Его будет генерировать Alembic по шаблону, который уже есть по умолчанию при установке Alembic + который мы можем изменить при большом желании (он называется `script.py.mako`).
### поставить и подключить alembic
Он ставится довольно просто:
```Bash
python -m pip install alembic
```
Дальше нужно инициализировать первичные файлы Alembic:
```Bash
alembic init <имя папки, где будут лежать файлы>
```
Я положил файлы в папку `app/migrations`. Структура проекта теперь выглядит так:
```Bash
.
├── __init__.py
├── alembic.ini  # файл конфигураций Alembic
├── app
│   ├── __init__.py
│   ├── config.py
│   ├── database
│   │   ├── __init__.py
│   │   ├── database.py
│   │   └── models.py
│   ├── homepage
│   │   ├── __init__.py
│   │   └── homepage.py
│   ├── migrations  # файлы Alembic
│   │   ├── README
│   │   ├── env.py
│   │   ├── script.py.mako
│   │   └── versions
│   ├── post
│   │   ├── __init__.py
│   │   ├── post.py
│   │   └── schemas.py
│   └── user
│       ├── __init__.py
│       ├── schemas.py
│       └── user.py
├── main.py
└── requirements.txt
```
В файле конфигураций `alembic.ini` нужно указать директорию, где содержаться остальные файлы `alembic`:
```
script_location = app/migrations
```
Далее можно идти в `app/migrations/env.py`, чтобы выставить конфигурацию проекта. Изначально Alembic собирается под синхронные движки SQLAlchemy, т.е. изначально он не подходит для работы с нашим асинхронным движком `asyncpg`. В документации к Alembic существует страница, которая посвящена специально этому вопросу: https://alembic.sqlalchemy.org/en/latest/cookbook.html#using-asyncio-with-alembic.
Так как я изначально побежал делать все в синхронном виде, не подумав, что мне нужно держать в голове асинхронный движок БД, теперь мне нужно изменить файл `env.py` с помощью экземпляров из документации:
```Python
import asyncio
from logging.config import fileConfig

from sqlalchemy.ext.asyncio import async_engine_from_config
from sqlalchemy.engine import Connection
from sqlalchemy import pool
from alembic import context

from app.database import models
from app.config import settings


config = context.config
postgresql_url_escaped = settings.POSTGRES_URL.replace('%', '%%')
config.set_main_option(
	"sqlalchemy.url", f"{postgresql_url_escaped}"
)
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = models.Base.metadata


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations():
	"""
	Запустить асинхронные миграции.
	"""
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await connectable.dispose()


def run_migrations_online():
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```
Из интересного: при передаче строки коннекта к БД у меня возникла проблема с использованием служебного символа `%` в пароле, поэтому я применил функцию  `replace()`, чтобы избежать конфликта:
```Python
postgresql_url_escaped = settings.POSTGRES_URL.replace('%', '%%')
config.set_main_option(
	"sqlalchemy.url", f"{postgresql_url_escaped}"
)
```
После первичных настроек можно сгенерировать первые миграции:
```Bash
alembic revision --autogenerate -m "init migrations"
```
И применить их последнюю версию:
```Bash
alembic upgrade head
```
Теперь можно убрать из процесса создания и удаление БД удаление БД. Для этого я в `database.py` просто закомментирую строку:
```Python
async def setup_db():
    async with engine.begin() as conn:
        # await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)
```
### вопросы
Я задался основным вопросом: "А что будет с данными, которые уже есть в БД, если для модели добавить новое поле?". Сейчас в моей БД есть одна запись о пользователе:
```SQL
blog=> table users;
 user_id | username
---------+----------
       1 | peter
(1 row)
```
Допустим я добавлю новое поле `email`. Что произойдет с пользователем `peter`, после применения миграций? Я добавил новую строку в модель юзера:
```Python
class User(Base):
    __tablename__ = "users"

    user_id: Mapped[int] = mapped_column(
	    BigInteger, primary_key=True
	)
    username: Mapped[str] = mapped_column(
	    String(200), unique=True
	)
    email: Mapped[str] = mapped_column(
	    String(200), unique=True
	)  # новое обязтельное поле

    __table_args__ = (
        CheckConstraint("length(username) > 0", name="chk_length_username"),
    )
```
Обновил схему:
```Python
from pydantic import BaseModel, EmailStr


class UserSchema(BaseModel):
    user_id: int
    username: str
    email: EmailStr

    class Config:
       from_attributes = True


class UserCreateSchema(BaseModel):
    username: str
    email: EmailStr
```
>[!important] К слову для использования `EmailStr` нужно установить не просто пакет `pydantic`, а пакет `pydantic[email]`.

И переписал метод `post` для создания юзера:
```Python
@router.post("/user")
async def add_user(user: UserCreateSchema, session: database.SessionDep):
    new_user = models.User(
        username = user.username
        email = user.email  # новая строка
    )
    session.add(new_user)
    await session.commit()
```
Нужно сгенерировать новые миграционные файлы и применить их к БД:
```Bash
alembic revision --autogenerate -m "user model updated email field added"
alembic upgrade head
```
И получить ошибку:
```Bash
sqlalchemy.exc.IntegrityError: (sqlalchemy.dialects.postgresql.asyncpg.IntegrityError) <class 'asyncpg.exceptions.NotNullViolationError'>: column "email" of relation "users" contains null values
[SQL: ALTER TABLE users ALTER COLUMN email SET NOT NULL]
```
Состояние БД при этом не изменится - Alembic просто не даст сделать миграцию, пока в БД есть сущности для которых не заполнены поля с электронной почтой.
Вместо того, чтобы делать поле для почты обязательным я хочу переделать его в необязательное.
Сначала изменю поле в модели:
```Python
email: Mapped[str | None] = mapped_column(
	String(200), unique=True, default=None
)
```
И добавлю новые значения в схему:
```Python
email: EmailStr | None = None
```
Теперь казалось бы нужно применить миграции и запушить новое состояние БД, однако все не так просто.
### список изменений и откат изменений
Из предыдущих примеров у нас осталась одна трешовая миграция - та, где я написал новое обязательное поле для модели. Она не применилась и упала с ошибкой, состояние БД осталось таким же, каким и было. Вроде все ок, но если попытаться применить новые миграции поверх этой трешовой, то получится следующая ошибка:
```Bash
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
ERROR [alembic.util.messaging] Target database is not up to date.
  FAILED: Target database is not up to date.
```
Мудрые люди из Stack Overflow [подсказывают](https://stackoverflow.com/a/17776558), что дело буквально в том, что мы обязаны после создания миграций принять их с помощью:
```Bash
alembic upgrade head
```
А если этот процесс упал с ошибкой (например, при создании миграции мы сделали что-то не то в моделях), то нужно напомнить Alembic о текущем состоянии БД. Чтобы это сделать нужно вбить команду:
```
alembic stamp head
```
Она подсказывает Alembic в каком состоянии находится сейчас БД. Вместо `head` можно поставить номер любой версии миграций. Сама команда не вносит никаких изменений в состояние БД и данные в ней, поэтому использовать ее безопасно.
Чтобы посмотреть какие вообще есть сейчас версии нужно вбить:
```Bash
alembic history
```
В моей истории отображены такие миграции:
```Bash
((.venv) ) ➜  blog git:(main) ✗ alembic history
b0b8662bd8eb -> cebf4db64277 (head), User.email default=None
99c42659819b -> b0b8662bd8eb, user model update email field update now its must be fullfilled
00abdff2e5c6 -> 99c42659819b, user model update email field was added
f3d60a9cb63a -> 00abdff2e5c6, user update email field was added
<base> -> f3d60a9cb63a, Initial migration
```
В самом верху списка текущая миграция `(head)`, а  в самом низу миграция `<base>` - инициирующая миграция. У всех миграций есть свои номера. Номер миграции отражен сразу после стрелки, а перед стрелкой стоит номер предыдущей миграции.
Чтобы выполнить откат БД к предыдущей версии нужно седлать команду:
```Bash
alembic downgrade -1
```
Это вернет состояние БД на версию назад. Можно использовать другие числа, чтобы вернуться на несколько шагов назад, а можно вставить идентификатор транзакции вместо числа и вернуться к ней.
`alembic history` не показывает ту ревизию, в которой сейчас находится состояние БД. Но эту ревизию может показать:
```
alembic current
```
## создание join
Допустим у меня есть две связанные таблицы, однако данные, которые я получаю от эндпоинта для постов возвращаются ко мне не с именем пользователя, а с его идентификатором. Для того, чтобы получить именно имя вместе с постом (+ может быть еще и почту) мне нужно сделать `join` таблиц `users` и `posts`.
Но еще я хочу научится правильно делать `select` по таблицам, с помощью ORM. Например, сейчас мой запрос к БД `posts` выглядит в общем случае так:
```Python
select(models.Post)
```
Что не очень хорошо с точки зрения SQL, потому что мы таким образом выбираем все поля из таблицы, даже те которые нам могут быть не нужны.
При попытке понять тему я руководствовался этой статьей: https://habr.com/ru/companies/true_engineering/articles/226521/.
### select
Самое банальное, что я попробовал сделать - написать такой запрос на чтение данных, какой я бы написал на SQL:
```Python
query = select(
	models.Post.post_id,
	models.Post.title,
	models.Post.content
)
```
Что примерно равно:
```SQL
select
	post_id,
	title,
	content
from posts;
```
И такой подход сработает. Не знаю насколько он верный. Однако он больше всего мне напоминает SQL, поэтому я решил и дальше действовать так.
### join
Чтобы сделать `join` нужно написать метод `join()` и передать ему на вход два аргумента - таблицу, с которой будет происходить соединение + признак, по которому оно будет происходить:
```Python
query = select(
	models.Post.post_id,
	models.User.username,
	models.User.email,
	models.Post.created_at,
	models.Post.title,
	models.Post.content
)
query = query.join(
	models.User, models.Post.user_id == models.User.user_id
)
```
Возможно, эту схему можно расписать попроще, т.к. в данный момент в моем коде сразу две функции содержат код выше.
### where
Для функции, которая принимает параметр пути - `post_id`, я должен прописать еще метод `where()`. Он должен найти нужный пост и отдать его обратно:
```Python
query = select(
	models.Post.post_id,
	models.User.username,
	models.User.email,
	models.Post.created_at,
	models.Post.title,
	models.Post.content
)
query = query.join(
	models.User, models.Post.user_id == models.User.user_id
)
query = query.where(
	models.Post.post_id == index
)
```
В общем случае порядок применения методов такой же как порядок стандартного SQL запроса.
### проблемы
По какой-то причине вместе с изменением модели данных (в т.ч. вместе с изменением схемы) изменилось и поведение `scalar()` и `scalars()`. Методы возвращают либо индексы, либо пустые данные. Дело сто процентов в моих кривых руках. Пытаясь разобраться я пришел к решению заменить `scalar()` и `scalars()` на старый `execute()` и использовать `all()` в случае с функцией, которая возвращает все посты, + `one()` в случае с функцией, которая возвращает только один пост. Получилось так:
```Python
@router.get("/p", response_model=list[PostGetSchema])
async def get_posts(session: database.SessionDep):
    query = select(
        models.Post.post_id,
        models.User.username,
        models.User.email,
        models.Post.created_at,
        models.Post.title,
        models.Post.content
    )
    query = query.join(models.User, models.Post.user_id == models.User.user_id)
    result = await session.execute(query)
    return result.all()


@router.get("/p/{index:int}", response_model=PostGetSchema)
async def get_post(index: int, session: database.SessionDep):
    query = select(
        models.Post.post_id,
        models.User.username,
        models.User.email,
        models.Post.created_at,
        models.Post.title,
        models.Post.content
    )
    query = query.join(models.User, models.Post.user_id == models.User.user_id)
    query = query.where(models.Post.post_id == index) 
    result = await session.execute(query)
    return result.one()
```
Так же есть проблема с очевидным повторением одного и того же кода в двух случаях. Лучше всего было бы ее решить с помощью отдельной функции, но мне бы не хотелось добавлять к двум уже существующим методам третий промежуточный шаг.
# глава 3 авторизация и аутентификация
Я всегда путал эти два процесса и мне всегда нужна была подсказка, чтобы их различить:
1) авторизация - это процесс предоставления пользователю прав доступа к определенным ресурсам или действиям.
2) аутентификация - это процесс проверки подлинности пользователя.
Грубо говоря, авторизация - это все те процессы, которые связаны с распределением прав пользователей на ресурсе: что может делать анонимный пользователь? что может делать администратор? а вот этот пользователь может изменить комментарий другого пользователя под постом?
Аутентификация же - это проверка пользователя на то действительно ли он является тем пользователем, за которого себя выдает. Такая проверка обычно выполняется с помощью паролей, логинов, писем на почту и т.д.
Оба процесса связаны, но теоретически могут существовать и автономно. Например, наш ресурс может не аутентифицировать пользователей.
## basic auth
Под базовой аутентификацией имеется в виду настройка аутентификации через Pydantic схему `HTTPBasicCredentials` и класс `HTTPBasic`.
Схема `HTTPBasicCredentials` представляет собой простой набор из двух полей - `username` и `password`:
```Python
class HTTPBasicCredentials(BaseModel):
	username:str
	password:str
```
Класс `HTTPBasic` принимает внутрь себя запрос (`request`) от пользователя, берет из этого запроса заголовок `Authorization` (куда по идее должно быть передано имя юзера и его пароль), проверяет их и пускает на ресурс (или отказывает в доступе).
Перед началом важно понимать, что пока никакой валидации данных (юзернейма и пароля) я не делал. В данной ситуации `HTTPBasic` выполняет просто роль шлагбаума перед нужным ресурсом.
### реализация простого входа
Допустим у меня появилась задача защитить один из моих ресурсов на сервисе. Для этого я могу использовать концепт `basic auth`:
```Python
from typing import Annotated

from fastapi import FastAPI, Depends
from fastapi.security import HTTPBasicCredentials, HTTPBasic


app = FastAPI()

security = HTTPBasic()
@router.get("/some_resource")
def get_info_from_some_resource(
	credentials: Annotated[HTTPBasicCredentials, Depends(security)]
):
	return {
		"message": "Hi!",
		"username": credentials.username,
		"password": credentials.password
	}
```
Теперь при попытке зайти на этот эндпоинт будет вызвана зависимость `HTTPBasic()`, которая покажет на странице форму для заполнения имени пользователя и пароля. 
Пока что не играет никакой роли то какие данные будут положены в форму - форма все равно пропустит пользователя. Однако теперь, когда я захочу снова зайти на ресурс, то `HTTPBasic` пропустит меня без повторения процедуры заполнения пароля. Почему? Потому что теперь вместе с запросом я передаю заполненный заголовок `Authorization`, который `HTTPBasic` может распарсить. Прочесть заголовок можно через инструменты разработчика:
![[basic auth header.PNG]]
Этот заголовок на самом деле является просто закодированной версией переданных мной данных. Проверить что я передал можно с помощью утилиты `base64` в Unix системах:
```Bash
echo <шифр> | base64 -d
```
У меня был такой ввод:
```Bash
echo cGV0ZXI6 | base64 -d
peter:%  # знак процента озанчает конец строки
```
Данные для входа на ресурс можно вставлять не только в форму - их еще можно предавать прямо вместе с запросом. Например, в адресной строке можно написать так:
```
http://john:doe@127.0.0.1:8000/some_resource
```
Так система на данном этапе посчитает, что мы совершили вход под другим пользователем. В этом случае считается, что перед двоеточием стоит `username`, а после `password`.
Когда мы попытаемся снова зайти на ресурс, то нас снова пропустят без проблем, потому что в данном случае наш браузер хранит ту самую закодированную строку и передает ее вместе с нашим запросом к ресурсу. Строка попадает на сервер, декодируется в `HTTPBasic`, данные в ней совпадают с теми данными, которые мы использовали в первый раз при заходе, и мы попадаем на ресурс без необходимости вводить имя пользователя и пароль снова.
### проверка данных входа
В [гайде](https://www.youtube.com/watch?v=Fg4tfUtJiT8), который я смотрел, по аутентификации автор решил перенести главу с хэшированием паролей и выбором паролей из БД на потом. У меня не слишком много терпения чтобы ждать, поэтому я решил применить все свои знания + документацию по FastAPI, чтобы прикрутить к текущему варианту авторизации серьезное хэширование и проверку паролей.
#### что такое хэш?
Было бы круто брать пароль пользователя при регистрации и сразу кидать в БД. Вполне логично - можно было бы доставать пароль при необходимости аутентификации пользователя. Однако что если над сервисом будет работать слишком много сотрудников, часть из которых имеет доступ к БД? А если злоумышленник получит БД? Звучит не очень круто. Как сделать так, чтобы БД хранила пароли, но при этом не давала бы возможность так просто зайти в таблицу и посмотреть данные пользователей в открытом виде?
Для реализации такой функции можно использовать хэш.
Хэш - это строка из символов, в которую преобразуется изначальная строка. Грубо говоря мы выполняем шифрование изначальных данных, но с той разницей, что итоговый вариант нельзя расшифровать. Хэширование от шифрования отличается именно этим: нельзя "расхэшировать" захэшированные данные.
В самом простом случае выглядит это так:
```Python
>>> from passlib.hash import pbkdf2_sha256
>>> hash = pbkdf2_sha256.hash("12345qwerty")
>>> hash
'$pbkdf2-sha256$29000$w9ib05pTqpXyPidEqNUagw$.qZHKYQFaegIaW5ZvS3J2EzwFZWgZbOtKSWtdQUauOs'
```
`pbkdf2_sha256` - это алгоритм хэширования, т.е. алгоритм преобразования изначальных данных в непонятные последовательности символов.
Важными особенностями хэша является:
1) Последовательности символов не случайны, поэтому все попытки составить несколько раз хэш от пароля `12345qwerty` вернут один и тот же результат.
2) Последовательности символов фиксированы. Не важно насколько длинны изначальные данные - хэш всегда будет содержать одинаковое количество символов с другими хэшами от других данных.
3) При малейшем изменении исходных данных изменится и сама последовательность.
Как сравнить пароль введенный пользователем и тот захэшированный пароль, что принадлежит ему в нашей БД? Нужно просто преобразовать введенный пароль в хэш и сравнить два хэша.
#### дополнение бд пароль
Прежде чем проверять пароли нужно сначала сделать инфраструктуру для их хранения - для модели и Pydentic схемы пользователя я добавил новые поля `password`. Оба поля выглядят так:
```Python
# модель
password: Mapped[str] = mapped_column(String(200))
# схема
class UserCreateSchema(BaseModel):
    username: str
    password: str
    email: EmailStr | None = None
```
Так как пароль теперь является обязательным полем для заполнения, нужно очистить от данных таблицу пользователей. Я сделал это через `psql`:
```SQL
truncate users restart identity cascade;
```
Теперь нужно создать миграционные файлы и применить миграции:
```Bash
alembic revision --autogenerate -m "User.password"
alembic upgrade head
```
И еще внести немного изменений в функцию по добавлению пользователей:
```Python
new_user = models.User(
	username = user.username,
	password = user.password,  # новое поле.
	email = user.email
)
```
А теперь можно писать запрос к эндпоинту:
```Bash
curl -X POST -H 'Content-Type: application/json' -d '{"username": "Steve", "email": "CoolEmail@gmail.com", "password": "12345qwerty"}'
```
Проверяю через `psql`:
```SQL
blog=> table users;
 user_id | username |        email        |  password
---------+----------+---------------------+-------------
       1 | Steve    | CoolEmail@gmail.com | 12345qwerty
(1 row)
```
#### хэширование пароля
Для хэширования я буду использовать библиотеку `passlib`.
Чтобы захэшировать какие-то данные нужен алгоритм хэширования и в качестве такого я буду использовать `bcrypt`. Поддержку этого алгоритма можно установить для `passlib` такой командой:
```Bash
python3.13 -m pip install "passlib[bcrypt]"
```
После установки нужно создать объект `CryptContext` мы дадим понять, что нужно использовать `bcrypt` в качестве алгоритма хэширования:
```Python
from passlib.context import CryptContext


pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
```
По сути параметр `deprecated` здесь не нужен, однако я его решил добавить, потому что он полезен, когда у нас есть много пользователей на ресурсе + несколько схем хэширования. `deprecated` помечает все схемы в списке схем, кроме первой, как устаревшие схемы хэширования. При проверке хэша с помощью `verify()` может случится так, что он записан с помощью другого алгоритма хэширования. В этом случае `deprecated` автоматически пометит хэш, как устаревший.
Устаревшие хэши можно менять сразу с помощью функции `verify_and_validate()` - она возвращает кортеж, где первое значение будет `True/False` (в зависимости от успеха процедуры верификации), а вторым значением будет обновленный хэш.
Я написал две функции - одна создает хэш из строки, а другая проверяет совпадает ли хэш одной строки с сохраненным хэшем у нас в БД:
```Python
def verify_password(
    plain_password: str, hashed_password: str
) -> tuple[bool, str | None]:
    return pwd_context.verify_and_update(plain_password, hashed_password)


def get_password_hash(
    password: str
) -> str:
    return pwd_context.hash(password)
```
Функцию `get_password_hash` сразу можно вызвать внутри функции для эндпоинта на создание пользователя, чтобы в БД отправлялись хэшированные версии паролей:
```Python
@router.post("/user")
async def add_user(
    user: UserCreateSchema,
    session: database.SessionDep
):
    hashed_password = get_password_hash(
        user.password
    )  # здесь делаем хэш
    new_user = models.User(
        username = user.username,
        password = hashed_password,  # здесь вставляем его в БД
        email = user.email
    )
    session.add(new_user)
    await session.commit()
```
Добавляю пользователя:
```Bash
curl -X POST -H "Content-Type: application/json" -d '{"username": "peter2", "email": "peter2@gmail.com", "password": "12345qwerty"}' http://127.0.0.1:8000/user
```
Проверяю через `psql`:
```SQL
blog=> table users;
 user_id | username |        email        |                           password
---------+----------+---------------------+--------------------------------------------------------------
       1 | Steve    | CoolEmail@gmail.com | 12345qwerty
       2 | peter    | peter@gmail.com     | $2b$12$LTYVftJh186LP/BxSH.AcuUqSlb90g8lx5Wsx1zwVd00T2anMA60S
       4 | peter2   | peter2@gmail.com    | $2b$12$BgnG8h/ZUUx.C1CNL9ewEOz59.CuTFEQPxHsjcJi04rcV1M.ohciG
(3 rows)
```
Важно! Здесь можно заметить, что я в первый раз добавил пользователя `peter`, однако в текст вошел именно вариант `peter2`. Почему? Потому что для хэширования пароля `peter` я использовал версию алгоритма `bcrypt==4.3.0` - это самая последняя версия на момент написания текста и она устанавливается вместе с пакетом `passlib`. Так вот похоже, что `bcrypt` обновляется быстрее, чем `passlib`, поэтому при использовании последней версии `bcrypt` возникает некритичная ошибка:
```Python
(trapped) error reading bcrypt version
Traceback (most recent call last):
  File "/home/lipkerton/lessons/fastapi_test_projects/blg_1/.venv/lib/python3.13/site-packages/passlib/handlers/bcrypt.py", line 620, in _load_backend_mixin
    version = _bcrypt.__about__.__version__
              ^^^^^^^^^^^^^^^^^
AttributeError: module 'bcrypt' has no attribute '__about__'
```
Хотя ошибка пока никак не влияет на процедуру хэширования, я не стал рисковать и снизил версию `bcrypt` на `bcrypt==4.0.1` по советам умных людей: https://github.com/pyca/bcrypt/issues/684
#### реализация проверки
Логика простая - взять переданные на вход аутентификационные данные (имя юзера + пароль), поискать соответствующее имя юзера в БД, если имя есть, то нужно будет сравнить хэш переданного на вход пароля и того пароля, который хранится в БД. При совпадении пускаем пользователя на ресурс, а при несовпадении выдаем ошибку.
Для начала я описал функцию, ее параметры и дал описание ошибке, которую буду отдавать на неверные данные:
```Python
from fastapi import Depends, HTTPException, status


async def credentials_check(
    credentials: Annotated[HTTPBasicCredentials, Depends(security)],
    session: database.SessionDep,
):
    unauthed_exc = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="invalid username or password",
        headers={"WWW-Authenticate": "Basic"}
    )
```
Чтобы проверить вводимые данные нам нужно сначала их получить из экземпляра класса `security`, который мы создавали ранее, поэтому здесь `credentials` зависят через `Depends` именно от такого класса.
`HTTPException` - это класс, который позволяет вызвать какую-нибудь ошибку HTTP и использовать ее в своем коде. В данном случае я дополнительно прописываю параметр `detail` и `headers`. В заголовке, который я передал будет содержаться информация о типе проваленной аутентификации, - это часть правил хорошего тона.
Теперь я попрошу у БД дать мне данные по переданному в `credentials` имени пользователя. Если таких данных не будет, то вернется `None` - для этого используется функция `one_or_none()`:
```Python
query = select(
	models.User.username,
	models.User.password
).where(
	models.User.username == credentials.username
)
db_credentials = await session.execute(query)
db_credentials = db_credentials.one_or_none()
```
После получения данных сразу проверяем их:
```Python
if db_credentials:
	is_verified, new_hash = verify_password(
		credentials.password,
		db_credentials.password
	)
```
Если данные будут не найдены в БД, то в переменную `db_credentials` будет передан `None` и проверка `if` упадет. Напротив, если в БД все было найдено, то отправляем данные в ранее созданную функцию `verify_password`. Она вернет нам кортеж с результатом проверки и новый хэш, если старый устарел.
Если новый хэш действительно есть, то я хочу обновить данные по пользователю, изменив ему пароль:
```Python
if new_hash:
	query = update(models.User).where(
		models.User.username == credentials.username
	).values(password=new_hash)
	await session.execute(query)
	await session.commit()
```
Далее, если статус проверки успешен я верну переданные изначально данные, а если неуспешен, то верну ошибку:
```Python
if is_verified:
	return credentials
raise unauthed_exc
```
Точно так же я сделаю, если пользователь вообще не был найден в БД (т.е. `one_or_none()` вернула `None`):
```Python
if db_credentials:
	...
raise unauthed_exc
```
Полностью код выглядит так:
```Python
async def credentials_check(
    credentials: Annotated[HTTPBasicCredentials, Depends(security)],
    session: database.SessionDep,
):
    unauthed_exc = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="invalid username or password",
        headers={"WWW-Authenticate": "Basic"}
    )
    query = select(
        models.User.username,
        models.User.password
    ).where(
        models.User.username == credentials.username
    )
    db_credentials = await session.execute(query)
    db_credentials = db_credentials.one_or_none()
    if db_credentials:
        is_verified, new_hash = verify_password(
            credentials.password,
            db_credentials.password
        )
        if new_hash:
            query = update(models.User).where(
                models.User.username == credentials.username
            ).values(password=new_hash)
            await session.execute(query)
            await session.commit()
        if is_verified:
            return credentials
        raise unauthed_exc
    raise unauthed_exc
```
Теперь нужно прикрутить эту функцию к какой-нибудь функции в нашем сервисе, чтобы она срабатывала каждый раз, когда пользователь пытается получить доступ к ресурсу. Делается это очень просто - опять через параметры функции и `Depends`:
```Python
@router.get("/user/{index:int}", response_model=UserGetSchema)
async def get_user(
    index: int,
    session: database.SessionDep,
    credentials: Annotated[tuple[str], Depends(credentials_check)]
):
	...
```
После того как проверка будет успешно завершена в параметр `credentials` будут переданы те `credentials`, которые мы отправили из `credentials_check`.
## token auth
Token auth - это способ прохождения аутентификации по токену. Токен чаще всего представляет собой набор символов, который выглядит как будто кто-то упал лицом на клавиатуру.
Гораздо важнее понимать - это уже не способ аутентификации чисто через пароль. Аутентификация по токену работает так:
1) пользователь отправляет запрос на ресурс
2) ресурс ищет в запросе особый заголовок, в котором должен содержаться уникальный токен.
3) если заголовок есть, то ресурс идет искать пользователя по этому токену в БД.
4) если пользователь находится, то он проходит процедуру аутентификации.
Можно вспомнить, что ранее я говорил, что данные пользователя при Basic Auth после первого захода на сервис сохраняются в браузере в виде закодированной последовательности символов в заголовке `Authorization`:
```
authorization = Basic cGV0ZXI6MTQ4OA==
```
У токена, о котором я говорю сейчас, не такая механика:
1) сохранить такой токен можно в любом заголовке HTTP запроса, потому что мы можем указать на сервере какой нужно проверять приложению заголовок для аутентификации.
2) сам токен может быть любой последовательностью символов - главное, чтобы он был уникальным, потому что только так можно будет найти конкретного пользователя в БД.
### реализация
Чтобы сделать такой тип аутентификации нужно импортировать специальную сущность из `fastapi` - `Header`. Для нее в параметрах можно указать имя заголовка, который мы хотим проверять при заходе пользователей.
Я напишу простую функцию:
```Python
async def static_token_check(
    static_token: Annotated[str, Header(alias="x-auth-token")],
    session: database.SessionDep
):
    unauthed_exc = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="invalid token"
    )
    query = select(
        models.User.username,
        models.User.static_token
    ).where(
        models.User.static_token == static_token
    )
    db_credentials = await session.execute(query)
    db_credentials = db_credentials.one_or_none()
    if db_credentials:
        return db_credentials
    raise unauthed_exc
```
Объяснение:
1) `Header` будет искать в заголовках пришедшего запроса заголовок `x-auth-token`.
2) `select()` достанет из БД имя пользователя и токен.
3) `one_or_none()` - вернет пользователя и токен, если они есть, а если их нет вернет `None`, а на клиенте появится ошибка 401.
### расширение БД под токен
Я решил добавить новое поле к БД пользователей - `static_token`. Конечно, так в открытую такие токены не хранят, но для демонстрации мне кажется такая реализация будет нормальной.
Модель обновилась так:
```Python
class User(Base):
    __tablename__ = "users"

    user_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    username: Mapped[str] = mapped_column(String(200), unique=True)
    password: Mapped[str] = mapped_column(String(200))
    email: Mapped[str | None] = mapped_column(
        String(200), unique=True, default=None
    )
    static_token: Mapped[str] = mapped_column(
        String(200), unique=True, default=token_hex(16)
    )  # поле для токена. 
    
    __table_args__ = (
        CheckConstraint("length(username) > 0", name="chk_length_username"),
    )
```
Тут интересно, что в качестве значения по умолчания я использую функцию из модуля `secrets` - `token_hex(16)`. Она будет автоматически генерировать токены для каждого нового пользователя.
Теперь нужно очистить и обновить БД:
```SQL
truncate users restart identity cascade;
```
Накатываем миграции Alembic:
```Bash
alembic revision --autogenerate -m "User.static_token"
alembic upgrade head
```
Теперь можно создать нового пользователя. Токен будет автоматически сгенерирован и его значение можно узнать через `psql`.
### используем вход по токену
Когда токен есть осталось только сделать функцию, которая будет требовать именно токен для работы. Я сделал такую - GET функция по всем юзерам:
```Python
@router.get("/user", response_model=list[UserGetSchema])
async def get_users(
    session: database.SessionDep,
    credentials: Annotated[tuple[str], Depends(static_token_check)]
):
    query = select(
        models.User.user_id,
        models.User.username,
        models.User.email
    )
    result = await session.execute(query)
    return result.all()
```
Она зависит от нашей функции проверки токена, поэтому, чтобы получить юзеров, нужно в дополнительном заголовке запроса указать токен:
```Bash
curl -X GET -H "x-auth-token: <токен из БД>" http://127.0.0.1:8000/user
```

## что такое JWT?
После того как я показал, что аутентификация может происходить по токену, который мы передаем в заголовке запроса, нужно разобраться с тем что из себя вообще может представлять этот токен.
В предыдущих примерах токен сам по себе не был так важен, т.к. он представлял просто последовательность сгенерированных символов, раскодировка которых не дала бы ничего важного для нашей программы. Но было бы круто, если бы в самом токене хранилась какая-то информация. Тогда при получении токена мы могли бы его раскодировать и использовать те вещи, которые в нем описаны. Например, в нем можно было бы сохранить статус пользователя на сервере (является он администратором или обычным пользователем?).
Так же было бы интересно совместить подход с паролем и подход с токеном, ведь и в том и в этом случае токен используется, чтобы повторно аутентифицировать пользователя на сервисе.
Именно такие механики (и несколько других) реализует JWT. 
JWT (JSON Web Token) - это данные в формате JSON. В такой JSON входят:
- `header` - заголовок с общей информацией по токену.
- `payload` - полезные данные (id пользователя, его роль и проч.)
- `signature` - подпись для токена.
С опорой на токены реализуется вся механика JWT:
1) Пользователь вводит аутентификационные данные (логин/пароль, либо заход через Google/Facebook/проч.). Его запрос приходит на сервер.
2) Если данные верны, то сервер генерирует JWT токен и отправляет назад пользователю.
3) Каждый раз когда пользователь будет делать новые запросы к ресурсу, его токен будет передан на сервер и проверен.
Зачем вообще нужна такая сложная механика? Все дело в том, что протокол HTTP не хранит состояния. То есть каждый запрос, который мы посылаем на один и тот же ресурс рассматривается как автономный запрос. Ни браузер, ни сервер (в стандартном представлении), ни один из протоколов не в курсе, что пользователь был только что аутентифицирован. Поэтому если не реализовать какой-то дополнительный механизм (типа JWT), то каждый переход по страницам, даже в рамках одного ресурса, требовал бы нового ввода пароля и логина. 
JWT решает эту проблему тем, что он отдает пользователю ключ, который свидетельствует о том, что этот пользователь недавно прошел аутентификацию и ему не нужно проходить ее снова в течении какого-то времени. Поэтому такой пользователь может прыгать по всем страницам в рамках ресурса и оставаться аутентифицированным.
### откровение
Я долго путался в этой системе:
1) Что делает `HTTPBearer()`?
2) Что делает `HTTPBasic()`?
3) Как реализовать эндпоинт `/login`?
4) И как будет он связан с другими эндпоинтами?
Так много вопросов и так мало ответов.
Здесь на меня снизошло откровение.
`HTTPBearer()` читает заголовок `Authorization` и достает оттуда токен, который сразу же передается в функцию. Класс совершенно никак не проверяет и не влияет на сам заголовок - он только достает из него токен. Точно так же поступает и `HTTPBasic()`.
Если в сервисе предусмотрено использование JWT токена, то нам нужно передать его на клиент. На клиенте должна произойти магия - JWT токен должен упасть в клиентское приложение. Мой бэкенд на Python никак не может повлиять на этот процесс.
### заголовки
Ранее я говорил, что у JWT токена есть три заголовка:
1) `header` в общем случае содержит два поля:
	1) `alg` - алгоритм шифрования токена.
	2) `typ`- тип токена.
```JSON
{
	"alg": "HS256",
	"typ": "JWT"
}
```
2) `payload` содержит сколько душе автора токена угодно полей, потому что в общем случае этот заголовок нужен для того, чтобы в нем передать всю необходимую информацию о пользователе, который создает токен. Например, сюда можно добавить идентификатор пользователя на ресурсе, его дату рождения, имя и проч. На [сайте JWT](https://jwt.io/) показан такой вариант:
```JSON
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true,
  "iat": 1516239022
}
```
3) `signature` - это подпись. По сути уникальный код, который генерируется из закодированных данных `payload` + `header`. Из первых двух создаются уникальные последовательности символов, а затем эти последовательности кодируются снова (+ добавляется дополнительный секретный ключ, который придумывает создатель) и получается подпись.
Токеном являются все три компонента, закодированные и объединенные в одну строку через точку.
### кодирование, шифрование, хэш
Важно понимать, что когда я пишу, что данные, которые составляют токен, "закодированы" - это не означают "зашифрованы". Кодирование имеет другое назначение, нежели шифрование. В ситуации с токеном это означает, что закодированные данные технически публично доступны каждому, кто применит верную кодировку для их раскодировки. Смысл использования кодировки в том, чтобы иметь возможность для альтернативной транспортировки данных.
Шифрование - это процесс трансформирования данных таким образом, чтобы они были доступны только кому-то конкретному. Это требует более сложного алгоритма и ключа, который можно использовать для расшифровки.
Хэширование - это процесс преобразования данных в такие последовательности, которые могут продемонстрировать цельность данных. Т.к. хэши меняются, если изменится хотя бы один символ в последовательности, то совпадение двух хэшей гарантирует идентичность данных, которые были преобразованы в эти хэши.
Подробнее про термины можно посмотреть тут: https://danielmiessler.com/blog/encoding-encryption-hashing-obfuscation
В общем случае нужно понимать, что кто угодно может посмотреть на запись JWT токена и подобрать ключи/механизм декодирования к нему, поэтому лучше не хранить в токене уж очень чувствительные данные.
### заход по JWT токену
Я реализую точно такую же схему, которую я делал в [[#token auth]], но с JWT токеном. Моя задача - сделать данные, которые я буду хранить в JWT токене, закодировать эти данные в токен, а потом выдавать токен всем пользователям при создании пользователя.
#### библиотеки
Я не специалист по библиотекам, которые могут помочь с аутентификацией и авторизацией пользователей на сервере. Однако, я точно знаю, что хочу использовать JWT и хэширование паролей. Для этого мне понадобятся:
```Bash
 python3.13 -m pip install pyjwt "passlib[bcrypt]"
```
Изначально я хотел установить `python-jose`, однако затем нашел этот топик: https://github.com/fastapi/fastapi/discussions/9587/; поэтому решил отказаться от затеи и обратиться к актуальным советам в доках FastAPI: https://fastapi.tiangolo.com/ru/tutorial/security/oauth2-jwt/.
Пояснение по библиотекам:
1) `pyjwt` дает интерфейс для создания JWT токена, проверки JWT токена и обновления JWT токена.
2) `passlib` - это библиотека для хэширования паролей.
#### создаем токены
Создание токена:
```Python
def create_jwt_token(data: dict) -> str:
    expire = (
        datetime.now(timezone.utc)
        + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    data.update({"exp": expire})
    jwt_token = jwt.encode(
        data, SECRET_KEY, algorithm=ALGORITHM
    )
    return jwt_token
```
Все довольно примитивно. Сначала описываем переменную `expire` - она будет хранить время, до которого наш токен будет актуален. После того как это время пройдет, нужно будет создавать новый токен (это я реализую в функции для верификации). 
На вход функции передается словарь `data` - в нем хранится информация, которая будет находится в нашем токене.
Теперь можно пойти в файл `user.py`, где я описывал все эндпоинты и описать эндпоинт `/login`. Он должен проверять пользователя с помощью ранее реализованной `basic auth` и при успехе отдавать токен JWT.
```Python
@router.post("/login", response_model=Token)
async def login(
    credentials: Annotated[HTTPBasicCredentials, Depends(credentials_check)]
):
    jwt_token = create_jwt_token(
        {
        "username": credentials.username,
        "password": credentials.password
        }
    )
    return {"access_token": jwt_token, "token_type": "Bearer"}
```
Схема `Token` состоит из двух полей:
```Python
class Token(BaseModel):
	access_token: str
	token_type: str
```
Здесь я отдаю в функцию на создание JWT токена то, что пользователь передает при входе, но вместо этой информации можно передать буквально все, что угодно. Например, можно было бы открыть БД, достать оттуда `email, static_token` и отдать их.
Сам токен я не сохраняю в БД - в этом нет необходимости, т.к. он живет всего полчаса и требует постоянных обновлений.
Кстати, а как сделать логин на такой эндпоинт через `curl`?
```Bash
curl -X POST http://127.0.0.1:8000/login --user peter:1488
```
#### проверяем пользователя
Я решил, что теперь функция на получение данных о пользователях будет доступна только по токену. Но сначала мне нужно написать функцию, которая будет проверять токены.
В этот раз нам нужно проверять заголовок `Authorization` не на `Basic`, а на `Bearer` тип токена, поэтому нужно будет использовать другой класс из `fastapi` - `HTTPBearer()`.
```Python
from fastapi import HTTPBearer


security = HTTPBearer()


def token_check(
    credentials: Annotated[str, Depends(security)]
):
    unauthed_exc = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="invalid or expired token",
        headers={"WWW-authenticate": "Bearer"}
    )
    jwt_token = credentials.credentials
    payload = verify_jwt_token(jwt_token)
    if payload:
        return payload
    raise unauthed_exc
```
Функция `verify_jwt_token()` выполняет такую проверку:
```Python
def verify_jwt_token(jwt_token: str) -> dict | None:
    try:
        decoded_jwt_token = jwt.decode(
            jwt_token, SECRET_KEY, algorithms=[ALGORITHM]
        )
        expire = decoded_jwt_token.get("exp", None)
        if (
            expire
            and datetime.fromtimestamp(expire)
            >= datetime.utcnow()
        ):
            return decoded_jwt_token
        return None
    except jwt.PyJWTError:
        return None
```
То есть мы просто принимаем токен и распаковываем его в обратную сторону, а если такого токена нет, то возвращаем ошибку. Из `token_check` должен вернуться словарь данных, который мы запаковали в токен при заходе пользователя на сервис.
<<<<<<< Updated upstream
Теперь можно передавать в параметры ресурса, который мы хотим защитить, зависимость:
```Python
@router.get("/user", response_model=list[UserGetSchema])
async def get_users(
=======
Теперь мне просто нужно передать `token_check` в параметры функции, которую я хочу защитить в качестве зависимости:
```Python
@router.post("/p")
async def add_post(
    post: PostCreateSchema,
>>>>>>> Stashed changes
    session: database.SessionDep,
    credentials: Annotated[dict, Depends(token_check)]
):
	pass
```
Чтобы обратиться к этой функции через `curl` нужно будет вставить дополнительный заголовок `Authorization`:
```Bash
curl -X GET -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6InN0ZXZlIiwicGFzc3dvcmQiOiIxMjM0NTYiLCJleHAiOjE3NTQ0NzE1NTF9.No_Jakp8oh0i3Nl_VVrLM27pyUv7KWvdz99cKkq6F9w" http://127.0.0.1:8000/user
```
С каждым запросом к этому эндпоинту понадобится передавать `Bearer` токен в заголовке `Authorization`. Я это делаю так:
```Bash
curl -X POST \
-H "Content-Type: application/json" \
-H "Authorization: Bearer <токен, который получили ранее>" \
-d '{"title": "my first post", "content": "prikol", "user_id": 2}' \
http://127.0.0.1:8000/p
```
# тесты
Когда у меня в проекте накопилось уже куча вещей я решил, что пора добавить тесты, чтобы не проебать весь аккуратный функционал.
Тесты буду делать на `pytest`, но из-за того, что у меня 100% функций на сервисе асинхронные, мне придется использовать `pytest-asyncio`.
Тесты не должны зависеть от основного кода, поэтому всю историю с подключением к БД придется организовать заново, но в этот раз без удобных фишек из `fastapi` типа `Depends`.
## фиксура на коннект + тесты БД
Сначала нужно создать как белый человек файл `conftest.py`. Это файл, который входит в круг "конфигурационных файлов" `pytest`. Не знаю как его правильно обозвать, но суть в том, что в нем я могу прописать функции-фиксуры, которые из этого же файла без всяких импортов можно будет использовать по всему набору тестов.
Внутри `conftest.py` я сделаю первую функцию-фиксуру - она будет выдавать сессии тестам:
```Python
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

from ..app.config import settings


@pytest.fixture
async def session(): 
	engine = create_async_engine(settings.POSTGRES_URL)
	async with engine.connect() as conn:
		async_session_maker = async_sessionmaker(bind=engine, expire_on_commit=False)
		with async_session_maker() as session:
			yield session
```
В сущности здесь происходит все то же самое, что происходило ранее при создании интерфейса коннекта к БД для приложения:
1) создается движок.
2) создается сессия;
3) сессия отдается по вызову фиксуры.
Теперь нужно написать первую тройку тестов - нужно проверить можно ли подключиться к БД. Это я сделаю в файле `test_db.py`:
```Python
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

from .app.config import settings


def test_engine():
    engine = create_async_engine(settings.POSTGRES_URL)
    assert isinstance(engine, AsyncEngine), \
        f"Не удалось подключиться к БД с помощью запроса: {settings.POSTGRES_URL}."


async def test_session():
    engine = create_async_engine(settings.POSTGRES_URL)
    async_session_maker = async_sessionmaker(bind=engine, expire_on_commit=False)
    async with async_session_maker() as session:
        assert isinstance(session, AsyncSession), \
            "Ошибка при попытке создать сессию."


async def test_query(session):
    query = select(1)
    result = await session.execute(query)
    assert result.scalar() == 1, \
        "Запрос `select(1)` вернул неверный результат."
```
Слэш в данном случае выполняет функцию переноса. Я использую именно слэш, а не скобочный подход, потому что в случае скобок `pytest` довольно часто пишет предупреждения о том, что в `assert` на проверку был передан кортеж, соответственно, возвращаться всегда будет `True`, т.к. кортеж не пуст.
Такое тестирование не запустится - `pytest` будет возмущаться, что неожиданно для него были сделаны асинхронные тесты. Чтобы исправить ситуацию нужно либо достать из библиотеки `pytest-asyncio` декоратор `@pytest.mark.asyncio`, либо прописать для `pytest.ini` настройки запуска тестирования:
```ini
addopts = 
    "--asyncio-mode=auto"
```
Для каждого `assert` прописана простейшая проверка + указано сообщение, которое будет появляться в случае провала проверки.
После проверки коннекта можно отправить пробный запрос - это делает третий тест.
### область фиксуры
Стандартная область фиксуры - `function`. То есть каждая функция, которая будет вызывать эту фиксуру, на самом деле запустит ее работу заново (это примерно то, как я это понял).
Область работы фиксуры можно изменить так:
```Python
@pytest.fixture(scope="class")
```
Список областей фиксур можно найти в доках.
Я это рассказываю только потому, что в одном из случаев мне показалось, что будет удобно создать модульную фиксуру, т.е. такую фиксуру, которая ставит предустановки для целого файла с тестами. Однако в этой фиксуре уже нельзя будет использовать ту фиксуру, которая создает нам сессии, потому что у них разные области действия.
При этом если мы поменяем область действия фиксуры для сессий, то асинхронные сессии будут работать некорректно.
### pytest.mark
Это такая удобная штука для разметки тестов. Допустим у меня есть куча разных файлов тестирования, которые при этом относятся примерно к одной категории тестов. Все эти файлы я могу пометить собственной меткой, например:
```Python
pytestmark = pytest.mark.db
```
А затем запускать скопом через эту же метку.
Метки - это гибкая штука. Кроме указания их через переменную `pytestmark` еще можно выставлять декораторы для каждого теста индивидуально + внутри `pytestmark` можно держать список меток, которые могут вызвать этот файл с тестами:
```Python
pytestmark = [pytest.mark.db, pytest.mark.joke]
```
После этого можно закинуть сессии на полку - в реальных тестах для проекта сначала нужно научиться вызвать эндпоинты.
Важно так же внести список марок в `pytest.ini`, иначе регулярно после тестирования будет возникать несколько ошибок категории `warning` - `pytest` скажет, что он нашел неизвестные ему метки.
```ini
[pytest]
markers =
    db: connection db tests.
    user: user endpoint tests.

addopts = 
    "--asyncio-mode=auto"
```

## тесты пользователя
Пользователь должен войти в сервис, должен быть способен получить доступ к списку других пользователей и доступ к конкретному пользователю. Так же пользователь должен быть способен удалить свой собственный аккаунт.
Нужно понимать, что это не тест возможностей пользователя, потому что тогда пришлось бы тестировать вообще возможности конкретного юзера на сервисе. Это тест методов для эндпоинтов `/user` и `/login`.
### объект юзера
Ранее, когда я делал авторизацию с помощью JWT, я не настроил выдачу объекта юзера при заходе на защищенный поинт.
По идее в нашем токене JWT храниться информация о пароле и юзернейме того, кто посещает сайт. В этом случае было бы неплохо при авторизации передавать в метод сразу объект юзера, который пытается его использовать. В частности это может понадобится при выполнении удаления аккаунта - нужно позволить удалять пользователю только свой аккаунт.
Реализация:
```Python
jwt_token = credentials.credentials  # type: ignore
payload = verify_jwt_token(jwt_token)

if payload:
	query = select(
		models.User.user_id,
		models.User.username,
		models.User.email
	).where(
		models.User.username == payload.get("username")
	)
	user = await session.execute(query)
	user = UserGetSchema.model_validate(user.one())
	return user
```
Это будет частью функции `token_check()`, с которой я настраивал зависимость ранее: если токен оказался верным, то с помощью его данных можно достать из БД пользователя по `username`, пропустить ответ БД через схему и отправить обратно в метод.
В методе это будет выглядеть так:
```Python
@router.delete("/p/{index:int}")
async def delete_post(
    index: int,
    session: database.SessionDep,
    user: CredentialsToken
):
    query = delete(models.Post).where(
        models.Post.post_id == index,
        models.Post.user_id == user.user_id  # type: ignore
    )
    await session.execute(query)
    await session.commit()
```
### user create
Для того, чтобы мой код тестов писал автоматические запросы к моему же приложению я буду использовать библиотеку `requests`. 
Однако лучше всего задаться вопросом: **что я хочу тестировать?**
На данном этапе материала не много - я описал его [[blog (проект)#тесты пользователя|здесь]].
С помощью библиотеки `requests` я отправлю запрос на создание пользователя и хочу получить ответ, что он действительно создался:
```Python
def test_create_user():
    """
    Тест высылает POST-запрос на эндпоинт
    `/user` с данными для создания пользователя.
    """
    response = requests.post(
        f"{socket.socket}{ENDPOINT}",
        headers={"Content-Type": "application/json"},
        data=user_json,
        timeout=TIMEOUT
    )
    assert response.status_code == 200, (
        "Не удалось создать пользователя.\n",
        ERROR_USER_MESSAGE_INFO.format(response.status_code)
    )
```
Перед написанием этого теста я сделал несколько вещей:
1) Создал класс `Socket`, который будет мне автоматически выдавать строку `хост+порт`. Зачем я это сделал? Для понтов. На самом деле эту штуку можно поместить в константу.
2) В константу вместо этого я поместил эндпоинт `/user` (константа называется `ENDPOINT`). Поэтому первый аргумент к функции `post()` является комбинацией сокета и эндпоинта в строке.
3) Сделал класс `User` с помощью `namedtuple`, а затем с помощью библиотеки `json` и функции `dumps()` превратил экземпляр класса `User` в JSON, чтобы можно было его отправить вместе с запросом. Эта магия выглядит так:
```Python
User = namedtuple("User", ["username", "password", "email"])
user = User("steve", "123456", "stevevach@gmail.com")
user_json = json.dumps(user._asdict())
```
4) Сделал константу, которая будет обозначать таймаут, чтобы Pylint не жаловался (константа называется `TIMEOUT`).
5) И сделал generic-строку с сообщением об ошибке в константе `ERROR_USER_MESSAGE_INFO`.
Тесты здесь и далее будут простыми и однотипными - обращаюсь к эндпоинту определенным методом, высылаю вместе с методом весь контент, который по идее должен метод принимать, а потом проверяю, что вернулся ожидаемый результат.
В этой части теста:
```Python
response = requests.post(
	f"{socket.socket}{ENDPOINT}",
	headers={"Content-Type": "application/json"},
	data=user_json,
	timeout=TIMEOUT
)   
```
Я использую функцию `post()` из библиотеки `requests`, чтобы сформировать POST-запрос на эндпоинт `http://localhost:8000/user`. Параметр `headers` содержит список заголовков, которые нужно передать вместе с запросом для корректной работы, в данном случае заголовок указывает, что в теле запроса (`data`) будет содержаться JSON. В `data` я размещаю JSON с `username`, `password` и `email` пользователя.
В результате работы функции `post()` в переменную `response` вернется объект класса `Response`. 
Далее, я пишу проверку:
```Python
assert response.status_code == 200, (
	"Не удалось создать пользователя.\n",
	ERROR_USER_MESSAGE_INFO.format(response.status_code)
)
```
Из объекта класса `Response` можно достать много классного. В данном случае я хочу получить статус-код ответа и проверить, что он равен `200`. В случае если проверка удастся, то я считаю, что тест успешен, а если она закончится провалом, то я выведу на экран то самое generic-сообщение об ошибке.
Теперь я хочу сделать более сложный тест - проверить, что юзер действительно был создан в БД после запроса. Для этого мне понадобится обратиться к БД с помощью сессии.
Так как я планирую обращаться к БД в нескольких разных тестах, то было бы правильно создать функцию, которая будет всем тестам обеспечивать коннект к БД. В `pytest` есть удобная реализация функций, которая называется "фиксура".
**Фиксура** - это механизм, который применяется перед тестом или после теста (или сразу в обоих случаях). Для такой штуки существует отдельный термин, потому что случаи, когда нужно создать какие-то данные перед тестированием и удалить эти же данные после очень часты.
Я хочу создать фиксуру, которая будет хранить в себе запрос к БД и перед началом теста делать этот запрос к БД, чтобы у теста сразу был в распоряжении результат запроса. Реализация:
```Python
@pytest.fixture(name="db_user")
async def get_db_user(session):
    """
    Фиксура должна отдавать пользователя из БД.
    """
    query = f"""
        select username, password, email from users
        where
            username = '{user.username}'
            and email = '{user.email}';
    """
    result = await session.execute(text(query))
    yield result
```
По сути это простая функция, но она подписана декоратором `@pytest.fixture()`. Для декоратора я передал параметр `name`, который будет определять по какому имени будет вызываться фиксура (это не обязательно, но Pylint ругается, если не использовать `name`). Внутри написано то, с чем я уже сталкивался ранее, задерживаться не буду. В конце стоит слово `yield`, которое будет возвращать результат запроса.

>[!important] Перед словом `yield` в фиксуре выполняются действия **до** тела теста, а после слова `yield` в фиксуре выполняются действия **после** теста.

Теперь сам тест:
```Python
def test_check_new_user_db(db_user):
    """
    Проверяем наличие созданного пользователя в БД.
    """
    hashed_password = password.get_password_hash(user.password)

    try:
        db_user = User._make(db_user.one())
    except TypeError as error:
        raise TypeError(
            "Запрос не удался: вернулся неполный результат.\n",
            ERROR_USER_MESSAGE_INFO.format(
                "Это был прямой запрос в БД."
            )
        ) from error
    except NoResultFound as error:
        raise NoResultFound(
            "Запрос не удался: вернулся пустой результат.\n",
            ERROR_USER_MESSAGE_INFO.format(
                "Это был прямой запрос в БД."
            )
        ) from error
    assert db_user.username == user.username, (
        "Не совпадает имя пользовтаеля.\n",
        f"db_username: {db_user.username}\n",
        f"init_username: {user.username}\n"
    )
    assert password.verify_password(user.password, db_user.password), (
        "Не совпадают пароли.\n",
        f"db_password: {db_user.password}\n",
        f"init_password: {hashed_password}\n"
    )
    assert db_user.email == user.email, (
        "Не совпадают эл.почты.\n"
        f"db_password: {db_user.password}\n"
        f"init_password: {hashed_password}\n"
    )
```
Как видно в параметрах функции как раз стоит имя нашей фиксуры. Это означает, что **до** тестирования она проделает всю работу и только потом начнется сам механизм теста.
Тест основан на фишке `namedtuple`, которая называется `_make()`. Она берет последовательность и делает из нее объект класса `User`. Если была передана неверная последовательность, то упадет ошибка `TypeError`. А если запрос вернет пустой результат, то функция `.one()` выдаст ошибку `NoResultFound`.

>[!important] У `sqlalchemy` есть своя библиотека исключений - ее можно импортировать так: `from sqlalchemy.exc import NoResultFound`.

Далее, идут проверки самого содержимого последовательности. Все они стандартно сравнивают то, что вернулось из БД благодаря фиксуре и то, что было изначально отправлено предыдущим тестом.
### user login
После создания юзера я хочу его залогинить. Логин должен вернуть мне токен, который я сохраню для последующего тестирования в экземпляре класса `Token`:
```Python
def test_login():
    """
    Тест высылает POST-запрос на эндпоинт
    `/login` с логином-паролем для входа.
    """
    endpoint = "/login"
    response = requests.post(
        f"{socket.socket}{endpoint}",
        auth=(user.username, user.password),
        timeout=TIMEOUT
    )
    assert response.status_code == 200, \
        f"""Логин не удался. Кредиты:
        {user.username}, {user.password}"""
    token.token = response.json()
```
Аргументы для функции `post()` отличаются. Параметр `auth` принимает в себя кортеж с параметрами аутентификации. Так как метод возвращает токен, то результат работы метода (токен) можно превратить в JSON, а потом сохранить, Это происходит тут:
```Python
token.token = response.json()
```
Все на этом крутой тест завершен.
### get user(s)
Все тесты на получение пользователя или списка пользователей требуют аутентификации:
```Python
def test_get_users():
    """
    Тест высылает GET-запрос на эндпоинт
    `/user`, чтобы получить список пользователей.
    Это защищенный эндпоинт, поэтому к запросу
    добавляем токен, полученный на логине.
    """
    response = requests.get(
        f"{socket.socket}{ENDPOINT}",
        headers={"Authorization": token.token},
        timeout=TIMEOUT
    )
    assert response.status_code == 200, \
        "Не удалось получить список пользователей."
    assert response.json(), \
        "Список пользователей пуст."


def test_get_user():
    """
    Тест высылает GET-запрос на эндпоинт
    `/user`, чтобы получить пользователя.
    Это защищенный эндпоинт, поэтому к запросу
    добавляем токен, полученный на логине.
    """
    username_path_param = f"/{user.username}"
    response = requests.get(
        f"{socket.socket}{ENDPOINT}{username_path_param}",
        headers={"Authorization": token.token},
        timeout=TIMEOUT
    )
    assert response.status_code == 200, \
        "Не удалось получить пользователя."
    result = response.json()
    assert result["username"] == user.username, (
        f"Имя пользователя не совпадает.\n"
        f"result_username: {result['username']}\n",
        f"user_username: {user.username}.\n"
    )
    assert result["email"] == user.email, (
        f"Почта пользователя не совпадает.\n"
        f"result_email: {result['email']}\n",
        f"user_username: {user.email}.\n"
    )
```
Собственно, принцип аутентификации здесь самый интересный - я передаю новый заголовок `Authorization` вместе с запросом.
### anon
Тесты на анонимного пользователя по сути должны проверять, что у него нет доступа туда, куда ему не нужно попадать. То есть я должен делать запросы к защищенным эндпоинтам и проверять, что вернулся статус код `403`.
```Python
def test_anon_get_users():
    """
    Провека доступа анонима (без токена) к GET `/user`.
    """
    response = requests.get(
        f"{socket.socket}{ENDPOINT}",
        timeout=TIMEOUT
    )
    assert response.status_code == 403, (
        f"Аноним не должен получать доступ к {socket.socket}{ENDPOINT}.\n",
        "HTTP-метод: GET."
    )


def test_anon_get_user():
    """
    Проверка доступ анонима (без токена) к GET `/user/{username}`.
    """
    username_path_param = f"/{user.username}"
    response = requests.get(
        f"{socket.socket}{ENDPOINT}{username_path_param}",
        timeout=TIMEOUT
    )
    assert response.status_code == 403, (
        f"Аноним не должен получать доступ к {socket.socket}{ENDPOINT}",
        "HTTP-метод: GET."
    )


def test_anon_delete_user():
    """
    Проверка доступ анонима (без токена) к DELETE `/user/{username}`.
    """
    response = requests.delete(
        f"{socket.socket}{ENDPOINT}",
        timeout=TIMEOUT
    )
    assert response.status_code == 403, (
        f"Аноним не должен получать доступ к {socket.socket}{ENDPOINT}",
        "HTTP-метод: DELETE."
    )
```
### user delete
Так как кейс пока что закончен, я хочу провести удаление тестового пользователя из БД.
```Python
def test_del_user():
    """
    Тест высылает DELETE-запрос на эндпоинт
    `/user`, чтобы удалить пользователя.
    Это защищенный эндпоинт, поэтому к запросу
    добавляем токен, полученный на логине.
    """
    response = requests.delete(
        f"{socket.socket}{ENDPOINT}",
        headers={"Authorization": token.token},
        timeout=TIMEOUT
    )


    assert response.status_code == 200, (
        "Не удалось удалить пользователя.\n" + \
        ERROR_USER_MESSAGE_INFO.format(status_code=response.response_code)
    )


def test_check_deleted_user_db(db_user):
    """
    Проверяем, что пользователь удалился из БД.
    """
    try:
        db_user.one()
    except NoResultFound:
        pass
    else:
        raise AssertionError(
            f"После удаления данные пользователя в БД: {db_user}."
        )
```
По аналогии с тестированием на добавление пользователя я хочу отправить запрос на удаление на свой сервис + удостовериться в удалении через проверку БД.
Если с запросом все понятно, то с БД  я делаю штуку интереснее: опять использую фиксуру для получения данных, но в этот раз, если функция `one()` вернет ошибку `NoResultFound` я буду считать, что тест успешно завершен.
# docker
Преимущественно я создаю этот проект, чтобы практиковаться в навыках деплоя на сервер. Поэтому первая задача из секции основных - упаковка проекта в контейнер и его публикация в DockerHub.
Сборку проекта всегда лучше начать с
## Dockerfile
В докерфайле описывается макет образа, по которому будет создан контейнер. Здесь должны быть инструкции по тому как будет развернут проект внутри контейнера.
Для текущего проекта я создал такой:
```Dockerfile
FROM python:3.13-slim
WORKDIR /blg_1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--proxy-headers", "--host", "0.0.0.0", "--port", "8000"]
```
Инструкции:
- `FROM` - это инструкция, которая диктует какой образ нужно установить внутри контейнера. В данном случае я использую облегченный образ `python:3.13-slim`.
- `WORKDIR` - указываем директорию внутри контейнера, внутрь которой образ при создании сразу перейдет. После такой команды все дальнейшие команды будут выполняться внутри этой директории.
- `COPY` - копируем файл `requirements.txt` в текущую директорию.
- `RUN` - запускаем внутри контейнера команду.
- `COPY` - копируем весь проект в директорию.
- `CMD` - опять запускаем команду внутри контейнера.
Сборка образа состоит из слоев - каждая строка с командой является слоем. Первый слой (инструкция `FROM`) называется *базовый слой*. Фишка слоев в том, что каждый слой кэшируется докером, чтобы при следующей сборке контейнера слои, которые не были затронуты изменениями, не собирались заново. Например, если я соберу контейнер, затем изменю команду установки зависимостей и попытаюсь собрать контейнер снова, то внутри контейнера будут использованы хэши первых трех слоев (они остались неизменными), а последующие слои будут перестроены.
За подробностями по всем командам лучше почитать доки: https://docs.docker.com/reference/dockerfile/#overview

---
Каждый докерфайл должен начинаться с команды `FROM`. Для команды `FROM` можно указать так же платформу, которая будет развернута внутри контейнера. Например:
```Dockerfile
ARG plat=linux/arm64
FROM --platform=${plat} python:3.13-slim
```
После указания платформы можно указать так же какой нам нужен образ внутри контейнера для запуска нашей программы. Этот образ будет взят из библиотеки образов Docker. Именно это я и сделал с помощью `python:3.13-slim`.
Хотя я сказал, что первой командой в файле должен быть `FROM`, первой командой здесь является инструкция `ARG`. Она позволяет делать аргументы внутри докерфайла. Этим я воспользовался, чтобы обойти ошибку, которая возникает при передаче в параметр `--platform` буквально текста `linux/arm64`.

---
Команда `WORKDIR` выполняет сразу две функции - создает внутри контейнера директорию и сразу в нее переходит. Более того она задает директорию в качестве рабочей для любых последующих команд типа `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, `ADD`. 
Такая инструкция может быть использована несколько раз в докерфайле, главное помнить, что при ее использовании мы не только делаем папку, но и переходим в нее.

---
Команда `COPY` копирует вещи из точки A в точку B. Все вроде просто, но почему тогда я использую эту команду дважды в пределах одного файла, если файлик `requirements.txt` лежит вместе со всем остальным проектом? Это сделано из-за слоеной структуры докерфайла - набор зависимостей меняется реже, чем сам проект, поэтому проще сначала установить зависимости, а потом копировать сами файлы проекта. В этом случае при изменении файлов проекта будет пересобран только их кэш, а кэши уровней с зависимостью останутся без изменений, поэтому не нужно будет заново устанавливать все пакеты проекта.

---
Инструкции `RUN` и `CMD`, запускают команду в терминале. Чем они различаются, кроме формы записи? Тем, что `RUN` запускается при инциализации контейнера единожды, а команда `CMD` запускается внутри контейнера и только когда контейнер уже готов.

---
Запустить стройку образа можно командой:
```Bash
docker build -t test .
```
Параметр `-t` позволяет ввести имя для образа. Точка после имени указывает где нужно искать `Dockerfile`.
Чтобы собрать и запустить контейнер по образу нужно написать:
```Bash
docker run --name test -p 8080:8000 test
```
Параметр `--name` позволяет ввести имя контейнера, параметр `-p` позволяет ввести пару `<порт, с которого хост сможет достучаться до контейнера:порт, на котором контейнер будет принимать сообщения от хоста>`, а имя `test` в конце указывает на образ, на основании которого мне нужно создавать контейнер.
## .dockerignore
При сборке образа мы копируем все файлы из локальной директории в директорию на контейнере. Это значит, что на контейнер попадут все файлы в т.ч. `.venv`, `.env` и проч. Я не хочу держать локальное виртуальное окружение в контейнере и не хочу перекладывать в контейнер файл `.env`, потому что тогда к моим секретам могут получить доступ все, кто умеет пользоваться командой:
```Bash
docker exec <имя контейнера> cat .env
```
Она, кстати, умеет исполнять скрипты командной строки внутри контейнера.
Поэтому, чтобы защититься и избавиться от мусора существует `.dockerignore`. Это такой же текстовый файл, как и `.gitignore` и механика их заполнения одинакова - просто пишу то, что хочу исключить из внимания Docker при сборках.
## логин в docker и dockerhub
Свои образы можно пушить в DockerHub. Это аналог GitHub, но для Docker. Преимущество в том, что после создания образа на основе Dockerfile этот образ можно разместить в DockerHub, а затем, вместо хранения образа локально, скачивать его именно с DockerHub. Это помогает, в частности, разворачивать проекты на удаленном сервере.
Хотя такой подход не обязателен и все собственные образы можно крафтить локально, все равно стоит знать о фиче пуша образов.
Сначала сделаю логин:
```Bash
docker login -u <username>
```
После этой команды докер предложит ввести пароль. Готово.
Когда логин закончен нужно сбилдить образ с особым именем:
```Bash
docker build -t <username>/<имя образа> .
```
И вот теперь можно запушить образ:
```Bash
docker push <username>/<имя образа>
```
Теперь образ будет добавлен в репо аккаунта на DockerHub, а это значит, что его можно будет скачать точно так же как образ, например, `python:3.13-slim`.
## docker-compose
Ок, допустим я запустил контейнер, но, как я сказал, это будет только само приложение внутри контейнера. Так как контейнер является полностью изолированной системой, я должен либо поставить в эту систему экземпляр PostgreSQL, чтобы мое приложение могло общаться с БД, либо заставить каким-то образом приложение общаться с экземпляром PostgreSQL на моем хосте.
Я предлагаю поступить так, как предлагают поступить разработчики Docker - запустить несколько приложений на нескольких разных контейнерах. В моем случае мне понадобится дополнительный контейнер с PostgreSQL внутри.
Запустить на нескольких контейнерах приложение поможет оркестрация и `docker-compose`. `docker-compose` - это дополнение к основной механике докера. Оно может собрать несколько контейнеров, на основании нескольких образов, запустить контейнеры в нужной последовательности и сделать еще много полезных штук.
`docker-compose` - это файл формата YAML. В нем нужно описать список сервисов, которые оркестрация должна запустить. Начну со своего приложения:
```YAML
services:
  blg_1:
    image: lipkerton/blog_1:v1.1
    ports:
      - "8000:8000"
```
Тэг `services` начинает список сервисов. `blg_1` - это название моего первого контейнера. `image` - это имя моего образа на DockerHub с версией. `ports` - это та же самая пара портов, которая была в Dockerfile.
Запустить такую штуку можно так:
```Bash
docker compose up
```
Результат будет примерно такой же, как если бы я запустил контейнер через `docker run`.
Опустить можно так:
```Bash
docker compose down
```
Я помню, что в моем приложении так же работают секреты из файла `.env` и этот файл я добавил в `.dockerignore`, но как сообщить внутрь контейнера пароли, логины и прч., чтобы приложение внутри взаимодействовало с БД?
Для этого у `docker compose` есть решение:
```YAML
services:
  blg_1:
    image: lipkerton/blog_1:v1.1
    ports:
      - "8000:8000"
    env_file: ".env"
```
Да, оно просто состоит в указании пути к файлу `.env`. Теперь контейнер подгрузит секреты, которые размещены локально внутрь, но не скопирует сам файл.
А где в этой схеме БД? БД - это отдельный сервис. Так как я пользуюсь PostgreSQL можно развернуть с помощью DockerHub готовый образ кластера без нужды писать Dockerfile. Как тогда сделать желаемое состояние БД внутри контейнера? С помощью миграций алембика! Какое чудо, я про них уже успел забыть.
Но сначала нужно дописать контейнер БД:
```YAML
services:
  blg_1:
    image: lipkerton/blog_1:v1.1
    networks:
      - blg_1_net
    ports:
      - "8000:8000"
    env_file: ".env"

  database:
    image: postgres:latest
    container_name: postgres_blg_1
    ports:
      - "5430:5432"
```
Здесь все работает идентично - скачиваем образ `postgres`, даем имя контейнеру, задаем порты. Однако я бы хотел получить со старта так же пользователя и базу данных внутри кластера. Такие вещи можно сделать с помощью передачи окружения:
```YAML
  database:
    image: postgres:latest
    container_name: postgres_blg_1
    environment:
	  POSTGRES_USER: ${POSTGRES_DB_USER}
      POSTGRES_PASSWORD: ${POSTGRES_DB_PASS}
      POSTGRES_DB: ${POSTGRES_DB_NAME}
    ports:
      - "${DB_PORT_HOST}:${DB_PORT_CONT}"
    env_file: ".env"
```
Запись `${POSTGRES_DB_USER}` означает, что я прошу переменную `POSTGRES_DB_USER` из своего файла `.env`. Это значит, что мне понадобится добавить референс на свой `.env` и в этот файл.
### сети
Сейчас у меня есть два контейнера, которые при этом никак друг с другом не связаны. Почему не связаны? Потому что они буквально являются двумя разными виртуальными устройствами. Чтобы их связать нужно использовать общую сеть - `network`. Это еще один тег:
```YAML
services:
  blg_1:
    image: lipkerton/blog_1:v1.1
    networks:
      - blg_1_net
    ports:
      - "8000:8000"
    env_file: ".env"

  database:
    image: postgres:latest
    container_name: postgres_blg_1
    environment:
      POSTGRES_USER: ${POSTGRES_DB_USER}
      POSTGRES_PASSWORD: ${POSTGRES_DB_PASS}
      POSTGRES_DB: ${POSTGRES_DB_NAME}
    networks:
      - blg_1_net
    ports:
      - "${DB_PORT_HOST}:${DB_PORT_CONT}"
    env_file: ".env"

networks:
  blg_1_net:
```
Здесь я добавил два типа записи - внес дополнительный тег внутрь сервиса с БД и еще дописал снаружи тега `services` тег `networks`. Тег внутри говорит, что этот контейнер будет принадлежать к общей сети контейнеров под названием `blg_1_net` и в такую же сеть записан сервис с приложением. Тег снаружи - это "регистратор" сетей, в котором можно прописать настройки сети, точно так же как я это делаю сейчас с настройками сервиса.
### тома
Существует так же более насущная проблема - допустим у меня есть система внутри контейнера с БД. Но что если мне понадобится сделать выкл-вкл по контейнеру? Куда денутся данные в этой БД?
Они испарятся. Каждый контейнер не зависит от предыдущей своей версии, поэтому данные, которые он хранит в течении сессии запуска, исчезнут вместе с ним. Такая механика плоха в контексте переносимости контейнера - что будет если я захочу перенести контейнеры с одного сервера на другой?
В этой ситуации выручают тома - `volume`. Это специальный сервис докера, который позволяет хранить определенные данные контейнера независимо от состояния его работы.
Создать том:
```YAML
  database:
    image: postgres:latest
    container_name: postgres_blg_1
    environment:
      POSTGRES_USER: ${POSTGRES_DB_USER}
      POSTGRES_PASSWORD: ${POSTGRES_DB_PASS}
      POSTGRES_DB: ${POSTGRES_DB_NAME}
    networks:
      - blg_1_net
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "${DB_PORT_HOST}:${DB_PORT_CONT}"
    env_file: ".env"

volumes:
  postgres_data:
```
Механика такая же, как и с сетями, но с одним отличием: внутри тега `volumes` я прописываю сначала название тома, затем ставлю двоеточие и после пишу директорию *внутри контейнера*, которую хочу хранить в томе. Точно так же как с сетью регистрирую том в отдельном списке томов.
### зависимости
Очевидно, что БД должна работать раньше запуска приложения, поэтому нужно прописать, что запуск контейнера с приложением зависит от запуска контейнера с БД. Это делается с помощью тега `depeneds_on`:
```YAML
services:
  blg_1:
    image: lipkerton/blog_1:v1.1
    depends_on:
	  database:
    networks:
      - blg_1_net
    ports:
      - "8000:8000"
    env_file: ".env"
```
Однако не все так просто - перед стартом БД еще может несколько секунд производить первичную инициализацию файлов и проч., поэтому лучше было бы перед запуском приложения еще и проверить готовность БД к работе. Это делается с помощью команды `healthcheck`:
```YAML
  database:
    image: postgres:latest
    container_name: postgres_blg_1
    environment:
      POSTGRES_USER: ${POSTGRES_DB_USER}
      POSTGRES_PASSWORD: ${POSTGRES_DB_PASS}
      POSTGRES_DB: ${POSTGRES_DB_NAME}
    networks:
      - blg_1_net
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "${DB_PORT_HOST}:${DB_PORT_CONT}"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_DB_USER} -d ${POSTGRES_DB_NAME}"]
      interval: 30s
      timeout: 10s
      retries: 5
    env_file: ".env"
```
Сначала внутри `healthcheck` пишется сам тест в виде команды в командную строку, затем прописываются настройки повторения, таймаутов и ретраев тестирования. Насколько я понял из [документации](https://docs.docker.com/reference/compose-file/services/#healthcheck) проверка будет запускаться именно в пределах всей сессии существования контейнера, а не только при его запуске.
Чтобы дать понять зависимости, что у нас существует проверка на здоровье я напишу:
```YAML
services:
  blg_1:
    image: lipkerton/blog_1:v1.1
    depends_on:
	  database:
	    condition: service_healthy
    networks:
      - blg_1_net
    ports:
      - "8000:8000"
    env_file: ".env"
```
О разновидностях таких проверок можно так же прочесть тут: https://docs.docker.com/reference/compose-file/services/#depends_on
### фишки запуска
Чтобы запустить:
```Bash
docker compose up
```
Чтобы запустить в фоновом режиме:
```Bash
docker compose up -d
```
Чтобы запустить с пересборкой образа:
```Bash
docker compose up --build
```
## контейнеры после запуска
Я набираю:
```Bash
docker compose up --build
```
И получаю примерно такой вывод:
![[docker_compose_1.png]]
Сервер, который запустился внутри контейнера уже принимает запросы. Я даже смогу запустить тесты, но сначала нужно привести БД в нормальное состояние с помощью миграций Alembic:
```Bash
docker exec blog-blg_1-1 alembic upgrade head
```
И запуск тестирования:
```
docker exec blog-blog_1-1 pytest py_tests/
```
Вывод:
![[docker_compose_2.png]]
В контейнере все успешно запускается.
### доп как я чистил миграции
Когда я учился в этом проекте работе миграций, то наделал кучу говеных экземпляров миграций, которые в истории миграций мне не нужны. Дополнительно я хочу сделать чистую начальную миграцию. Как это сделать?
Сначала стоит проверить журнал миграций на хосте, где строится образ:
```Bash
alembic history
```
У меня они чистые, поэтому здесь будет только инициализирующая миграция:
![[alembic_1.png]]
Но допустим тут куча миграций. В любом случае каждая миграция - это файл. Файл лежит внутри текущего проекта в папке `migrations/versions`. Файл можно удалить и миграция пропадет. Однако не нужно забывать так же чистануть используемый кластер от таблицы `alembic_version`, иначе алембик будет помнить, что у него какие-то миграции таинственно исчезли и не даст сгенерировать новую миграцию.
Теперь я локально создаю новый файл миграций:
```Bash
alembic revision --autogenerate -m "init migration"
```
И пересобираю образ:
```Bash
docker compose up --build
```
# Jenkins
## установка
Jenkins - это масштабная система по настройке пайплайнов. По крайней мере именно в таком смысле я ее буду использовать.
Перед тем как начать работать с Jenkins его сначала нужно установить. Так как это программа, которая должна работать на постоянке (= демон), я буду ее устанавливать на свой Debian-сервер, который так же постоянно работает.
В [официальном доке](https://www.jenkins.io/doc/book/installing/linux/#debian-java) требуют поставить версию JDK 21. Однако в моем пакете `apt` такой версии нет (и мне не особо ясно почему разработчик Jenkins об этом не знает), поэтому я поставлю последнюю доступную в нем (https://wiki.debian.org/Java):
```Bash
sudo apt-get install default-jdk
java -version
```
Когда поставил JDK можно качать пак Jenkins:
```Bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins
```
Шаги, которые Jenkins сделает при установке описаны в [доке](https://www.jenkins.io/doc/book/installing/linux/#debian-stable). Основной момент - доступ к браузерному интерфейсу Jenkins, где можно настраивать пайплайны. Зайти туда можно по связке `<localhost>:8080`. Платформа попросит ввести пароль, который написан в логах Jenkins. Открыть логи можно так:
```Bash
sudo journalctl -u jenkins.service
```
## пайплайн
От Jenkins мне нужно несколько вещей:
1) Взять мою ветку репозитория с Git
2) Запустить для этой ветки тесты
3) Сообщить о том прошли тесты хорошо или плохо
На Jenkins есть шаблоны создания различных пайплайнов:
![[jenkins_1.png]]
В официальных доках рекомендуют экспериментировать с *Multibranch Pipeline* или с *Organisation Folder*, однако я захотел повторить фокус знакомого девопса - сделать возможность выбирать ветку, которую следует билдить и на которой следует проверять тесты. Для этого нужно сделать параметризированный *Pipeline*.
Для установки параметров я указываю:
![[jenkins_2.png]]
Так как я хочу именно указывать нужную мне ветку я добавлю параметр `String Parameter` с именем `branch`. Теперь нужно сообщить гиту, что значение из этого параметра нужно использовать для поиска ветки. Однако сначала нужно настроить сам гит:
![[jenkins_3.png]]
Jenkins для работы с веткой потребуются кредиты для входа от лица владельца репо, либо от лица кого-то, кто имеет доступ к репо. Кредиты можно заполнить разными способами - я использовал SSH.
После этого в параметрах к SCM можно указать, что использовать следует именно то значение, которое будет передано вместе с параметром `branch`:
![[jenkins_4.png]]
И в конце выбираю путь до Jenkinsfile.
## Jenkisfile
Jenkinsfile пишется на чем-то, что подобно языку Groovy. Для начала я сделаю что-нибудь простое:
```Groovy
pipeline {
    agent any
    stages {
        stage('hello') {
            steps {
		    	echo_begin_stage()
		    	echo 'hello world!'
				echo "Building ${env.JOB_NAME}"
            }
        }
    }
}


def echo_begin_stage() {
    last = env.STAGE_NAME
    echo "---------------------{$last}---------------------"
}
```
Здесь я начинаю сам пайплайн с ключевого слова `pipline`, затем объявляю агента и описываю первый шаг. На первом шаге я хочу использовать функцию, которая будет печатать разделитель между шагами, напечатать `hello world!` и напечатать `Building <имя джоба>`.
Теперь нужно разобраться че это все вообще такое.
Jenkins называет язык, на котором для него пишутся инструкции "[Pipeline domain-specific language (DSL) syntax](https://www.jenkins.io/doc/book/pipeline/syntax)". Пайплайн, который построен через такой тип файла, называется "декларативный пайплайн".
Jenkinsfile наследует некоторые (но не все) правила из языка Groovy. Набор таких правил описан [тут](https://www.jenkins.io/doc/book/pipeline/syntax/).
Для одного Jenkinsfile существует один пайплайн. Пайплайном считается то, что написано внутри ключевого слова `pipeline`:
```Groovy
pipeline {
	<тут весь код>
}
```
### agent
Следующий важный параметр - `agent`. Он определяет локацию, где будет выполняться наш пайплайн, запускает этапы пайплайна, выполняет тесты и т.д. После команды `agent` обычно идет три метки: `any`, `label`, `node`.
- `any` - любая доступная среда выполнения.
- `label` - простое указание метки среды запуска. Список меток есть в `Настроить Jenkins -> Nodes`.
- `node` - это указание метки среды запуска + возможность гибкой настройки этой среды. Дополнительно [тут](https://www.jenkins.io/doc/book/pipeline/syntax/#agent).
Агент так же может иметь значение `none`, в этом случае для каждого этапа пайплайна должен быть описан собственный агент.
Прежде чем использовать агента нужно сначала создать себе агента. Агент - это буквально отдельная система или система внутри той же системы, где стоит Jenkins-controller (лучше, чтобы это была отдельная система). Официальная инструкция по развертыванию агента: https://www.jenkins.io/doc/book/using/using-agents/#creating-your-docker-agent
#### docker-agent
Я пренебрег советами официальной документации и развернул агента прямо в той же системе, где у меня стоит Jenkins-controller. Для этого мне пришлось поставить Docker и сгенерировать еще один SSH ключ. Ключ нужен, чтобы Jenkins мог получить доступ к управлению агентом внутри контейнера: приватный ключ будет находится на сервере Jenkins, а публичный я буду передавать при старте контейнера в контейнер.
В документации к созданию агента написана такая команда запуска контейнера:
```Bash
docker run -d --rm --name=agent1 -p 22:22 \
-e "JENKINS_AGENT_SSH_PUBKEY=<your_public_key>" \
jenkins/ssh-agent:alpine-jdk21
```
И по умолчанию разработчик предлагает использовать 22-й порт в качестве порта, на котором контейнер будет отслеживать подключения. Проблема в том, что 22-й порт - это порт, который утилита `ssh` использует для установления коннекта с удаленных устройств. Так как я запускал контейнер в виртуалке, подключенный через `ssh`, то у меня возник конфликт - контейнер не хочет запускаться, пока порт хоста занят.
Чтобы это исправить я внес изменения в конфиг `sshd` - поменял стандартный порт подключения на другой порт. Нужно иметь в виду, что, проделав такой фокус, нужно сообщить Jenkins новый порт, через который он сможет подключаться к виртуалке, чтобы обнаружить на ней агента.
#### agent без docker

### этапы
После агента можно наконец описывать этапы. Они маркируются ключевым словом `stages`. Все этапы прописываются внутри и называются `stage`:
```Groovy
pipeline {
	agent {
		label "linux && docker"
	}
	stages {
		stage {
			<первый этап>
		}
	}
}
```
Для начала я поставлю в `stage` команду `echo`:
```Groovy
pipeline {
	agent {
		label "linux"
	}
	stages {
		stage {
			echo "Hello world!"
		}
	}
}
```
