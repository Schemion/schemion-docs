---
title: ML-пайплайны
description: Обучение, инференс, тайлинг и форматы результатов
weight: 30
---

# ML-пайплайны

## Модели и архитектуры

Система работает с моделями детекции объектов. В таблице `models` хранятся:

- `architecture` - логический тип модели, например `yolo` или `faster_rcnn`;
- `architecture_profile` - профиль или конкретная разновидность архитектуры;
- `classes` - массив имен классов, если он известен;
- `minio_model_path` - объектный путь весов в MinIO;
- `metrics_path` - объектный путь JSON-метрик;
- `is_system` - признак системной базовой модели;
- `base_model_id` - ссылка на базовую модель для fine-tuned моделей;
- `dataset_id` - ссылка на датасет, на котором обучалась модель.

## Training pipeline

Training pipeline запускается сообщением в `training_queue`.

Сообщение имеет вид:

```json
{
  "task_id": "00000000-0000-0000-0000-000000000000",
  "task_type": "training",
  "model_id": "00000000-0000-0000-0000-000000000000",
  "dataset_id": "11111111-1111-1111-1111-111111111111",
  "image_size": 640,
  "epochs": 20,
  "name": "experiment-001",
  "timestamp": "2026-05-13T00:00:00+00:00"
}
```

Последовательность обработки:

1. Training-воркер получает сообщение из Bobber.
2. Загружает из PostgreSQL задачу, базовую модель и датасет.
3. Переводит задачу в статус `running`.
4. Проверяет, что модель является системной. Fine-tuning пользовательской модели в текущей реализации запрещен.
5. Скачивает веса базовой модели из бакета `models`.
6. Скачивает ZIP-датасет из бакета `datasets`, распаковывает его и находит YAML.
7. Создает тренер по `architecture` и `architecture_profile`.
8. Загружает веса в тренер.
9. Выполняет обучение.
10. Формирует JSON-метрики и сохраняет их в бакет `metrics`.
11. Экспортирует новые веса.
12. Загружает веса в бакет `models`.
13. Обновляет задачу: `output_path`, `updated_at`, `status=succeeded`.
14. Создает новую модель с `is_system=False`.
15. Удаляет локальные временные файлы.

При исключении воркер:

- пишет текст ошибки в `task.error_msg`;
- обновляет `updated_at`;
- ставит `status=failed`;
- удаляет временные файлы, если они были созданы.

## YOLO trainer

YOLO trainer находится в `schemion-training/app/infrastructure/trainers/yolo_trainer.py`.

Ключевые свойства:

- использует `ultralytics.YOLO`;
- загружает веса через `YOLO(weights_path)`;
- обучает через `self.model.train`;
- сохраняет результат через `self.model.save`;
- экспортирует файл `.pt`;
- метрики читает из `results.csv`, который генерирует Ultralytics.

Параметры обучения:

```text
epochs: из задачи или 10
imgsz: из задачи или 640
batch: 4
workers: 4
device: cuda
deterministic: true
exist_ok: true
```

Формат метрик:

```json
{
  "run_id": "task_uuid",
  "model_id": "base_model_uuid",
  "started_at": "2026-05-13T00:00:00Z",
  "epochs": [
    {
      "epoch": 0,
      "train/box_loss": 1.23,
      "metrics/mAP50(B)": 0.42
    }
  ]
}
```

Точный набор ключей зависит от версии Ultralytics и содержимого `results.csv`.

## Faster R-CNN trainer

Faster R-CNN trainer находится в `schemion-training/app/infrastructure/trainers/fasterrcnn_trainer.py`.

Поддерживаемые профили:

- `resnet50_fpn`
- `resnet50_fpn_v2`

В конфигурационных файлах также описаны профили MobileNet, но training factory в текущей реализации создает `FasterRCNNTrainer`, который выбирает между `fasterrcnn_resnet50_fpn` и `fasterrcnn_resnet50_fpn_v2` по наличию `v2` в `architecture_profile`.

Особенности:

- использует `torchvision.models.detection`;
- перестраивает модель под количество классов `len(names) + 1`, где дополнительный класс - background;
- загружает веса с `strict=False`, исключая параметры `roi_heads.box_predictor.*`;
- читает YOLO labels и преобразует их в формат TorchVision detection target;
- использует `SGD(lr=0.005, momentum=0.9, weight_decay=0.0005)`;
- использует `StepLR(step_size=3, gamma=0.1)`;
- batch size равен `2`;
- экспортирует файл `.pth`.

