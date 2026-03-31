# Trigger-Based vs Data Interval Scheduling

## Airflow 2.10.5 & Airflow 3

---

## Kluczowa zmiana: domyślna interpretacja crona

| Cecha                       | Airflow 2.10.5               | Airflow 3                         |
| :--------------------------- | :---------------------------- | :--------------------------------- |
| **Domyślny `schedule`**     | `timedelta(days=1)`          | `None`                            |
| **Cron string → timetable** | `CronDataIntervalTimetable`  | `CronTriggerTimetable`            |
| **`logical_date`**          | = `data_interval_start`      | = `run_after` (czas triggera)     |
| **Catchup po pauzie**       | Nadrabia pominięte interwały | Pomija, czeka na następny trigger |

---

## 1. CronDataIntervalTimetable (domyślne w Airflow 2)

Cron definiuje **granice przedziału danych**. DAG odpala się **po zakończeniu** interwału.

```
Cron: @daily (0 0 * * *)

Uruchomienie o 2026-01-02 00:00 przetwarza dane z:
  data_interval_start = 2026-01-01 00:00  ← logical_date
  data_interval_end   = 2026-01-02 00:00
```

### Przykład – Airflow 2.10.5

```python
from airflow.decorators import dag, task
from pendulum import datetime

# Sposób 1: domyślny w 2.10 – string cron = DataInterval
@dag(
    dag_id="etl_data_interval",
    schedule="0 6 * * *",        # cron string → CronDataIntervalTimetable
    start_date=datetime(2026, 1, 1),
    catchup=False,
)
def etl_data_interval():
    @task
    def extract(**context):
        # logical_date = data_interval_start = wczoraj 06:00
        # data_interval_end = dziś 06:00
        start = context["data_interval_start"]
        end = context["data_interval_end"]
        print(f"Pobieram dane od {start} do {end}")

    extract()

etl_data_interval()


# Sposób 2: jawny timetable (identyczny efekt)
from airflow.timetables.interval import CronDataIntervalTimetable

@dag(
    dag_id="etl_data_interval_explicit",
    schedule=CronDataIntervalTimetable("0 6 * * *", timezone="UTC"),
    start_date=datetime(2026, 1, 1),
    catchup=False,
)
def etl_data_interval_explicit():
    @task
    def extract(**context):
        start = context["data_interval_start"]
        end = context["data_interval_end"]
        print(f"Pobieram dane od {start} do {end}")

    extract()

etl_data_interval_explicit()
```

### Zachowanie:
- DAG z `start_date=2026-01-01` i `@daily` uruchomi się **pierwszy raz 2026-01-02**
  (bo czeka aż interwał 01-01→01-02 się zakończy)
- `logical_date` = `data_interval_start` = 2026-01-01
- Po pauzie i odpauzowaniu → nadrabia wszystkie pominięte interwały (jeśli `catchup=True`)

---

## 2. CronTriggerTimetable (domyślne w Airflow 3)

Cron to po prostu **timer** – DAG odpala się **dokładnie w momencie** cron.

```
Cron: @daily (0 0 * * *)

Uruchomienie o 2026-01-01 00:00:
  data_interval_start = 2026-01-01 00:00
  data_interval_end   = 2026-01-01 00:00  ← RÓWNE!
  logical_date        = 2026-01-01 00:00
```

### Przykład – Airflow 2.10.5 (jawny import)

```python
from airflow.decorators import dag, task
from airflow.timetables.trigger import CronTriggerTimetable
from pendulum import datetime

@dag(
    dag_id="notification_trigger",
    schedule=CronTriggerTimetable("0 6 * * *", timezone="UTC"),
    start_date=datetime(2026, 1, 1),
    catchup=False,
)
def notification_trigger():
    @task
    def send_report(**context):
        # logical_date = data_interval_start = data_interval_end = czas triggera
        run_time = context["logical_date"]
        print(f"Wysyłam raport o {run_time}")

    send_report()

notification_trigger()
```

### Przykład – Airflow 3 (domyślne zachowanie)

```python
from airflow.decorators import dag, task
from pendulum import datetime

# W Airflow 3 cron string → CronTriggerTimetable automatycznie!
@dag(
    dag_id="notification_trigger_af3",
    schedule="0 6 * * *",        # w AF3 to jest CronTriggerTimetable
    start_date=datetime(2026, 1, 1),
    catchup=False,
)
def notification_trigger_af3():
    @task
    def send_report(**context):
        run_time = context["logical_date"]
        print(f"Wysyłam raport o {run_time}")

    send_report()

notification_trigger_af3()
```

