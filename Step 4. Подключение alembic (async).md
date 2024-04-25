В ходе развития приложения структура данных в нем может многократно изменяться. Для того, чтобы <span style="color:#ffc000">фиксировать</span> <span style="color:#ffc000">эти</span> <span style="color:#ffc000">изменения</span> подобно коммитам в git, а так-же иметь возможность <span style="color:#ffc000">обновлять</span> <span style="color:#ffc000">таблицы</span> <span style="color:#ffc000">в БД</span> одновременно с обновлением таблиц в приложении необходимо использовать <span style="color:#ffc000">миграции</span>, в нашем случае при помощи <span style="color:#ffc000">alembic</span>.

## Установка и настройка
Устанавливаем, а затем находясь в корне проекта инициализируем alembic.
``` shell
pip install alembic
alembic init alembic
```

Далее переходим в alembic.ini, который автоматически создался в корне проекта и модифицируем его. По желанию убираем комментарии, на самом деле нас интересует лишь одна строка в файле:
``` ini
sqlalchemy.url = postgresql+%(DB_DRIVER_ASYNC)s://%(DB_USER)s:%(DB_PASSWORD)s@%(DB_HOST)s:%(DB_PORT)s/%(DB_NAME)s
```

Здесь происходит форматирование при помощи %()s, параметры мы укажем в файле env.py, который находится в директории alembic. Но с этим файлом дела обстоят посложнее.

``` python
from logging.config import fileConfig  
import os  # Добавили импорт os для извлечения из окружения наших переменных  
import sys  # Добавили импорт модуля sys для работы с путями  
  
# Тут добавили в пути нашу папку src, чтобы алембик её увидел.  
sys.path.append(os.path.join(sys.path[0], 'src'))  
  
from logging.config import fileConfig  
from sqlalchemy import engine_from_config  
from sqlalchemy import pool  
from alembic import context  
# Импортируем настройки из конфиг
from src.config import settings  
# чтобы алембик увидел таблицы, их нужно импортировать  
from src.database.models import *  
  
# this is the Alembic Config object, which provides  
# access to the values within the .ini file in use.  
config = context.config  
section = config.config_ini_section  
config.set_section_option(section, 'DB_PORT', settings.DB_PORT)  
config.set_section_option(section, 'DB_USER', settings.DB_USER)  
config.set_section_option(section, 'DB_PASSWORD', settings.DB_PASSWORD)  
config.set_section_option(section, 'DB_HOST', settings.DB_HOST)  
config.set_section_option(section, 'DB_NAME', settings.DB_NAME)  
config.set_section_option(section, 'DB_DRIVER_ASYNC', settings.DB_DRIVER_SYNC)  
  
# Interpret the config file for Python logging.  
# This line sets up loggers basically.  
if config.config_file_name is not None:  
    fileConfig(config.config_file_name)  
  
target_metadata = Base.metadata  
  
  
# other values from the config, defined by the needs of env.py,  
# can be acquired:  
# my_important_option = config.get_main_option("my_important_option")  
# ... etc.  
  
  
def run_migrations_offline() -> None:  
    """Run migrations in 'offline' mode.  
  
    This configures the context with just a URL    and not an Engine, though an Engine is acceptable    here as well.  By skipping the Engine creation    we don't even need a DBAPI to be available.  
    Calls to context.execute() here emit the given string to the    script output.  
    """    url = config.get_main_option("sqlalchemy.url")  
    context.configure(  
        url=url,  
        target_metadata=target_metadata,  
        literal_binds=True,  
        dialect_opts={"paramstyle": "named"},  
    )  
  
    with context.begin_transaction():  
        context.run_migrations()  
  
  
def run_migrations_online() -> None:  
    """Run migrations in 'online' mode.  
  
    In this scenario we need to create an Engine    and associate a connection with the context.  
    """    connectable = engine_from_config(  
        config.get_section(config.config_ini_section, {}),  
        prefix="sqlalchemy.",  
        poolclass=pool.NullPool,  
    )  
  
    with connectable.connect() as connection:  
        context.configure(  
            connection=connection, target_metadata=target_metadata  
        )  
  
        with context.begin_transaction():  
            context.run_migrations()  
  
  
if context.is_offline_mode():  
    run_migrations_offline()  
else:  
    run_migrations_online()
```

## Создание миграций
После настройки Alembic вы можете приступить к созданию миграций, которые представляют собой изменения в схеме вашей базы данных. Миграция - это скрипт на Python, который определяет, как применять изменения (upgrade) и, при необходимости, отменять эти изменения (downgrade).

Для создания миграции в автоматическом режиме используем команду:
``` cmd
alembic revision --autogenerate -m "rev_name"
```

Для обновления БД до актуальной версии необходимо использовать команду:
``` cmd
alembic upgrade head
```

Для обновления/даунгрейда БД до конкретной версии можно использовать следующие команды:
``` cmd
alembic upgrade +2
alembic downgrade -1
```

Для получения текущей информации:
``` cmd
alembic current
```

В случае, если сделали какую-то поганую ревизию, мы не можем просто удалить ее файл из папки с ревизиями, нам придется удалять запись об этой ревизии и из БД.
### Ревизия для миграции с нуля
Если хотим получить миграцию для установки базы данных с нуля, мы можем поступить следующим образом:

If you are introducing `alembic/sqlalchemy` to an existing database, and you want a migration file that given an empty, fresh database would reproduce the current state- follow these steps.

1. Ensure that your `metadata` is truly in line with your current `database`(i.e. ensure that running `alembic revision --autogenerate` creates a migration with zero operations).
    
2. Create a new `temp_db` that is empty and point your `sqlalchemy.url` in `alembic.ini` to this new `temp_db.`
    
3. Run `alembic revision --autogenerate`. This will create your desired bulk migration that brings a fresh db in line with the current one.
    
4. Remove `temp_db` and re-point `sqlalchemy.url` to your existing database.
    
5. Run `alembic stamp head`. This tells sqlalchemy that the current migration represents the state of the database- so next time you run `alembic upgrade head` it will begin from this migration.