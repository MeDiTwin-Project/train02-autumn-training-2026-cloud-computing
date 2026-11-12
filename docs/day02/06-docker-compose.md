# 6. Microservices with Docker Compose

Modern computational systems are distributed multi-tier applications comprising distinct services:
web application servers, relational databases, cache brokers, and reverse proxies.

Managing multi-container topologies using imperative `docker run` commands introduces configuration
drift and operational complexity. **Docker Compose** provides a declarative, version-controlled YAML
specification (**Compose File Specification**) to model, network, and orchestrate multi-container
application stacks on a single host.

**Symbols:** ⌨️ you type the commands · 📖 reference material: read it after the lesson, or when you need it. The lesson does not cover it in class.

---

## 6.1 The whole stack in one file

With Docker Compose, infrastructure requirements are codified within `compose.yaml`:

```{mermaid}
flowchart TB
    Client([External Client / Ingestion Gateway]) -->|Port 8000:8000| API["Application Service (Python)<br/>Service Name: 'api'"]

    subgraph StackNet["Custom Stack Bridge Network: climate-stack_default (Internal DNS)"]
        direction TB
        API -->|tcp://db:5432| DB[("Relational Database (PostgreSQL 16)<br/>Service Name: 'db'")]
        API -->|tcp://cache:6379| Cache[("In-Memory Cache (Redis 7)<br/>Service Name: 'cache'")]
    end

    subgraph Storage["Persistent Host Storage"]
        Vol[("Managed Volume: postgres_data")]
    end

    DB <===>|Mounts /var/lib/postgresql/data| Vol
```

Executing `docker compose up -d` directs the runtime to:
1. Initialize a dedicated custom bridge network for the stack.
2. Instantiate defined persistent volumes.
3. Build custom container images specified in build contexts.
4. Launch container instances according to declared dependency graphs and health checks.
5. Register service names within Docker's embedded DNS server (`127.0.0.11`).

:::{note} Jargon: *project*
Compose calls the stack a **project**. Its name comes from `name:` in `compose.yaml` (or from the directory name), and Compose puts it in front of everything it creates: the network `climate-stack_default`, the volume `climate-stack_postgres_data`, the container `climate-stack-db-1`.
:::

---

## 6.2 ⌨️ The MeDiTwin climate stack: API, database and cache

Consider a multi-tier containerized system for the **MeDiTwin project** (Mediterranean Digital Twin Network for Understanding Climate Extremes), consisting of:
- An **API Service** handling Mediterranean weather station telemetry.
- A **PostgreSQL Database** storing structured climate observations (heatwaves, droughts, storms).
- A **Redis Cache** caching recent station queries.

Create a project directory:

```bash
mkdir -p ~/climate-stack && cd ~/climate-stack
```

### 1. The application (`app.py`)

Create `app.py`:

This is a Flask service, like the one of section 5, with a second route that uses the two backends:

```python
import os

import psycopg2
import redis
from flask import Flask

DB_HOST = os.getenv("DB_HOST", "db")
DB_NAME = os.getenv("POSTGRES_DB", "climate_db")
DB_USER = os.getenv("POSTGRES_USER", "ecmwf_admin")
DB_PASS = os.getenv("POSTGRES_PASSWORD", "EcmwfPassphrase123!")
REDIS_HOST = os.getenv("REDIS_HOST", "cache")

app = Flask(__name__)


@app.get("/health")
def health():
    return {"status": "healthy"}


@app.get("/stats")
def stats():
    try:
        # 1. Increment access metric in Redis cache
        r = redis.Redis(host=REDIS_HOST, port=6379, socket_connect_timeout=2)
        access_count = r.incr("telemetry:access_count")

        # 2. Query record count from PostgreSQL
        conn = psycopg2.connect(
            host=DB_HOST, dbname=DB_NAME, user=DB_USER, password=DB_PASS, connect_timeout=3
        )
        cur = conn.cursor()
        cur.execute("SELECT COUNT(*) FROM climate_telemetry;")
        total_records = cur.fetchone()[0]
        cur.close()
        conn.close()
    except Exception as e:
        return {"error": str(e)}, 500

    return {
        "cache_access_count": access_count,
        "database_record_count": total_records,
        "service_state": "synchronized"
    }


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

`return {…}, 500` sends the dictionary as JSON with the status code `500`.

### 2. Dependencies and image (`requirements.txt` and `Dockerfile`)

Create `requirements.txt`:
```text
flask==3.1.3
gunicorn==26.2.0
psycopg2-binary==2.9.9
redis==5.0.8
```

Create `Dockerfile`. It follows the same rules as the `Dockerfile` of [section 5.2](05-application-containerisation.md#id-5-2-anatomy-of-a-production-grade-dockerfile): pinned base image, dependencies before the code, a non-root user with a fixed UID, exec form.

```dockerfile
FROM python:3.12-slim-bookworm

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Unprivileged user and group with a fixed ID, as in section 5
RUN groupadd -g 10001 ecmwf && useradd -u 10001 -g ecmwf -d /app -s /sbin/nologin appuser

