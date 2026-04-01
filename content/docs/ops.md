---
title: Эксплуатация
description: Переменные окружения, импорт системных моделей и ограничения
---

# Эксплуатация

## Переменные окружения
| Переменная | Значение по умолчанию | Где используется |
| --- | --- | --- |
| DATABASE_URL | `postgresql+asyncpg://admin:admin@database:5432/schemion` | `schemion-api` |
| DATABASE_URL | `postgresql://admin:admin@database:5432/schemion` | `schemion-training`, `schemion-inference` |
| REDIS_URL | `redis://:adminpass@redis:6379/0` | `schemion-api` |
| MINIO_ENDPOINT | `minio:9000` | все сервисы |
| MINIO_ACCESS_KEY | `minioadmin` | все сервисы |
| MINIO_SECRET_KEY | `minioadmin` | все сервисы |
| BOBBER_HOST | `bob-the-broker` | API и воркеры |
| BOBBER_PORT | `50051` | API и воркеры |
| JWT_SECRET | `supersecret` | `schemion-api` |

## Системные модели
1. Поместите файлы `.pt` или `.pth` в `system_model_importer/models`.
2. Убедитесь, что Postgres и MinIO доступны на `localhost:5432` и `localhost:9000`.
3. Запустите `python main.py` из `system_model_importer`.
4. Скрипт загрузит файлы в бакет `models` и создаст записи `is_system=true`.

## Форматы данных
- Датасет хранится как ZIP и должен содержать YAML и файлы аннотаций.
- Веса модели хранятся в MinIO как `.pt` или `.pth`.
- Результат инференса сохраняется как JSON с полями `task_id`, `model_id`, `predictions`, `image_width`, `image_height`.

## GPU
- В `docker-compose.yml` для `schemion-training` указан доступ к GPU через NVIDIA runtime.

## Известные ограничения
- В `schemion-training` реализован только `yolo` trainer, `faster_rcnn` пока пустой.
- Объединение предсказаний в инференсе выполняется без NMS. (в процессе для faster rcnn)