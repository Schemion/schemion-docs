---
title: API Reference
description: Подробное описание HTTP API, авторизации, лимитов и эндпоинтов
weight: 20
---

# API Reference

## Общие сведения

HTTP API реализован на FastAPI. При обращении напрямую к контейнеру `server` базовый адрес обычно:

```text
http://localhost:8000
```

При обращении через nginx API доступен с префиксом:

```text
http://localhost/api
```

В конфигурации nginx `location /api/` проксирует запросы в API-сервис с удалением префикса `/api/`. Например, внешний путь `/api/auth/login` становится внутренним `/auth/login`.

FastAPI по умолчанию предоставляет интерактивную OpenAPI-документацию:

- напрямую: `http://localhost:8000/docs`
- через nginx: `http://localhost/api/docs`

## Аутентификация

Большинство эндпоинтов требуют заголовок:

```http
Authorization: Bearer <access_token>
```

Access token и refresh token выдаются эндпоинтами `/auth/register` и `/auth/login`. JWT содержит:

- `sub` - UUID пользователя;
- `roles` - список ролей;
- `permissions` - список разрешений;
- `type` - тип токена для refresh token.

## Ошибки

Пользовательские исключения API преобразуются в JSON:

```json
{"detail": "Описание ошибки"}
```

Основные коды:

- `400` - ошибка валидации бизнес-логики или файла;
- `401` - пользователь не аутентифицирован или токен некорректен;
- `403` - недостаточно прав, если исключение FastAPI не перехвачено отдельно;
- `404` - объект не найден;
- `408` - чтение файла или создание задачи превысило timeout;
- `413` - файл модели больше допустимого лимита;
- `429` - превышен rate limit.

## Rate limits

Глобальный лимит:

```text
100/minute
```

Специализированные лимиты:

- `POST /models/create` - `5/minute`;
- `POST /tasks/create/inference` - `10/minute`;
- `POST /tasks/create/training` - `2/hour`.

Лимит считается по IP-адресу клиента через `slowapi.util.get_remote_address`.

## Auth

### `POST /auth/register`

Создает пользователя и возвращает пару токенов.

Тело запроса:

```json
{
  "email": "user@example.com",
  "password": "strong-password",
  "role": "user"
}
```

Поле `role` имеет значение по умолчанию `user`. В коде также определена роль `admin`.

Ответ `201 Created`:

```json
{
  "access_token": "<jwt>",
  "refresh_token": "<jwt>",
  "token_type": "bearer"
}
```

### `POST /auth/login`

Аутентифицирует пользователя по email и паролю.

Тело запроса:

```json
{
  "email": "user@example.com",
  "password": "strong-password"
}
```

Ответ:

```json
{
  "access_token": "<jwt>",
  "refresh_token": "<jwt>",
  "token_type": "bearer"
}
```

### `POST /auth/refresh`

Выдает новый access token по refresh token.

Тело запроса:

```json
{
  "refresh_token": "<refresh_jwt>"
}
```

Ответ:

```json
{
  "access_token": "<new_access_jwt>",
  "token_type": "bearer"
}
```

## Datasets

### `POST /datasets/create`

Создает датасет. Требует `multipart/form-data`.

Поля формы:

- `name` - имя датасета;
- `description` - необязательное описание;
- `file` - ZIP-архив датасета.

Пример:

```bash
curl -X POST http://localhost:8000/datasets/create \
  -H "Authorization: Bearer $TOKEN" \
  -F "name=Road defects dataset" \
  -F "description=YOLO-format detection dataset" \
  -F "file=@dataset.zip"
```

Ответ `201 Created`:

```json
{
  "name": "Road defects dataset",
  "minio_path": "uuid_dataset.zip",
  "description": "YOLO-format detection dataset",
  "id": "00000000-0000-0000-0000-000000000000",
  "user_id": "00000000-0000-0000-0000-000000000000"
}
```

### `GET /datasets/`

Возвращает список датасетов текущего пользователя.

Query parameters:

- `skip` - смещение, по умолчанию `0`;
- `limit` - лимит, по умолчанию `100`;
- `name_contains` - необязательный фильтр по имени.

### `GET /datasets/{dataset_id}`

Возвращает один датасет по UUID. Доступ разрешен только владельцу датасета.

### `GET /datasets/download/{dataset_id}`

Возвращает presigned URL для скачивания ZIP-архива датасета.

Ответ:

```json
{
  "download_url": "http://localhost:9000/datasets/..."
}
```

### `DELETE /datasets/{dataset_id}`

Удаляет датасет из MinIO и PostgreSQL. Ответ `204 No Content`.

## Models

### `POST /models/create`

Загружает пользовательские веса модели. Требует `multipart/form-data`.

Поля формы:

