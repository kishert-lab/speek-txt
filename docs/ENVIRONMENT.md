# Параметры окружения Meeting Protocol API

Версия документа: 8 ноября 2026 г.

Документ описывает переменные `.env`, используемые API, Celery worker, потоковым worker, Docker Compose и frontend. Рекомендуемый шаблон находится в `.env.example`.

## 1. Правила заполнения

- Формат строки: `ИМЯ=значение`, без пробелов вокруг `=`.
- Логические значения: `true` или `false`.
- Дробные числа записываются через точку: `0.25`.
- Пустое значение у необязательного параметра означает «использовать автоматический режим» или `None`.
- Секреты нельзя помещать в Git. В production используйте отдельный `.env`, Docker secrets или менеджер секретов.
- Пути внутри `.env` относятся к файловой системе контейнера. Модели монтируются в `/models`, словари — в `/dictionaries`, рабочие данные — в `/data`.
- Изменение backend-переменных требует перезапуска соответствующих контейнеров. Переменные `VITE_*` встраиваются в JavaScript при сборке frontend и требуют его пересборки.

## 2. Docker Compose и образы

| Переменная | Пример / значение по умолчанию | Назначение | Применение |
|---|---|---|---|
| `DOCKER_REGISTRY` | `registry.dev.it59com.ru` | Адрес Docker Registry. | Следующая сборка, pull или push. |
| `DOCKER_IMAGE_NAMESPACE` | `meeting-protocol` | Пространство имён образов. | Следующая сборка, pull или push. |
| `IMAGE_TAG` | `latest` | Тег образов API, frontend, worker и worker-base. Для production лучше уникальный тег версии. | Пересоздание контейнеров. |
| `HTTP_PORT` | `80` | Порт хоста, публикуемый контейнером nginx. | Пересоздание nginx. |

Эти переменные подставляются самим Docker Compose и не передаются в Python-приложение.

## 3. Общие параметры приложения

| Переменная | Значение по умолчанию | Назначение | Рекомендации |
|---|---|---|---|
| `APP_NAME` | `Meeting Protocol API` | Отображаемое имя API. | Можно менять без влияния на обработку. |
| `APP_ENV` | `production` | Название окружения. | Используйте `production`, `staging` или `development`. |
| `LOG_LEVEL` | `INFO` | Уровень журналирования. | `DEBUG`, `INFO`, `WARNING`, `ERROR`. На production — `INFO`. |
| `SERVICE_NAME` | `meeting-protocol-api` | Служебное имя компонента для мониторинга и логов. | В текущем коде зарезервировано для интеграций. |

После изменения перезапустите `api`, `worker` и `stream-worker`.

## 4. Redis и хранение данных

| Переменная | Значение по умолчанию | Назначение | Рекомендации |
|---|---|---|---|
| `REDIS_URL` | `redis://redis:6379/0` | Redis для Celery, заданий, очереди и потоковых событий. | Для внешнего Redis укажите пароль и TLS при необходимости. |
| `STORAGE_ROOT` | `/data` | Корень заданий, исходных аудио, результатов и экспортов. | Должен находиться на постоянном volume. |
| `TRAINING_DATA_DIR` | `/data/training` | Каталог исправленных стенограмм, отложенных для обучения. | Должен быть внутри резервируемого хранилища. |
| `MAX_UPLOAD_SIZE_MB` | `4096` | Максимальный суммарный размер multipart-запроса в МБ. | Согласуйте с `client_max_body_size` и timeout nginx. |
| `RESULT_RETENTION_DAYS` | `7` | Срок хранения обычных протоколов и число дней в статистике страницы состояния. | Исправленные обучающие данные хранятся отдельно. |
| `KEEP_AUDIO_SEGMENTS` | `false` | Сохранять временные WAV-фрагменты после ASR. | Включайте только для диагностики: быстро расходует диск. |

Изменение требует перезапуска API и worker. Уменьшение `RESULT_RETENTION_DAYS` может привести к удалению старых результатов при следующей очистке.

## 5. Модель GigaAM

