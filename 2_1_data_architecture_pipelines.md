# 2.1 Data Architecture & Pipelines

Protocol of session 2.1, module 2 (Data Engineering). The session project is `mle-data-pipeline-and-modeling`: a local PostgreSQL database in Docker, an ETL pipeline for the NYC Yellow Taxi data, and a star schema built on top of it.

**Date:** DD.MM.YYYY

---

## Basic Overview

* Started PostgreSQL 17 in Docker on a private network and connected with `psql`
* Built an ETL pipeline: download Parquet, inspect with pandas, load into PostgreSQL in chunks, validate
* Packaged the ETL as a Click CLI (`data_ingestion.py`) and ran it from a Docker image
* Practiced SQL on the raw `yellow_taxi` table
* Compared normalized and dimensional data models
* Built a star schema (`fact_trip` plus five dimensions) and validated it
* Debugged two real problems: a SQLAlchemy driver change and a full disk

---

## Schedule

|Time|Content|
|---|---|
|09:00 - 09:45|Check in + Daily Review|
|09:45 - 12:45|Data Architecture & Pipelines|
|12:45 - 13:30|Q&A|
|13:30 - 14:00|Lunch Break|
|14:00 - 14:45|Check in + Active Learning & Guided work|
|14:45 - 17:45|Active Learning & Guided work|
|17:45 - 18:30|Journaling + Guided work|

---

## Content Notes

**Goal of the session: move raw data into a database reliably (ETL), then reshape it so analysts can query it easily (data modeling).**

```mermaid
flowchart LR
    A["Parquet file<br>(extract)"] --> B["pandas / pyarrow<br>(transform)"]
    B --> C["PostgreSQL<br>yellow_taxi (load)"]
    C --> D["Star schema<br>fact_trip + dim_*"]
    D --> E["Analytical SQL"]
```

The project folder has three lesson units:

```text
mle-data-pipeline-and-modeling/
├── 01-setup-your-db.md        # PostgreSQL in Docker
├── 02-load-data.ipynb         # ETL notebook + SQL practice
├── 03-data-modeling.ipynb     # star schema
├── data_ingestion.py          # CLI version of the ETL
├── Dockerfile                 # image that runs the CLI
├── pyproject.toml / uv.lock   # dependencies
└── db-data/                   # local PostgreSQL files (git-ignored)
```

---

### **01 Setup Your Database**

```bash
docker network create ny-taxi
mkdir -p db-data
docker run -d --name ny-taxi-db --network ny-taxi \
  -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=ny_taxi \
  -v "$(pwd)/db-data:/var/lib/postgresql/data" \
  -p 5432:5432 postgres:17
docker exec -it ny-taxi-db psql -U postgres -d ny_taxi
```

* The **network** lets other containers reach the database by name (`ny-taxi-db:5432`) instead of by IP.
* The **volume** (`db-data`) keeps the data when the container is removed.
* `-p 5432:5432` publishes the port so notebooks on the laptop use `localhost:5432`.

| Client runs... | Host to use |
|---|---|
| on your laptop (Jupyter, DBeaver) | `localhost` |
| in another container on `ny-taxi` | `ny-taxi-db` |

Step is done when the container is running, port 5432 is reachable, and `psql` connects to `ny_taxi`.

---

### **02 Load Data (ETL)**

| Phase | What happens here |
|---|---|
| **Extract** | Download `yellow_tripdata_2025-01.parquet` (skipped if the file already exists, so reruns are safe) |
| **Transform** | Inspect with pandas; infer a table schema with `psql.get_schema` |
| **Load** | Create an empty table, then append the file in 100,000-row batches |
| **Validate** | Compare the PostgreSQL row count with the Parquet metadata |

Why batches? The file has 3,475,226 rows. Streaming with `ParquetFile.iter_batches` keeps memory use small. `COPY` would be faster, but the batch approach is easier to read.

**CLI and Docker.** `data_ingestion.py` repeats the same logic with Click options (`--user`, `--password`, `--host`, `--port`, `--db`, `--table_name`, `--url`, `--file_path`).

```bash
docker build -t data_ingestion .
docker run --rm --network=ny-taxi data_ingestion \
  --host ny-taxi-db --url <parquet-url> --file_path /tmp/yellow_tripdata_2025-01.parquet
```

Inside the container `localhost` means the container itself, so the host is `ny-taxi-db`.

**Result:** 35 batches (34 x 100,000 rows plus 75,226), 3,475,226 rows in `yellow_taxi`, equal to the Parquet row count.

