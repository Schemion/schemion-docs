---
title: База, хранилище и кэш
description: PostgreSQL, MinIO, бакеты, presigned URLs и кэширование
weight: 10
---

# База данных, объектное хранилище и кэш

## PostgreSQL

Основная база данных - PostgreSQL 16. API использует async SQLAlchemy, а воркеры используют синхронные SQLAlchemy-сессии. Миграции находятся в `schemion-api/alembic`.

Строка подключения по умолчанию для API:

```text
postgresql+asyncpg://admin:admin@database:5432/schemion
```

Строка подключения по умолчанию для воркеров:

```text
postgresql://admin:admin@database:5432/schemion
```

## Основные таблицы

### `users`

Пользователи системы.

Поля:

- `id` - UUID primary key;
- `email` - уникальный email;
- `hashed_password` - хэш пароля;
- `created_at` - время создания.

Связи:

- `tasks`;
- `datasets`;
- `models`;
- `user_roles`.

### `datasets`

Метаданные загруженных датасетов.

Поля:

- `id` - UUID primary key;
- `name` - имя датасета;
- `minio_path` - объектный путь ZIP-архива в MinIO;
- `user_id` - владелец датасета;
- `description` - описание;
- `created_at` - время создания.

Связи:

- `models`;
- `tasks`;
- `user`.

### `models`

Метаданные моделей.

Поля:

- `id` - UUID primary key;
- `name` - имя модели;
- `architecture` - тип архитектуры;
- `architecture_profile` - профиль архитектуры;
- `classes` - массив классов;
- `minio_model_path` - путь к весам модели;
- `metrics_path` - путь к JSON-метрикам;
- `user_id` - владелец модели; для системных моделей может быть `NULL`;
- `is_system` - системная ли модель;
- `base_model_id` - ссылка на базовую модель;
- `dataset_id` - датасет, использованный при обучении;
- `created_at` - время создания.

Связи:

- `dataset`;
- `tasks`;
- `base_model`;
- `derived_models`;
- `user`.

### `tasks`

Асинхронные задачи инференса и обучения.

Поля:

- `id` - UUID primary key;
- `user_id` - владелец задачи;
- `task_type` - `inference` или `training`;
- `status` - `queued`, `running`, `succeeded`, `failed`;
- `model_id` - связанная модель;
- `dataset_id` - связанный датасет;
- `input_path` - входной объект MinIO;
- `output_path` - выходной объект MinIO;
- `error_msg` - текст ошибки;
- `created_at` - время создания;
- `updated_at` - время последнего обновления.

## RBAC-таблицы

В API присутствуют модели:

- `roles`;
- `permissions`;
- `user_roles`;
- `role_permissions`.

JWT включает роли и разрешения пользователя. Middleware административной панели проверяет наличие роли `admin`.

## MinIO

MinIO используется как S3-compatible object storage. В базе данных хранятся только object names, а не сами файлы.

Настройки по умолчанию:

```text
endpoint: minio:9000
public endpoint для ссылок API: localhost:9000
access key: minioadmin
secret key: minioadmin
```

## Бакеты

### `schemas-images`

Входные файлы для инференса. API загружает сюда изображения или PDF из `POST /tasks/create/inference`.

### `datasets`

ZIP-архивы датасетов, загруженные через `POST /datasets/create`.

### `models`

Файлы весов моделей:

- системные модели, загруженные через `system_model_importer`;
- пользовательские модели, загруженные через API;
- fine-tuned модели, созданные training-воркером.

### `metrics`

JSON-файлы метрик обучения.

### `inference-results`

JSON-файлы результатов инференса.

## Object naming

API-слой при загрузке файла в MinIO добавляет UUID-префикс:

```text
<uuid>_<filename>
```

Для датасетов и входных файлов перед загрузкой формируется путь вида:

```text
<user_id>/<filename>
```

Но итоговый object name все равно получает UUID-префикс на уровне `MinioStorage.upload_file`.

`system_model_importer` сохраняет системные модели в:

```text
system/<uuid>_<model_name>.<ext>
```

## Presigned URLs

API не отдает файлы напрямую. Для скачивания используются presigned URLs:

- `/datasets/download/{dataset_id}`;
- `/models/download/{model_id}`;
- `/models/metrics/{model_id}`;
- поле `output_url` в `TaskRead`.

Для задач срок жизни `output_url` равен TTL кэша задач. По умолчанию это около 5 минут с jitter.

## Кэширование

В API реализован in-memory async cache. Это не Redis, несмотря на наличие Redis в `docker-compose.mac.yml`. Кэш живет внутри процесса API и сбрасывается при перезапуске контейнера.

Кэшируются:

- отдельные датасеты;
- списки датасетов;
- отдельные модели;
- списки моделей;
- отдельные задачи;
- списки задач.

TTL:

- списки - `60` секунд;
- датасеты - `5 * 60` секунд;
- модели - `5 * 60` секунд;
- задачи - `5 * 60` секунд;
- пользователь - `5 * 60` секунд.

К TTL добавляется jitter до 30%, чтобы снизить вероятность одновременного истечения большого числа ключей.

## Инвалидация кэша

При создании, удалении и изменении объектов сервисы удаляют точечные ключи и ключи списков по pattern.

Примеры ключей:

```text
dataset:<dataset_id>
model:<model_id>
task:<task_id>
datasets:<user_id>:<skip>:<limit>:<name_filter>
models:<user_id>:<skip>:<limit>:<dataset_id_or_all>
tasks:<user_id>:<skip>:<limit>
```

## Консистентность

Система имеет eventual consistency между API, брокером и воркерами. После создания задачи запись в PostgreSQL появляется сразу, но результат будет доступен только после обработки воркером.

Если воркер недоступен, задача останется в `queued`. Если обработка упала, задача перейдет в `failed`, а причина будет записана в `error_msg`.
