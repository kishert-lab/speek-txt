# Сбор аналитики обработки на production

Скрипты формируют технический отчёт по сохранённым заданиям. Они работают только на чтение и не изменяют Redis, результаты распознавания или аудиофайлы.

В отчёт не включаются:

- текст стенограммы;
- название встречи;
- имя исходного аудиофайла;
- имена участников.

## Быстрый запуск

Запускайте из каталога проекта, где находится `docker-compose.yml`:

```bash
chmod +x scripts/collect-production-analytics.sh
./scripts/collect-production-analytics.sh 7
```

Аргумент `7` означает количество последних дней. Каталог отчёта будет создан в `analytics/<UTC timestamp>`.

Указать период и каталог явно:

```bash
./scripts/collect-production-analytics.sh 7 /tmp/meeting-analytics
```

Собрать все результаты, которые ещё физически сохранены в volume:

```bash
./scripts/collect-production-analytics.sh 0 /tmp/meeting-analytics-all
```

Скрипт запускает Python внутри контейнера `api`, читает общий volume `/data`, копирует готовый отчёт на хост и удаляет временный каталог контейнера. Перезапуск сервисов не требуется.

## Прямой запуск Python

Если данные доступны на хосте без Docker:

```bash
python3 scripts/collect-processing-analytics.py \
  --data-root /path/to/data \
  --output-dir /tmp/meeting-analytics \
  --since-days 7
```

Для конкретного задания параметр можно повторять:

```bash
python3 scripts/collect-processing-analytics.py \
  --data-root /path/to/data \
  --output-dir /tmp/one-job \
  --since-days 0 \
  --job-id 11111111-1111-1111-1111-111111111111
```

## Файлы отчёта

- `summary.json` — общие показатели, p50/p95, RTF и доли проблемных результатов;
- `jobs.csv` — метрики этапов по каждому заданию;
- `asr_fragments.csv` — обезличенные данные фрагментов GigaAM;
- `problem_jobs.csv` — только задания, требующие проверки.

Основные признаки в `problem_flags`:

- `diagnostics_missing` — запись создана до появления `diagnostics.json`;
- `asr_empty` — GigaAM вернула пустой первичный результат хотя бы для одного фрагмента;
- `short_speech_unrecognized` — короткую реплику не удалось восстановить;
- `transcript_builder_excluded` — подготовленный ASR-фрагмент не вошёл в итоговую стенограмму;
- `rtf_above_1` — обработка заняла больше времени, чем длительность записи.

## Ограничение хранения

Сервис удаляет результаты старше `RESULT_RETENTION_DAYS`. Скрипт не может собрать уже удалённые задания. Для накопительной статистики запускайте сбор до очистки данных, например ежедневно через cron, и храните полученные отчёты отдельно.

Пример ежедневного запуска в 02:15:

```cron
15 2 * * * cd /home/clicker/speek-text && ./scripts/collect-production-analytics.sh 2 /var/lib/meeting-analytics/$(date -u +\%Y\%m\%dT\%H\%M\%SZ) >> /var/log/meeting-analytics.log 2>&1
```
