В чем смысл? Чтобы избежать паттерна, в котором логика работы с данными будет разбросана по всему проекту мы можем прибегнуть к использованию <span style="color:#ffc000">репозиториев</span>.
Мы создаем <span style="color:#ffc000">базовый абстрактный репозиторий</span>, от которого наследуемся создавая родительский репозиторий, в котором имплементируем базовый функционал по добавлению и считыванию данных <span style="color:#ffc000">без привязки к конкретной модели</span>.

Тоесть, мы пишем репозиторий, который умеет:
- <span style="color:#ffc000">считывать</span> <span style="color:#ffc000">один</span> элемент по id;
- <span style="color:#ffc000">считывать</span> <span style="color:#ffc000">все</span> элементы;
- <span style="color:#ffc000">добавлять</span> элемент;
- <span style="color:#ffc000">удалять</span> элемент.
Возможно что-то еще, в случае, если это действие не имеет привязки к конкретной модели данных. Это позволяет нам создавать репозитории <span style="color:#ffc000">для работы с разными таблицами</span> не заморачиваясь о необходимости каждый раз разрабатывать один и тот же интерфейс для <span style="color:#ffc000">CRD операций над данными</span>.
Для того, чтобы получить данные для конкретной таблицы, мы будем наследоваться от родительского репозитория и создавать атрибут <span style="color:#00b0f0">self.model</span>, в который передадим значение <span style="color:#ffc000">модели базы данных</span>, с которой будут производиться манипуляции.

## Реализация
Рядом с main.py создаем каталог repositories, в котором будет основной файл base_repository.py и непосредственно сами репозитории, по одному на модель.
```
app/
├─ src/
│  ├─ repositories/
│  │  ├─ base_repository.py
│  │  ├─ user_repository.py
│  ├─ main.py
```

Листинг классического кода создания абстрактного и родительского репозиториев:
``` python
from abc import ABC, abstractmethod  
  
from sqlalchemy import select, insert  
from sqlalchemy.ext.asyncio import AsyncSession  
from src.schemas.user import User as UserSchema  
  
  
class AbstractRepository(ABC):  
    @abstractmethod  
    async def get_all(self) -> list:  
        raise NotImplementedError  
  
    @abstractmethod  
    async def add_one(self, data: dict) -> dict:  
        raise NotImplementedError  
  
  
class Repository(AbstractRepository):  
    model = None  
  
    def __init__(self, session: AsyncSession):  
        self.session = session  
  
    async def get_all(self) -> list[model]:  
        stmt = select(self.model)  
        result = await self.session.execute(stmt)  
        return [self.model.to_model_view(model) for model in result.scalars().all()]  
  
    async def add_one(self, data: dict) -> UserSchema:  
        stmt = insert(self.model).values(data).returning(self.model)  
        result = await self.session.execute(stmt)  
        result = result.scalar_one().to_model_view()  
        await self.session.commit()  
        return result
```

И рядом создаются уже конкретные репозитории, которые привязываются к моделям.
``` python
from sqlalchemy import select  
  
from src.repositories.base_repository import Repository  
  
from src.database.models import Users  
  
  
class UserRepository(Repository):  
    model = Users  
  
    async def get_by_username(self, username: str):  
        stmt = select(self.model).where(Users.username == username)  
        result = await self.session.execute(stmt)  
        return result.scalar_one()
```

В случае, если нам необходимо разработать <span style="color:#ffc000">дополнительный</span> <span style="color:#ffc000">функционал</span>, который будет привязан к <span style="color:#ffc000">конкретной</span> <span style="color:#ffc000">модели</span> <span style="color:#ffc000">данных</span> и актуален только для нее, мы можем создать дополнительные функции для <span style="color:#ffc000">CRUD</span> операций.

``` python
import datetime  
  
from sqlalchemy import select, insert, update  
  
from src.repositories.base_repository import Repository  
from src.database.models import Tasks  
  
  
class TaskRepository(Repository):  
    model = Tasks  
  
    async def get_all(self, username: str | None = None):  
        stmt = select(self.model)  
        if username:  
            stmt = stmt.where(self.model.username == username)  
        result = await self.session.execute(stmt)  
        return [task.to_read_model() for task in result.scalars().all()]  
  
    async def get_task(self, task_id: int):  
        stmt = select(self.model).where(self.model.id == task_id)  
        result = await self.session.execute(stmt)  
        result = result.scalars().first()  
        if result:  
            return result.to_read_model()  
  
    async def update_task(self, task_id: int, status: str, updated_at: datetime.datetime):  
        stmt = update(self.model).values(status=status, updated_at=updated_at).where(self.model.id == task_id).returning(self.model)  
        result = await self.session.execute(stmt)  
        result = result.scalars().first()  
        if result:  
            return result
```