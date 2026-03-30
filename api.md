---
title: API
description: Эндпоинты и форматы запросов
---

# API

## Аутентификация
- `POST /auth/register` — регистрация.
- `POST /auth/login` — логин и выдача JWT.
- Заголовок для защищенных эндпоинтов: `Authorization: Bearer <token>`.

## Формат запросов
- `/datasets/create` принимает `multipart/form-data` с `name`, `description` и `file`.
- `/models/create` принимает `multipart/form-data` с `name`, `architecture`, `architecture_profile`, опциональным `dataset_id` и `file`.
- `/tasks/create/inference` принимает `multipart/form-data` с `model_id` и `file`.
- `/tasks/create/training` принимает form‑параметры `model_id` и `dataset_id`.

## Датасеты
| Method | Path | Auth | Описание |
| --- | --- | --- | --- |
| POST | /datasets/create | yes | Загрузка ZIP датасета |
| GET | /datasets | yes | Список датасетов |
| GET | /datasets/{dataset_id} | yes | Получить датасет |
| GET | /datasets/download/{dataset_id} | yes | Ссылка на скачивание |
| DELETE | /datasets/{dataset_id} | yes | Удаление |

## Модели
| Method | Path | Auth | Описание |
| --- | --- | --- | --- |
| POST | /models/create | yes | Загрузка `.pt` или `.pth` |
| GET | /models | yes | Список моделей |
| GET | /models/{model_id} | yes | Получить модель |
| GET | /models/download/{model_id} | yes | Ссылка на скачивание |
| DELETE | /models/{model_id} | yes | Удаление |

## Задачи
| Method | Path | Auth | Описание |
| --- | --- | --- | --- |
| POST | /tasks/create/inference | yes | Инференс по файлу |
| POST | /tasks/create/training | yes | Запуск обучения |
| GET | /tasks | yes | Список задач |
| GET | /tasks/{task_id} | yes | Получить задачу |
| GET | /tasks/subscribe/{task_id} | yes | SSE обновления |
| DELETE | /tasks/{task_id} | yes | Удалить задачу |

## Админка
- `GET /admin` — SQLAdmin UI. Доступ только для роли `admin`.

## Параметры списков
- `/datasets`: `skip`, `limit`, `name_contains`.
- `/models`: `skip`, `limit`, `dataset_id`, `include_system`.
- `/tasks`: `skip`, `limit`.

## Ограничения по запросам
- `POST /tasks/create/inference` — 10 запросов в минуту.
- `POST /tasks/create/training` — 2 запроса в час.
- `POST /models/create` — 5 запросов в минуту.
- Глобальный лимит — 100 запросов в минуту.

## Валидация файлов
- Модель: `.pt` или `.pth`, размер до 1 GB, MIME `application/octet-stream` или `application/x-pickle`.
- Датасет: `.zip`, размер до 5 GB, внутри должны быть изображения и файлы аннотаций.
- Инференс: `image/jpeg`, `image/png`, `application/pdf`.

## Формат SSE события
```json
{
  "event": "task_update",
  "data": {
    "id": "UUID",
    "task_type": "inference",
    "status": "running",
    "model_id": "UUID",
    "dataset_id": null,
    "input_path": "minio/object",
    "output_path": "minio/object",
    "output_url": "https://...",
    "error_msg": null
  }
}