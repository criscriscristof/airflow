# Jinja2 w Airflow 2.10.5 — kompletny przewodnik

## Jak to działa?

Airflow używa Jinja2 do **renderowania parametrów tasków w runtime**. Zamiast hardkodować wartości, wstawiasz template `{{ ... }}`, a Airflow podmienia je w momencie wykonywania taska — nie podczas parsowania DAG-a.

```python
BashOperator(
    task_id="print_date",
    bash_command="echo 'Przetwarzam dane za {{ ds }}'",
)
# W runtime: echo 'Przetwarzam dane za 2024-03-15'
```

> [!IMPORTANT]
> Nie każdy parametr operatora obsługuje templating! Tylko pola wymienione w `template_fields` danego operatora są renderowane. Np. w `BashOperator` templateable jest `bash_command` i `env`, ale nie `task_id`.

---

## Dostępne zmienne (Template Variables)

### Daty i interwały

| Zmienna                     | Typ                 | Opis                                          |
| --------------------------- | ------------------- | --------------------------------------------- |
| `{{ logical_date }}`        | `pendulum.DateTime` | Logiczna data runu (dawniej `execution_date`) |
| `{{ data_interval_start }}` | `pendulum.DateTime` | Początek interwału danych                     |
| `{{ data_interval_end }}`   | `pendulum.DateTime` | Koniec interwału danych                       |
| `{{ ds }}`                  | `str`               | `logical_date` jako `YYYY-MM-DD`              |
| `{{ ds_nodash }}`           | `str`               | `logical_date` jako `YYYYMMDD`                |
| `{{ ts }}`                  | `str`               | `logical_date` jako ISO timestamp             |
| `{{ ts_nodash }}`           | `str`               | Timestamp bez myślników i dwukropków          |

> [!NOTE]
> `{{ ds }}` to skrót — równoważny z `{{ logical_date | ds }}`. Oba dają `"2024-03-15"`.

### Informacje o DAG-u i tasku

| Zmienna                              | Opis                                             |
| ------------------------------------ | ------------------------------------------------ |
| `{{ dag }}`                          | Obiekt DAG                                       |
| `{{ dag.dag_id }}`                   | ID DAG-a                                         |
| `{{ task }}`                         | Obiekt bieżącego taska                           |
| `{{ task.task_id }}`                 | ID taska                                         |
| `{{ task_instance }}` lub `{{ ti }}` | Obiekt TaskInstance                              |
| `{{ run_id }}`                       | ID runu DAG-a                                    |
| `{{ dag_run }}`                      | Obiekt DagRun                                    |
| `{{ dag_run.conf }}`                 | Konfiguracja przekazana przy manualnym triggerze |

### Połączenia i zmienne

| Zmienna                       | Opis                                  |
| ----------------------------- | ------------------------------------- |
| `{{ conn.my_connection_id }}` | Obiekt Connection                     |
| `{{ var.value.my_variable }}` | Wartość Airflow Variable (string)     |
| `{{ var.json.my_variable }}`  | Airflow Variable sparsowany jako JSON |

### Poprzedni / następny run

| Zmienna                                  | Opis                                          |
| ---------------------------------------- | --------------------------------------------- |
| `{{ prev_data_interval_start_success }}` | Start interwału ostatniego UDANEGO runu       |
| `{{ prev_data_interval_end_success }}`   | Koniec interwału ostatniego UDANEGO runu      |
| `{{ prev_start_date_success }}`          | Faktyczny czas startu ostatniego UDANEGO runu |

---

## Filtry (Filters)

Filtry transformują wartości. Składnia: `{{ wartość | filtr }}`.

### Filtry Airflow (custom)

Wszystkie filtry Airflow przyjmują **datetime** i zwracają **string**:

