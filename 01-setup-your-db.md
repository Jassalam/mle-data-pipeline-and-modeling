# Setup Your Database

This step starts a local PostgreSQL database in Docker for the rest of the project. After it is running:

- notebooks on your machine will connect to `localhost:5432`,
- other containers on the same Docker network will connect to `ny-taxi-db:5432`,
- the database name will be `ny_taxi`.

```mermaid
flowchart LR
    A["Your laptop / Jupyter / DBeaver"] -->|"localhost:5432"| B["PostgreSQL container<br/>ny-taxi-db"]
    C["Future ingestion container"] -->|"ny-taxi-db:5432"| B
    C --> D["Docker network<br/>ny-taxi"]
    B --> D
```

The same database is reachable in two different ways depending on where the client is running: `localhost` from your machine, and `ny-taxi-db` from another container on the `ny-taxi` network.

We will use the official `postgres:17` image directly, because this project does not need a custom PostgreSQL image for the setup step.

## Step 0: Start Docker

Make sure Docker Desktop is running before you execute any Docker commands.

You can confirm that Docker is available with:

```bash
docker --version
docker info
```

If `docker info` fails, Docker is installed but the Docker daemon is not running yet.

## Step 1: Create a Docker network

```bash
docker network create ny-taxi
```

This creates a private Docker network named `ny-taxi`. We use it so later containers, such as the ingestion container in the next exercise, can talk to the database by container name instead of by IP address.

If Docker prints `network with name ny-taxi already exists`, that is fine. It just means you created it earlier.

Success looks like this:

- Docker returns a network ID, or
- Docker tells you the network already exists.

Either outcome means you can move on.

## Step 2: Create a local data directory

```bash
mkdir -p db-data
```

This folder stores the PostgreSQL data files on your machine. By mounting it into the container, your database contents survive container restarts.

Success looks like this:

- the `db-data/` folder exists in the repository root,
- rerunning the command does nothing harmful.

## Step 3: Start PostgreSQL

```bash
docker run -d \
  --name ny-taxi-db \
  --network ny-taxi \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=ny_taxi \
  -v "$(pwd)/db-data:/var/lib/postgresql/data" \
  -p 5432:5432 \
  postgres:17
```

What each part does:

- `-d` runs the container in the background.
- `--name ny-taxi-db` gives the container a stable name other Docker containers can use.
- `--network ny-taxi` attaches the container to the network you created in step 1.
- `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` create the initial login and database.
- `-v "$(pwd)/db-data:/var/lib/postgresql/data"` stores database files in the local `db-data` folder.
- `-p 5432:5432` publishes PostgreSQL to your machine so notebooks can connect through `localhost:5432`.
- `postgres:17` is the official PostgreSQL image.

Use `localhost` when you connect from your laptop, Jupyter, or DBeaver. Use `ny-taxi-db` when you connect from another container on the `ny-taxi` network.

Success looks like this:

- Docker prints a container ID,
- `docker ps` shows a container named `ny-taxi-db`,
- the container keeps running instead of exiting immediately.

## Step 4: Verify that the container is running

Check that the container started:

```bash
docker ps --filter "name=ny-taxi-db"
```

If you do not see the container yet, inspect the logs:

```bash
docker logs ny-taxi-db
```

PostgreSQL is ready when the logs show `database system is ready to accept connections`.

Success looks like this:

- `docker ps --filter "name=ny-taxi-db"` returns one running container,
- the logs mention that PostgreSQL is ready to accept connections.

## Step 5: Connect with `psql`

```bash
docker exec -it ny-taxi-db psql -U postgres -d ny_taxi
```

This opens the `psql` command-line client inside the running container and connects to the `ny_taxi` database.

### Useful `psql` commands

| Command | Description |
| --- | --- |
| `\l` | List all databases |
| `\c ny_taxi` | Reconnect to the `ny_taxi` database if needed |
| `\dt` | List tables in the current database |
| `\q` | Exit client |

At this point, `\dt` may be empty, which is expected. The tables are created later when you load data.

Success looks like this:

- your terminal prompt changes to `ny_taxi=#` or similar,
- `\l` lists databases,
- `\dt` runs without an error, even if it shows no tables yet.

## Connection details for the next steps

These are the values used throughout the rest of the project:

- Host from your machine: `localhost`
- Host from another container: `ny-taxi-db`
- Port: `5432`
- Database: `ny_taxi`
- User: `postgres`
- Password: `postgres`

The notebook in the next step uses this connection string:

```text
postgresql://postgres:postgres@localhost:5432/ny_taxi
```

## Troubleshooting

### Docker commands fail immediately

If `docker info` returns an error about the Docker daemon or socket, Docker Desktop is not running yet. Start Docker Desktop and try again.

### Port `5432` is already in use

Another PostgreSQL instance may already be using that port on your machine. Find and stop the conflicting service, or remove the existing Docker container that is using it.

### The container name already exists

If you see an error about `ny-taxi-db` already existing, remove the old container and start again:

```bash
docker rm -f ny-taxi-db
```

### You want to start over completely

To remove the container but keep your saved database files:

```bash
docker rm -f ny-taxi-db
```

To remove both the container and the saved database files:

```bash
docker rm -f ny-taxi-db
rm -rf db-data
```
