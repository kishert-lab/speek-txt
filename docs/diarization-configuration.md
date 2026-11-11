# Настройка диаризации через ENV

## Источники конфигурации

Параметры применяются в следующем порядке:

1. Непустое значение из ENV.
2. Значение из локального `/models/diarization/config.yaml`.
3. Значение по умолчанию конструктора конкретного pipeline, если оно предусмотрено pyannote.audio.

Пустая переменная, например `DIARIZATION_CLUSTERING_THRESHOLD=`, считается отсутствующей. Значения `0` и `false` являются явными переопределениями. Локальный YAML и веса моделей не изменяются.

`Pipeline.from_pretrained()` создает pipeline из локального YAML. Параметры `clustering.*` и `segmentation.min_duration_off` фильтруются по `pipeline.parameters(instantiated=False)` и передаются в `pipeline.instantiate()`. Размеры batch и `embedding_exclude_overlap` относятся к конструктору pipeline и применяются как runtime-атрибуты, а не через `instantiate()`.

`DIARIZATION_MODEL_ID` приложением не используется. Загрузка моделей из Hugging Face при старте запрещена: источником служит только `DIARIZATION_MODEL_DIR`.

## Переменные

| Переменная | Тип и диапазон | Пример | Назначение |
| --- | --- | --- | --- |
| `DIARIZATION_ENABLED` | bool | `true` | Включает PyAnnote. |
| `DIARIZATION_DEVICE` | `auto`, `cuda`, `cpu` | `auto` | Выбирает устройство. Явный `cuda` завершится ошибкой, если CUDA недоступна. |
| `DIARIZATION_MODEL_DIR` | path | `/models/diarization` | Каталог локального `config.yaml` и весов. |
| `DIARIZATION_AUDIO_FILTER` | строка FFmpeg или пусто | `highpass=f=70,lowpass=f=7800` | Фильтр только для отдельной дорожки диаризации. |
| `DIARIZATION_AUDIO_NORMALIZATION_ENABLED` | bool или пусто | `true` | Включает применение фильтра; при `false` сохраняется только перевод в mono PCM16 16 kHz. |
| `DIARIZATION_CLUSTERING_THRESHOLD` | float, `0..2` или пусто | `0.6` | Порог агломеративной кластеризации. |
| `DIARIZATION_CLUSTERING_METHOD` | enum или пусто | `centroid` | `average`, `centroid`, `complete`, `median`, `single`, `ward`, `weighted`. |
| `DIARIZATION_CLUSTERING_MIN_CLUSTER_SIZE` | int, `1..20` или пусто | `15` | Минимальный размер кластера. |
| `DIARIZATION_SEGMENTATION_MIN_DURATION_OFF` | float, `0..1` или пусто | `0.0` | Заполняет короткие паузы между участками речи. |
| `DIARIZATION_SEGMENTATION_BATCH_SIZE` | int, `1..1024` или пусто | `32` | Batch сегментации. |
| `DIARIZATION_EMBEDDING_BATCH_SIZE` | int, `1..1024` или пусто | `32` | Batch извлечения speaker embeddings. |
| `DIARIZATION_EMBEDDING_EXCLUDE_OVERLAP` | bool или пусто | `true` | Исключает перекрывающуюся речь при расчете embeddings. |
| `DIARIZATION_ALLOW_TF32` | bool | `false` | Управляет TF32 для CUDA matmul и cuDNN. |
| `ALLOW_SINGLE_SPEAKER_FALLBACK` | bool | `true` | Разрешает результат с одним говорящим при недоступности PyAnnote. Ошибки конфигурации не скрываются fallback-механизмом. |
| `DIARIZATION_DEBUG` | bool | `false` | Пишет диагностическую сводку в лог. |
| `DIARIZATION_SAVE_DEBUG_ARTIFACTS` | bool | `false` | Сохраняет `diarization-debug.json` в каталоге задания. |

В `pyannote.audio 3.3.2` класс `SpeakerDiarization` не предоставляет рабочий `default_parameters()` для отсутствующих обязательных параметров. Поэтому обязательные значения должны присутствовать в YAML или ENV; иначе запуск завершится понятной ошибкой конфигурации.

## Аудиодорожки

Исходный файл не изменяется. Процессор создает две независимые дорожки:

- `normalized.wav` для GigaAM с прежней обработкой ASR;
- `diarization.wav` для PyAnnote с отдельным ENV-фильтром.

Отключение фильтра диаризации не меняет ASR. FFmpeg вызывается без shell, поэтому значение фильтра не исполняется как команда. Ошибка синтаксиса фильтра возвращается как `FFMPEG_ERROR`.

## Диагностика

При `DIARIZATION_DEBUG=true` фиксируются эффективная конфигурация, время выполнения, количество говорящих и сегментов, границы и длительности сегментов, количество смен говорящего и быстрые переключения без существенной паузы.

PyAnnote 3.3.2 не публикует similarity/confidence для кластеров в результате pipeline. Поэтому такие показатели отмечаются как недоступные и не вычисляются искусственно. Диагностика не сохраняет аудио и embeddings и не объединяет спикеров автоматически.

Для сохранения JSON дополнительно установите `DIARIZATION_SAVE_DEBUG_ARTIFACTS=true`. Этот флаг имеет смысл только вместе с `DIARIZATION_DEBUG=true`.

## Безопасное развертывание

1. Измените нужные значения в `.env`.
2. Проверьте, что `/models/diarization/config.yaml` и локальные веса доступны в `./models`, смонтированном read-only.
3. Пересоздайте прикладные воркеры без сборки образов:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d --no-build --force-recreate worker stream-worker
```

4. Проверьте запуск и фактические значения:

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml logs -f worker
```

В логе должны появиться `Diarization device selected`, `Diarization pipeline ready` и `Diarization effective configuration`. Модель загружается один раз на процесс Celery и повторно используется следующими задачами. При нескольких процессах или контейнерах каждый процесс имеет собственный экземпляр модели.

`stream-worker` принимает поток и выполняет предварительное online-сопровождение. После завершения записи он ставит финальную задачу в Celery, где применяется описанная конфигурация PyAnnote.

## Методика оценки качества

Используйте фиксированный набор реальных совещаний с ручной RTTM-разметкой, включая тихую речь, реверберацию, шум, короткие реплики и перекрытия. Разделите записи на tuning и test: параметры выбираются только на tuning, итоговая оценка выполняется один раз на test.

Для каждой конфигурации сохраняйте:

- DER и его составляющие: missed speech, false alarm и speaker confusion;
- число ложных смен говорящего на час;
- число лишних идентификаторов спикеров относительно эталона;
- число пропущенных говорящих;
- время обработки и пиковую VRAM/RAM.

Сравнивайте baseline и один измененный параметр за эксперимент при одинаковом аудио, версии модели и collar/правиле учета overlap. Рекомендуется дополнительно вручную проверить участки, отмеченные как подозрительные в диагностике. Снижение DER не считается достаточным, если одновременно резко растет число ложных смен или исчезают тихие говорящие.

## Выявленные ограничения

- Поддержка параметров определяется фактическим объектом pipeline; неизвестные значения YAML игнорируются с предупреждением, а неподдерживаемое явное ENV-переопределение завершает задачу ошибкой.
- Кэш модели локален процессу и не разделяется между контейнерами.
- Online speaker tracker потоковой записи не является финальной PyAnnote-диаризацией.
- Автоматическое объединение похожих идентификаторов спикеров намеренно не реализовано без измерений качества.