| Filtr               | Wejście    | Zwraca               | Przykład wyniku               |
| ------------------- | ---------- | -------------------- | ----------------------------- |
| `ds`                | `datetime` | `str` (`YYYY-MM-DD`) | `"2024-03-15"`                |
| `ds_nodash`         | `datetime` | `str` (`YYYYMMDD`)   | `"20240315"`                  |
| `ts`                | `datetime` | `str` (ISO 8601)     | `"2024-03-15T06:00:00+00:00"` |
| `ts_nodash`         | `datetime` | `str`                | `"20240315T060000"`           |
| `ts_nodash_with_tz` | `datetime` | `str`                | `"20240315T060000+0000"`      |

### Filtry wbudowane Jinja2 (przydatne)

| Filtr              | Opis                     | Przykład                                          |
| ------------------ | ------------------------ | ------------------------------------------------- |
| `int`              | Konwersja na int         | `{{ "42" \| int }}` → `42`                        |
| `default(val)`     | Wartość domyślna         | `{{ var \| default("brak") }}`                    |
| `replace(a, b)`    | Zamiana tekstu           | `{{ ds \| replace("-", "/") }}` → `"2024/03/15"` |
| `upper` / `lower`  | Zmiana wielkości liter   | `{{ "abc" \| upper }}` → `"ABC"`                  |
| `trim`             | Usunięcie białych znaków | `{{ " abc " \| trim }}`                           |
| `tojson`           | Serializacja do JSON     | `{{ dict_var \| tojson }}`                         |

### Łączenie filtrów (chaining)

```python
# Można łączyć wiele filtrów:
"{{ data_interval_start | ds | replace('-', '') }}"
# → "20240315"

# choć w tym przypadku prościej:
"{{ data_interval_start | ds_nodash }}"
```

---

## Makra (Macros)

Makra to funkcje i moduły dostępne w szablonach przez `{{ macros.* }}`.

### Wbudowane makra

```python
# timedelta — przesunięcie dat
"{{ (logical_date - macros.timedelta(days=1)) | ds }}"
# → "2024-03-14" (wczoraj)

"{{ (logical_date + macros.timedelta(hours=6)) | ts }}"

# datetime
"{{ macros.datetime(2024, 1, 1) }}"

# random (losowa wartość)
"{{ macros.random() }}"

# uuid
"{{ macros.uuid.uuid4() }}"
```

### Dostępne moduły w macros

| Moduł              | Opis                 |
| ------------------ | -------------------- |
| `macros.datetime`  | `datetime.datetime`  |
| `macros.timedelta` | `datetime.timedelta` |
| `macros.dateutil`  | Moduł `dateutil`     |
| `macros.time`      | Moduł `time`         |
| `macros.uuid`      | Moduł `uuid`         |
| `macros.random`    | `random.random`      |

---

## Praktyczne przykłady

### SQL z interwałem dat

```python
SqlOperator(
    task_id="load_data",
    sql="""
        SELECT * FROM orders
        WHERE order_date >= '{{ data_interval_start | ds }}'
          AND order_date <  '{{ data_interval_end | ds }}'
    """,
)
```

### Dynamiczna ścieżka pliku

```python
BashOperator(
    task_id="export",
    bash_command="python export.py --date={{ ds }} --output=/data/export_{{ ds_nodash }}.csv",
)
```

### Użycie Airflow Variables

```python
BashOperator(
    task_id="call_api",
    bash_command='curl -H "Authorization: Bearer {{ var.value.api_token }}" {{ var.value.api_url }}/data',
)
```

### Konfiguracja z manualnego triggera (dag_run.conf)

```python
# Trigger DAG z conf: {"target_date": "2024-03-01", "mode": "full"}

BashOperator(
    task_id="process",
    bash_command="""
        python process.py \
            --date={{ dag_run.conf.get("target_date", ds) }} \
            --mode={{ dag_run.conf.get("mode", "incremental") }}
    """,
)
```

### Logika warunkowa w szablonie

```python
BashOperator(
    task_id="conditional",
    bash_command="""
        {% if dag_run.conf and dag_run.conf.get("full_reload") %}
            python full_reload.py
        {% else %}
            python incremental.py --since='{{ data_interval_start | ds }}'
        {% endif %}
    """,
)
```

### Pętla w szablonie

