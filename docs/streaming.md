# Потоковое распознавание

Потоковая обработка не создаёт Celery-задачу на каждый фрагмент. Один сеанс
живёт в API-процессе, а Celery запускается только после команды `stop`.

## WebSocket

Подключение:

```text
ws://host/api/v1/streams/{stream_id}?api_key=change-me
```

Первое сообщение JSON:

```json
{
  "type": "start",
  "metadata": {
    "title": "Совещание",
    "meeting_type": "general",
    "expected_speakers": 2,
    "apply_dictionary": true,
    "generate_protocol": true
  }
}
```

После `stream.ready` клиент отправляет бинарные PCM-фрагменты:

- PCM signed 16-bit little-endian;
- mono;
- 16 000 Hz;
- размер одного сообщения не более `STREAM_CHUNK_MAX_BYTES`.

Завершение:

```json
{"type": "stop"}
```

Сервер публикует события `stream-events:{stream_id}` в Redis Streams и
отправляет их всем подключённым клиентам. Основные события:

- `stream.started`;
- `stream.backpressure`;
- `transcript.final` с `speaker_provisional: true`;
- `stream.offline_queued` с `job_id` и `task_id`;
- `stream.error`.

## Ограничение нагрузки

`STREAM_QUEUE_MAX_CHUNKS` задаёт размер bounded queue **на каждую сессию**. При переполнении сервер
возвращает `STREAM_BACKPRESSURE` и клиент должен уменьшить частоту/размер
отправки. Celery не участвует в обработке отдельных фрагментов.

Отдельный `stream-worker` обслуживает все активные сессии. У него один общий
экземпляр GigaAM и GPU-семафор допускает одну ML-операцию одновременно.
Это важно для GTX 1630 с 4 ГБ VRAM: параллельные копии GigaAM/PyAnnote могут
привести к OOM. Сессии остаются независимыми, но их GPU-фрагменты выполняются
по очереди.

## Диаризация

В потоковом режиме `OnlineSpeakerTracker` выдаёт временные идентификаторы для
немедленного UI. Это не финальная диаризация: после `stop` создаётся обычный
job с `original.wav`, и существующая offline-цепочка запускает PyAnnote,
GigaAM, коррекцию словаря и экспорт DOCX/TXT/JSON. Именно результат offline
обработки является окончательным и может переименовать speaker IDs.

Если браузер отключился, сессия продолжает работать в отдельном stream-worker.
Состояние и события сохраняются в Redis, поэтому при восстановлении
соединения клиент может получить состояние из `stream:{stream_id}` и историю
из Redis Stream. После перезапуска API незавершённую сессию нужно завершить
новым подключением или пометить как interrupted отдельной задачей очистки.
