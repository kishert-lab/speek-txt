# Эксплуатация и диагностика

## 1. Ежедневная проверка

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml ps
curl -fsS http://localhost/health/live
curl -fsS http://localhost/health/ready
curl -fsS http://localhost/api/v1/health/metrics | jq
docker compose exec redis redis-cli PING
nvidia-smi
```

Проверяйте свободное место для Docker volumes, длину очереди, возраст самой старой задачи, GPU memory/utilization и наличие повторяющихся ошибок worker.

## 2. Логи

```bash
docker compose logs --since=30m api
docker compose logs --since=30m worker
docker compose logs --since=30m stream-worker
docker compose logs --since=30m nginx
docker compose logs -f worker
```

Ключевые сообщения worker:

- `Audio source`: длительность и параметры исходника;
- `Audio normalization finished`: время FFmpeg;
- `Diarization device selected: cuda`: выбор GPU;
- `Diarization pipeline ready`: pipeline загружен и перенесён на устройство;
- `segmentation/embeddings/clustering`: текущий этап PyAnnote;
- `Segment transcribed`: GigaAM закончил сегмент;
- `Job completed`: результат экспортирован.

Предупреждение `torch.load(weights_only=False)` не является причиной остановки. Предупреждение `std(): degrees of freedom is <= 0` обычно связано с очень короткими фрагментами embeddings; единичное сообщение допустимо, массовое требует проверки сегментации и качества записи.

## 3. Очередь Redis и Celery

Число сообщений, ещё не взятых worker:

```bash
docker compose exec redis redis-cli LLEN celery
```

Число задач со статусом `queued` в метаданных приложения:

```bash
docker compose exec -T redis sh -c \
  'for key in $(redis-cli --raw --scan --pattern "job:*"); do redis-cli --raw GET "$key"; done' \
  | jq -s '[.[] | select(.status == "queued")] | length'
```

Список задач по дате с длительностью:

```bash
docker compose exec -T redis sh -c \
  'for key in $(redis-cli --raw --scan --pattern "job:*"); do redis-cli --raw GET "$key"; done' \
  | jq -s -r 'sort_by(.created_at) | reverse[] | [.id, (.title // ""), .status, .created_at] | @tsv'
```

Celery inspect при `--pool=solo` может не ответить, пока исполняется длинная задача. Это ограничение remote control solo pool, а не доказательство зависания.

Перезапуск worker без удаления очереди:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml restart worker
docker compose logs -f worker
```

Благодаря late ack незавершённая задача должна вернуться в очередь при потере worker. Не очищайте Redis и не выполняйте `FLUSHALL`, если требуется сохранить задания.

## 4. Проверка GPU

На хосте:

```bash
nvidia-smi
watch -n 1 nvidia-smi
```

В worker:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml exec worker \
  python -c 'import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0)); x=torch.randn(2048,2048,device="cuda"); y=x@x; torch.cuda.synchronize(); print(y.device)'
```

Проверка Docker reservation:

```bash
docker inspect speek-text-worker-1 --format '{{json .HostConfig.DeviceRequests}}'
```

Если `nvidia-smi` на хосте работает, а `torch.cuda.is_available()` в контейнере возвращает `False`, причина находится в NVIDIA Container Toolkit/runtime или контейнер был создан без GPU overlay. Выполните `up -d --force-recreate`, а не `restart`.

Нулевая текущая GPU utilization при занятой VRAM нормальна между batch-операциями: модель остаётся в памяти. Оценивайте график за весь этап, а не одну выборку.

## 5. Медленная обработка

Сначала сравните длительности этапов в логах. Для записи длиной `D` вычисляйте Real Time Factor:

```text
RTF = processing_seconds / duration_seconds
```

`RTF < 1` означает обработку быстрее реального времени.

Если медленная диаризация:

1. Убедитесь, что `pipeline_device=cuda` и `cuda_available=True`.
2. Проверьте, что batch size не вызывает paging/OOM.
3. Сравните `segmentation elapsed`, количество embeddings и время clustering.
4. Проверьте, не создаётся ли pipeline заново для каждого задания (`cached=False` означает холодную загрузку).
5. Не запускайте несколько ML workers на GPU с 4 GiB.

Если медленное распознавание:

1. Проверьте `ASR_DEVICE=cuda` или `auto` и FP16.
2. Посчитайте количество и длину сегментов.
3. Не уменьшайте `ASR_MAX_SEGMENT_SECONDS` без необходимости: слишком много коротких файлов увеличивает overhead.
4. Проверьте скорость диска volume `/data`.

CPU-потоки (`OMP_NUM_THREADS`, `MKL_NUM_THREADS`, `OPENBLAS_NUM_THREADS`) помогают отдельным операциям, но не распараллеливают Celery solo worker по заданиям.

## 6. Плохое разделение говорящих

Диагностику проводите на одном и том же контрольном аудио и фиксируйте число speakers, DER либо ручную оценку, пропущенные тихие ответы и ошибки overlap.

Безопасный порядок настройки:

1. Прослушайте оригинал и `diarization.wav`.
2. Укажите разумный `min_speakers/max_speakers`, если известен диапазон.
3. Сравните без дополнительного `DIARIZATION_AUDIO_FILTER` и с щадящим профилем.
4. Меняйте `DIARIZATION_CLUSTERING_THRESHOLD` малыми шагами, например 0.05.
5. После каждого изменения повторяйте один контрольный набор.

Интерпретация clustering threshold зависит от pipeline. Для текущей конфигурации нельзя заранее считать, что больше всегда означает больше speakers. Проверяйте экспериментально на размеченном наборе.

Не используйте для PyAnnote агрессивный denoise, compressor или высокий dynaudnorm max gain без A/B-теста: они меняют тембр и расстояния embeddings. Тихие реплики прежде всего исправляются размещением нескольких микрофонов, снижением реверберации и отсутствием клиппинга.

## 7. Большие файлы и обрывы браузера

Проверить итоговую конфигурацию Nginx:

```bash
docker compose exec nginx nginx -T 2>&1 | grep -E \
  'client_max_body_size|client_body_timeout|proxy_request_buffering|proxy_read_timeout|proxy_send_timeout'