```python
BashOperator(
    task_id="multi_table",
    bash_command="""
        {% for table in ["users", "orders", "products"] %}
            echo "Processing {{ table }} for {{ ds }}"
            python etl.py --table={{ table }} --date={{ ds }}
        {% endfor %}
    """,
)
```

---

## Które pola są templateable?

Każdy operator deklaruje `template_fields` — tylko te pola są renderowane.

```python
# Sprawdzenie w kodzie:
print(BashOperator.template_fields)
# → ('bash_command', 'env', 'cwd')

print(PythonOperator.template_fields)
# → ('templates_dict', 'op_args', 'op_kwargs')
```

> [!WARNING]
> **PythonOperator**: Argumenty przekazywane przez `op_kwargs` SĄ templateable, ale sam `python_callable` NIE. Jeśli chcesz użyć templateowanych wartości w Pythonie, przekaż je przez `op_kwargs`:
> ```python
> def my_func(date_str, **kwargs):
>     print(f"Processing {date_str}")
> 
> PythonOperator(
>     task_id="process",
>     python_callable=my_func,
>     op_kwargs={"date_str": "{{ ds }}"},
> )
> ```

### Renderowanie plików SQL

Jeśli operator ma `template_ext`, Airflow automatycznie renderuje pliki z tymi rozszerzeniami:

```python
# Zamiast inline SQL:
SqlOperator(
    task_id="load",
    sql="queries/load_orders.sql",  # Airflow znajdzie plik i wyrenderuje Jinja2 w nim
)
```

```sql
-- queries/load_orders.sql
SELECT * FROM orders
WHERE order_date >= '{{ data_interval_start | ds }}'
  AND order_date <  '{{ data_interval_end | ds }}'
```

Domyślnie Airflow szuka plików SQL w folderze DAG-a. Można zmienić `template_searchpath` na poziomie DAG-a:

```python
DAG(
    dag_id="my_dag",
    template_searchpath="/opt/airflow/sql",
)
```

---

## Debugowanie szablonów

### W UI Airflow

**Task Instance → Rendered Template** — pokazuje wyrenderowany szablon z podstawionymi wartościami.

### W CLI

```bash
airflow tasks render my_dag my_task 2024-03-15
```

### W kodzie (testowo)

```python
from airflow.models import DagBag

dagbag = DagBag()
dag = dagbag.get_dag("my_dag")
task = dag.get_task("my_task")
ti = task.render_template(task.bash_command, context)
```

---

## Częste pułapki

> [!CAUTION]
> **1. Templating działa tylko w runtime, nie podczas parsowania DAG-a**
> ```python
> # ❌ TO NIE ZADZIAŁA — wartość jest ewaluowana przy parsowaniu:
> if "{{ ds }}" > "2024-01-01":
>     do_something()
> 
> # ✅ Użyj BranchPythonOperator lub logiki w szablonie
> ```

> [!CAUTION]
> **2. Cudzysłowy w SQL**
> ```sql
> -- ❌ Podwójne cudzysłowy — ds już jest stringiem:
> WHERE date = "'{{ ds }}'"
> 
> -- ✅ Poprawnie:
> WHERE date = '{{ ds }}'
> ```

> [!WARNING]
> **3. Brak renderowania w polach spoza template_fields**
> ```python
> # ❌ task_id NIE jest renderowane:
> BashOperator(task_id="task_{{ ds }}", ...)  # zostanie literalnie "task_{{ ds }}"
> ```

---

## Zaawansowane: user_defined_macros

Możesz dodać własne zmienne i funkcje dostępne w szablonach Jinja2 na poziomie DAG-a:

```python
def format_quarter(dt):
    """Zwraca kwartał w formacie '2024-Q1'"""
    return f"{dt.year}-Q{(dt.month - 1) // 3 + 1}"

with DAG(
    dag_id="my_dag",
    schedule="0 6 * * *",
    user_defined_macros={
        "project": "analytics",
        "env": "prod",
        "quarter": format_quarter,
    },
):
    BashOperator(
        task_id="run",
        bash_command="""
            echo "Projekt: {{ project }}"
            echo "Środowisko: {{ env }}"
            echo "Kwartał: {{ quarter(logical_date) }}"
        """,
    )
# W runtime:
# Projekt: analytics
# Środowisko: prod
# Kwartał: 2024-Q1
```

