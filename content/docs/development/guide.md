---
title: Руководство разработчика
description: Структура репозитория, локальный запуск, тесты и расширение сервисов
weight: 10
---

# Руководство разработчика

## Структура репозитория

```text
.
├── docker-compose.yml
├── docker-compose.mac.yml
├── nginx/
├── schemion-api/
├── schemion-inference/
├── schemion-training/
└── system_model_importer/
```

Каждый из трех основных сервисов является отдельным Python-проектом со своими `requirements.txt`, `Dockerfile`, тестами и внутренней архитектурой.

## Локальный запуск

Для полного интеграционного запуска предпочтительно использовать Docker Compose:

```bash
docker compose up --build
```

На macOS:

```bash
docker compose -f docker-compose.mac.yml up --build
```

После запуска:

- API: `http://localhost:8000`
- API через nginx: `http://localhost/api`
- OpenAPI: `http://localhost:8000/docs`
- MinIO console: `http://localhost:9001`
- Bobber: `localhost:50051`
- PostgreSQL: `localhost:5432`

## Разработка API

Код API находится в `schemion-api`.

Ключевые директории:

- `app/presentation/routers` - FastAPI маршруты;
- `app/presentation/schemas` - Pydantic-схемы запросов и ответов;
- `app/core/services` - бизнес-логика;
- `app/core/validation` - валидация файлов;
- `app/infrastructure/persistence` - SQLAlchemy-модели и репозитории;
- `app/infrastructure/services` - MinIO, cache, Bobber publisher;
- `app/infrastructure/di` - DI-контейнер Dishka;
- `alembic` - миграции БД.

При добавлении нового endpoint желательно сохранять текущий паттерн:

1. Pydantic schema в `presentation/schemas`.
2. Метод сервиса в `core/services`.
3. Интерфейс репозитория в `core/interfaces`, если нужен новый persistence contract.
4. Реализация в `infrastructure/persistence/repositories`.
5. Роут в `presentation/routers`.
6. Инвалидация кэша, если endpoint меняет состояние.
7. Тесты в `schemion-api/tests`.

## Разработка воркеров

Training и inference не являются HTTP-сервисами. Это долгоживущие процессы, которые подписываются на Bobber.

При изменении формата сообщения нужно синхронно обновлять:

- publisher в `schemion-api`;
- parser/use case в соответствующем воркере;
- документацию [Архитектура системы](/docs/architecture/system/) и материалы по ML-пайплайнам;
- тесты use case.

## Добавление новой архитектуры модели

Для inference:

1. Реализовать детектор, совместимый с `IDetector`.
2. Добавить mapping или ветку в `DetectorFactory`.
3. Обновить `models_config.py`.
4. Убедиться, что формат весов и список классов корректно загружаются.

Для training:

1. Реализовать trainer, совместимый с `IDetectorTrainer`.
2. Добавить создание trainer в `DetectorTrainerFactory`.
3. Описать требования к датасету.
4. Реализовать экспорт весов в `.pt` или `.pth`.
5. Добавить сбор метрик.

Для API:

1. Обновить `ModelArchitectures`, если появляется новый верхнеуровневый тип.
2. Обновить валидацию и документацию.

## Тесты

В репозитории есть тесты для:

- API services;
- API schemas;
- API MinIO storage;
- API Bobber publisher;
- training use cases and factories;
- inference use cases, repositories, services and factories.

Запуск зависит от окружения конкретного submodule. Общий паттерн:

```bash
cd schemion-api
pytest
```

```bash
cd schemion-training
pytest
```

```bash
cd schemion-inference
pytest
```

Если тесты требуют внешних сервисов, поднимите Docker Compose или используйте test doubles, уже описанные в `tests/conftest.py`.

## Безопасность

Текущие значения секретов в compose и config являются development defaults:

```text
JWT_SECRET=supersecret
POSTGRES_PASSWORD=admin
MINIO_SECRET_KEY=minioadmin
```

Для production необходимо:

- заменить все секреты;
- ограничить CORS вместо `allow_origins=["*"]`;
- включить TLS на внешнем контуре;
- не публиковать PostgreSQL и MinIO API наружу без необходимости;
- пересмотреть срок жизни access и refresh token;
- добавить постоянное распределенное кэш-хранилище, если API масштабируется горизонтально.

## Эксплуатационные замечания

- Статус задачи является главным источником правды для клиента.
- Если задача зависла в `queued`, проверьте доступность Bobber и соответствующего воркера.
- Если задача зависла в `running`, смотрите логи воркера и наличие GPU/CUDA.
- Если `output_url` не открывается, проверьте доступность MinIO на `localhost:9000` и срок жизни presigned URL.
- Если обучение YOLO падает на macOS, причина может быть в жестком `device="cuda"`.
- Если inference на PDF падает, требуется отдельная конвертация PDF в изображение перед `PIL.Image.open`.

## Известные места для улучшения

- Добавить постоянное распределенное кэш-хранилище, если предполагается несколько API-инстансов.
- Добавить NMS при объединении тайлов inference.
- Унифицировать `MINIO_PUBLIC_ENDPOINT` и фактический endpoint, используемый для presigned URL.
- Явно валидировать структуру датасета на API-слое, а не только в training-воркере.
- Разделить admin endpoints и публичный API на уровне маршрутизации и политик доступа.
- Добавить поддержку CPU fallback для YOLO training.
- Уточнить и зафиксировать профили Faster R-CNN между training и inference.
