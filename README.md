# Meeting Protocol API

Полная русскоязычная документация: [docs/README.md](docs/README.md).

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
2. Однократно загрузить модели в каталог хоста `./models`. Для diarization временно укажите `HF_TOKEN` в окружении команды:

```bash
export HF_TOKEN='<token>'
make models-download
unset HF_TOKEN
```

Команда проверяет существующие файлы и не скачивает их повторно. Модели не включаются в Docker-образы и не отправляются в registry.

3. При необходимости положить словари в `dictionaries/`.
4. Собрать и запустить сервис:

```bash
make docker-build
make docker-up
```

`make docker-up` подключает GPU к worker только при наличии NVIDIA GPU. На CPU-хосте worker стартует без CUDA.

Внешний доступ идёт только через Nginx:

```text
http://<host>/              frontend
http://<host>/api/...       API
http://<host>/health/live   health-check
```

Порты API и frontend не публикуются отдельно. Порт Nginx настраивается переменной `HTTP_PORT` (по умолчанию `80`).

## Docker-образы

- `api`: HTTP API без ML-зависимостей и моделей.
- `worker`: PyTorch/CUDA, GigaAM и pyannote, без файлов моделей.
- `frontend`: статическая Vite-сборка.
- `nginx`: единая внешняя точка входа.

`docker compose push` публикует только эти образы. Модели хранятся в `./models` на сервере и повторно не скачиваются при сборке или push.

## API

Swagger: `http://localhost/docs`

Загрузка файла:

```bash
curl -X POST http://localhost/api/v1/protocols \
  -F "file=@meeting.m4a" \
  -F "title=Рабочее совещание" \
  -F "meeting_type=conference" \
  -F "expected_speakers=4" \
  -F "apply_dictionary=true" \
  -F "generate_protocol=true"
```

Получение статуса:

```bash
curl http://localhost/api/v1/protocols/<job_id>
```

Получение результата:

```bash
curl http://localhost/api/v1/protocols/<job_id>/result
```

Назначение имён спикерам:

```bash
curl -X PATCH http://localhost/api/v1/protocols/<job_id>/speakers \
  -H "Content-Type: application/json" \
  -d '{"speakers":{"SPEAKER_00":"Иванов Иван Иванович"}}'
```

Скачивание файлов:

```bash
curl http://localhost/api/v1/protocols/<job_id>/files/json
curl http://localhost/api/v1/protocols/<job_id>/files/txt
curl http://localhost/api/v1/protocols/<job_id>/files/docx
```

## Health checks

- `GET /health/live`
- `GET /health/ready`

`ready` проверяет Redis, `ffmpeg`, `ffprobe`, каталог `/data`, checkpoint и tokenizer GigaAM при выключенной автозагрузке, а также наличие CUDA при `ASR_DEVICE=cuda`.

## Тесты

```bash
python -m pip install -r requirements-dev.txt
python -m pytest
```

## Типичные ошибки

- `MODEL_NOT_FOUND`: отсутствует `models/gigaam/v3_e2e_rnnt.ckpt` при `GIGAAM_DOWNLOAD_ENABLED=false`
- `TOKENIZER_NOT_FOUND`: отсутствует `models/gigaam/v3_e2e_rnnt_tokenizer.model` при `GIGAAM_DOWNLOAD_ENABLED=false`
- `MODEL_DOWNLOAD_FAILED`: не удалось загрузить или проверить опубликованную GigaAM-модель

После загрузки оставьте `GIGAAM_DOWNLOAD_ENABLED=false`, чтобы worker работал только с локальными файлами.
- `UNSUPPORTED_MEDIA`: неподдерживаемый формат или файл не проходит `ffprobe`
- `DIARIZATION_UNAVAILABLE`: локальная модель диаризации не найдена, включён single-speaker fallback

## Ограничения MVP

- Поле `protocol.decisions` пока не заполняется автоматически.
- Реальная диаризация зависит от локально установленной модели и `pyannote.audio`.
- Модели загружаются отдельной командой `make models-download` и хранятся в `models/` рядом с Compose-файлом.
- Периодическая очистка старых заданий ещё не вынесена в отдельный beat-процесс.

## Приватность

Сервис не отправляет аудио, стенограммы и протоколы во внешние сервисы. При включённой автозагрузке наружу обращается только за публичными файлами GigaAM; после скачивания обработка данных выполняется локально.