---

## Zaawansowane: user_defined_filters

Analogicznie możesz dodać własne **filtry** Jinja2:

```python
def to_oracle_date(dt):
    """Konwertuje datetime na format Oracle TO_DATE"""
    return dt.strftime("%d-%b-%Y").upper()

with DAG(
    dag_id="oracle_dag",
    schedule="0 6 * * *",
    user_defined_filters={
        "oracle_date": to_oracle_date,
    },
):
    SqlOperator(
        task_id="query",
        sql="""
            SELECT * FROM orders
            WHERE order_date >= TO_DATE('{{ data_interval_start | oracle_date }}', 'DD-MON-YYYY')
        """,
    )
# W runtime: TO_DATE('15-MAR-2024', 'DD-MON-YYYY')
```

---

## Params — parametry z walidacją

`params` to parametry DAG-a/taska z opcjonalną walidacją, dostępne w szablonach przez `{{ params.nazwa }}`:

```python
from airflow.models.param import Param

with DAG(
    dag_id="parametrized_dag",
    schedule=None,
    params={
        "target_table": Param("orders", type="string", description="Tabela docelowa"),
        "limit": Param(1000, type="integer", minimum=1, maximum=100000),
        "full_reload": Param(False, type="boolean"),
    },
):
    BashOperator(
        task_id="load",
        bash_command="""
            python etl.py \
                --table={{ params.target_table }} \
                --limit={{ params.limit }} \
                {% if params.full_reload %}--full-reload{% endif %}
        """,
    )
```

> [!TIP]
> Gdy triggerujesz DAG manualnie z UI, Airflow **automatycznie wygeneruje formularz** na podstawie `params` — z polami tekstowymi, checkboxami itp. Typy `Param` (`string`, `integer`, `boolean`, `array`) mapują się na odpowiednie kontrolki.

### Różnica: `params` vs `dag_run.conf`

| Cecha               | `params`                            | `dag_run.conf`             |
| ------------------- | ----------------------------------- | -------------------------- |
| Walidacja typów     | ✅ Tak (JSON Schema)                 | ❌ Nie                      |
| Formularz w UI      | ✅ Automatyczny                      | ❌ Tylko surowy JSON        |
| Wartości domyślne   | ✅ Wbudowane                         | Trzeba obsługiwać ręcznie  |
| Dostęp w szablonie  | `{{ params.nazwa }}`                | `{{ dag_run.conf.nazwa }}` |
| Override przez conf | ✅ `dag_run.conf` nadpisuje `params` | —                          |

---

## XCom w szablonach

Możesz odczytywać wartości przekazane między taskami (XCom) bezpośrednio w szablonach:

```python
# Task 1: pushuje wartość do XCom
def get_row_count(**kwargs):
    count = run_query("SELECT COUNT(*) FROM orders")
    return count  # automatycznie pushowane jako XCom (return_value)

count_task = PythonOperator(
    task_id="get_count",
    python_callable=get_row_count,
)

# Task 2: pulluje wartość z XCom w szablonie
report_task = BashOperator(
    task_id="report",
    bash_command='echo "Przetworzono {{ ti.xcom_pull(task_ids="get_count") }} wierszy"',
)

count_task >> report_task
```

### Skrótowa notacja XCom (Airflow 2.x)

```python
# Zamiast ti.xcom_pull można użyć:
"{{ task_instance.xcom_pull(task_ids='get_count', key='return_value') }}"

# Lub z wieloma taskami:
"{{ ti.xcom_pull(task_ids=['task_a', 'task_b']) }}"
# → zwraca listę wartości
```

> [!WARNING]
> XCom jest serializowany (domyślnie jako JSON/pickle). Duże obiekty (DataFrames, pliki) **nie powinny** przechodzić przez XCom — przekazuj ścieżki plików lub klucze.

