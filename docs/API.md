# REST API и WebSocket

Базовый префикс REST API: `/api/v1`. Тело ошибок имеет единый формат:

```json
{
  "detail": {
    "code": "ERROR_CODE",
    "message": "Описание на русском языке",
    "context": {}
  }
}
```

## 1. Создание протокола

`POST /api/v1/protocols`, `multipart/form-data`, ответ `202 Accepted`.

| Поле | Тип | Обязательное | Описание |
|---|---|---|---|
| `file` | binary | одно из `file/files` | Одиночный файл, обратная совместимость |
| `files` | binary[] | одно из `file/files` | Части встречи в порядке объединения |
| `title` | string | нет | Название |
| `meeting_type` | enum | нет | `general`, `conference`, `court`, `interview` |
| `expected_speakers` | int >= 1 | нет | Точное число говорящих |
| `min_speakers` | int >= 1 | нет | Нижняя граница |
| `max_speakers` | int >= 1 | нет | Верхняя граница |
| `apply_dictionary` | bool | нет | Применять словари, по умолчанию `true` |
| `generate_protocol` | bool | нет | Создать черновик, по умолчанию `true` |
| `diplomatic_mode` | bool | нет | Сформировать обобщение, выводы и поручения в уважительном литературном стиле; по умолчанию `false` |
| `language` | string | нет | Только `ru` |

При включённом `diplomatic_mode` объект `protocol` содержит литературную версию в `transcript_text` и массив `replacements`. Элемент `replacements` включает `segment_index`, `speaker_id`, исходное выражение `source`, литературную замену `replacement` и имя словаря `dictionary`.

Одиночный файл:

```bash
curl -F "file=@meeting.mp3" -F "title=Заседание" \
  http://localhost/api/v1/protocols
```

Несколько файлов:

```bash
curl -F "files=@part1.wav" -F "files=@part2.wav" \
  -F "meeting_type=court" -F "min_speakers=2" -F "max_speakers=8" \
  http://localhost/api/v1/protocols
```

Ответ:

```json
{
  "id": "18b1ca20-a1ab-44f5-8b78-7fa73fcf1324",
  "status": "queued",
  "created_at": "2026-09-11T08:00:00Z",
  "status_url": "/api/v1/protocols/18b1ca20-a1ab-44f5-8b78-7fa73fcf1324",
  "result_url": "/api/v1/protocols/18b1ca20-a1ab-44f5-8b78-7fa73fcf1324/result",
  "queue_position": 2,
  "queue_size": 5
}
```

## 2. Статус

`GET /api/v1/protocols/{job_id}`

Возвращает `status`, `stage`, `progress`, временные метки, предупреждения и ошибку. Для `queued` дополнительно рассчитываются `queue_position` и `queue_size`.

## 3. Итоговый результат

`GET /api/v1/protocols/{job_id}/result`

До завершения возвращает `409 JOB_NOT_COMPLETED`. Результат содержит общую длительность, время обработки, параметры аудио, speakers, сегменты стенограммы, черновик протокола, предупреждения, метрики и ссылки на файлы.

## 4. Имена говорящих

`PATCH /api/v1/protocols/{job_id}/speakers`

```json
{
  "speakers": {
    "SPEAKER_00": "Иванов Иван Иванович",
    "SPEAKER_01": "Петров Пётр Петрович"
  }
}
```

Сервис обновляет сегменты, JSON, TXT и DOCX. Если имя не назначено, в DOCX системный идентификатор `SPEAKER_<номер>` отображается как читаемая метка `Говорящий <номер + 1>`; например, `SPEAKER_00` — `Говорящий 1`. Явно назначенное имя имеет приоритет над этой меткой.

## 5. Редактирование стенограммы

`PATCH /api/v1/protocols/{job_id}/transcript`

```json
{
  "segments": [
    {"index": 1, "text": "Исправленный текст"},
    {"index": 2, "text": ""}
  ]
}
```

Пустая строка допустима и удаляет ошибочно распознанную реплику из текста. Неизвестный индекс возвращает `422 INVALID_REQUEST`. Изменения сохраняются как обучающие примеры.

## 6. Файлы

`GET /api/v1/protocols/{job_id}/files/{file_type}`

| `file_type` | Содержимое |
|---|---|
| `json` | Полный `result.json` |
| `txt` | Стенограмма и черновик протокола |
| `docx` | Документ Word с таймкодами и именами или метками говорящих |
| `audio` | Нормализованный WAV |
| `audio-original` | Исходная/объединённая дорожка |

Для аудио FastAPI/FileResponse и Nginx поддерживают Range-запросы браузерного плеера.

## 7. Удаление

`DELETE /api/v1/protocols/{job_id}`

Если известен Celery task ID, задача отзывается с `terminate=true`, затем удаляются каталог задания и ключ Redis.

## 8. Health и метрики

| Маршрут | Назначение |
|---|---|
| `GET /health/live` | Процесс API запущен |
| `GET /health/ready` | Redis, FFmpeg, storage, модели и выбранное устройство готовы |
| `GET /api/v1/health/metrics` | completed, queue, CPU, GPU и VRAM |

GPU-метрики публикует worker в Redis каждые 5 секунд с TTL 15 секунд. Если worker остановлен, pynvml не видит GPU или API подключён к другому Redis, поля GPU равны `null`.

## 9. WebSocket потоковой записи

Endpoint:

```text
ws(s)://host/api/v1/streams/{stream_id}?api_key=<key>&last_event_id=0-0
```

Клиент сначала отправляет JSON:

```json
{
  "type": "start",
  "metadata": {
    "title": "Заседание",
    "meeting_type": "general",
    "apply_dictionary": true,
    "generate_protocol": true
  }
}
```

После `stream.ready` передаются binary frames: PCM signed 16-bit little-endian, mono, 16 kHz. Размер одного сообщения не должен превышать `STREAM_CHUNK_MAX_BYTES`.

Завершение:

```json
{"type": "stop"}
```

События сервера:

- `stream.ready`: параметры принимаемого PCM;
- `stream.started`: сессия создана;
- `transcript.final`: предварительный текст и временные метки;
- `stream.backpressure`: очередь переполнена, отправку нужно приостановить;
- `stream.offline_queued`: WAV сохранён и поставлен на точную обработку;
- `stream.error`: структурированная ошибка;
- `pong`: ответ на `ping`.

`speaker_id` в live-событиях предварительный (`speaker_provisional=true`) и может не совпасть с итоговой разметкой PyAnnote.

### Важное замечание текущей реализации

Маршрут обращается к `settings.api_key`. Перед включением streaming необходимо убедиться, что поле `api_key` определено в `Settings` и одинаково задано для API и frontend (`VITE_API_KEY`). В текущем справочном `.env.example` этот контракт не оформлен полностью; потоковый режим следует считать экспериментальным до устранения этого несоответствия.
