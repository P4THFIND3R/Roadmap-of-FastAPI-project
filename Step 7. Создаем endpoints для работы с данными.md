Поскольку у нас и UoW, и сервисы, и чего только нет, работа с данными требует некоторых приготовлений.
Для того, чтобы сократить количество кода в эндпоинтах необходимо в каталоге с конечными точками создать файл dependencies.py, в котором будет хранить псевдонимы для зависимостей.
```
app/
├─ src/
│  ├─ api/
│  │  ├─ new_folder/
│  │  │  ├─ dependencies.py
│  │  │  ├─ user_router.py
```

Листинг dependencies.py
``` python
from typing import Annotated  
from fastapi import Depends  
from src.services.user_service import UserService  
from src.utils.uow import IUnitOfWork, UnitOfWork  


UOWDep = Annotated[IUnitOfWork, Depends(UnitOfWork)]  
  
def get_user_service(uow: UOWDep) -> UserService:  
    return UserService(uow)  
  
user_service_dep = Annotated[UserService, Depends(get_user_service)]  
```

Сначала мы создаем аннотацию с типом интерфейса IUoW и при помощи Depends получим экземпляр UoW. Создадим функцию для получения экземпляра сервиса, и аннотацию для зависимости от этого сервиса. 

Благодаря этому в эндпоинтах нам не придется многократно повторять код. user_router.py
``` python
from fastapi import APIRouter  
  
from src.api.endpoints.dependencies import user_service_dep  
from src.schemas.user import User as UserSchema  
  
router = APIRouter(prefix='/users', tags=["users"])  
  
  
@router.get('')  
async def get_users(username: str, user_service: user_service_dep):  
    user = await user_service.get_user(username)  
    return user  
  
  
@router.post('')  
async def add_user(userdata: UserSchema, user_service: user_service_dep):  
    user = await user_service.add_user(userdata)  
    return user
```