| Переменная | Рекомендуемое значение | Назначение |
|---|---|---|
| `GIGAAM_MODEL_DIR` | `/models/gigaam` | Каталог локальной модели внутри worker. |
| `GIGAAM_CHECKPOINT` | `v3_e2e_rnnt.ckpt` | Имя checkpoint относительно `GIGAAM_MODEL_DIR`. |
| `GIGAAM_TOKENIZER` | `v3_e2e_rnnt_tokenizer.model` | Имя tokenizer относительно каталога модели. |
| `GIGAAM_MODEL_NAME` | `v3_e2e_rnnt` | Имя архитектуры GigaAM. |
| `GIGAAM_DOWNLOAD_ENABLED` | `false` | Разрешить загрузку модели во время работы. | 

На production оставляйте `GIGAAM_DOWNLOAD_ENABLED=false`: модель монтируется с хоста и не должна скачиваться или попадать в каждый образ worker.

## 6. Распознавание речи ASR

| Переменная | Значение | Влияние и настройка |
|---|---|---|
| `ASR_DEVICE` | `auto` | `auto`, `cuda` или `cpu`. Для production с GPU можно указать `cuda`, чтобы ошибка CUDA не маскировалась переходом на CPU. |
| `ASR_USE_FP16` | `true` | FP16 уменьшает VRAM и ускоряет GigaAM на NVIDIA GPU. Для CPU установите `false`. |
| `ASR_MAX_SEGMENT_SECONDS` | `22` | Жёсткий верхний предел обычного ASR-фрагмента. Большие значения требуют больше VRAM; слишком малые чаще рвут фразы. |
| `REPROCESS_MAX_DURATION_SECONDS` | `120` | Максимальная длительность одного интервала выборочного повторного распознавания. |
| `REPROCESS_CANDIDATE_RETENTION_DAYS` | `7` | Срок хранения завершённых и отклонённых кандидатов повторной обработки. Опубликованные ревизии хранятся вместе с заданием. |
| `ASR_SEGMENT_PADDING_SECONDS` | `0.25` | Добавляет контекст в свободных промежутках. Padding не пересекает границы соседних говорящих и overlap. |
| `ASR_BATCH_SIZE` | `4` | Число фрагментов в одном batch GigaAM. Для GTX 1630 4 ГБ — `4`; при OOM уменьшить до `2` или `1`. Для 8–12 ГБ обычно `8`. |
| `ASR_MERGE_GAP_SECONDS` | `0.35` | Склеивает только последовательные фрагменты одного говорящего с короткой паузой. Не пересекает смену говорящего или overlap. |
| `ASR_BUCKET_BY_DURATION` | `true` | Группирует похожие длительности и уменьшает лишний zero-padding внутри batch. |
| `ASR_SPLIT_ON_SILENCE` | `true` | Ищет паузу перед жёсткой границей `ASR_MAX_SEGMENT_SECONDS`. |
| `ASR_SILENCE_THRESHOLD_DB` | `-38` | Порог тишины в dBFS. Для шумной комнаты можно снизить до `-42…-48`; слишком низкий порог перестанет находить паузы. |
| `ASR_MIN_SILENCE_SECONDS` | `0.18` | Минимальная длительность паузы для разреза. Диапазон `0.15–0.35`. |
| `ASR_PAUSE_SEARCH_SECONDS` | `2.0` | На сколько секунд назад от жёсткой границы искать подходящую паузу. |
| `ASR_SHORT_SEGMENT_RECOVERY_ENABLED` | `true` | Повторяет пустой ASR для коротких реплик, не меняя чувствительность PyAnnote. |
| `ASR_SHORT_SEGMENT_MAX_SECONDS` | `1.5` | Максимальная длительность реплики для дополнительной попытки. |
| `ASR_SHORT_SEGMENT_CONTEXT_SECONDS` | `1.5` | Желаемый контекст слева и справа; фактически ограничивается соседними речевыми интервалами. |
| `ASR_SHORT_SEGMENT_MAX_RETRIES` | `1` | Число последовательных повторов. Для GTX 1630 рекомендуется `1`. |
| `OVERLAP_SEPARATION_MAX_SECONDS` | `8.0` | Максимальный размер одного unresolved-overlap фрагмента. Он распознаётся один раз с `speaker_id=UNKNOWN` и подготавливается для будущего последовательного speech separation. |

