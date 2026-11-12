# 7. Capstone lab: the MeDiTwin climate extremes pipeline in containers

This hands-on capstone laboratory synthesizes the concepts covered throughout Day 2. You will construct, harden, orchestrate, and audit a multi-tier containerized data platform for the **MeDiTwin project** ([Mediterranean Digital Twin Network for Understanding Climate Extremes](https://meditwin-project.eu/)).

The platform ingests real-time weather observations and climate extreme indicators (temperature, humidity, atmospheric pressure, wind speed) from sensor stations across the Mediterranean basin (monitoring heatwaves, droughts, wildfires, and medicanes), caches latest station readings in Redis, and commits historical records to PostgreSQL.

The API of this lab uses **FastAPI**, another common Python web framework, served by `uvicorn`, instead of the Flask and gunicorn of sections 5 and 6. Nothing else changes: the `Dockerfile`, the image and `compose.yaml` work the same way for any framework. The container does not care what runs inside it.

**Duration:** 90 – 120 minutes.

**Symbols:** ⌨️ you type the commands · 📖 reference material: read it after the lesson, or when you need it. The lesson does not cover it in class.

---

## What you build

```{mermaid}
flowchart TB
    subgraph ClientLayer["External Network Ingress"]
        Station([Mediterranean Weather Station / Sensor])
        Scientist([Climate Scientist / EO Analyst])
        Admin([System Operator])
    end

    subgraph PortainerUI["Web Management Console (Port 9000)"]
        Portainer["Portainer CE Container<br/>(Monitors system resources, logs, and volumes)"]
    end

    subgraph StackNet["Docker Compose Network: 'meditwin-climate-net'"]
        direction TB
        API["MeDiTwin Climate Ingestion API (FastAPI)<br/>- Multi-stage image<br/>- Non-root user: appuser<br/>- Port 8000"]
        DB[("Climate Database (PostgreSQL 16)<br/>- Healthchecked<br/>- Port 5432 (Internal)")]
        Cache[("Telemetry Cache (Redis 7)<br/>- Port 6379 (Internal)")]

        API -->|Persists observation| DB
        API -->|Caches latest reading| Cache
    end

    subgraph HostDisk["Host Storage System"]
        DBVol[("Named Volume: meditwin_db_data")]
        PortVol[("Named Volume: portainer_data")]
    end

    Station -->|HTTP / POST :8000| API
    Scientist -->|HTTP / GET :8000| API
    Admin -->|HTTP :9000| Portainer
    Portainer -.->|Unix Socket /var/run/docker.sock| StackNet
    DB <===> DBVol
    Portainer <===> PortVol
```

---

## ⌨️ Phase 1 — Create the project

Establish a clean project directory hierarchy:

```bash
mkdir -p ~/meditwin-climate-capstone/{src,tests}
cd ~/meditwin-climate-capstone
```

### 1. Dependencies (`requirements.txt`)

Create `requirements.txt`:

```text
fastapi==0.111.0
uvicorn==0.30.1
psycopg2-binary==2.9.9
redis==5.0.8
pydantic==2.8.2
pytest==8.2.2
httpx==0.27.0
```

### 2. The application (`src/main.py`)

Create `src/main.py`:

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, Field
from typing import Optional
import os
import psycopg2
import redis
import socket
import json

app = FastAPI(title="MeDiTwin Climate Extremes Ingestion Gateway", version="1.0.0")

DB_HOST = os.getenv("DB_HOST", "db")
DB_NAME = os.getenv("POSTGRES_DB", "meditwin_climate_db")
DB_USER = os.getenv("POSTGRES_USER", "meditwin_admin")
DB_PASS = os.getenv("POSTGRES_PASSWORD", "SuperSecurePassword987!")
REDIS_HOST = os.getenv("REDIS_HOST", "cache")

class ObservationPayload(BaseModel):
    station_id: str = Field(..., example="STATION-ATHENS-01", description="Identifier (e.g. Athens, Bologna, Valencia)")
    temperature: float = Field(..., ge=-50.0, le=60.0, example=38.4, description="Temperature in Celsius (Heatwave monitoring)")
    humidity: float = Field(..., ge=0.0, le=100.0, example=28.5, description="Relative humidity in % (Drought / Fire weather index)")
    pressure: float = Field(..., ge=850.0, le=1100.0, example=1012.3, description="Atmospheric pressure in hPa (Medicane tracking)")
    wind_speed: Optional[float] = Field(None, ge=0.0, le=150.0, example=12.4, description="Wind speed in m/s (Wildfire spread index)")

def get_db():
    return psycopg2.connect(
        host=DB_HOST, dbname=DB_NAME, user=DB_USER, password=DB_PASS, connect_timeout=3
    )

def get_redis():
    return redis.Redis(host=REDIS_HOST, port=6379, socket_connect_timeout=2)

@app.get("/health")
def health_check():
    return {"status": "healthy", "service": "meditwin-climate-telemetry", "host": socket.gethostname()}

@app.post("/api/v1/observations", status_code=status.HTTP_201_CREATED)
def record_observation(payload: ObservationPayload):
    try:
        # 1. Cache latest station reading in Redis (serialize as JSON)
        r = get_redis()
        cache_data = {
            "temperature": payload.temperature,
            "humidity": payload.humidity,
            "pressure": payload.pressure,
            "wind_speed": payload.wind_speed,
        }
        r.set(f"station:{payload.station_id}:latest", json.dumps(cache_data))

        # 2. Commit observation into PostgreSQL
        conn = get_db()
        cur = conn.cursor()
        cur.execute(
            """
            INSERT INTO climate_observations (station_id, temperature, humidity, pressure, wind_speed)
            VALUES (%s, %s, %s, %s, %s) RETURNING id;
            """,
            (payload.station_id, payload.temperature, payload.humidity, payload.pressure, payload.wind_speed)
        )
        record_id = cur.fetchone()[0]
        conn.commit()
        cur.close()
        conn.close()

        return {
            "recorded": True,
            "record_id": record_id,
            "station_id": payload.station_id,
            "cached": True
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/v1/observations/{station_id}")
def get_latest_observation(station_id: str):
    try:
        # Check cache first (Redis)
        r = get_redis()
        cached_data = r.get(f"station:{station_id}:latest")
        if cached_data:
            parsed = json.loads(cached_data)
            return {
                "station_id": station_id,
                "latest_observation": parsed,
                "source": "cache"
            }

        # Fallback to PostgreSQL database on cache miss
        conn = get_db()
        cur = conn.cursor()
        cur.execute(
            """
            SELECT temperature, humidity, pressure, wind_speed, recorded_at 
            FROM climate_observations 
            WHERE station_id = %s 
            ORDER BY recorded_at DESC LIMIT 1;
            """,
            (station_id,)
        )
        row = cur.fetchone()
        cur.close()
        conn.close()

        if not row:
            raise HTTPException(status_code=404, detail=f"Observation records for station '{station_id}' not found")

        return {
            "station_id": station_id,
            "latest_observation": {
                "temperature": float(row[0]),
                "humidity": float(row[1]),
                "pressure": float(row[2]),
                "wind_speed": float(row[3]) if row[3] is not None else None,
            },
            "recorded_at": str(row[4]),
            "source": "database"
        }
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### 3. Unit tests (`tests/test_api.py`)

Create `tests/test_api.py`:

```python
from fastapi.testclient import TestClient
import sys
import os

sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), "..", "src")))
from main import app

client = TestClient(app)

def test_health():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"
    assert response.json()["service"] == "meditwin-climate-telemetry"
```

---

(phase-2-multi-stage-image-compilation)=
## ⌨️ Phase 2 — A multi-stage image

Author a multi-stage `Dockerfile` enforcing non-root execution and layer optimization.

Create `Dockerfile`:

```dockerfile
# Stage 1: Build Dependencies
FROM python:3.12-slim-bookworm AS builder

WORKDIR /build

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir=/build/wheels -r requirements.txt

# Stage 2: Minimal Runtime Image
FROM python:3.12-slim-bookworm AS runner

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /build/wheels /wheels
RUN if ls -1 /wheels/*.whl >/dev/null 2>&1; then pip install --no-cache-dir --no-index --find-links=/wheels /wheels/*.whl; fi && rm -rf /wheels

# Enforce non-root execution (meditwin user)
RUN groupadd -r meditwin && useradd -r -g meditwin -d /app -s /sbin/nologin appuser

COPY --chown=appuser:meditwin src/ /app/src/

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Create `.dockerignore`:

```text
.git/
.venv/
__pycache__/
*.pyc
tests/
.pytest_cache/
```

:::{note} Checkpoint 1: Image Compilation Verification
Execute the build command and verify the resulting image:
```bash
docker build -t climate-ingestion-api:1.0.0 .
docker images | grep climate-ingestion-api
```
Notice that thanks to the multi-stage build, the final runner image does not contain `gcc` or compilation toolchains, keeping the image size compact (~160 MB).
:::

---

## ⌨️ Phase 3 — The database schema

Create `init.sql`:

```sql
CREATE TABLE IF NOT EXISTS climate_observations (
    id SERIAL PRIMARY KEY,
    station_id VARCHAR(50) NOT NULL,
    temperature NUMERIC(5, 2) NOT NULL,
    humidity NUMERIC(5, 2) NOT NULL,
    pressure NUMERIC(6, 2) NOT NULL,
    wind_speed NUMERIC(5, 2),
    recorded_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_station_id ON climate_observations(station_id);
```

---

## ⌨️ Phase 4 — The stack (`compose.yaml`)

Construct `compose.yaml`:

```yaml
name: meditwin-climate-stack

services:
  # -------------------------------------------------------------
  # 1. Management Console: Portainer CE
  # -------------------------------------------------------------
  portainer:
    image: portainer/portainer-ce:2.21.0
    container_name: capstone-portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - climate-net

  # -------------------------------------------------------------
  # 2. Ingestion Application Service (FastAPI)
  # -------------------------------------------------------------
  api:
    build:
      context: .
      dockerfile: Dockerfile
    image: climate-ingestion-api:1.0.0
    container_name: capstone-api
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - DB_HOST=db
      - POSTGRES_DB=meditwin_climate_db
      - POSTGRES_USER=meditwin_admin
      - POSTGRES_PASSWORD=SuperSecurePassword987!
      - REDIS_HOST=cache
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    networks:
      - climate-net

  # -------------------------------------------------------------
  # 3. Relational Database Service (PostgreSQL 16)
  # -------------------------------------------------------------
  db:
    image: postgres:16-alpine
    container_name: capstone-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=meditwin_climate_db
      - POSTGRES_USER=meditwin_admin
      - POSTGRES_PASSWORD=SuperSecurePassword987!
    volumes:
      - meditwin_db_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U meditwin_admin -d meditwin_climate_db"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s
    networks:
      - climate-net

  # -------------------------------------------------------------
  # 4. In-Memory Cache Service (Redis 7)
  # -------------------------------------------------------------
  cache:
    image: redis:7-alpine
    container_name: capstone-cache
    restart: unless-stopped
    networks:
      - climate-net

volumes:
  meditwin_db_data:
  portainer_data:

networks:
  climate-net:
    driver: bridge
```

---

## ⌨️ Phase 5 — Start the stack and check it

### Step 1: start the stack

```bash
docker compose up -d
```

### Step 2: check the services

```bash
docker compose ps
```

```text
NAME                 IMAGE                         STATUS                    PORTS
capstone-api         climate-ingestion-api:1.0.0   Up 15 seconds (healthy)   0.0.0.0:8000->8000/tcp
capstone-cache       redis:7-alpine                Up 25 seconds             6379/tcp
capstone-db          postgres:16-alpine            Up 25 seconds (healthy)   5432/tcp
capstone-portainer   portainer/portainer-ce:2.21.0 Up 25 seconds             0.0.0.0:9000->9000/tcp
```

### Step 3: send and read climate data

Transmit Mediterranean climate observation telemetry payloads to the API:

```bash
# 1. Ingest reading from station 'STATION-ATHENS-01' (NTUA)
curl -X POST http://localhost:8000/api/v1/observations \
  -H "Content-Type: application/json" \
  -d '{"station_id": "STATION-ATHENS-01", "temperature": 38.4, "humidity": 28.5, "pressure": 1012.3, "wind_speed": 12.4}'
```

```json
{"recorded":true,"record_id":1,"station_id":"STATION-ATHENS-01","cached":true}
```

```bash
# 2. Ingest reading from station 'STATION-VALENCIA-02' (UV)
curl -X POST http://localhost:8000/api/v1/observations \
  -H "Content-Type: application/json" \
  -d '{"station_id": "STATION-VALENCIA-02", "temperature": 34.2, "humidity": 35.0, "pressure": 1014.1, "wind_speed": 8.1}'
```

```bash
# 3. Query station 'STATION-ATHENS-01' (Returns from Redis cache)
curl http://localhost:8000/api/v1/observations/STATION-ATHENS-01
```

```json
{"station_id":"STATION-ATHENS-01","latest_observation":{"temperature":38.4,"humidity":28.5,"pressure":1012.3,"wind_speed":12.4},"source":"cache"}
```

---

## ⌨️ Phase 6 — Break it on purpose

(experiment-6-1-container-eviction-and-state-persistence)=
### Experiment 6.1: remove the database container, keep the data

Simulate sudden database container failure:

```bash
# Forcibly destroy the database container
docker rm -f capstone-db

# Restore stack state
docker compose up -d
```

Flush the Redis cache to force the API to query the database:
```bash
docker exec -i capstone-cache redis-cli FLUSHALL
curl http://localhost:8000/api/v1/observations/STATION-ATHENS-01
```

```json
{"station_id":"STATION-ATHENS-01","latest_observation":{"temperature":38.4,"humidity":28.5,"pressure":1012.3,"wind_speed":12.4},"recorded_at":"2026-09-15 ...","source":"database"}
```

*Analysis:* Data persisted across container destruction because storage was bound to the `meditwin_db_data` named volume rather than the container's mutable layer.

(experiment-6-2-control-group-memory-boundary-enforcement)=
### Experiment 6.2: go over the memory limit

Verify kernel cgroup enforcement under memory allocation stress:

```bash
docker run --rm -m 30m --memory-swap 30m python:3.12-alpine python3 -c 'x = bytearray(50 * 1024 * 1024)' || echo "Process Exit Code: $?"
```

```text
Killed
Process Exit Code: 137
```

*Analysis:* Exit code **137** confirms that the host kernel's OOM killer issued `SIGKILL` ($128 + 9$) when process RSS exceeded the declared 30 MB boundary. (`--memory-swap 30m` caps memory + swap at 30 MB: without it Docker also allows 30 MB of swap, and on a host with swap the allocation may succeed.)

---

## ⌨️ Phase 7 — Stop the stack

Halt the stack while preserving volume allocations:

```bash
docker compose down
```

*(To purge all storage volumes: `docker compose down -v`)*

---

## Evaluation

Students are evaluated based upon:
1. **Dockerfile Optimization:** Successful multi-stage separation and verification of unprivileged non-root `appuser` execution.
2. **Topology Correctness:** Proper network isolation and `depends_on: condition: service_healthy` implementation.
3. **Data Durability:** Preservation of relational records post container eviction.

Building and scanning this image automatically, in a CI pipeline, is covered on Day 4.

---

Continue with: [8. Cheat sheet & troubleshooting](08-cheatsheet.md).