WORKDIR /app

# Dependencies first: this layer is rebuilt only when requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:ecmwf app.py .

USER 10001

EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "1", "--threads", "4", "app:app"]
```

### 3. The database schema (`init.sql`)

Create `init.sql`:
```sql
CREATE TABLE IF NOT EXISTS climate_telemetry (
    id SERIAL PRIMARY KEY,
    station_id VARCHAR(50) NOT NULL,
    temperature_celsius NUMERIC(6, 2) NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO climate_telemetry (station_id, temperature_celsius) VALUES
('STATION-ATHENS-01', 37.4),
('STATION-BOLOGNA-02', 29.8),
('STATION-VALENCIA-03', 34.1);
```

---

(id-6-3-the-compose-specification-compose-yaml)=
## 6.3 ⌨️ The `compose.yaml` file

Assemble the stack topology in `compose.yaml`:

```yaml
name: climate-stack

services:
  # -------------------------------------------------------------
  # 1. API Application Service
  # -------------------------------------------------------------
  api:
    build:
      context: .
      dockerfile: Dockerfile
    image: climate-stats-api:1.0.0
    ports:
      - "8000:8000"
    environment:
      - DB_HOST=db
      - POSTGRES_DB=climate_db
      - POSTGRES_USER=ecmwf_admin
      - POSTGRES_PASSWORD=EcmwfPassphrase123!
      - REDIS_HOST=cache
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped

  # -------------------------------------------------------------
  # 2. Relational Database Service (PostgreSQL 16)
  # -------------------------------------------------------------
  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=climate_db
      - POSTGRES_USER=ecmwf_admin
      - POSTGRES_PASSWORD=EcmwfPassphrase123!
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ecmwf_admin -d climate_db"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    restart: unless-stopped

  # -------------------------------------------------------------
  # 3. Cache Service (Redis 7)
  # -------------------------------------------------------------
  cache:
    image: redis:7-alpine
    restart: unless-stopped

# ---------------------------------------------------------------
# Persistent Storage Declarations
# ---------------------------------------------------------------
volumes:
  postgres_data:
```

### What each key does

Almost every key is a `docker run` flag that you already know, written in a file:

| Key in `compose.yaml` | What it does | Same as |
|---|---|---|
| `name:` | Name of the project: prefix of the containers, the network and the volumes | — |
| `services:` | One entry for each container: `api`, `db`, `cache`. The key is also its **DNS name** (6.4) | `--name` |
| `build:` | Build the image from this directory and its `Dockerfile` | `docker build .` |
| `image:` | Image to run; with `build:`, the name of the image that Compose builds | the image in `docker run` |
| `ports:` | Publish a container port on the host | `-p 8000:8000` |
| `environment:` | Environment variables | `-e NAME=value` |
| `volumes:` (in a service) | Mount a named volume or a host file | `-v postgres_data:/var/lib/…` |
| `healthcheck:` | Command that tells when the service is ready (6.5) | `--health-cmd` |
| `depends_on:` | Start order: here, `api` waits for the health check of `db` (6.5) | — |
| `restart:` | Restart the container if it stops | `--restart unless-stopped` |
| `volumes:` (top level) | Declare the named volumes; Compose creates them | `docker volume create` |

On Day 3 the same keys become Kubernetes objects: see "Your `compose.yaml` seen through Kubernetes eyes" in the [Day 3 cheat sheet](../day03/13-cheatsheet.md).

---

(id-6-4-service-discovery-and-embedded-dns-resolution)=
## 6.4 Services find each other by name

In `compose.yaml`, the API service resolves its dependencies using:
```yaml
- DB_HOST=db
- REDIS_HOST=cache
```

Docker Compose establishes network routing as follows:
1. An isolated bridge network named `<project>_default` is created.
2. All services declared within the specification join this network automatically.
3. When the application opens a socket to hostname `db`, Docker's embedded DNS server (`127.0.0.11`)
   resolves the service name to the private container IP address of the database instance.

---

(id-6-5-synchronization-and-healthcheck-gated-dependencies)=
## 6.5 Start order: wait until the database is healthy

A distributed systems race condition occurs if an application process launches before its backing
database has completed initialization:

(unsynchronized-dependency-depends-on-without-conditions)=
### Without a condition (`depends_on` only)
```yaml
# ❌ NOT ENOUGH:
depends_on:
  - db
```
This constraint evaluates to `true` as soon as the database container process is spawned. However,
PostgreSQL requires several seconds to initialize filesystem clusters and bind TCP sockets. Requests
that reach the database in that window fail with `Connection Refused`: with this API, `/stats` returns
HTTP 500 until PostgreSQL is ready; an application that connects only once at startup would crash.

### With a health check (`condition: service_healthy`)
```yaml
# ✅ WAITS FOR THE DATABASE:
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ecmwf_admin -d climate_db"]
  interval: 5s
  timeout: 5s
  retries: 5

api:
  depends_on:
    db:
      condition: service_healthy
```
With this declaration, the runtime blocks instantiation of the `api` container until PostgreSQL's
health check probe exits with return code `0`.

:::{note} On Day 3
Kubernetes has no `depends_on`. Each container must retry until its database answers, and **probes** take the place of `healthcheck:`: the readiness probe decides when a Pod receives traffic ([Day 3, section 3](../day03/03-deploying-an-application.md) and [section 10](../day03/10-from-compose-to-kubernetes.md)).
:::

---

(id-6-6-operational-lifecycle-management)=
## 6.6 ⌨️ Hands-on 13: start and check the stack

### Step 1: start the stack

Deploy the stack in detached mode:

```bash
cd ~/climate-stack
docker compose up -d
```

```text
[+] Running 5/5
 ✔ Network climate-stack_default         Created
 ✔ Volume "climate-stack_postgres_data"  Created
 ✔ Container climate-stack-cache-1       Started
 ✔ Container climate-stack-db-1          Healthy
 ✔ Container climate-stack-api-1         Started
```

### Step 2: check the status (`docker compose ps`)

```bash
docker compose ps
```

```text
NAME                   IMAGE                     COMMAND                  SERVICE   STATUS                    PORTS
climate-stack-api-1    climate-stats-api:1.0.0   "gunicorn --bind 0.…"   api       Up 20 seconds             0.0.0.0:8000->8000/tcp
climate-stack-cache-1  redis:7-alpine            "docker-entrypoint.s…"   cache     Up 30 seconds             6379/tcp
climate-stack-db-1     postgres:16-alpine        "docker-entrypoint.s…"   db        Up 30 seconds (healthy)   5432/tcp
```

### Step 3: call the API

```bash
curl http://localhost:8000/stats
```

```json
{"cache_access_count":1,"database_record_count":3,"service_state":"synchronized"}
```

### Step 4: query the database and read the logs

Run a query directly in the `db` service container. `docker compose exec` works like `docker exec`, but takes the **service** name (`db`) instead of the container name:

```bash
docker compose exec db psql -U ecmwf_admin -d climate_db -c "SELECT * FROM climate_telemetry;"
```

You see the three rows of `init.sql`.

Then read the last 20 lines of the logs of all the services together; each line starts with the name of the service that wrote it:

```bash
docker compose logs --tail 20
```

**Checkpoint 13** — the three services are up, the database is `healthy`, and the API reads from both PostgreSQL and Redis.

```bash
docker compose ps --format '{{.Service}}: {{.Status}}'
curl -s http://localhost:8000/stats
```

```text
api: Up 2 minutes
cache: Up 2 minutes
db: Up 2 minutes (healthy)
{"cache_access_count":2,"database_record_count":3,"service_state":"synchronized"}
```

`cache_access_count` grows by one at every call.

:::{dropdown} Not there?
- `api` restarts, or `/stats` returns `{"error": …}`: read `docker compose logs api`. `could not translate host name "db"` means the service name in `DB_HOST` does not match the name under `services:`.
- `db` is not `healthy` after a minute: `docker compose logs db`. If you changed `POSTGRES_USER` or `POSTGRES_DB`, the old volume still has the old database: `docker compose down -v` and `up -d` again (this deletes the data).
- `database_record_count` is `0` or the table does not exist: `init.sql` runs only when the volume is empty. Same fix: `docker compose down -v`, then `up -d`.
- `bind: address already in use` on port 8000: another container uses it (for example `climate-telemetry` from section 5). `docker ps --filter publish=8000`, then remove it.
:::

(id-6-7-teardown-and-volume-durability-semantics)=
## 6.7 ⌨️ Stop the stack and keep the data

To halt and remove containers and network devices while preserving database volumes:

```bash
docker compose down
```

```text
[+] Running 4/4
 ✔ Container climate-stack-api-1    Removed
 ✔ Container climate-stack-db-1     Removed
 ✔ Container climate-stack-cache-1  Removed
 ✔ Network climate-stack_default    Removed
```

Re-invoking `docker compose up -d` restores the stack with all pre-existing records intact.

:::{warning} `-v` deletes the data
Executing `docker compose down -v` instructs the engine to purge all persistent volumes declared in the
file alongside container instances. Use strictly when complete data resetting is intended.
:::

Keep the `~/climate-stack` directory: Day 3 runs the same stack on Kubernetes (sections 9 and 10), and Day 5 adds metrics to it.

## 6.8 Core takeaway

> **`compose.yaml` describes the whole stack; `docker compose up -d` creates it, and `docker compose down` removes it but keeps the data.**

- Each entry under `services:` becomes a container; Compose creates one network for the project, and the services find each other **by service name**.
- Named volumes under `volumes:` survive `down`; `down -v` deletes them.
- `depends_on` alone only orders the start; with `condition: service_healthy` the API waits for the database **health check**.
- `docker compose ps`, `logs` and `exec` work like `docker ps`, `logs` and `exec`, for the whole stack.

## 6.9 📖 Check yourself

1. The API connects to `db:5432`. Where does the name `db` come from, and who resolves it?
2. With `depends_on: [db]` only, `/stats` returns errors for the first seconds. Why, and what do you change?
3. You run `docker compose down`, then `up -d`. Are the rows in PostgreSQL still there? And after `down -v`?
4. You change `init.sql` and run `docker compose up -d` again. The new table does not appear. Why?
5. Why does the API container not publish port `5432` of the database, and why is that good?

:::{dropdown} Solutions
**1.** From the service name under `services:`. Docker's DNS server (`127.0.0.11`) on the project network resolves it to the IP address of the `db` container.

**2.** `depends_on` waits only until the database container starts, not until PostgreSQL accepts connections. Add a `healthcheck` to `db` and `condition: service_healthy` to `api`.

**3.** After `down` and `up -d`: yes, the volume `climate-stack_postgres_data` is kept. After `down -v`: no, the volume is deleted.

**4.** PostgreSQL runs the scripts in `/docker-entrypoint-initdb.d/` only when its data directory is empty. The volume already has a database: `docker compose down -v` and `up -d` (this deletes the data).

**5.** Only the API must be reachable from outside. The database is reachable by the API on the project network, without a published port, so nobody outside the host can connect to it.
:::

---

Continue with: [7. Capstone lab](07-lab.md).
