``` cmd
pip install pytest pytest-asyncio
```

### pyproject.toml и старт тестирования
Основной конфигурационный файл для pytest это файл <span style="color:#00b0f0">pyproject.toml</span>
``` pyproject.toml
[tool.pytest.ini_options]  
pythonpath = [".", "src"]  
asyncio_mode = "auto"  
  
[tool.pytest_env]  
APP_MODE="test"
```

Для тестирования необходимо создать побочную базу данных, чтобы не затрагивать основную БД, не забыть прописать новые параметры в .env, либо вовсе создать файл .test.env, в котором будут значения для тестирования, главное не забыть переключиться в режим этого самого тестирования.

Для удобства я через docker compose разворачиваю весь необходимый стек, чтобы тестирование не было обременительным х)
Для начала необходимо войти в режим виртуального окружения
``` cmd
cd venv/Scripts
activate
cd ..
cd ..
```

Затем развернуть тестовую среду
``` cmd
cd tests
docker compose up
cd ..
```

Для тестирования вводим команду
``` cmd
pytest -p no:warnings -s -v
```
	- -p no:warnings убирает depricated предупреждения;
	- -s (stdout), выводим принты и логи
	- -v (verbose), подробный вывод

docker-compose.yml для развертывания postgres & redis будет выглядеть следующим образом:
``` docker-compose.yml
version: "3.7"  
services:  
  db:  
    image: postgres:15  
    container_name: test_db_app  
    ports:  
      - "5433:5432"  
    env_file:  
      - ../src/.test.env  
    #volumes:  
      #- /home/data/postgresql:/var/lib/postgresql/data
  redis:  
	image: redis:alpine3.19  
	container_name: test_db_redis  
	ports:  
	  - "6379:6379"  
	#volumes:  
	  #- /home/data/redis:/data
```

.test.env выглядит следующим образом
``` .test.env
APP_NAME=task_manager  
APP_DESCRIPTION="task_manager pet project"  
  
# authentication  
SECRET_KEY=TimeIsLikeMusic-PlayItTillTheEndAndThenReset  
JWT_ALGORITHM=HS256  
JWT_ACCESS_TOKEN_EXPIRES_MINUTES=1  
JWT_REFRESH_TOKEN_EXPIRES_DAYS=1  
  
#YNC: str DB parameters  
DB_HOST=localhost  
DB_PORT=5433  
DB_USER=postgres  
DB_PASSWORD=admin  
DB_NAME=tasks  
DB_DRIVER_SYNC=psycopg2  
DB_DRIVER_ASYNC=asyncpg  
  
# postgres parameters  
POSTGRES_DB=tasks  
POSTGRES_USER=postgres  
POSTGRES_PASSWORD=admin  
  
REDIS_HOST=localhost
```

### Конфигурирование pytest
Структура каталогов для тестирования
```
app/
├─ test/
│  ├─ auth_test.py
│  ├─ conftest.py
│  ├─ docker-compose.yml
py-project.toml
```

Файл <span style="color:#00b0f0">conftest.py</span> выглядит следующим образом
``` python
import asyncio  
import pytest  
from fastapi.testclient import TestClient  
from httpx import AsyncClient  
from sqlalchemy.pool import NullPool  
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker  
from typing import AsyncGenerator  
  
from src.database.db import Base  
  
from src.config import settings  
from src.main import app  
  
engine_test = create_async_engine(settings.ASYNC_DATABASE_URL, poolclass=NullPool)  
async_session_maker = async_sessionmaker(engine_test, class_=AsyncSession, expire_on_commit=False)  
Base.metadata.bind = engine_test  
  
  
@pytest.fixture(autouse=True, scope='session')  
async def prepare_database():  
    async with engine_test.begin() as conn:  
        await conn.run_sync(Base.metadata.create_all)  
    yield  
    async with engine_test.begin() as conn:  
        await conn.run_sync(Base.metadata.drop_all)  
  
  
# SETUP  
@pytest.fixture(scope='session')  
def event_loop(request):  
    loop = asyncio.get_event_loop_policy().new_event_loop()  
    yield loop  
    loop.close()  
  
  
client = TestClient(app)  
  
  
@pytest.fixture(scope="session")  
async def ac() -> AsyncGenerator[AsyncClient, None]:  
    async with AsyncClient(app=app, base_url="http://127.0.0.1") as ac:  
        yield ac
```

Что такое fixture? - функции, которые выполняются ДО старта тестов pytest. В них мы можем настроить или подготовить тестовую среду.
В нашем случае мы создаем две фикстуры:
- prepare_database(), которая при старте тестов создаст все таблицы в БД, а при завершении тестирования дропнет их;
- event_loop(), создаем (получаем?) event loop;
- ac(), создаем асинхронный клиент для асинхронных функций.

Так же создаем асинхронный клиент тестирования, через который в дальнейшем будем вызывать HTTP-методы GET, POST, etc...