После изменения перезапустите worker. Пересборка образа не нужна.

## 7. Нормализация аудио

| Переменная | Значение | Назначение |
|---|---|---|
| `AUDIO_NORMALIZATION_ENABLED` | `true` | Включает стандартный динамический профиль нормализации. |
| `AUDIO_NORMALIZE_FILTER` | пусто | Полная пользовательская цепочка FFmpeg `-af`. Если заполнена, заменяет стандартно собираемую цепочку. |
| `AUDIO_HIGHPASS_HZ` | `70` | Убирает гул ниже частоты речи. Обычно `60–100` Гц. |
| `AUDIO_LOWPASS_HZ` | `7600` | Убирает высокочастотные помехи выше полезного диапазона речи. |
| `AUDIO_DYNAUDNORM_FRAME` | `500` | Размер окна `dynaudnorm` в миллисекундах. |
| `AUDIO_DYNAUDNORM_GAUSS` | `31` | Сглаживание изменения усиления. Большие значения уменьшают «качание» громкости. |
| `AUDIO_DYNAUDNORM_PEAK` | `0.90` | Целевой пик сигнала. Допустимый диапазон `0–1`. |
| `AUDIO_DYNAUDNORM_MAX_GAIN` | `8` | Максимальное динамическое усиление. Слишком большое значение поднимает шум и реверберацию вместе с тихой речью. |
| `AUDIO_LIMITER` | `0.95` | Финальный limiter от цифрового перегруза. |

Если `AUDIO_NORMALIZE_FILTER` заполнен, он применяется как явная цепочка даже при выключенном стандартном профиле. Для сравнения интерфейс сохраняет оригинальную и нормализованную дорожки.

Рекомендуемый осторожный профиль для шумной комнаты:

```dotenv
AUDIO_NORMALIZATION_ENABLED=true
AUDIO_HIGHPASS_HZ=70
AUDIO_LOWPASS_HZ=7600
AUDIO_DYNAUDNORM_FRAME=500
AUDIO_DYNAUDNORM_GAUSS=31
AUDIO_DYNAUDNORM_PEAK=0.90
AUDIO_DYNAUDNORM_MAX_GAIN=8
AUDIO_LIMITER=0.95
```

## 8. Диаризация PyAnnote

| Переменная | Значение | Влияние и настройка |
|---|---|---|
| `DIARIZATION_ENABLED` | `true` | Включает определение говорящих. При `false` запись считается одним говорящим. |
| `DIARIZATION_DEVICE` | `auto` | `auto`, `cuda` или `cpu`. Для гарантированного GPU используйте `cuda`. |
| `DIARIZATION_MODEL_DIR` | `/models/diarization` | Каталог локальной pipeline и checkpoint-файлов. |
| `DIARIZATION_MODEL_ID` | `pyannote/speaker-diarization-community-1` | Идентификатор модели для первоначальной загрузки. Локальный `config.yaml` может соответствовать PyAnnote 3.3.2. |
| `DIARIZATION_AUDIO_FILTER` | см. `.env.example` | Отдельная щадящая FFmpeg-цепочка для speaker embeddings. Не применяйте агрессивный denoise, меняющий тембр. |
| `DIARIZATION_CLUSTERING_THRESHOLD` | пусто | Необязательный порог кластеризации. Обычно увеличение порога сильнее объединяет похожие голоса и уменьшает число спикеров; калибруйте по DER/JER. |
| `DIARIZATION_SEGMENTATION_MIN_DURATION_OFF` | пусто | Минимальная пауза между речевыми участками. Увеличение может склеивать короткие паузы. |
| `DIARIZATION_SEGMENTATION_BATCH_SIZE` | `32` | Batch segmentation. Уменьшайте при CUDA OOM. |
| `DIARIZATION_EMBEDDING_BATCH_SIZE` | `32` | Batch speaker embeddings — основной тяжёлый этап. Для GTX 1630 проверенное значение `32`; при OOM — `16`, затем `8`. |
| `DIARIZATION_ALLOW_TF32` | `false` | TF32 ускоряет Ampere и новее ценой воспроизводимости. GTX 1630 не получает от TF32 пользы. |
| `ALLOW_SINGLE_SPEAKER_FALLBACK` | `true` | При недоступной диаризации разрешает завершить задачу как одного говорящего. Для строгого контроля качества установите `false`. |
| `HF_TOKEN` | пусто | Токен Hugging Face только для первоначальной загрузки закрытых моделей. Не хранить в Git и удалить из production `.env` после загрузки. |
| `HF_HUB_DISABLE_XET` | `1` | Отключает Xet-механизм Hugging Face при загрузке модели. На распознавание не влияет. |

