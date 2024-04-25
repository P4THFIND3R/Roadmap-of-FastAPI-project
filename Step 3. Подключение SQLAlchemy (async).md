SQLAlchemy ORM (Object-Relational Mapping) — это технология, которая позволяет сопоставлять модели, типы которых несовместимы. Например: таблица базы данных и объект языка программирования.

В SQLAlchemy есть понятие <span style="color:#ffc000">декларативных</span> и <span style="color:#ffc000">недекларативных</span> определений моделей.

<span style="color:#ffc000">Недекларативные</span> определения являются новым методом создания моделей и подразумевают использования mapper(), описывающего сопоставление каждой колонки БД и классом модели.

Пример <span style="color:#ffc000">декларативного</span> создания модели:
``` python
class City(Base):
    __tablename__ = "cities"

    id = Column(Integer, autoincrement=True, primary_key=True, index=True)
    name = Column(String, unique=True)
    population = Column(Integer)
```
Пример <span style="color:#ffc000">недекларативного</span> создания модели:
``` python
class Tasks(Base):  
    __tablename__ = "tasks"  
  
    id: Mapped[BigInteger] = mapped_column(BigInteger, primary_key=True, nullable=False, autoincrement=True, index=True)  
    title: Mapped[String] = mapped_column(String, nullable=False)  
```
## Step 3.1 - Создание подключения
Файлик database.py должен располагаться рядом с main.py, либо в каталоге database на одном уровне с main.py
```
app/
├─ src/
│  ├─ database/
│  │  ├─ database.py
│  │  ├─ models.py
│  ├─ main.py

```

Сам файлик выглядит следующим образом.
``` python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine  
from sqlalchemy.orm import DeclarativeBase  
  
from src.config import settings  
  
  
class Base(DeclarativeBase):  
    pass  
  
  
async_engine = create_async_engine(settings.ASYNC_DATABASE_URL)  
async_session_maker = async_sessionmaker(async_engine, class_=AsyncSession)  
  
  
async def get_async_session() -> AsyncSession:  
    async with async_sessionmaker() as session:  
        yield session
```

## Step 3.2 - Создание моделей БД
Модели БД принято хранить в отдельной папке models, либо в файле рядом с конфигурацией БД.
``` python
app/
├─ src/
│  ├─ database/
│  │  ├─ database.py
│  │  ├─ models.py
```

Листинг файла models.py, в котором расположены модели для одного из моих предыдущих проектов.
``` python
from sqlalchemy.orm import Mapped, mapped_column  
from sqlalchemy import BigInteger, String, ForeignKey, DateTime  
  
from src.database.db import Base  
from src.api.schemas.user import User  
from src.api.schemas.task import Task, TaskFromDB  
  
  
class Tasks(Base):  
    __tablename__ = "tasks"  
  
    id: Mapped[BigInteger] = mapped_column(BigInteger, primary_key=True, nullable=False, autoincrement=True, index=True)  
    title: Mapped[String] = mapped_column(String, nullable=False)  
    description: Mapped[String] = mapped_column(String, nullable=True)  
    status: Mapped[String] = mapped_column(String, nullable=False, default="pending")  
  
    created_at: Mapped[DateTime] = mapped_column(DateTime, nullable=False)  
    updated_at: Mapped[DateTime] = mapped_column(DateTime, nullable=False)  
    username: Mapped[BigInteger] = mapped_column(String, ForeignKey("users.username"), nullable=False)  
  
    def to_read_model(self) -> TaskFromDB:  
        return TaskFromDB(  
            id=self.id,  
            title=self.title,  
            description=self.description,  
            status=self.status,  
            created_at=self.created_at,  
            updated_at=self.updated_at,  
            username=self.username  
        )  
  
  
class Users(Base):  
    __tablename__ = "users"  
  
    username: Mapped[String] = mapped_column(String, primary_key=True, nullable=False)  
    password: Mapped[String] = mapped_column(String, nullable=False)  
  
    def to_read_model(self) -> User:  
        return User(  
            username=self.username,  
            password=self.password  
        )
```

В чем смысл, дабы не писать голые SQL-запросы, мы при помощи SQLAlchemy создаем <span style="color:#ffc000">модели</span> <span style="color:#ffc000">таблиц</span> <span style="color:#ffc000">БД</span>, а затем при помощи этих моделей будем взаимодействовать с базой данных, таблицы которых будут представлены <span style="color:#ffc000">в виде</span> <span style="color:#ffc000">объектов</span> <span style="color:#ffc000">Python</span>.
<span style="color:#ffc000">Base</span> агрегирует в себе все таблицы, создается на этапе инициализации подключения к БД.
<span style="color:#00b0f0">to_read_model</span><span style="color:#00b0f0">()</span> - функция, при помощи которой можно преобразовать модель БД в схему Pydantic.