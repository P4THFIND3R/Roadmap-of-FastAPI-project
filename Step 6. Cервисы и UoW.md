## База
### UnitOfWork
Для чего вообще сервисы и UoW (UnitOfWork)? - Поясняю.

"Паттерн Unit of Work часто используется в сочетании с шаблоном репозитория для <span style="color:#ffc000">управления</span> <span style="color:#ffc000">координацией</span> и <span style="color:#ffc000">отслеживанием</span> <span style="color:#ffc000">изменений</span> в базе данных. Unit of Work отвечает за обеспечение того, чтобы изменения, внесенные в ходе бизнес-транзакции, <span style="color:#ffc000">последовательно</span> <span style="color:#ffc000">фиксировались</span> или <span style="color:#ffc000">откатывались</span>."

То есть, если в процессе работы ты взаимодействуешь не только с одним репозиторием, и тебе нужно соблюсти условие <span style="color:#ffc000">атомарности</span>, для этого необходимости использовать UoW, который <span style="color:#ffc000">аккумулирует</span> в себе некоторое количество <span style="color:#ffc000">репозиториев</span>, проверяет, чтоб <span style="color:#ffc000">транзакция</span> <span style="color:#ffc000">была выполнена</span> от и до, и только в случае успешного выполнения каждого из этапов производит commit в базу данных.
### Сервисы (контроллер)
Для того, чтобы <span style="color:#ffc000">инкапсулировать</span> <span style="color:#ffc000">бизнес-логику</span> приложения от логики работы с данными можно (необходимо) ввести еще один уровень <span style="color:#ffc000">абстракции</span>, который будет называться <span style="color:#ffc000">сервисом</span>. Сервисный уровень будет <span style="color:#ffc000">защищать</span> слой API(его эндпоинтов) <span style="color:#ffc000">от изменения</span> в механизмах <span style="color:#ffc000">доступа к данным</span>.
Благодаря ему можно централизовать механизм <span style="color:#ffc000">логирования</span> ошибок и <span style="color:#ffc000">сбора</span> <span style="color:#ffc000">дополнительной</span> <span style="color:#ffc000">информации</span>, без привязки к конкретным эндпоинтам.

Получается, что <span style="color:#00b0f0">конечные точки</span> будут дергать <span style="color:#00b0f0">методы сервиса</span>, а <span style="color:#00b0f0">сервис</span> уже будет взаимодействовать <span style="color:#00b0f0">с данными</span> при помощи <span style="color:#00b0f0">UoW</span>.
Так-же, <span style="color:#ffc000">на уровне сервисов</span> происходит <span style="color:#ffc000">проверка</span> <span style="color:#ffc000">и</span> <span style="color:#ffc000">форматирование</span> <span style="color:#ffc000">данных</span> перед тем, как отправлять их в БД!
## Реализация
### UnitOfWork
Обычно, UoW реализуют в каталоге utils, в котором размещают всякие побочные утилиты, по типу штук доступа к стороннему API. Не уверен в том, что это действительно подходящее место для его размещения, ну и ладно.

Весь UoW будет располагаться в одном файле, сначала реализуем абстрактный интерфейс, чтоб наследоваться от него и передавать его в аннотации типов.
Листинг таков:
``` python
from abc import ABC, abstractmethod  
  
from src.database.db import async_session_maker  
from src.repositories.user_repository import UserRepository  

  
class IUnitOfWork(ABC):  
    user_repos: UserRepository  
  
    @abstractmethod  
    def __init__(self):  
        ...  
  
    @abstractmethod  
    async def __aenter__(self):  
        ...  
  
    @abstractmethod  
    async def __aexit__(self, exc_type, exc_val, exc_tb):  
        ...  
  
    @abstractmethod  
    async def commit(self):  
        ...  
  
    @abstractmethod  
    async def rollback(self):  
        ...  
```

Затем наследуемся от него и создаем полноценный UoW:
``` python
class UnitOfWork(IUnitOfWork):  
    def __init__(self):  
        self.session_factory = async_session_maker  
  
    async def __aenter__(self):  
        self.session = self.session_factory()  
  
        # repositories  
        self.user_repos = UserRepository(self.session)  
        self.task_repos = TaskRepository(self.session)  
  
    async def __aexit__(self, exc_type, exc_val, exc_tb):  
        await self.rollback()  
        await self.session.close()  
  
    async def rollback(self):  
        await self.session.rollback()  
  
    async def commit(self):  
        await self.session.commit()
```

При инициализации мы создает атрибут асинхронного создателя сессий, при входе в контекстный менеджер <span style="color:#ffc000">async with</span> <span style="color:#ffc000">создаем сессию</span>, а так-же <span style="color:#ffc000">истанцируем</span> <span style="color:#ffc000">экземпляры</span> всех <span style="color:#ffc000">репозиториев</span>, которые существуют в проекте. Возможно в дальнейшем придется как-то масштабировать создание экземпляров репозиториев, но я до этого еще не доходил, по словам автора курса, такого решения хватает для практически всех проектов.

При выходи из асинхронного контекстного менеджера без <span style="color:#ffc000">commit</span> мы <span style="color:#ffc000">отменяем</span> все <span style="color:#ffc000">изменения</span> и <span style="color:#ffc000">закрываем</span> <span style="color:#ffc000">сессию</span>. При коммите в конце транзакции мы успешно <span style="color:#ffc000">сохраняем</span> все данные <span style="color:#ffc000">в БД</span>, после чего rollback при выходе из менеджера уже не повлияет на данные.

### Сервисы
Обычно, для каждой группы роутов создается свой сервис, тоесть, одни сервис для пользователей покроет все потребности в работе с пользователем, и т.д.

Сервис будет выглядеть следующим образом:
``` python
from src.schemas.user import User as UserSchema  
from src.utils.uow import IUnitOfWork  
  
  
class UserService:  
    def __init__(self, uow: IUnitOfWork):  
        self.uow = uow  
  
    async def get_user(self, username: str):  
        async with self.uow:  
            res = await self.uow.user_repos.get_by_username(username)  
            return res.to_model_view()
```

При инициализации <span style="color:#ffc000">определяем UoW</span>, и далее пишем <span style="color:#ffc000">функции</span>, в которых <span style="color:#ffc000">описываем</span> <span style="color:#ffc000">бизнес-логику</span> приложения. <span style="color:#ffc000">Обязательно</span> в начале функции <span style="color:#ffc000">входим</span> <span style="color:#ffc000">в</span> асинхронный контекстный менеджер <span style="color:#ffc000">async with self.uow</span>.
В случае, если внутри функции производятся какие-либо <span style="color:#ffc000">манипуляции</span><span style="color:#ffc000"> с данными</span>, в конце функции <span style="color:#ffc000">перед</span> <span style="color:#ffc000">возвратом</span> значения (концом функции) необходимо <span style="color:#ffc000">совершить</span> <span style="color:#ffc000">self.uow.commit()</span>, чтобы данные <span style="color:#ffc000">транзакции</span> были успешно <span style="color:#ffc000">зафиксированы</span> базой данных.