Overlap сохраняется как один временной интервал `UNKNOWN`, а кандидаты говорящих записываются в `overlap_speaker_ids`. Одна и та же монофоническая смесь не отправляется в GigaAM по одному разу на каждого кандидата.

## 9. Словари и дипломатический режим

| Переменная | Значение по умолчанию | Назначение |
|---|---|---|
| `DICTIONARIES_DIR` | `/dictionaries` | Каталог обычных файлов замен `ошибка>исправление;`. |
| `DICTIONARY_ENABLED` | `true` | Разрешает применение словарей, если пользователь отметил опцию в форме. |
| `DIPLOMATIC_DICTIONARY_PATH` | `/dictionaries/diplomaticReplacement.txt` | Файл литературных и уважительных замен дипломатического режима. |

Словари смонтированы read-only. После изменения текстового файла обычно достаточно запустить новое задание; если worker уже кешировал словарь, перезапустите worker.

## 10. Celery

| Переменная | Значение | Назначение |
|---|---|---|
| `CELERY_TASK_ALWAYS_EAGER` | `false` | При `true` задача выполняется синхронно в процессе API. Только для тестов; production — `false`. |

Фактическая параллельность worker задаётся командой Compose: `--concurrency=1 --pool=solo`. Это безопасно для одной GPU 4 ГБ. Изменение числа CPU-потоков не создаёт дополнительные одновременные GPU-задачи.

## 11. Потоковая запись

| Переменная | Значение | Назначение |
|---|---|---|
| `STREAM_QUEUE_MAX_CHUNKS` | `32` | Максимум PCM chunks в bounded queue. При заполнении включается backpressure. |
| `STREAM_CHUNK_MAX_BYTES` | `32000` | Максимальный размер одного бинарного WebSocket-сообщения. Для PCM16 mono 16 кГц это примерно 1 секунда. |
| `STREAM_EVENT_MAX_LENGTH` | `1000` | Максимальная длина Redis Stream событий с приблизительным trim. |
| `STREAM_IDLE_TTL_SECONDS` | `3600` | Время жизни неактивной stream-сессии. |

Потоковая запись является экспериментальной. Endpoint сравнивает ключ с `settings.api_key`, но поле `API_KEY` пока не объявлено в `Settings`. До исправления этого контракта держите `VITE_ENABLE_STREAMING_RECORDING=false`.

## 12. Frontend: переменные времени сборки

| Переменная | Значение по умолчанию | Назначение |
|---|---|---|
| `VITE_ENABLE_STREAMING_RECORDING` | `false` | Показывает отдельный пункт потоковой записи. Передаётся Dockerfile frontend через Compose. |
| `VITE_API_BASE_URL` | пусто | Базовый URL API. Пусто означает тот же origin, что рекомендуется при nginx reverse proxy. |
| `VITE_API_KEY` | `change-me` в текущем коде | Ключ экспериментального WebSocket. Не включать streaming до исправления backend-контракта. |
| `VITE_MAX_UPLOAD_SIZE_MB` | `4096` | Клиентская проверка суммарного размера файлов. Должна совпадать с `MAX_UPLOAD_SIZE_MB`. |
| `VITE_STATUS_POLL_INTERVAL_MS` | `3000` | Интервал опроса состояния задания в миллисекундах. |

