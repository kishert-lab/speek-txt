# Meeting Protocol API

Локальный сервис распознавания и протоколирования заседаний, совещаний и конференций. Сервис принимает длинные аудио- и видеофайлы, сохраняет исходник, ставит задачу в очередь Celery, формирует стенограмму с говорящими и экспортирует результат в `JSON`, `TXT` и `DOCX`.

## Требования

- Python `3.11`
- Docker и Docker Compose
- `ffmpeg` и `ffprobe`
- Redis
- Для GPU-режима: NVIDIA Container Toolkit и совместимая CUDA

Рекомендуемые ресурсы:

- CPU-only: от `4 vCPU`, `8 GB RAM`
- GPU: `8+ GB VRAM`, `16 GB RAM`

## Структура моделей и словарей

```text
models/
├── gigaam/
│   ├── v3_e2e_rnnt.ckpt
│   └── v3_e2e_rnnt_tokenizer.model
└── diarization/
    └── <локальная модель диаризации>

dictionaries/
├── phraseReplacement
├── phraseReplacementLocal
├── GoodWords.lst
└── bugWords.lst
```

`tokenizer.model` не используется вместо `v3_e2e_rnnt_tokenizer.model`, если библиотека этого явно не требует.

## Запуск

1. Создать `.env` на основе `.env.example`.
2. Положить локальные модели в `models/`.
3. При необходимости положить словари в `dictionaries/`.
4. Собрать и запустить сервис:

```bash
docker compose build
docker compose up -d
docker compose logs -f worker
```

CPU-вариант:

```bash
docker compose up -d api worker redis
```

GPU-вариант:

```bash
docker compose up -d
```

`worker` в `docker-compose.yml` настроен на `gpus: all`.

## API

Swagger: `http://localhost:8000/docs`

Загрузка файла:

```bash
curl -X POST http://localhost:8000/api/v1/protocols \
  -H "X-API-Key: change-me" \
  -F "file=@meeting.m4a" \
  -F "title=Рабочее совещание" \
  -F "meeting_type=conference" \
  -F "expected_speakers=4" \
  -F "apply_dictionary=true" \
  -F "generate_protocol=true"
```

Получение статуса:

```bash
curl -H "X-API-Key: change-me" http://localhost:8000/api/v1/protocols/<job_id>
```

Получение результата:

```bash
curl -H "X-API-Key: change-me" http://localhost:8000/api/v1/protocols/<job_id>/result
```

Назначение имён спикерам:

```bash
curl -X PATCH http://localhost:8000/api/v1/protocols/<job_id>/speakers \
  -H "X-API-Key: change-me" \
  -H "Content-Type: application/json" \
  -d '{"speakers":{"SPEAKER_00":"Иванов Иван Иванович"}}'
```

Скачивание файлов:

```bash
curl -H "X-API-Key: change-me" http://localhost:8000/api/v1/protocols/<job_id>/files/json
curl -H "X-API-Key: change-me" http://localhost:8000/api/v1/protocols/<job_id>/files/txt
curl -H "X-API-Key: change-me" http://localhost:8000/api/v1/protocols/<job_id>/files/docx
```

## Health checks

- `GET /health/live`
- `GET /health/ready`

`ready` проверяет Redis, `ffmpeg`, `ffprobe`, каталог `/data`, checkpoint и tokenizer GigaAM, а также наличие CUDA при `ASR_DEVICE=cuda`.

## Тесты

Настройка PyAnnote через ENV, диагностика и методика оценки качества описаны в [docs/diarization-configuration.md](docs/diarization-configuration.md).

```bash
python -m pip install -r requirements-dev.txt
python -m pytest
```

## Типичные ошибки

- `MODEL_NOT_FOUND`: отсутствует `models/gigaam/v3_e2e_rnnt.ckpt`
- `TOKENIZER_NOT_FOUND`: отсутствует `models/gigaam/v3_e2e_rnnt_tokenizer.model`
- `UNSUPPORTED_MEDIA`: неподдерживаемый формат или файл не проходит `ffprobe`
- `DIARIZATION_UNAVAILABLE`: локальная модель диаризации не найдена, включён single-speaker fallback

## Ограничения MVP

- Поле `protocol.decisions` пока не заполняется автоматически.
- Реальная диаризация зависит от локально установленной модели и `pyannote.audio`.
- Интеграция GigaAM реализована только через локальные файлы; сервис принципиально не скачивает модели автоматически.
- Периодическая очистка старых заданий ещё не вынесена в отдельный beat-процесс.

## Приватность

Сервис не отправляет аудио, стенограммы и протоколы во внешние облачные сервисы. Все данные обрабатываются локально, модели должны быть размещены заранее в локальных каталогах `models/` и `dictionaries/`.