### Zachowanie:
- DAG z `start_date=2026-01-01` i `0 6 * * *` uruchomi się **pierwszy raz 2026-01-01 06:00**
- `data_interval_start` = `data_interval_end` = czas odpalenia (brak notion interwału)
- Po pauzie → **nie nadrabia**, czeka na następny trigger

---

## 3. Porównanie na timeline

```
Schedule: @daily (0 0 * * *)

─── DataInterval ───────────────────────────────────────────────
Jan 1         Jan 2         Jan 3         Jan 4
|-------------|-------------|-------------|
  interval 1    interval 2    interval 3
              ^             ^             ^
              RUN 1         RUN 2         RUN 3
              logical=Jan1  logical=Jan2  logical=Jan3


─── Trigger ────────────────────────────────────────────────────
Jan 1         Jan 2         Jan 3         Jan 4
^             ^             ^             ^
RUN 1         RUN 2         RUN 3         RUN 4
logical=Jan1  logical=Jan2  logical=Jan3  logical=Jan4
```

---

## 4. Kiedy który wybrać?

### ✅ CronDataIntervalTimetable – ETL / przetwarzanie danych
- Przetwarzasz dane **za konkretny okres** ("wczorajsze zamówienia")
- Potrzebujesz `data_interval_start` i `data_interval_end` do filtrowania SQL
- Chcesz catchup / backfill pominiętych okresów

```python
# Typowy ETL
@task
def load_sales(**context):
    query = f"""
        SELECT * FROM sales
        WHERE created_at >= '{context["data_interval_start"]}'
          AND created_at <  '{context["data_interval_end"]}'
    """
```

### ✅ CronTriggerTimetable – joby operacyjne / powiadomienia
- Maintenance, monitoring, wysyłka raportów
- Nie przetwarzasz danych za konkretny okres
- Nie potrzebujesz backfill
- Chcesz intuicyjne zachowanie "odpali się o 6:00"

```python
# Monitoring / powiadomienia
@task
def check_system_health(**context):
    print(f"Health check at {context['logical_date']}")
```

---

## 5. Migracja z Airflow 2.10.5 → Airflow 3

### ⚠️ UWAGA: zmiana domyślnego zachowania crona!

Jeśli w Airflow 2.10.5 masz:
```python
schedule="0 6 * * *"   # → CronDataIntervalTimetable
```

W Airflow 3 ten sam string daje:
```python
schedule="0 6 * * *"   # → CronTriggerTimetable (!!!)
```

### Opcje migracji:

#### Opcja A: Zachowaj stare zachowanie (jawny timetable)
```python
# Airflow 2.10.5 - działa także w Airflow 3
from airflow.timetables.interval import CronDataIntervalTimetable

@dag(
    schedule=CronDataIntervalTimetable("0 6 * * *", timezone="UTC"),
    ...
)
```

#### Opcja B: Globalny config w Airflow 3
```ini
# airflow.cfg
[scheduler]
create_cron_data_intervals = True
```
To przywraca domyślną interpretację cron → CronDataIntervalTimetable.

#### Opcja C: Przejdź na trigger (zalecane dla jobów operacyjnych)
```python
# Airflow 2.10.5 – przygotuj się na AF3
from airflow.timetables.trigger import CronTriggerTimetable

@dag(
    schedule=CronTriggerTimetable("0 6 * * *", timezone="UTC"),
    ...
)
```

---

## 6. Nowe w Airflow 3

### logical_date może być None
```python
# Airflow 3 – równoległe uruchomienia tego samego DAGa
@dag(schedule=None)
def ml_inference():
    # Wiele runów może wystartować jednocześnie
    pass
```

### Assets (dawniej Datasets) + Watchers
```python
# Airflow 3 – event-driven scheduling
from airflow.sdk.definitions.asset import Asset

sales_data = Asset("s3://bucket/sales/")

@dag(schedule=[sales_data])   # Uruchom gdy dane się pojawią
def process_sales():
    pass
```

### DeltaTriggerTimetable / DeltaDataIntervalTimetable
```python
from datetime import timedelta
from airflow.timetables.trigger import DeltaTriggerTimetable

@dag(
    schedule=DeltaTriggerTimetable(timedelta(hours=2)),  # co 2h, trigger-style
    ...
)
```

---


## Import paths – ściągawka

```python
# Airflow 2.10.5
from airflow.timetables.trigger import CronTriggerTimetable
from airflow.timetables.trigger import DeltaTriggerTimetable

from airflow.timetables.interval import CronDataIntervalTimetable

# Airflow 3
from airflow.timetables.trigger import CronTriggerTimetable
from airflow.timetables.interval import CronDataIntervalTimetable
```