---

## Kontekst w PythonOperator

W `PythonOperator` masz dwa sposoby dostępu do zmiennych szablonowych:

### Sposób 1: przez `**kwargs` (zalecany)

```python
def process_data(**kwargs):
    logical_date = kwargs["logical_date"]      # pendulum.DateTime
    ds = kwargs["ds"]                            # str "2024-03-15"
    ti = kwargs["ti"]                            # TaskInstance
    dag_run = kwargs["dag_run"]                  # DagRun
    params = kwargs["params"]                    # dict z params
    
    start = kwargs["data_interval_start"]
    end = kwargs["data_interval_end"]
    
    print(f"Przetwarzam dane od {start} do {end}")

PythonOperator(
    task_id="process",
    python_callable=process_data,
)
```

### Sposób 2: przez `get_current_context()` (od Airflow 2.x)

```python
from airflow.operators.python import get_current_context

def process_data():
    context = get_current_context()
    ds = context["ds"]
    logical_date = context["logical_date"]
    ti = context["ti"]

PythonOperator(
    task_id="process",
    python_callable=process_data,
)
```

> [!TIP]
> `get_current_context()` jest przydatne gdy wywołujesz głęboko zagnieżdżone funkcje i nie chcesz przekazywać `**kwargs` przez wiele warstw.

### Sposób 3: Jinja2 przez `op_kwargs`

```python
def process_data(date_str, interval_start, interval_end):
    print(f"Data: {date_str}, okres: {interval_start} - {interval_end}")

PythonOperator(
    task_id="process",
    python_callable=process_data,
    op_kwargs={
        "date_str": "{{ ds }}",
        "interval_start": "{{ data_interval_start | ds }}",
        "interval_end": "{{ data_interval_end | ds }}",
    },
)
```

> [!NOTE]
> Różnica: `**kwargs` daje obiekt `pendulum.DateTime`, a `op_kwargs` z Jinja2 daje **string**. Wybieraj w zależności od tego, czego potrzebujesz.

---

## Pełna lista zmiennych kontekstowych

Dla referencji — wszystkie zmienne dostępne w szablonach i w `**kwargs`:

| Zmienna                            | Typ                          | Opis                             |
| ---------------------------------- | ---------------------------- | -------------------------------- |
| `dag`                              | `DAG`                        | Obiekt DAG                       |
| `dag_run`                          | `DagRun`                     | Bieżący run                      |
| `task`                             | `BaseOperator`               | Bieżący task                     |
| `task_instance` / `ti`             | `TaskInstance`               | Instancja taska                  |
| `logical_date`                     | `pendulum.DateTime`          | Logiczna data                    |
| `data_interval_start`              | `pendulum.DateTime`          | Początek interwału               |
| `data_interval_end`                | `pendulum.DateTime`          | Koniec interwału                 |
| `ds`                               | `str`                        | `YYYY-MM-DD`                     |
| `ds_nodash`                        | `str`                        | `YYYYMMDD`                       |
| `ts`                               | `str`                        | ISO timestamp                    |
| `ts_nodash`                        | `str`                        | Timestamp bez separatorów        |
| `run_id`                           | `str`                        | ID runu                          |
| `params`                           | `dict`                       | Parametry DAG-a/taska            |
| `var.value`                        | accessor                     | Airflow Variables (str)          |
| `var.json`                         | accessor                     | Airflow Variables (JSON)         |
| `conn`                             | accessor                     | Airflow Connections              |
| `macros`                           | module                       | Moduł z makrami                  |
| `prev_data_interval_start_success` | `pendulum.DateTime` / `None` | Interwał poprzedniego sukcesu    |
| `prev_data_interval_end_success`   | `pendulum.DateTime` / `None` | Interwał poprzedniego sukcesu    |
| `prev_start_date_success`          | `pendulum.DateTime` / `None` | Czas startu poprzedniego sukcesu |
| `inlets`                           | `list`                       | Data lineage — wejścia           |
| `outlets`                          | `list`                       | Data lineage — wyjścia           |
