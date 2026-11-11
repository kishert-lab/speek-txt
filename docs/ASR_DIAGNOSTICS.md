# Диагностика потери текста ASR

Для каждого нового задания создаётся файл `diagnostics.json`. Он доступен через:

```text
GET /api/v1/protocols/{job_id}/files/diagnostics
```

Файл позволяет сопоставить:

1. речевые интервалы, найденные PyAnnote;
2. подготовленные фрагменты GigaAM;
3. результаты каждого batch без содержимого текста;
4. исходные `hypothesis_id`, вошедшие в каждый итоговый сегмент;
5. количество исключённых и объединённых фрагментов.

## Счётчики

- `pyannote_speech_intervals_found`;
- `asr_fragments_prepared`;
- `asr_fragments_sent_to_gigaam`;
- `asr_successfully_recognized`;
- `asr_empty_results`;
- `transcript_builder_excluded`;
- `transcript_builder_merged`.

`asr_empty_results` относится к первичной попытке GigaAM. Успешное восстановление короткой реплики отражается отдельно в `short_speech_recovery_status`.

## Время batch-инференса

GigaAM обрабатывает несколько фрагментов одним batch. Точное CUDA-время отдельного элемента batch определить без дополнительной синхронизации невозможно. Поэтому:

- `batch_inference_seconds` — фактическое время вызова batch;
- `inference_seconds` — равная доля времени batch на один фрагмент плюс время последовательного Short Speech Recovery, если оно запускалось.

## Конфиденциальность

Диагностический файл и структурированные production-логи не содержат распознанный текст. Сохраняются только количество символов, слов, временные границы и технические идентификаторы. Результаты Celery также сокращены до технического статуса, чтобы стандартный Celery logger не печатал стенограмму.
