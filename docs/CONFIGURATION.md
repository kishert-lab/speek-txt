# Установка и конфигурация

## 1. Требования

Минимально необходимы Linux, Docker Engine, Docker Compose v2, 8 GiB RAM и свободное место под модели и записи. Для GPU-режима требуются NVIDIA GPU, драйвер, NVIDIA Container Toolkit и совместимая CUDA-сборка PyTorch. Текущий worker использует PyTorch `2.4.1+cu124`.

Рекомендуемая конфигурация для длинных совещаний:

- 8 и более CPU-ядер;
- 16 GiB RAM;
- NVIDIA GPU с 6–8 GiB VRAM или больше;
- SSD для Docker volumes;
- синхронизация времени и корректный часовой пояс.

GTX 1630 с 4 GiB работает, но почти полностью занимает VRAM и не подходит для нескольких одновременных ML-worker процессов.

## 2. Подготовка хоста с NVIDIA

Проверить драйвер на хосте:

```bash
nvidia-smi
```

Проверить доступ Docker к GPU:

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

Если в контейнере GPU не виден, проверить NVIDIA Container Toolkit и runtime:

```bash
docker info | grep -i runtime
docker inspect speek-text-worker-1 --format '{{json .HostConfig.DeviceRequests}}'
```

## 3. Модели

Модели находятся на хосте и монтируются в worker, а не копируются в Docker image:

```text
models/
├── gigaam/
│   ├── v3_e2e_rnnt.ckpt
│   └── v3_e2e_rnnt_tokenizer.model
└── diarization/
    ├── config.yaml
    ├── segmentation/
    └── embedding/
```

Первичная загрузка:

```bash
cp .env.example .env
# Временно укажите HF_TOKEN, если он нужен для PyAnnote.
make models-download
```

После скачивания установите `GIGAAM_DOWNLOAD_ENABLED=false` и удалите `HF_TOKEN` из production `.env`. Каталоги моделей уже исключены из `.dockerignore`, поэтому push worker не должен передавать многогигабайтные веса.

Проверка checkpoint PyAnnote без сборки worker:

```bash
docker run --rm --user 0:0 \
  -v "$PWD/models/diarization:/models/diarization:ro" \
  -v "$PWD/scripts/check-diarization-checkpoints.py:/scripts/check.py:ro" \
  registry.dev.it59com.ru/meeting-protocol/worker:latest \
  python /scripts/check.py --model-dir /models/diarization
```

## 4. Запуск

CPU:

```bash
docker compose up -d --build
```

GPU:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d --build
```

Пересоздание после изменения `.env` или compose-файла:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d --force-recreate
```

Обычный `restart` не применяет изменённую конфигурацию контейнера.

Проверка:

```bash
docker compose ps
curl http://localhost/health/live
curl http://localhost/health/ready
docker compose -f docker-compose.yml -f docker-compose.gpu.yml exec worker \
  python -c 'import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))'
```

## 5. Переменные окружения

### Общие

| Переменная | По умолчанию | Назначение |
|---|---:|---|
| `APP_NAME` | `Meeting Protocol API` | Имя FastAPI-приложения |
| `APP_ENV` | `production` | Имя окружения |
| `LOG_LEVEL` | `INFO` | Уровень логирования |
| `HTTP_PORT` | `80` | Порт внешнего Nginx |
| `REDIS_URL` | `redis://redis:6379/0` | Redis broker/backend |
| `STORAGE_ROOT` | `/data` | Корень файлового хранилища |
| `MAX_UPLOAD_SIZE_MB` | `4096` | Суммарный размер файлов одной задачи |
| `RESULT_RETENTION_DAYS` | `7` | Срок хранения неизменённых завершённых задач |
| `KEEP_AUDIO_SEGMENTS` | `false` | Сохранять временные ASR-сегменты |
| `DIPLOMATIC_DICTIONARY_PATH` | `/dictionaries/diplomaticReplacement.txt` | Пополняемый словарь литературных замен для дипломатического режима |

Словарь дипломатического режима находится на хосте в `dictionaries/diplomaticReplacement.txt` и монтируется в контейнер read-only. Формат строки:

```text
исходное выражение>литературная замена;
```

Пустые строки и строки, начинающиеся с `#`, игнорируются. Регистр исходного выражения не учитывается, более длинные правила применяются первыми. Файл перечитывается при формировании каждого нового протокола, поэтому после пополнения словаря пересборка образа и перезапуск worker не требуются. Исходная стенограмма не изменяется; применённые пары «оригинал → замена» сохраняются в поле `protocol.replacements`.

### GigaAM