**SQL practice on the raw table:**

| Question | Result |
|---|---|
| Trips that started and ended on 2025-01-15 | 124,772 |
| Largest single trip | 2025-01-22, 276,423.57 miles |
| 2025-01-01 trips with 2 / 3 passengers | 15,166 / 4,721 |
| Pickup location with the most tips on 2025-01-01 | `PULocationID` 132 (44,091.46) |

Distances above 200,000 miles are bad records in the source data. A successful load does not mean the data is clean, so analysts still need to check quality.

---

### **03 Data Modeling: Star Schema**

**Data models, from abstract to concrete:** conceptual (business ideas), logical (entities, attributes, relations), physical (tables, keys, types).

| | Normalized | Dimensional |
|---|---|---|
| Goal | Avoid redundancy, safe updates | Easy, fast analytical queries |
| Typical use | Operational systems | Reporting and analytics |
| Cost | Many joins | Some repetition |

**Process:** pick the business event, **choose the grain**, separate context into dimensions, define the physical schema, load dimensions first and then the fact table, validate.

**Grain: one row per taxi trip.** Deciding this first keeps the rest of the model clear.

```mermaid
erDiagram
    DIM_DATE ||--o{ FACT_TRIP : "pickup_date_key"
    DIM_DATE ||--o{ FACT_TRIP : "dropoff_date_key"
    DIM_VENDOR ||--o{ FACT_TRIP : "vendor_key"
    DIM_RATE_CODE ||--o{ FACT_TRIP : "rate_code_key"
    DIM_PAYMENT_TYPE ||--o{ FACT_TRIP : "payment_type_key"
    DIM_LOCATION ||--o{ FACT_TRIP : "pickup_location_key"
    DIM_LOCATION ||--o{ FACT_TRIP : "dropoff_location_key"
```

* A cleaned **staging view** `yellow_taxi_clean` renames mixed-case columns (`"VendorID"`, `"PULocationID"`) to snake_case and adds `pickup_date`, `dropoff_date` and `trip_duration_minutes`.
* Tables created: `dim_date`, `dim_location`, `dim_payment_type`, `dim_rate_code`, `dim_vendor`, `fact_trip`.
* **Validation:** `fact_trip` has 3,475,226 rows, the same as `yellow_taxi`, so no trip was lost or duplicated.

---

### **Problems we ran into**

**1. `ModuleNotFoundError: No module named 'psycopg'`**

* Cause: SQLAlchemy 2.1.6 now uses the `psycopg` (v3) driver for a plain `postgresql://` URL, but the project installs `psycopg2-binary`.
* Fix: name the driver in the URL: `postgresql+psycopg2://postgres:postgres@localhost:5432/ny_taxi`. Changed in `data_ingestion.py`, both notebooks and `01-setup-your-db.md`.

**2. `psycopg2.errors.DiskFull: No space left on device`**

* Cause: the profiling query (several `COUNT(DISTINCT ...)` over 3.5M rows) wrote temporary files to `db-data/` on a Mac with only about 500 MB free.
* Fix: free disk space (for example unused Docker volumes and images), then rerun the cell.
* Lesson: an error message from the database can point to the host machine, not to the SQL.

---

### **Key Takeaways**

| Idea | In one sentence |
|---|---|
| Docker network | Containers reach each other by name; the laptop uses `localhost`. |
| Volume | Keeps database files after the container is removed. |
| ETL | Extract, transform, load, then check the result. |
| Chunked loading | Handle big files in batches to save memory. |
| Validation | Compare row counts between source and target. |
| Grain | Decide what one fact row means before building tables. |
| Star schema | One fact table with dimension tables around it makes analytics easier to read. |
| Pin the driver | Use `postgresql+psycopg2://` so a library default change does not break the code. |

---

## Helpful References

* [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)
* [PostgreSQL Docker image](https://hub.docker.com/_/postgres)
* [SQLAlchemy: PostgreSQL dialects](https://docs.sqlalchemy.org/en/20/dialects/postgresql.html)
* [pandas `to_sql`](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_sql.html)
* [PyArrow `ParquetFile.iter_batches`](https://arrow.apache.org/docs/python/generated/pyarrow.parquet.ParquetFile.html)
* [Click documentation](https://click.palletsprojects.com/en/stable/)
* [Star schema (GeeksforGeeks)](https://www.geeksforgeeks.org/star-schema-in-data-warehouse-modeling/)