- `name` - имя модели;
- `architecture` - `yolo` или `faster_rcnn`;
- `architecture_profile` - профиль архитектуры, например `default`, `resnet50_fpn`, `resnet50_fpn_v2`;
- `dataset_id` - необязательная связь с датасетом;
- `file` - файл весов `.pt` или `.pth`.

Пример:

```bash
curl -X POST http://localhost:8000/models/create \
  -H "Authorization: Bearer $TOKEN" \
  -F "name=custom-yolo" \
  -F "architecture=yolo" \
  -F "architecture_profile=default" \
  -F "file=@model.pt"
```

Ответ `201 Created`:

```json
{
  "name": "custom-yolo",
  "architecture": "yolo",
  "architecture_profile": "default",
  "minio_model_path": "uuid_model.pt",
  "id": "00000000-0000-0000-0000-000000000000",
  "user_id": "00000000-0000-0000-0000-000000000000",
  "is_system": false,
  "dataset_id": null,
  "base_model_id": null
}
```

### `GET /models/`

Возвращает список моделей.

Query parameters:

- `skip` - смещение, по умолчанию `0`;
- `limit` - лимит, по умолчанию `100`;
- `dataset_id` - необязательный фильтр по датасету;
- `include_system` - включать системные модели, по умолчанию `true`.

### `GET /models/{model_id}`

Возвращает одну модель. В текущей реализации сервис проверяет принадлежность модели пользователю; системные модели в этом методе могут быть недоступны как обычные пользовательские.

### `GET /models/download/{model_id}`

Возвращает presigned URL для скачивания пользовательской модели. Для системных моделей скачивание запрещено.

### `GET /models/metrics/{model_id}`

Возвращает presigned URL на JSON-метрики обучения. Для системных моделей метрики недоступны.

Ответ:

```json
{
  "metrics_url": "http://localhost:9000/metrics/..."
}
```

### `DELETE /models/{model_id}`

Удаляет пользовательскую модель из MinIO и PostgreSQL. Системные модели удалять запрещено.

## Tasks

### `POST /tasks/create/inference`

Создает задачу инференса. Требует `multipart/form-data`.

Поля формы:

- `model_id` - UUID модели;
- `file` - изображение или PDF.

Допустимые MIME-типы:

- `image/jpeg`
- `image/png`
- `application/pdf`

Пример:

```bash
curl -X POST http://localhost:8000/tasks/create/inference \
  -H "Authorization: Bearer $TOKEN" \
  -F "model_id=00000000-0000-0000-0000-000000000000" \
  -F "file=@image.png"
```

Ответ `201 Created`:

```json
{
  "task_type": "inference",
  "status": "queued",
  "model_id": "00000000-0000-0000-0000-000000000000",
  "dataset_id": null,
  "image_size": null,
  "epochs": null,
  "name": null,
  "input_path": "uuid_image.png",
  "output_path": null,
  "error_msg": null,
  "id": "00000000-0000-0000-0000-000000000000",
  "user_id": "00000000-0000-0000-0000-000000000000",
  "output_url": null
}
```

### `POST /tasks/create/training`

Создает задачу обучения. Требует `multipart/form-data`.

Поля формы:

- `model_id` - UUID базовой модели;
- `dataset_id` - UUID датасета;
- `image_size` - размер входного изображения для обучения;
- `num_epochs` - число эпох;
- `name` - имя training run.

В training-воркере fine-tuning разрешен только от системных моделей (`is_system=True`).

Пример:

```bash
curl -X POST http://localhost:8000/tasks/create/training \
  -H "Authorization: Bearer $TOKEN" \
  -F "model_id=00000000-0000-0000-0000-000000000000" \
  -F "dataset_id=11111111-1111-1111-1111-111111111111" \
  -F "image_size=640" \
  -F "num_epochs=20" \
  -F "name=experiment-001"
```

### `GET /tasks/`

Возвращает задачи текущего пользователя.

Query parameters:

- `skip` - смещение, по умолчанию `0`;
- `limit` - лимит, по умолчанию `100`.

### `GET /tasks/{task_id}`

Возвращает задачу по UUID. Если у задачи заполнен `output_path`, API добавляет `output_url` - presigned URL на результат.

### `GET /tasks/subscribe/{task_id}`

SSE-подписка на состояние задачи. Сервер отправляет событие:

```text
event: task_update
data: {...TaskRead...}
```

События отправляются один раз в секунду до достижения терминального статуса `succeeded` или `failed`.

### `DELETE /tasks/{task_id}`

Удаляет запись задачи из PostgreSQL. Ответ `204 No Content`.

## Admin

Административная панель доступна по `/admin`. Middleware скрывает панель за `404`, если пользователь не имеет роли `admin`. Токен может быть передан через session или заголовок `Authorization: Bearer <token>`.
