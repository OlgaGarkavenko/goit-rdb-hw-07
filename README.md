# goit-rdb-hw-07 - Time-series ML features, JSONB та GIN-індекси у PostgreSQL

Домашнє завдання 7 курсу з реляційних БД. Усе виконується в одному notebook **`hw7_time_series_jsonb.ipynb`** (Google Colab) на локальному PostgreSQL 16, який запускається через [`pgserver`](https://pypi.org/project/pgserver/). Окремо встановлювати PostgreSQL не потрібно.

## Джерело даних

| Частина | Джерело |
|---|---|
| Time-series | **NYC TLC Yellow Taxi Trip Records**, лютий і березень 2024 (`yellow_tripdata_2024-02.parquet`, `yellow_tripdata_2024-03.parquet`) з офіційного CDN `https://d37ci6vzurychx.cloudfront.net/trip-data/` ([сторінка TLC](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)) |
| Fallback | **Stub-schema** із завдання: 5 zones × 60 днів = 300 рядків з фіксованою датою `2024-03-31` |
| JSONB event-log | Синтетичні дані: 50 000 подій, згенеровані через `generate_series` |

**Як отримуються файли.** Notebook сам завантажує Parquet через `urlretrieve` у `/content` (у Colab) або в системну temp-папку. Тому Parquet і CSV у репозиторій не додаються.

**Fallback.** Якщо завантаження не вдалося (немає мережі, 403 тощо), notebook автоматично переходить на **stub-schema (Варіант B)** і друкує `DATA_SOURCE = stub`. Яке джерело використано в збереженому запуску, видно в output клітинки завантаження.

> ⚠️ **У збережених output-ах використано stub-schema fallback.** Під час виконання завантаження Parquet повернуло `403 Forbidden` (обмеження мережі в середовищі виконання). Тому `t7_hw_taxi_daily` містить stub-дані (300 рядків). JSONB-частина та всі EXPLAIN-и від джерела taxi-даних не залежать.

### Відхилення від шаблону для real TLC (і чому)
- Беремо **2 місяці з випадковою вибіркою по 300 000 поїздок** (`random_state=42`) замість `head(500_000)` з одного місяця. TLC-файли впорядковані за часом, тому `head` покриває лише ~5 днів, і `trips_lag_30d` був би NULL у всіх рядках.
- Daily aggregation обмежена періодом `2024-02-01 … 2024-03-31`, бо в сирих даних трапляються помилкові дати pickup.
- Для `engine` явно задано драйвер `postgresql+psycopg2://`. SQLAlchemy 2.1+ для `postgresql://` за замовчуванням шукає `psycopg` (v3), якого немає в залежностях.
- Перед `DROP TABLE t7_hw_taxi_daily` виконується `DROP VIEW IF EXISTS v_t7_hw_taxi_features`. Без цього повторний запуск у тому самому runtime падає на залежності view від таблиці.

## Time-series features (`t7_hw_taxi_daily` → `v_t7_hw_taxi_features`)

Гранулярність: один рядок на `(pickup_zone_id, ts_day)`. **Кожне window-визначення містить `PARTITION BY pickup_zone_id`.**

| Feature | Реалізація |
|---|---|
| `trips_lag_1d`, `trips_lag_7d`, `trips_lag_30d` | `LAG(trip_count, n)` по `ORDER BY ts_day` |
| `rolling_avg_fare_7d` | `AVG(avg_fare)` з `RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW` (7 календарних днів) |
| `rolling_std_fare_30d` | `STDDEV_SAMP(avg_fare)` з `RANGE BETWEEN INTERVAL '29 days' PRECEDING AND CURRENT ROW` |
| `cumulative_trips`, `expanding_avg_trips` | `SUM` / `AVG` з `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |

View фільтрує рядки без повної 30-денної історії (`trips_lag_30d IS NOT NULL`). Окрема клітинка показує leakage, що з'являється без `PARTITION BY`.

## JSONB-частина

- `t7_hw_events`: 5 типів подій (`page_view`, `click`, `add_to_cart`, `checkout`, `recommendation_shown`), кожен зі своєю структурою `payload`.
- Запити: containment `@>`, key existence `?` / `?|`, JSONPath (`jsonb_path_query`, `jsonb_path_exists`).
- Індекси з `EXPLAIN (ANALYZE, BUFFERS)` до і після:
  - `GIN (payload jsonb_path_ops)` для `@>`: Seq Scan -> Bitmap Index Scan + Bitmap Heap Scan;
  - expression B-tree `((payload ->> 'device'))` для path-equality.
- Додатковий вимір write-cost GIN: час вставки 50 000 рядків без індексу і з ним.
- `t7_hw_checkouts_norm`: нормалізована таблиця checkout і той самий агрегат у JSONB- та normalized-версіях.
- Bonus: MongoDB aggregation pipeline як pseudocode у markdown.
- Reflection з конкретними cost, time і buffers з output-ів.

Усі службові таблиці мають префікс `t7_hw_`. Кожен DDL-блок починається з `DROP … IF EXISTS`, тому notebook можна перезапускати.

## Як запустити

1. Відкрийте `hw7_time_series_jsonb.ipynb` у Google Colab (*File → Upload notebook* або *Open from GitHub*).
2. *Runtime → Restart session and run all* (або *Restart & Run All*).
3. Перша клітинка встановить `pgserver psycopg2-binary sqlalchemy pandas pyarrow requests`, запустить PostgreSQL у `/tmp/hw7_pg` і виведе `SELECT version()`.

Локально (Python 3.9–3.12, бо для 3.13 wheel `pgserver` немає):

```bash
python -m venv .venv && source .venv/bin/activate
pip install jupyter pgserver psycopg2-binary sqlalchemy pandas pyarrow requests
jupyter notebook hw7_time_series_jsonb.ipynb
```

## Структура репозиторію

```
goit-rdb-hw-07/
├── hw7_time_series_jsonb.ipynb   # notebook з output-ами клітинок
└── README.md
```
