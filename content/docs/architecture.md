---
title: Архитектура
description: Потоки данных и компоненты платформы
---

# Архитектура

## Схема компонентов
```text
Клиент -> schemion-api -> PostgreSQL
Клиент -> schemion-api -> MinIO
schemion-api -> Redis
schemion-api -> Bobber -> schemion-inference -> PostgreSQL
schemion-api -> Bobber -> schemion-training -> PostgreSQL
schemion-inference -> MinIO
schemion-training -> MinIO
```

## Поток инференса

1. Клиент вызывает `POST /tasks/create/inference` и передает `model_id` и файл.
1. API загружает файл в MinIO и создает задачу в БД.
1. API публикует сообщение в очередь inference_queue.
1. `schemion-inference` получает задачу и загружает веса модели и изображение.
1. Изображение режется на тайлы `640×640` с перекрытием `100 px`.
1. Детектор предсказывает объекты по каждому тайлу.
1. Координаты предсказаний сдвигаются и объединяются.
1. Результат сохраняется в MinIO как `JSON`, задача помечается succeeded или failed.

## Поток обучения

1. Клиент вызывает `POST /tasks/create/training` и передает `model_id` и `dataset_id`.
1. API создает задачу и публикует сообщение в очередь `training_queue`.
1. `schemion-training` загружает базовые веса и архив датасета из MinIO.
1. Датасет распаковывается, YAML приводится к локальному пути.
1. Запускается обучение через `Ultralytics YOLO`.
1. Метрики сохраняются в MinIO как `metrics_{task_id}.json`.
1. Экспортированный `.pt` файл загружается в MinIO.
1. Создается новая модель `*_fine_tuned`, задача обновляется.

## Поддерживаемые архитектуры

- `yolo` для обучения и инференса.
- `faster_rcnn` для инференса. (дообучение в процессе)
- Профили Faster R-CNN: `resnet50_fpn`, `resnet50_fpn_v2`, `mobilenet_v3_large_fpn`, `mobilenet_v3_large_320_fpn`.
- Алиасы профилей: `resnet50`, `resnet50v2`, `mobilenet`, `mobilenet_320`, `fpn`, `fpnv2`.

## Бакеты MinIO
- `schemas-images` для входных изображений.
- `datasets` для архивов датасетов.
- `models` для весов моделей.
- `metrics` для метрик обучения.
- `inference-results` для JSON результатов инференса.

## Кэш и лимиты

- Redis используется для кэша сущностей и списков.
- TTL по умолчанию: датасеты 1 час, модели 2 часа, задачи 30 минут, пользователь 30 минут.
- Глобальный лимит запросов: 100 в минуту.

## Безопасность

- JWT авторизация с ролями и правами.
- Админ‑панель скрыта middleware и возвращает `404` без валидного токена.