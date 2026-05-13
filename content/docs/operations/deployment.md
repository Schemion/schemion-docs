---
title: Развертывание и конфигурация
description: Docker Compose, сервисы, порты, переменные окружения и миграции
weight: 20
---

# Развертывание и конфигурация

## Docker Compose

В корне проекта есть два основных compose-файла:

- `docker-compose.yml` - основной вариант с GPU reservation для training-сервиса.
- `docker-compose.mac.yml` - вариант для macOS без GPU reservation и с Redis-сервисом.

Запуск основного окружения:

```bash
docker compose up --build
```

Запуск macOS-варианта:

```bash
docker compose -f docker-compose.mac.yml up --build
```

## Сервисы и порты

### `nginx`

- image: `nginx:1.27-alpine`
- порт: `80:80`
- проксирует `/api/` в `server:8000`
- раздает фронтенд из `nginx/dist`

### `server`

- build context: `./schemion-api`
- порт: `8000:8000`
- команда запуска:

```text
alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 1
```

Перед стартом ждет PostgreSQL через `wait-for-it.sh`.

### `trainer`

- build context: `./schemion-training`
- команда запуска: `python -m app.main`
- зависит от PostgreSQL, MinIO и Bobber.

В `docker-compose.yml` запрашивает один NVIDIA GPU:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: 1
          capabilities: [gpu]
```

### `inference`

- build context: `./schemion-inference`
- команда запуска: `python -m app.infrastructure.main`
- зависит от PostgreSQL, MinIO и Bobber.

### `bob-the-broker`

- image: `ghcr.io/schemion/bob-the-broker:v2.2`
- порт: `50051:50051`

### `database`

- image: `postgres:16`
- порт: `5432:5432`
- база: `schemion`
- пользователь: `admin`
- пароль: `admin`

### `minio`

- image: `minio/minio:RELEASE.2025-09-07T16-13-09Z`
- API порт: `9000:9000`
- console порт: `9001:9001`
- пользователь: `minioadmin`
- пароль: `minioadmin`

### `redis`

Redis присутствует в `docker-compose.mac.yml`:

- image: `redis:7-alpine`
- порт: `6379:6379`
- пароль: `adminpass`

В текущем API-коде кэш реализован in-memory, поэтому Redis не является обязательной частью runtime-пути API.

## Переменные окружения API

`schemion-api/app/infrastructure/config.py` читает `.env` через Pydantic Settings.

Основные переменные:

```text
DATABASE_URL=postgresql+asyncpg://admin:admin@database:5432/schemion
JWT_SECRET=supersecret
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=10
REFRESH_TOKEN_EXPIRE_MINUTES=10080
MINIO_ENDPOINT=minio:9000
MINIO_PUBLIC_ENDPOINT=files.localhost
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_SCHEMAS_BUCKET=schemas-images
MINIO_MODELS_BUCKET=models
MINIO_DATASETS_BUCKET=datasets
MINIO_METRICS_BUCKET=metrics
MINIO_INFERENCES_BUCKET=inference-results
BOBBER_HOST=bob-the-broker
BOBBER_PORT=50051
```

Примечание: в реализации `MinioStorage` API presigned URLs подписываются через hardcoded public endpoint `localhost:9000`, а не через `MINIO_PUBLIC_ENDPOINT`.

## Переменные окружения training-воркера

`schemion-training/app/config.py` читает `.env` через `python-dotenv`.

```text
DATABASE_URL=postgresql://admin:admin@database:5432/schemion
REDIS_BROKER_URL=redis://:adminpass@redis:6379/1
RABBITMQ_URL=amqp://admin:admin@rabbitmq:5672/
MINIO_ENDPOINT=minio:9000
MINIO_PUBLIC_ENDPOINT=files.localhost
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_SCHEMAS_BUCKET=schemas-images
MINIO_MODELS_BUCKET=models
MINIO_DATASETS_BUCKET=datasets
MINIO_METRICS_BUCKET=metrics
BOBBER_HOST=bob-the-broker
BOBBER_PORT=50051
```

`REDIS_BROKER_URL` и `RABBITMQ_URL` присутствуют как настройки, но текущий runtime-путь использует Bobber.

## Переменные окружения inference-воркера

`schemion-inference/app/config.py`:

```text
DATABASE_URL=postgresql://admin:admin@database:5432/schemion
RABBITMQ_URL=amqp://admin:admin@rabbitmq:5672/
REDIS_BROKER_URL=redis://:adminpass@redis:6379/1
BOBBER_HOST=bob-the-broker
BOBBER_PORT=50051
JWT_SECRET=supersecret
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60
MINIO_ENDPOINT=minio:9000
MINIO_PUBLIC_ENDPOINT=files.localhost
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_SCHEMAS_BUCKET=schemas-images
MINIO_MODELS_BUCKET=models
MINIO_DATASETS_BUCKET=datasets
MINIO_INFERENCE_RESULTS_BUCKET=inference-results
```

## Nginx

Конфигурация находится в `nginx/nginx.conf`.

Важные правила:

```nginx
location /api/ {
    proxy_pass http://schemion/;
}

location / {
    try_files $uri $uri/ /index.html;
}
```

Поскольку `proxy_pass` заканчивается на `/`, префикс `/api/` заменяется на `/` при проксировании.

## Миграции

API-контейнер автоматически выполняет:

```bash
alembic upgrade head
```

перед запуском Uvicorn. Конфигурация Alembic находится в:

```text
schemion-api/alembic.ini
schemion-api/alembic/
```

## Загрузка системных моделей

Утилита `system_model_importer/main.py` рассчитана на локальную загрузку файлов из директории:

```text
system_model_importer/models
```

По умолчанию она подключается к:

```text
PostgreSQL: localhost:5432
MinIO: localhost:9000
```

Файлы `.pt` и `.pth` загружаются в бакет `models`, затем создается запись в таблице `models` с:

- `is_system=True`;
- `architecture="yolo"`, если имя файла содержит `yolo`;
- `architecture="faster_rcnn"`, если имя файла содержит `faster`;
- `architecture="unknown"` иначе;
- `architecture_profile="default"`.

## Замечания по зависимостям

В корневом `README.md` указано, что в submodules Dockerfile использует китайское зеркало PyPI:

```bash
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple/ ...
```

Для возврата к стандартному PyPI используется:

```bash
pip install --no-cache-dir -r requirements.txt
```

Это важно учитывать при сборке контейнеров в сетях, где внешние индексы пакетов ограничены.