```

Контрольная загрузка:

```bash
curl -v -F "file=@/tmp/audio_test.mp3" -F "title=test" \
  http://localhost/api/v1/protocols
```

Windows CMD:

```bat
curl -v -F "file=@c:\tmp\audio_test.MP3" -F "title=test" http://server/api/v1/protocols
```

Если curl проходит, а браузер обрывается, серверный лимит исключён. Проверяйте DevTools Network, клиентский timeout, антивирус/прокси браузера, frontend bundle и промежуточные reverse proxies. Ответ `408` формирует Nginx при таймауте чтения тела запроса; `400` от API требует просмотра JSON-тела ответа и логов API.

Предупреждение `upstream response is buffered to a temporary file` при скачивании аудио означает буферизацию ответа, а не ошибку. Для audio location в проекте `proxy_buffering off` и передаются Range/If-Range.

## 8. WebSocket и микрофон

Проверить upgrade:

```bash
docker compose exec nginx nginx -T | grep -A15 '/api/v1/streams/'
docker compose logs -f nginx api stream-worker
```

Правила:

- HTTP-страница использует `ws://`, HTTPS-страница использует `wss://`;
- микрофон в браузере доступен только через HTTPS или localhost;
- внешний proxy обязан передавать `Upgrade` и `Connection: upgrade`;
- frontend feature flag применяется во время сборки;
- при backpressure клиент должен остановить отправку, а не накапливать память.

## 9. Метрики страницы состояния

API endpoint:

```bash
curl http://localhost/api/v1/health/metrics | jq
```

Если GPU показывает «Нет данных»:

```bash
docker compose exec redis redis-cli GET server:worker-metrics
docker compose logs worker | grep -iE 'pynvml|metrics|gpu'
```

Ключ живёт 15 секунд. Проверьте, что worker запущен, имеет GPU и использует тот же `REDIS_URL`, что API.

## 10. Исправленные стенограммы и датасет

Список исправленных заданий:

```bash
docker compose exec worker sh -lc \
  'find /data/training/corrections -maxdepth 1 -type f -name "*.json" -printf "%TY-%Tm-%Td %TH:%TM\t%f\n" | sort -r'
```

Экспорт:

```bash
make training-export
```

Перед обучением требуется ручная проверка примеров. Пустые исправления могут означать намеренное удаление ложного сегмента и не должны автоматически становиться ASR target.

## 11. Очистка и резервное копирование

Автоматическая очистка запускается при `worker_ready`, затем раз в сутки. Исправленные задачи сохраняются. Для резервного копирования нужны оба volumes:

- `protocol_data`: записи, результаты, исправления;
- `redis_data`: состояния и очередь.

Перед консистентным cold backup остановите API и workers либо используйте средства snapshot конкретного storage backend. Не копируйте только Redis: аудио и экспорт находятся в `/data`.

## 12. Типовые ошибки

| Код/симптом | Причина | Действие |
|---|---|---|
| `FILE_TOO_LARGE` | Сумма файлов выше лимита | Согласовать Nginx и `MAX_UPLOAD_SIZE_MB` |
| `UNSUPPORTED_MEDIA` | Расширение/ffprobe | Перекодировать или проверить повреждение |
| `MODEL_NOT_FOUND` | Нет checkpoint GigaAM | Смонтировать `models/gigaam` |
| `TOKENIZER_NOT_FOUND` | Нет tokenizer | Смонтировать tokenizer |
| `DIARIZATION_UNAVAILABLE` | Pipeline PyAnnote не загрузилась | Проверить config/checkpoints и совместимость версий |
| `JOB_NOT_FOUND` | Redis-ключ удалён | Проверить retention и согласованность volume/Redis |
| `JOB_NOT_COMPLETED` | Результат запрошен рано | Продолжить polling status endpoint |
| `STREAM_BACKPRESSURE` | Bounded queue заполнена | Приостановить отправку |
| GPU `null` | Нет свежей worker metric | Проверить worker, pynvml, GPU и Redis |
| `No nodes replied` | solo worker занят | Смотреть логи и Redis, повторить после задачи |

## 13. Тестирование и выпуск

```bash
python -m pip install -r requirements-dev.txt
python -m pytest
cd frontend
npm ci
npm run test
npm run build
```

Перед production-релизом пройдите [Excel-чек-лист](Чек-лист_оценки_сервиса.xlsx), выполните тест на длинной записи, нескольких файлах, тихом говорящем, overlap и загрузке максимально допустимого размера.