### Создание тестов
Пример теста для аутентификации JWT
``` python
from httpx import AsyncClient  
  
from tests.errors import OPERATION_FAIL, DATA_CONVERT_FAIL  
  
global access_token, refresh_token  
access_token: str | None = None  
refresh_token: str | None = None  
  
  
async def test_registration(ac: AsyncClient):  
    response = await ac.post("/auth/signin", json={  
        "username": "user_test",  
        "password": "password_test"  
    })  
  
    assert response.status_code == 200, OPERATION_FAIL  
    assert response.json().get("username") == "user_test", DATA_CONVERT_FAIL  
  
  
async def test_authentication(ac: AsyncClient):  
    response = await ac.post("/auth/login", data={  
        "username": "user_test",  
        "password": "password_test"  
    })  
  
    assert response.status_code == 200, OPERATION_FAIL  
    assert response.cookies.get('access_token') == response.json().get('access_token'), \  
        "access_token не установлен/ошибка конвертации!"  
    assert response.cookies.get('refresh_token'), "refresh_token не установлен!"  
  
    global access_token, refresh_token  
    access_token, refresh_token = response.cookies.get('access_token'), response.cookies.get('refresh_token')  
  
  
async def test_update(ac: AsyncClient):  
    global access_token, refresh_token  
  
    response = await ac.post("/auth/update", cookies={'access_token': access_token, 'refresh_token': refresh_token})  
  
    assert response.status_code == 200, OPERATION_FAIL  
    assert response.cookies.get('access_token') != access_token, "Выданный access токен совпадает с предыдущим!"  
    assert response.cookies.get('refresh_token') != refresh_token, "Выданный refresh токен совпадает с предыдущим!"  
  
    access_token, refresh_token = response.cookies.get('access_token'), response.cookies.get('refresh_token')  
  
  
async def test_authorization(ac: AsyncClient):  
    response = await ac.post("/auth/authorize", cookies={'access_token': access_token, 'refresh_token': refresh_token})  
  
    assert response.status_code == 200, OPERATION_FAIL  
    assert response.json()  
  
  
async def test_autoupdate(ac: AsyncClient):  
    """ Тест направлен на проверку работоспособности автоматического обновления access&refresh tokens,  
    в случае истечения срока действия JWT токена, или в случае отсутствия access_token в куках.    """    global access_token, refresh_token  
  
    ''' Процедура обновления токенов происходит равнозначно в случаях, если access_token истек или отсутствует в cookies  
        Если необходимо так-же проверить случай истечения срока жизни токена - раскомментируйте код '''    # await asyncio.sleep(60)  
    # response = await ac.post("/auth/authorize", cookies={'access_token': access_token, 'refresh_token': refresh_token})  
    # в случае отсутствия access_token в куках начнется процедура обновления токенов.    response = await ac.post("/auth/authorize", cookies={'access_token': '', 'refresh_token': refresh_token})  
  
    assert response.status_code == 200, OPERATION_FAIL  
    assert response.cookies.get('refresh_token') != refresh_token, "Обновление токенов произведено не было!"
```

Чуть попроще
``` python
from httpx import AsyncClient  
  
  
async def get_headers(ac: AsyncClient) -> dict:  
    response = await ac.post("/auth/login", data={  
        "username": "user_test",  
        "password": "password_test"  
    })  
    my_headers = {'access_token': response.cookies.get('access_token'),  
                  'refresh_token': response.cookies.get('refresh_token')}  
    return my_headers  
  
  
async def test_add_task(ac: AsyncClient):  
    my_headers = await get_headers(ac)  
  
    json = {  
        "title": "test",  
        "description": "test",  
        "username": "user_test",  
        "status": "planned",  
    }  
    response = await ac.post(  
        "/tasks/",  
        json=json, headers=my_headers  
    )  
    assert response.status_code == 201  
    for k, v in json.items():  
        assert response.json()[k] == v  
  
  
async def test_get_task(ac: AsyncClient):  
    my_headers = await get_headers(ac)  
  
    task_id = 1  
    response = await ac.get(  
        "/tasks/?task_id={}".format(task_id), headers=my_headers  
    )  
    assert response.status_code == 200  
    assert response.json()  
  
  
async def test_update_task(ac: AsyncClient):  
    my_headers = await get_headers(ac)  
  
    task_id = 1  
    response = await ac.patch(  
        "/tasks/?task_id={}".format(task_id), json={'status': 'pending'}, headers=my_headers  
    )  
    assert response.status_code == 200  
    assert response.json()['status'] == 'pending'  
  
  
async def test_delete_task(ac: AsyncClient):  
    my_headers = await get_headers(ac)  
  
    task_id = 1  
    response = await ac.delete(  
        "/tasks/?task_id={}".format(task_id), headers=my_headers  
    )  
    assert response.json()['status'] == 'deleted'
```