| Переменная | По умолчанию | Назначение |
|---|---:|---|
| `GIGAAM_MODEL_DIR` | `/models/gigaam` | Каталог модели |
| `GIGAAM_CHECKPOINT` | `v3_e2e_rnnt.ckpt` | Файл checkpoint |
| `GIGAAM_TOKENIZER` | `v3_e2e_rnnt_tokenizer.model` | Файл tokenizer |
| `GIGAAM_MODEL_NAME` | `v3_e2e_rnnt` | Имя архитектуры |
| `GIGAAM_DOWNLOAD_ENABLED` | `false` в example | Разрешить автоматическую загрузку |
| `ASR_DEVICE` | `auto` | `auto`, `cuda` или `cpu` |
| `ASR_USE_FP16` | `true` | FP16 на CUDA |
| `ASR_MAX_SEGMENT_SECONDS` | `22` | Максимальная длина ASR-фрагмента |
| `ASR_SEGMENT_PADDING_SECONDS` | `0.25` | Запас по границам сегмента |

### Нормализация

| Переменная | По умолчанию | Назначение |
|---|---:|---|
| `AUDIO_NORMALIZATION_ENABLED` | `true` | Включить профиль нормализации |
| `AUDIO_NORMALIZE_FILTER` | пусто | Полная пользовательская FFmpeg-цепочка |
| `AUDIO_HIGHPASS_HZ` | `70` | Частота high-pass |
| `AUDIO_LOWPASS_HZ` | `7600` | Частота low-pass для удаления ВЧ-помех выше речевого диапазона |
| `AUDIO_DYNAUDNORM_FRAME` | `500` | Окно dynaudnorm, мс |
| `AUDIO_DYNAUDNORM_GAUSS` | `31` | Сглаживание усиления |
| `AUDIO_DYNAUDNORM_PEAK` | `0.90` | Целевой пик |
| `AUDIO_DYNAUDNORM_MAX_GAIN` | `8` | Максимальное усиление; большие значения поднимают шум и реверберацию |
| `AUDIO_LIMITER` | `0.95` | Ограничитель пика |

Если `AUDIO_NORMALIZE_FILTER` заполнен, он заменяет составной профиль из отдельных параметров.

### PyAnnote

| Переменная | По умолчанию | Назначение |
|---|---:|---|
| `DIARIZATION_ENABLED` | `true` | Включить диаризацию |
| `DIARIZATION_DEVICE` | `auto` | `auto`, `cuda` или `cpu` |
| `DIARIZATION_MODEL_DIR` | `/models/diarization` | Каталог локальной pipeline |
| `DIARIZATION_MODEL_ID` | community-1 | Идентификатор модели |
| `DIARIZATION_AUDIO_FILTER` | см. `.env.example` | Щадящий FFmpeg-профиль speaker embeddings |
| `DIARIZATION_CLUSTERING_THRESHOLD` | пусто | Порог кластеризации или параметры модели |
| `DIARIZATION_SEGMENTATION_MIN_DURATION_OFF` | пусто | Склейка коротких пауз |
| `DIARIZATION_SEGMENTATION_BATCH_SIZE` | `32` | Batch сегментации |
| `DIARIZATION_EMBEDDING_BATCH_SIZE` | `32` | Batch embeddings |
| `DIARIZATION_ALLOW_TF32` | `false` | TF32: быстрее, но возможна потеря воспроизводимости |
| `ALLOW_SINGLE_SPEAKER_FALLBACK` | `true` | Продолжить одним speaker при отказе PyAnnote |
| `HF_TOKEN` | пусто | Только для скачивания закрытых моделей |

Batch size увеличивайте только после контроля VRAM. При OOM сначала уменьшайте `DIARIZATION_EMBEDDING_BATCH_SIZE`, затем `DIARIZATION_SEGMENTATION_BATCH_SIZE`.

### Словари и потоки

| Переменная | По умолчанию | Назначение |
|---|---:|---|
| `DICTIONARIES_DIR` | `/dictionaries` | Каталог текстовых словарей |
| `DICTIONARY_ENABLED` | `true` | Глобальное включение словарей |
| `VITE_ENABLE_STREAMING_RECORDING` | `false` | Показывать потоковую запись во frontend |
| `STREAM_QUEUE_MAX_CHUNKS` | `32` | Bounded queue одной stream-сессии |
| `STREAM_CHUNK_MAX_BYTES` | `32000` | Максимальный PCM chunk |
| `STREAM_EVENT_MAX_LENGTH` | `1000` | Длина Redis Stream событий |
| `STREAM_IDLE_TTL_SECONDS` | `3600` | Плановый TTL неактивной сессии |

Переменные `VITE_*` встраиваются во frontend на этапе сборки. Изменения требуют пересборки образа frontend, а не только перезапуска.

## 6. Nginx и HTTPS

Внутренний `nginx/default.conf` уже содержит отдельные location для WebSocket, больших upload и Range-запросов аудиоплеера. Для внешнего reverse proxy используйте:

```nginx
location /api/v1/streams/ {
    proxy_pass http://192.168.0.50;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
    proxy_buffering off;
}

location / {
    proxy_pass http://192.168.0.50;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    client_max_body_size 4096m;
    client_body_timeout 3600s;
    proxy_request_buffering off;
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

Браузер разрешает `getUserMedia()` только в secure context: HTTPS либо localhost. Отключить это требование на стороне WebSocket/API нельзя. На HTTPS-странице используйте `wss://`.
