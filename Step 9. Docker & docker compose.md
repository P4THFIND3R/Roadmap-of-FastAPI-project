В чем суть? Если вкратце, поднимаем обрезанные линукс-образы, на которых уже установлены те или иные дистрибутивы, дабы они работали независимо от других сервисов, которые есть в нашем проекте.

## Dockerfile
Для того, чтобы упаковать наше собственное приложение в контейнер необходимо создать Dockerfile, в котором будет описана инструкция по развертыванию нашего "образа".

``` Dockerfile
FROM python:3.11  
  
RUN mkdir /fastapi_app  
  
WORKDIR /fastapi_app  
  
COPY requirements.txt .  
  
RUN pip install -r requirements.txt  
  
COPY . .
```

- Мы создаем image из образа python:3.11;
- Создаем директорию, в котором будет хранится наше приложение;
- Далее назначаем эту директорию рабочей директорий;
- Копируем из текущего каталога файл requirements.txt в workdir;
- Устанавливаем все зависимости для приложения;
- Копируем все файлы из текущей директории в workdir.

После чего необходимо запустить процесс сборки образа командой
``` cmd
docker build . -t fastapi_app:latest
```
	 - . означает текущую директорию;
	 - -t, создаем имя нашему образу

Чтобы запустить контейнер из образа
``` cmd
docker run -p 8080:8080 fastapi_app
```

## docker-compose.yml
В чем суть? Если нам необходимо собрать некоторое количество сервисов вместе, чтобы они запускались одновременно, имели доступ друг к другу и т.д.

docker-compose.yml
``` yml
version: "3.7"  
services:  
  db:  
    image: postgres:15  
    container_name: db_app  
    expose:  
      - 5432  
    env_file:  
      - /src/.prod.env  
    #volumes:  
      #- /home/data/postgresql:/var/lib/postgresql/data  redis:  
    image: redis:alpine3.19  
    container_name: redis_db  
    expose:  
      - 6379  
    #volumes:  
      #- /home/data/redis:/data  app:  
    build:  
      context: .  
    env_file:  
      - /src/.prod.env  
    container_name: app  
    restart: on-failure  
    environment:  
      - APP_MODE=prod  
    command: bash -c "alembic upgrade head && uvicorn src.main:app --host 0.0.0.0 --port 80"  
    ports:  
      - "80:80"  
    depends_on:  
      - db
```

## Запуск
В директории с файлом docker-compose.yml, в случае, если нам нужно создать образ приложения, Dockerfile должен находиться здесь же.
``` cmd
docker compose up --build
```

Либо просто
``` cmd
docker compose up
```

Чтобы остановить docker compose
``` cmd
docker compose down

# в случае, если надо дропнуть volumes
docker compose down --volumes
```