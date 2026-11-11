# Диагностика производительности ML-конвейера

## Фактическое распределение вычислений

| Этап | Устройство | Реализация |
|---|---|---|
| FFmpeg, чтение и нарезка WAV | CPU и диск | `AudioService` |
| Segmentation pyannote | GPU при `DIARIZATION_DEVICE=cuda` | `LocalDiarizer` |
| Speaker embeddings pyannote | GPU при `DIARIZATION_DEVICE=cuda` | `LocalDiarizer` |
| Speaker counting и clustering | преимущественно CPU | pyannote pipeline |
| Encoder и decoder GigaAM | GPU при `ASR_DEVICE=cuda` | `GigaAMTranscriber` |
| Словари, сборка стенограммы и протокола | CPU | `PhraseCorrector`, `TranscriptBuilder`, `NullProtocolSummarizer` |
| JSON, TXT и DOCX | CPU и диск | `Exporter` |

`build_processor()` кэшируется на время жизни Celery-процесса. Модели pyannote и GigaAM
инициализируются по одному разу на worker-процесс. `stream-worker` является отдельным
процессом и загружает собственный экземпляр GigaAM.

Pyannote получает декодированный waveform из RAM один раз, но выполняет отдельные
CPU-to-GPU передачи для внутренних segmentation- и embedding-батчей. Их количество
определяется самой библиотекой и размером батча. GigaAM формирует padded batch на CPU
и выполняет одну передачу аудио и одну передачу lengths на GPU для каждого ASR-батча.

Для каждого объединённого ASR-фрагмента пока создаётся временный WAV. Это необходимо
для совместимости с закреплённой версией GigaAM. Файлы удаляются после обработки, если
`KEEP_AUDIO_SEGMENTS=false`.

## Пакетная обработка

- `DIARIZATION_SEGMENTATION_BATCH_SIZE=32` управляет segmentation pyannote.
- `DIARIZATION_EMBEDDING_BATCH_SIZE=32` управляет speaker embeddings pyannote.
- `ASR_BATCH_SIZE=4` управляет пакетами GigaAM.
- `ASR_MERGE_GAP_SECONDS=0.35` объединяет только соседние интервалы одного спикера.
- `ASR_BUCKET_BY_DURATION=true` группирует близкие длительности и уменьшает zero-padding.
- При CUDA OOM GigaAM автоматически делит пакет пополам и повторяет inference.

Для GPU с 4 ГБ VRAM начальное безопасное значение `ASR_BATCH_SIZE` равно 4. Повышение
pyannote embedding batch с 32 до 64 следует делать только после проверки пикового VRAM.
Больший batch обычно повышает throughput, но не улучшает качество и может вызвать OOM.

Увеличение `ASR_MERGE_GAP_SECONDS` уменьшает число ASR-вызовов, но слишком большое
значение может захватить паузу или реплику другого человека. Значение 0.35 не объединяет
интервалы через сегмент другого спикера и сохраняет перекрывающуюся речь.

## Синхронные участки

Offline-конвейер внутри Celery-задачи синхронный: FFmpeg, pyannote, последовательность
ASR-батчей, коррекция и экспорт выполняются по очереди. API работает в отдельном
контейнере и этим не блокируется. Worker использует `solo` и обрабатывает одну offline-задачу
за раз. Несколько процессов на одной 4-ГБ GPU не рекомендуются.

`worker` и `stream-worker` не имеют общего межпроцессного GPU lock. Одновременная offline-
обработка и потоковая сессия могут увеличить задержку и потребление VRAM.

## Метрики результата

Каждый `result.json` содержит:

- `audio_preprocessing_seconds` — probe, конвертация, фильтры и объединение файлов;
- `diarization_segmentation_seconds` — GPU-время segmentation;
- `diarization_embedding_seconds` — GPU-время speaker embeddings;
- `diarization_clustering_seconds` — CPU-интервал clustering;
- `diarization_total_seconds` — полный wall time этапа, включая первичную загрузку модели;
- `asr_total_seconds` — загрузка GigaAM, подготовка сегментов и все ASR-батчи;
- `asr_segments_count` и `asr_average_segment_seconds`;
- `postprocessing_seconds` — словари, сборка стенограммы и протокола;
- `export_seconds` — JSON, TXT и DOCX;
- `processing_total_seconds`, `gpu_peak_memory_mb` и `rtf`.

CUDA Events измеряют segmentation и embeddings. Синхронизация выполняется один раз после
всего pyannote pipeline, а не после каждого батча. ASR wall time корректен, потому что
декодирование строк завершает CUDA-работу перед возвратом результата.

Последние 100 наборов метрик хранятся в Redis. Endpoint
`GET /api/v1/health/metrics` публикует `processing.latest`, `processing.average` и
`processing.samples`.