Только `VITE_ENABLE_STREAMING_RECORDING` сейчас явно передаётся как build argument стандартного Dockerfile. Остальные значения работают при локальной Vite-сборке или после добавления соответствующих `ARG/ENV` в Dockerfile frontend. Все `VITE_*` видны пользователю в JavaScript и не должны содержать настоящие секреты.

## 13. GPU и системные переменные

`docker-compose.gpu.yml` задаёт следующие значения автоматически:

| Переменная | Значение | Назначение |
|---|---|---|
| `NVIDIA_VISIBLE_DEVICES` | `all` | Какие GPU видны worker и stream-worker. Можно указать индекс `0`. |
| `NVIDIA_DRIVER_CAPABILITIES` | `compute,utility` | CUDA compute и доступ к телеметрии `nvidia-smi/NVML`. |

Дополнительные необязательные переменные:

| Переменная | Пример | Назначение |
|---|---|---|
| `OMP_NUM_THREADS` | `8` | CPU-потоки OpenMP для одного процесса. |
| `MKL_NUM_THREADS` | `8` | CPU-потоки Intel MKL. |
| `OPENBLAS_NUM_THREADS` | `8` | CPU-потоки OpenBLAS. |
| `TZ` | `Asia/Yekaterinburg` | Часовой пояс логов и системных операций контейнера. |

Не устанавливайте число потоков выше числа доступных физических/логических CPU. Эти параметры не меняют Celery `concurrency`.

## 14. Рекомендуемый профиль GTX 1630 4 ГБ

```dotenv
ASR_DEVICE=cuda
DIARIZATION_DEVICE=cuda
ASR_USE_FP16=true
ASR_BATCH_SIZE=4
ASR_MAX_SEGMENT_SECONDS=22
OVERLAP_SEPARATION_MAX_SECONDS=8
DIARIZATION_SEGMENTATION_BATCH_SIZE=32
DIARIZATION_EMBEDDING_BATCH_SIZE=32
DIARIZATION_ALLOW_TF32=false
CELERY_TASK_ALWAYS_EAGER=false
```

При CUDA OOM сначала уменьшайте `ASR_BATCH_SIZE` до `2`, затем `DIARIZATION_EMBEDDING_BATCH_SIZE` до `16`. Не запускайте несколько GPU Celery workers на одной GTX 1630.

## 15. Применение изменений

Backend и worker без пересборки образов:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d --force-recreate api worker stream-worker
```

Изменения `VITE_*`:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml build frontend
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d frontend nginx
```

Проверка итоговой конфигурации:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml config
docker compose exec worker python -c "from app.config import get_settings; print(get_settings())"
docker compose exec worker python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

Не публикуйте вывод `get_settings()` без проверки: он может содержать адреса инфраструктуры; `HF_TOKEN` маскируется типом `SecretStr`.

## 16. Диагностика типичных ошибок

- `ValidationError ... boolean`: в boolean-переменную записано `cuda` или другое недопустимое значение. `DIARIZATION_ENABLED=true`, а устройство задаётся отдельно: `DIARIZATION_DEVICE=cuda`.
- GPU не виден: запускайте обе Compose-конфигурации и проверьте `docker inspect ... DeviceRequests`.
- Frontend не изменился после правки `VITE_*`: требуется пересборка frontend, простого restart недостаточно.
- Файл отклоняется браузером: проверьте одновременно `VITE_MAX_UPLOAD_SIZE_MB`, `MAX_UPLOAD_SIZE_MB`, nginx `client_max_body_size` и timeout.
- Слишком много спикеров: калибруйте `DIARIZATION_CLUSTERING_THRESHOLD` на эталонных RTTM, не по одной записи.
- Тихая речь пропускается: сначала сравните оригинальную и нормализованную дорожки; чрезмерный gain также усиливает шум и ухудшает embeddings.
