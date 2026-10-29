# Data Pipeline and Data Modeling

This repository is a small hands-on course that walks you through building a data pipeline and a simple analytics model on top of it. You will spin up a local PostgreSQL database in Docker, load the January 2025 [NYC Yellow Taxi dataset](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) into it with a Python ETL script, and then remodel the raw data into a star schema for analytical queries.

## Learning Objectives

By the end of this repository, you should be able to:

- Start and verify a local PostgreSQL database with Docker.
- Load Parquet data into PostgreSQL with a small Python ETL pipeline.
- Validate raw data loads with SQL and pandas.
- Explain the difference between normalized and dimensional data models.
- Build and query a simple star schema for analytics.

## Learning Path

| File / Folder | Description |
|---|---|
| [**01 - Setup Your Database**](01-setup-your-db.md) | Start PostgreSQL in Docker and verify you can connect to it. |
| [**02 - Load Data**](02-load-data.ipynb) | Download the Yellow Taxi Parquet file and load it into PostgreSQL. |
| [**03 - Data Modeling**](03-data-modeling.ipynb) | Remodel the raw table into a star schema and query it. |

### Additional Folders and Files

| File / Folder | Description |
|---|---|
| [**data**](data/) | Local storage for the downloaded Parquet file. |
| [**assets**](assets/) | Visual aids referenced in the repository. |
| [**data_ingestion.py**](data_ingestion.py) | CLI script and Dockerized entry point for loading Parquet data into PostgreSQL. |
| [**Dockerfile**](Dockerfile) | Container image for running the ingestion script. |
| [**pyproject.toml**](pyproject.toml) | Project configuration and dependencies. |
| [**uv.lock**](uv.lock) | Dependency lock file. |

## Prerequisites

- **Docker Desktop**: required to run the local PostgreSQL database used throughout this course. Follow the [installation instructions](https://docs.docker.com/get-docker/) if you do not have it yet. Make sure it is **installed and running** before you start with the exercises.
- **DBeaver** (optional): a GUI client for exploring the PostgreSQL database. Follow the [installation instructions](https://dbeaver.io/download/), or install it with the commands below.

  **`macOS`**

    ```bash
    brew install --cask dbeaver-community
    ```

  **`Windows`**

    ```powershell
    choco install dbeaver
    ```

## Setup

Use [uv](https://docs.astral.sh/uv/getting-started/installation/) 0.13.5 or newer
and Python 3.13.17 or a newer 3.13 patch release, as defined in
[`.python-version`](.python-version).

Rebuild existing Docker images with `--pull --no-cache` after this update
to refresh their system certificates and dependencies.

> [!NOTE]
> Throughout these steps, text in angle brackets like `<repo-name>` is a **placeholder**. Replace it, including the `< >` brackets, with your own value. For example, `cd <repo-name>` becomes `cd mle-data-pipeline-and-modeling`.

### 1. Use This Repository as a Template

Click **Use this template** on GitHub.

When creating the repository:

- Set yourself as the **Owner**
- Choose a repository name
- Disable **Include all branches**
- Click **Create repository**

> [!IMPORTANT]
> If you are working in pairs or groups, only **one person** should complete this step.

---

### 2. Add Collaborators (Pairs/Groups Only)

If working with teammates:

1. Open the repository on GitHub
2. Go to **Settings → Collaborators**
3. Add your teammates as collaborators
4. Share the repository link with your team

Teammates should accept the invitation before continuing.

---

### 3. Clone the Repository

Copy the SSH URL from the **Code** button on GitHub, then run:

```bash
git clone <copied-ssh-url>
```

The copied SSH URL will look like `git@github.com:<your-username>/<repo-name>.git`.

---

### 4. Move into the Project Folder and Install Dependencies

This installs all dependencies and creates a virtual environment in `.venv/`.

```bash
cd <repo-name>
uv sync
```

---

### 5. Open the Repository in VS Code

> [!NOTE]
> Make sure you open VS Code from the project root so it automatically detects the environment created by `uv sync`.

Launch VS Code in the project root folder:

```bash
code .
```

Then open a notebook and select the Python environment created by `uv sync` as the kernel.

## Walkthrough

### Step 1: Setup the database

In [`01-setup-your-db.md`](./01-setup-your-db.md), you will start PostgreSQL in Docker and verify that you can connect to it.

You are done with step 1 when:

- the `ny-taxi-db` container is running,
- PostgreSQL is reachable on `localhost:5432`,
- you can connect to the `ny_taxi` database with `psql`.

### Step 2: Build the ETL pipeline

In [`02-load-data.ipynb`](./02-load-data.ipynb), you will download the Yellow Taxi Parquet file, inspect it with pandas, and load it into PostgreSQL in chunks.

You are done with step 2 when:

- the `yellow_taxi` table exists in `ny_taxi`,
- the notebook validation confirms the PostgreSQL row count matches the Parquet row count,
- the CLI and Dockerized ingestion flow both make sense from the notebook walkthrough.

### Step 3: Model the data for analytics

In [`03-data-modeling.ipynb`](./03-data-modeling.ipynb), you will turn the raw `yellow_taxi` table into a small star schema for analytical queries.

You are done with step 3 when:

- the dimension tables and `fact_trip` exist in PostgreSQL,
- `fact_trip` has the same number of rows as `yellow_taxi`,
- you can answer the notebook exercises with the modeled tables.
