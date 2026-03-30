---
title: Schemion
description: Платформа для обучения и инференса детекторов объектов
---

# Schemion

Schemion — платформа для управления датасетами и моделями, обучения и инференса детекторов объектов. Проект разделен на API и два воркера, которые общаются через брокер задач.

## Ключевые возможности
- REST API на FastAPI для пользователей, датасетов, моделей и задач.
- Асинхронные воркеры обучения и инференса через брокер bob-the-broker (библиотека bobber).
- Хранение артефактов в MinIO и метаданных в PostgreSQL.
- Кэширование и лимитирование через Redis и slowapi.
- Админ‑панель SQLAdmin под JWT.

## Состав проекта
- `schemion-api` — API, авторизация, CRUD, постановка задач.
- `schemion-training` — обучение и fine‑tune моделей.
- `schemion-inference` — инференс и сохранение результатов.
- `bob-the-broker` — легковесный in-memory gRPC брокер сообщений на Golang.
- `bobber-lib` — библиотека для взаимодействия с bob-the-broker.

## Типовой сценарий
1. Зарегистрировать пользователя и получить JWT.
2. Загрузить датасет и базовую модель.
3. Создать задачу обучения или инференса.
4. Отслеживать статус через SSE или запросы к задачам.

## Быстрый старт Docker
1. Склонировать репозиторий `schemion-dev`
2. Инициализировать сабмодули.
3. Поднять сервисы.
4. Проверить доступность API.

```bash
git submodule update --init --recursive
docker compose up --build
```

## Порты по умолчанию
- API http://localhost:8000
- MinIO http://localhost:9000
- MinIO Console http://localhost:9001
- Postgres localhost:5432
- Redis localhost:6379
- Bobber localhost:50051

## Примечания
В docker-compose.yml для schemion-training включен доступ к GPU через NVIDIA runtime.  
В README есть пример установки зависимостей через зеркало PyPI на случай проблем с основным репозиторием.