Метрики Faster R-CNN в текущей реализации являются training-loss метриками по эпохам:

```json
{
  "epoch": 0,
  "loss_classifier": 0.42,
  "loss_box_reg": 0.18,
  "loss_objectness": 0.09,
  "loss_rpn_box_reg": 0.04,
  "loss_total": 0.73
}
```

## Inference pipeline

Inference pipeline запускается сообщением в `inference_queue`.

Сообщение имеет вид:

```json
{
  "task_id": "00000000-0000-0000-0000-000000000000",
  "task_type": "inference",
  "model_id": "00000000-0000-0000-0000-000000000000",
  "model_arch": "yolo",
  "input_path": "uuid_image.png",
  "timestamp": "2026-05-13T00:00:00+00:00"
}
```

Последовательность обработки:

1. Inference-воркер получает сообщение из Bobber.
2. Загружает задачу и модель из PostgreSQL.
3. Переводит задачу в статус `running`.
4. Загружает входное изображение из бакета `schemas-images`.
5. Загружает веса модели из бакета `models`.
6. Создает детектор через `DetectorFactory`.
7. Разбивает изображение на тайлы.
8. Выполняет предсказание на каждом тайле.
9. Сдвигает координаты предсказаний из локальной системы координат тайла в координаты исходного изображения.
10. Объединяет все предсказания в единый список.
11. Сохраняет JSON-результат в бакет `inference-results`.
12. Обновляет задачу: `output_path`, `updated_at`, `status=succeeded`.

При ошибке воркер записывает `error_msg` и ставит `status=failed`.

## Тайлинг изображений

`ImageTilerService` используется для обработки крупных изображений.

Параметры по умолчанию:

- `tile_size=640`;
- `overlap=100`;
- `step=tile_size-overlap=540`.

Если изображение не превышает `640x640`, оно обрабатывается одним тайлом. Иначе создается сетка тайлов с перекрытием. Тайлы на правой и нижней границе сдвигаются так, чтобы покрыть край изображения.

В текущей реализации merge является простым объединением списков предсказаний. Non-maximum suppression между тайлами не выполняется, поэтому в зонах перекрытия возможны дублирующиеся детекции.

## Формат результата инференса

Результат сохраняется как JSON:

```json
{
  "task_id": "00000000-0000-0000-0000-000000000000",
  "model_id": "00000000-0000-0000-0000-000000000000",
  "model_arch": "yolo",
  "predictions": [
    {
      "class": "defect",
      "confidence": 0.93,
      "bbox": [10.0, 20.0, 110.0, 160.0]
    }
  ],
  "image_width": 1920,
  "image_height": 1080,
  "processed_at": "2026-05-13T00:00:00+00:00"
}
```

`bbox` представлен в формате absolute `xyxy`: `[x1, y1, x2, y2]` в пикселях исходного изображения.

## YOLO detector

YOLO detector:

- загружает веса через `YOLO(model_weights_path)`;
- вызывает модель на `PIL.Image`;
- возвращает class name из `results.names`;
- возвращает confidence и bbox в формате `xyxy`.

## Faster R-CNN detector

Faster R-CNN detector:

- создает архитектуру TorchVision по `architecture_profile`;
- определяет число классов из формы весов classifier head;
- заменяет `roi_heads.box_predictor`;
- загружает state dict;
- переводит модель в `eval`;
- использует confidence threshold `0.5`;
- добавляет искусственный background class при наличии списка классов.

## Известные технические ограничения

- Inference API допускает `application/pdf`, но `ImageLoader` открывает файл через PIL как изображение. PDF может потребовать дополнительной конвертации, если PIL в окружении не поддерживает данный входной формат.
- Merge предсказаний после тайлинга не удаляет дубли.
- Training YOLO жестко использует `device="cuda"`, поэтому контейнеру нужен доступ к GPU.
- В `docker-compose.mac.yml` GPU reservation для trainer отсутствует, что удобно для macOS, но YOLO training с `device="cuda"` без CUDA завершится ошибкой.
- Fine-tuning разрешен только от системных моделей.
