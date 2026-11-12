# 3. Group project: package and serve a model

**Objective:** your group packages a trained model as a web service and deploys it to the
training cluster **through Git only**. The service uses HTTPS, needs a password and is
monitored. When the load increases, it scales out.

This page uses what you learned on all five days. Each step says what to do and where you
learned it. When you are stuck, open the **hidden instructions** under each step: click their
title. Try without them first.

| Step | Task | Who | Concepts from |
| --- | --- | --- | --- |
| [1](#gp-step-1) | Make the model; put it in the bucket | Everyone; host | Day 1 (object storage) |
| [2](#gp-step-2) | Write the API: `/predict`, `/health`, `/metrics` | Application | Day 5 (metrics, Basic Auth) |
| [3](#gp-step-3) | Multi-stage, non-root image; test with Compose | Container | Day 2 |
| [4](#gp-step-4) | GitLab projects, members, protected `main` | Host | Day 4 (Git) |
| [5](#gp-step-5) | Pipeline: test, build, scan, release | Pipeline | Day 4 (CI/CD, security, releases) |
| [6](#gp-step-6) | Helm chart: Deployment, Service, Ingress, HPA, monitoring | Platform | Day 3, Day 5 |
| [7](#gp-step-7) | Secrets, then deploy with Fleet | Host | Day 4 (GitOps, Fleet) |
| [8](#gp-step-8) | Check HTTPS and authentication | Everyone | Day 5 (security) |
| [9](#gp-step-9) | Metrics in Prometheus, dashboard in Grafana | Everyone | Day 5 (monitoring) |
| [10](#gp-step-10) | Load test: the HPA adds pods | Everyone | Day 3 (scaling) |

## 3.1 How the group works

**Groups** have 3 or 4 participants. The instructor gives each group:

- a **group name**, for example `group03`;
- a **host**: one member of the group.

You make the model yourselves, in step 1.

**The host.** Each of you has your own namespace, bucket and Fleet workspace. The group uses the
**host's**: the host's namespace, the host's bucket, the host's Fleet workspace, and two GitLab
projects in the host's personal namespace. So some steps can only be done by the host; the table
above marks them. The other members work through Git: branches, merge requests, reviews.

**Roles.** Share the work. The host takes **Platform** (only the host can change the cluster).

| Role | Responsibility |
| --- | --- |
| Application | `app.py`, tests, metrics |
| Container | `Dockerfile`, `compose.yaml`, local tests |
| Pipeline | `.gitlab-ci.yml`, scans, releases |
| Platform (host) | Helm chart, secrets, Fleet, Ingress, scaling |

Every member reviews the merge requests of the others.

**Rules** (the evaluation checks them):

- The model file is **never** in Git or in the image. It is read from the bucket when the
  service starts.
- No secret is in Git.
- Only Fleet deploys. Nobody runs `kubectl apply` or `helm install`. The only manual cluster
  changes are the three Secrets of step 7.

**Tools.** Work on your training VM. The `course` Python environment, active when you log in,
already has every package the project uses, at the same versions as the image. Do not create
another virtual environment.

## 3.2 Set up your shell (everyone)

Every command on this page reads a few variables. Put them in a file and load it from
`~/.bashrc`, so that every new terminal has them.

```bash
cat > ~/day5.env <<'EOF'
export GROUP=group03                 # ← your group name
export HOST_ACCOUNT=trainee07        # ← your group's host
export NS=$HOST_ACCOUNT
export BUCKET=meditwin-training-2026-$HOST_ACCOUNT
export APP=$HOST_ACCOUNT/model-api-$GROUP
export DEPLOY=$HOST_ACCOUNT/model-api-$GROUP-deploy
export APP_HOST=model-$GROUP.training.meditwin-project.eu
export REG=gitlab.data.meditwin-project.eu:6443
EOF
echo 'source ~/day5.env' >> ~/.bashrc && source ~/day5.env
```

(gp-step-1)=
## Step 1. Make the model and put it in the bucket

**Everyone.** Write a script, `make_model.py`, that trains the model the service will serve, and
run it in `~/day5`. The model predicts a class of weather extreme from one day of station data.

| Input feature | Meaning |
| --- | --- |
| `tmax_c`, `tmin_c` | daily maximum and minimum temperature, °C |
| `precip_mm` | 24-hour precipitation, mm |
| `rel_humidity` | mean relative humidity, % |
| `wind_max_ms` | maximum gust, m/s |
| `pressure_hpa` | mean sea-level pressure, hPa |

The four classes are `none`, `heatwave`, `heavy_rain` and `windstorm`. The script:

- makes synthetic training data, with a fixed random seed;
- trains a scikit-learn `RandomForestClassifier`;
- writes three files:
  - `model.joblib`: the model, with its feature order, classes and version;
  - `model-card.json`: the versions used, the features, the classes and the file's SHA-256;
  - `sample-input.json`: a request body for `POST /predict`, with five station days.

With the same package versions and the same seed, every member gets the **same file, byte for
byte**. Compare your SHA-256 with your group's.

The model is a Python pickle. The service must load it with the **scikit-learn version** that
wrote it. `model-card.json` records that version.

**Host only.** Then copy `model.joblib` and `model-card.json` into your bucket, and check that
the copy in the bucket is your file (Day 1).

:::{dropdown} Hidden file: `make_model.py`
```bash
mkdir -p ~/day5 && cd ~/day5
cat > make_model.py <<'EOF'
#!/usr/bin/env python3
"""Make the group project's model, its model card and a sample input.

The data are synthetic "station days": six weather variables, with a plausible
relation to four classes of extreme: none, heatwave, heavy_rain, windstorm.
The project is about packaging and serving a model, so the model only has to
be small, believable and reproducible: the same versions give the same file,
byte for byte, on every VM.

    python make_model.py --out ~/day5

Writes, in --out:
    model.joblib        {"model": RandomForestClassifier, "features": [...],
                         "classes": [...], "version": "..."}
    model-card.json     versions, feature order, classes, sha256 of model.joblib
    sample-input.json   the body of a POST /predict, five station days

The model is a pickle: the service must load it with the scikit-learn version
that wrote it (model-card.json says which).
"""
from __future__ import annotations

import argparse
import hashlib
import json
import platform
from datetime import datetime, timezone
from pathlib import Path

import joblib
import numpy as np
import sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split

FEATURES = [
    "tmax_c",          # daily maximum temperature, degC
    "tmin_c",          # daily minimum temperature, degC
    "precip_mm",       # 24 h precipitation, mm
    "rel_humidity",    # mean relative humidity, %
    "wind_max_ms",     # maximum gust, m/s
    "pressure_hpa",    # mean sea-level pressure, hPa
]
CLASSES = ["none", "heatwave", "heavy_rain", "windstorm"]


def synthesise(n: int, rng: np.random.Generator) -> tuple[np.ndarray, np.ndarray]:
    tmax = rng.normal(24, 7, n)
    tmin = tmax - rng.gamma(4, 2.5, n)
    precip = rng.gamma(0.6, 8, n)
    rh = np.clip(rng.normal(65, 15, n) + precip * 0.6 - (tmax - 24) * 0.8, 5, 100)
    pressure = rng.normal(1013, 9, n)
    wind = np.clip(rng.gamma(3, 3, n) + (1013 - pressure) * 0.6, 0, None)

    y = np.zeros(n, dtype=int)
    heat = (tmax > 35) & (tmin > 20) & (rh < 55)
    rain = (precip > 40) & (pressure < 1010)
    wind_ = (wind > 22) & (pressure < 1000)
    y[heat] = 1
    y[rain] = 2
    y[wind_] = 3          # a windstorm wins over rain on the same day
    flip = rng.random(n) < 0.03           # 3 % label noise: no perfect score
    y[flip] = rng.integers(0, 4, flip.sum())

    X = np.column_stack([tmax, tmin, precip, rh, wind, pressure])
    return X.round(2), y


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--out", type=Path, default=Path("dist"))
    ap.add_argument("--version", default="2026.11-1")
    ap.add_argument("--seed", type=int, default=20261120)
    args = ap.parse_args()
    args.out.mkdir(parents=True, exist_ok=True)

    rng = np.random.default_rng(args.seed)
    X, y = synthesise(60_000, rng)
    X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=0, stratify=y)
    # Small on purpose: ~2-4 MB on disk, ~5 ms per prediction on one core.
    # Enough CPU per request that 20 concurrent clients push the HPA.
    clf = RandomForestClassifier(n_estimators=120, max_depth=12, n_jobs=1, random_state=0)
    clf.fit(X_tr, y_tr)
    accuracy = float(clf.score(X_te, y_te))

    model_path = args.out / "model.joblib"
    joblib.dump(
        {"model": clf, "features": FEATURES, "classes": CLASSES, "version": args.version},
        model_path,
        compress=3,
    )
    sha = hashlib.sha256(model_path.read_bytes()).hexdigest()

    card = {
        "name": "meditwin-climate-extremes",
        "version": args.version,
        "created": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "sha256": sha,
        "size_bytes": model_path.stat().st_size,
        "python": platform.python_version(),
        "scikit_learn": sklearn.__version__,
        "numpy": np.__version__,
        "joblib": joblib.__version__,
        "features": FEATURES,
        "classes": CLASSES,
        "test_accuracy": round(accuracy, 4),
        "input": {"instances": [{f: "<number>" for f in FEATURES}]},
        "note": "Synthetic training data. For the course only.",
    }
    (args.out / "model-card.json").write_text(json.dumps(card, indent=2) + "\n")

    samples = [
        {"tmax_c": 38.4, "tmin_c": 23.1, "precip_mm": 0.0, "rel_humidity": 31, "wind_max_ms": 6.2, "pressure_hpa": 1016},
        {"tmax_c": 17.9, "tmin_c": 11.0, "precip_mm": 72.5, "rel_humidity": 96, "wind_max_ms": 12.0, "pressure_hpa": 1002},
        {"tmax_c": 12.3, "tmin_c": 6.4, "precip_mm": 9.1, "rel_humidity": 81, "wind_max_ms": 31.5, "pressure_hpa": 982},
        {"tmax_c": 22.0, "tmin_c": 14.2, "precip_mm": 1.2, "rel_humidity": 63, "wind_max_ms": 7.4, "pressure_hpa": 1014},
        {"tmax_c": 36.1, "tmin_c": 21.5, "precip_mm": 0.4, "rel_humidity": 48, "wind_max_ms": 4.0, "pressure_hpa": 1019},
    ]
    (args.out / "sample-input.json").write_text(json.dumps({"instances": samples}, indent=2) + "\n")

    preds = [CLASSES[i] for i in clf.predict(np.array([[s[f] for f in FEATURES] for s in samples]))]
    print(f"model.joblib  {card['size_bytes']} bytes  sha256 {sha[:16]}...  accuracy {accuracy:.3f}")
    print(f"scikit-learn {sklearn.__version__}  numpy {np.__version__}  python {platform.python_version()}")
    print("sample predictions:", preds)


if __name__ == "__main__":
    main()
EOF
```
:::

:::{dropdown} Hidden instructions: make the model
In the `course` environment, which has the pinned scikit-learn:

```bash
cd ~/day5
python make_model.py --out ~/day5
# model.joblib  1683476 bytes  sha256 3218d14bab716f7e...  accuracy 0.976
# scikit-learn 1.9.1  numpy 2.5.4  python 3.12.x
# sample predictions: ['heatwave', 'heavy_rain', 'windstorm', 'none', 'heatwave']
jq . model-card.json
```

It takes about a minute. Your accuracy should be 0.976, and the SHA-256 should start with
`3218d14b`. A different value means different package versions: check `pip list | grep -E
'scikit|numpy|scipy|joblib'`.
:::

:::{dropdown} Hidden instructions: the bucket (host)
`s3cmd` is already set up with your keys (`~/.s3cfg`).

```bash
cd ~/day5
s3cmd put model.joblib model-card.json s3://$BUCKET/
s3cmd ls s3://$BUCKET/
sha256sum model.joblib && jq -r .sha256 model-card.json     # the two values are the same
```

Try another trainee's bucket: `s3cmd ls s3://meditwin-training-2026-trainee99/`. The S3
gateway refuses it with `AccessDenied`. Your keys open your bucket only.
:::

(gp-step-2)=
## Step 2. The API

**Application.** Write a Flask service with three endpoints:

| Endpoint | Access | Answer |
| --- | --- | --- |
| `GET /health` | open | `{"status": "ok", "model_version": …}` |
| `POST /predict` | HTTP Basic Auth | body `{"instances": [{feature: number, …}]}` → `{"predictions": [label, …]}`; `401` without valid credentials; `400` for bad input |
| `GET /metrics` | open (not exposed outside the cluster) | Prometheus metrics: requests by endpoint and status, request duration, refused logins |

Requirements:

- The service reads the model **when it starts**: from the file `MODEL_LOCAL_PATH` if that is
  set (local tests), otherwise from `s3://$MODEL_BUCKET/$MODEL_KEY`. Then a broken S3 setup
  stops the pod immediately, with the reason in its log.
- The credentials come from the environment, `API_USER` and `API_PASSWORD`. Without them,
  nobody gets in.
- Unit tests that need neither S3 nor the real model.

Create a directory `~/model-api` for the application. It becomes the group's application
repository in step 4.

:::{dropdown} Hidden file: `app.py`
Two details are worth a question in the demonstration:

- `addressing_style: path`: the training S3 gateway authorises a request by the first part of
  its path, which is the bucket.
- `KNOWN_ENDPOINTS`: any other path is counted as `other`. If the raw path were a label value,
  every scanner probe would create a new time series in Prometheus.

```bash
mkdir -p ~/model-api/tests && cd ~/model-api
cat > app.py <<'EOF'
"""Model API: GET /health (open), POST /predict (Basic Auth), GET /metrics."""
import base64
import logging
import os
import secrets
import time

import joblib
import numpy as np
from flask import Flask, Response, g, jsonify, request
from prometheus_client import CONTENT_TYPE_LATEST, Counter, Histogram, generate_latest

log = logging.getLogger("model-api")

REQUESTS = Counter("model_api_requests_total", "HTTP requests", ["method", "path", "status"])
LATENCY = Histogram("model_api_request_duration_seconds", "Request duration", ["path"])
AUTH_FAILURES = Counter("model_api_auth_failures_total", "Refused /predict calls (401)")
KNOWN_ENDPOINTS = {"/health", "/predict", "/metrics"}   # anything else is labelled "other"


def load_model():
    """Read the model from MODEL_LOCAL_PATH, or download it from S3."""
    path = os.environ.get("MODEL_LOCAL_PATH")
    if not path:
        import boto3
        from botocore.config import Config

        path = "/tmp/model.joblib"            # /tmp: the only writable place in the pod
        s3 = boto3.client("s3", config=Config(s3={"addressing_style": "path"}))
        log.info("downloading s3://%s/%s", os.environ["MODEL_BUCKET"], os.environ["MODEL_KEY"])
        s3.download_file(os.environ["MODEL_BUCKET"], os.environ["MODEL_KEY"], path)
    return joblib.load(path)


def create_app(bundle=None):
    logging.basicConfig(level="INFO")
    bundle = bundle or load_model()           # at start: a broken S3 setup fails the pod
    model, features, classes = bundle["model"], bundle["features"], bundle["classes"]
    version = bundle["version"]
    api_user = os.environ.get("API_USER", "")
    api_password = os.environ.get("API_PASSWORD", "")
    log.info("model %s loaded", version)

    app = Flask(__name__)

    @app.before_request
    def start_timer():
        g.t0 = time.perf_counter()

    @app.after_request
    def count(resp):
        path = request.path if request.path in KNOWN_ENDPOINTS else "other"
        REQUESTS.labels(request.method, path, str(resp.status_code)).inc()
        LATENCY.labels(path).observe(time.perf_counter() - g.t0)
        return resp

    def authorised():
        header = request.headers.get("Authorization", "")
        if not (api_user and api_password and header.startswith("Basic ")):
            return False                      # fail closed
        try:
            user, _, password = base64.b64decode(header[6:]).decode().partition(":")
        except ValueError:
            return False
        return secrets.compare_digest(user, api_user) & secrets.compare_digest(password, api_password)

    @app.get("/health")
    def health():
        return jsonify(status="ok", model_version=version)

    @app.post("/predict")
    def predict():
        if not authorised():
            AUTH_FAILURES.inc()
            return jsonify(error="authentication required"), 401, {
                "WWW-Authenticate": 'Basic realm="predict"'}
        body = request.get_json(silent=True) or {}
        try:
            X = np.array([[float(row[f]) for f in features] for row in body["instances"]])
        except (KeyError, TypeError, ValueError):
            return jsonify(error="expected {\"instances\": [{feature: number}]}", features=features), 400
        labels = [classes[i] for i in model.predict(X)]
        return jsonify(model_version=version, predictions=labels)

    @app.get("/metrics")
    def metrics():
        return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

    return app
EOF
```
:::

:::{dropdown} Hidden file: `requirements.txt`
The same versions as the `course` environment. `scikit-learn` **must** match `model-card.json`.

```bash
cat > requirements.txt <<'EOF'
scikit-learn==1.9.1
numpy==2.5.4
scipy==1.18.1
joblib==1.6.0
flask==3.1.3
gunicorn==26.2.0
boto3==1.43.115
prometheus-client==0.26.0
EOF
```
:::

:::{dropdown} Hidden file: `tests/test_app.py`
A tiny model is trained inside the test. The pipeline needs neither the model file nor S3.

```bash
cat > tests/test_app.py <<'EOF'
import base64

import numpy as np
import pytest
from sklearn.tree import DecisionTreeClassifier

import app as service

FEATURES = ["tmax_c", "tmin_c", "precip_mm", "rel_humidity", "wind_max_ms", "pressure_hpa"]
ROW = dict(zip(FEATURES, [38.4, 23.1, 0.0, 31, 6.2, 1016]))
AUTH = {"Authorization": "Basic " + base64.b64encode(b"group:secret").decode()}


@pytest.fixture()
def client(monkeypatch):
    monkeypatch.setenv("API_USER", "group")
    monkeypatch.setenv("API_PASSWORD", "secret")
    rng = np.random.default_rng(0)
    model = DecisionTreeClassifier().fit(rng.normal(size=(50, 6)), rng.integers(0, 2, 50))
    bundle = {"model": model, "features": FEATURES, "classes": ["none", "heatwave"], "version": "test"}
    return service.create_app(bundle).test_client()


def test_health(client):
    assert client.get("/health").json == {"status": "ok", "model_version": "test"}


def test_predict_needs_credentials(client):
    assert client.post("/predict", json={"instances": [ROW]}).status_code == 401


def test_predict(client):
    r = client.post("/predict", json={"instances": [ROW, ROW]}, headers=AUTH)
    assert r.status_code == 200 and len(r.json["predictions"]) == 2


def test_bad_input(client):
    assert client.post("/predict", json={"instances": [{"tmax_c": 1}]}, headers=AUTH).status_code == 400


def test_metrics(client):
    client.post("/predict", json={"instances": [ROW]})
    text = client.get("/metrics").get_data(as_text=True)
    assert 'model_api_requests_total{method="POST",path="/predict",status="401"}' in text
EOF
```
:::

:::{dropdown} Hidden instructions: run it on the VM
```bash
cd ~/model-api
python -m pytest -q                                   # 5 passed
mkdir -p model && cp ~/day5/model.joblib model/
API_USER=group API_PASSWORD=dev MODEL_LOCAL_PATH=model/model.joblib \
  gunicorn -b 127.0.0.1:8000 --workers 1 --threads 4 "app:create_app()" &
sleep 3
curl -s localhost:8000/health
curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8000/predict              # 401
curl -s -u group:dev -H 'Content-Type: application/json' \
     -d @$HOME/day5/sample-input.json localhost:8000/predict
# {"model_version":"2026.11-1","predictions":["heatwave","heavy_rain","windstorm","none","heatwave"]}
curl -s localhost:8000/metrics | grep '^model_api_'
kill %1
```

Use `python -m pytest`, not `pytest`. `python -m` puts the current directory on the import
path, so `tests/` can `import app`.

**One gunicorn worker, four threads.** `prometheus_client` keeps its counters inside the
process. With several worker processes, each worker has its own counters, and `/metrics`
answers with those of whichever worker took the scrape. Scale with pods (step 10), not with
workers.
:::

(gp-step-3)=
## Step 3. The container

**Container.** Write a multi-stage `Dockerfile` ([Day 2, section 5.4](../day02/05-application-containerisation.md#id-5-4-multi-stage-builds)). The image:

- contains the code and its dependencies, **not** the model and not the tests;
- runs as a **numeric** non-root user (Kubernetes can only check `runAsNonRoot` against a
  number);
- works with a **read-only** root file system, because the cluster will enforce it.

Test it with Docker Compose. The model file is mounted read-only from `./model`.

:::{dropdown} Hidden file: `Dockerfile`
`--worker-tmp-dir /dev/shm`: gunicorn writes a heartbeat file, and the root file system is
read-only.

```bash
cd ~/model-api
cat > Dockerfile <<'EOF'
FROM python:3.12-slim-bookworm AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir=/build/wheels -r requirements.txt

FROM python:3.12-slim-bookworm AS runner
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
RUN groupadd -g 10001 app && useradd -u 10001 -g app -d /app -s /sbin/nologin app
WORKDIR /app
COPY --from=builder /build/wheels /wheels
RUN pip install --no-cache-dir /wheels/* && rm -rf /wheels
COPY app.py .
USER 10001
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "1", "--threads", "4", \
     "--worker-tmp-dir", "/dev/shm", "--access-logfile", "-", "app:create_app()"]
EOF
```
:::

:::{dropdown} Hidden file: `.dockerignore`
Without it, `COPY` and the build context would also carry the model, your `.env` and Git's
history.

```bash
cat > .dockerignore <<'EOF'
.git
.venv
__pycache__
.pytest_cache
.env
model
*.joblib
tests
EOF
```
:::

:::{dropdown} Hidden files: `compose.yaml` and `.env`
`.env` holds the local test password. It is never committed: `.gitignore` in step 4 excludes it.

```bash
cat > compose.yaml <<'EOF'
services:
  model-api:
    build: .
    image: model-api:dev
    ports: ["8000:8000"]
    env_file: .env
    environment:
      MODEL_LOCAL_PATH: /models/model.joblib
    volumes:
      - ./model:/models:ro
    read_only: true
    tmpfs: [/tmp]
EOF
printf 'API_USER=group\nAPI_PASSWORD=dev\n' > .env
```
:::

:::{dropdown} Hidden instructions: build and check
```bash
docker compose up --build -d
curl -s localhost:8000/health
docker compose exec model-api id                    # uid=10001(app) gid=10001(app)
docker compose exec model-api touch /x              # Read-only file system
docker run --rm model-api:dev find / -name '*.joblib' 2>/dev/null   # nothing: no model in the image
docker image ls model-api:dev                       # about 500 MB: numpy, scipy, scikit-learn
docker compose down
```
:::

(gp-step-4)=
## Step 4. The GitLab projects

**Host.** Create two **private** projects in your personal namespace in the GitLab web UI:

| Project | Holds | Read by |
| --- | --- | --- |
| `model-api-<group>` | the application: code, `Dockerfile`, pipeline | the pipeline, which builds the image |
| `model-api-<group>-deploy` | the Helm chart and `fleet.yaml` | Fleet |

Two projects, as on Day 4. Fleet only needs to read the configuration, and a deployment or a
rollback is one commit in the deploy project.

In both projects:

1. Invite the other members as **Maintainer**.
2. Protect `main`, once it exists (after the first push). The release job of step 5 needs this.

Then push the application. All later changes go through branches and merge requests.

:::{dropdown} Hidden instructions: create, invite, protect
1. GitLab → **New project → Create blank project**.
   - *Project name* `model-api-<group>`.
   - *Project URL*: your username.
   - *Visibility* **Private**.
   - **Untick** *Initialize repository with a README*: you push the first commit yourself.
2. In the project, open **Manage → Members → Invite members**. Choose each member, role
   **Maintainer**.
3. Repeat 1 and 2 for `model-api-<group>-deploy`.
:::

:::{dropdown} Hidden file: `.gitignore`, and the first push
```bash
cd ~/model-api
cat > .gitignore <<'EOF'
.venv/
__pycache__/
.pytest_cache/
.env
model/
*.joblib
EOF
git init -b main
git add . && git status --short     # check: no .env, no model/, no *.joblib
git commit -m "feat: serve the climate-extremes model"
git remote add origin git@gitlab.data.meditwin-project.eu:$APP.git
git push -u origin main
```

Now protect `main` with a branch rule: left sidebar **Settings → Repository**, open the section
**Branch rules** (click its title) and open the rule for `main` (GitLab creates it with the first push;
if it is missing: **Add branch rule → Branch name or pattern**, `main`, **Create branch rule**).
Under **Protect branch**: *Allowed to merge* **Maintainers**, *Allowed to push and merge*
**Maintainers** (each with its **Edit** button), as the release job's token needs. Nobody pushes to
`main` directly anyway: it is the team's rule, and the checklist checks that every change arrived
through a merge request.

The other members move their own copy aside and clone the project:
`mv ~/model-api ~/model-api.mine && git clone git@gitlab.data.meditwin-project.eu:$APP.git ~/model-api`.
They work on branches (`git switch -c feat/...`) and open merge requests. A file of steps 2 and 3
that only a member wrote, and the host did not push, arrives the same way: copied from
`~/model-api.mine` into a branch, then a merge request.
:::

(gp-step-5)=
## Step 5. The pipeline

**Pipeline.** Write `.gitlab-ci.yml` as on Day 4: test → build with Kaniko → scan with Trivy →
release with semantic-release → the version as a second tag on the scanned image, exactly as in Day 4
Hands-on 6.

| Event | Jobs | Result |
| --- | --- | --- |
| A merge request or a branch | `run-unit-tests` | the tests |
| A commit on `main` | tests, `build-container-image`, `security-audit`, `release`, `tag-release-image` | image `:<short-sha>`; if a `feat:` or `fix:` commit arrived: the Git tag `vX.Y.Z`, a GitLab Release with its notes, `CHANGELOG.md` updated, and `:vX.Y.Z` on the same image, which `fleet.yaml` will use |
| The release commit `chore(release): … [skip ci]`, and the tag | none | skipped, on purpose (Day 4, section 5.4) |

Requirements:

- A **fixable CRITICAL** vulnerability stops the pipeline. HIGH ones are listed in a report but
  do not stop it.
- Version numbers come from the commit messages (conventional commits). The changelog is the
  release notes of each GitLab Release.

Before you push the pipeline, the **host** gives semantic-release a token: it must push the
`CHANGELOG.md` commit and the tag, and create the release, and the job's own token cannot push.

:::{dropdown} Hidden instructions: the token for semantic-release
In `model-api-<group>`, as the host:

1. **Settings → Access tokens → Add new token**.
   - Name `semantic-release`, role **Maintainer**.
   - Scopes **`api`** and **`write_repository`**.
   - Copy the value.
2. **Settings → CI/CD → Variables → Add variable**.
   - Key `GITLAB_TOKEN`, value the token.
   - **Masked**, **Protect variable** ticked.

A protected variable only reaches pipelines of **protected** branches. That is why `main` was
protected in step 4. If `main` is not protected, the `release` job fails with
`ENOGLTOKEN No GitLab token specified`, then with `You are not allowed to push code` (403).
:::

:::{dropdown} Hidden file: `.gitlab-ci.yml`
`.kaniko` is a hidden job: `build-container-image` takes its settings with `extends`.

```bash
cd ~/model-api && git switch -c ci/pipeline
cat > .gitlab-ci.yml <<'EOF'
stages: [test, build, scan, release]

variables:
  IMAGE: "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"
  PIP_CACHE_DIR: "${CI_PROJECT_DIR}/.cache/pip"

.kaniko:
  image:
    name: gcr.io/kaniko-project/executor:v1.23.0-debug
    entrypoint: [""]
  before_script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "${CI_REGISTRY_USER}" "${CI_REGISTRY_PASSWORD}" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json

run-unit-tests:
  stage: test
  image: python:3.12-slim-bookworm
  cache:
    key:
      files: [requirements.txt]
    paths: [.cache/pip]
  script:
    - pip install -r requirements.txt pytest
    - python -m pytest --junitxml=report.xml
  artifacts:
    when: always
    expire_in: 1 week
    reports:
      junit: report.xml
  rules:
    - if: $CI_COMMIT_BRANCH

build-container-image:
  extends: .kaniko
  stage: build
  needs: [run-unit-tests]
  script:
    - /kaniko/executor --context "$CI_PROJECT_DIR" --dockerfile "$CI_PROJECT_DIR/Dockerfile"
      --destination "$IMAGE" --cache=true
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

security-audit:
  stage: scan
  needs: [build-container-image]
  image:
    name: aquasec/trivy:0.69.3
    entrypoint: [""]
  variables:
    TRIVY_USERNAME: "$CI_REGISTRY_USER"
    TRIVY_PASSWORD: "$CI_REGISTRY_PASSWORD"
    TRIVY_CACHE_DIR: "$CI_PROJECT_DIR/.cache/trivy"
  cache:
    key: trivy-db
    paths: [.cache/trivy]
  script:
    - trivy image --no-progress --severity HIGH,CRITICAL --format table --output trivy-report.txt "$IMAGE"
    # The project's gate is stricter on what it lists than on what it stops: only a fixable CRITICAL
    # fails the job (Day 4 gates on HIGH too).
    - trivy image --no-progress --severity CRITICAL --ignore-unfixed --exit-code 1 "$IMAGE"
  artifacts:
    when: always
    expire_in: 1 week
    paths: [trivy-report.txt]
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# As in Day 4, Hands-on 6 (section 5.6 explains every line).
release:
  stage: release
  needs: [security-audit]
  image: node:20
  variables:
    GIT_DEPTH: 0
  before_script:
    - git fetch --tags --force
    - npm install -g semantic-release @semantic-release/gitlab
        @semantic-release/changelog @semantic-release/git
  script:
    - git tag --list > tags-before.txt
    - semantic-release
    - echo "RELEASE_TAG=$(git tag --list | grep -vxFf tags-before.txt)" > release.env
  artifacts:
    reports:
      dotenv: release.env
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

tag-release-image:
  stage: release
  needs: [release]
  image:
    name: gcr.io/go-containerregistry/crane/debug:v0.22.1
    entrypoint: [""]
  script:
    - if [ -z "$RELEASE_TAG" ]; then echo "No release in this pipeline"; exit 0; fi
    - crane auth login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - crane tag "$IMAGE" "$RELEASE_TAG"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
EOF
```
:::

:::{dropdown} Hidden file: `.releaserc.json`, then the merge request
The same as Day 4 Hands-on 6: semantic-release writes `CHANGELOG.md`, commits it to `main` with
`[skip ci]`, tags that commit `vX.Y.Z` and creates the GitLab Release. `[skip ci]` skips the
pipelines of that commit and of the tag; `tag-release-image`, in the pipeline that made the release,
has already put `vX.Y.Z` on the scanned image.

```bash
cat > .releaserc.json <<'EOF'
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", { "changelogFile": "CHANGELOG.md" }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md"],
      "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
    }],
    "@semantic-release/gitlab"
  ]
}
EOF
git add .gitlab-ci.yml .releaserc.json
git commit -m "ci: test, build, scan and release"
git push -u origin ci/pipeline        # open the merge request; a member reviews and merges it
```

After the merge, the pipeline on `main` runs. semantic-release reads every commit since the
last tag. There is no tag yet, so it finds the `feat:` commit of step 4 and makes `v1.0.0`. Later,
`feat:` makes the next minor version, `fix:` the next patch, and `ci:`, `docs:` or `chore:` make
none. Then check:

- **Deploy → Releases**: `v1.0.0` with its notes, and `CHANGELOG.md` in the repository.
- **Build → Pipelines**: in the pipeline of the merge, `tag-release-image` ran after `release`; the
  pipelines of `chore(release): 1.0.0 [skip ci]` and of the tag `v1.0.0` are **skipped**.
- **Deploy → Container Registry**: the tags `<short-sha>` and `v1.0.0`, with the same digest.
- `security-audit` → **Browse** artifacts → `trivy-report.txt`: the HIGH and CRITICAL list.

To prove the gate, put `FROM python:3.12.0-slim-bookworm AS runner` in a branch: the first
Python 3.12 image, from 2023, whose system libraries have fixable CRITICAL vulnerabilities (13 when
this was written). Merge it in a throw-away merge request and watch `security-audit` fail and
`release` not run. Then revert it.
:::

(gp-step-6)=
## Step 6. The Helm chart

**Platform.** Write a chart in the deploy project with:

| Object | Requirements |
| --- | --- |
| Deployment | the release image; the three Secrets of step 7 as environment; non-root, read-only root file system, no service account token; requests and limits; probes on `/health`, with a startup probe that waits for the model download; **no `replicas:`**, because the HPA owns the number |
| ConfigMap | bucket, key, S3 endpoint, region |
| Service | port 80 → the container's `http` port |
| Ingress | `https://model-<group>.training.meditwin-project.eu`, TLS from cert-manager; **only** `/predict` and `/health`; **HTTPS only**, so the password is never sent in clear |
| HorizontalPodAutoscaler | 2 to 5 pods, 60 % CPU |
| ServiceMonitor | Prometheus scrapes `/metrics` every 15 s. This cluster's Prometheus watches every namespace and needs no extra label |
| Dashboard ConfigMap | label `grafana_dashboard: "1"`: Grafana imports it by itself |

`fleet.yaml` sets the namespace, the release name and the **image tag**. A deployment is a
change of that one line.

:::{dropdown} Hidden instructions: the deploy project
As the host (or any member, then push through a merge request), create the files with the
hidden blocks below, then:

```bash
cd ~/model-api-deploy
sed -i "s|__APP__|$APP|; s|__BUCKET__|$BUCKET|; s|__APP_HOST__|$APP_HOST|; s|__NS__|$NS|" helm/values.yaml helm/fleet.yaml
helm lint helm --set image.tag=v1.0.0
helm template model-api helm -n $NS --set image.tag=v1.0.0 | kubectl apply --dry-run=server -f -
git init -b main && git add . && git commit -m "feat: deploy model-api v1.0.0"
git remote add origin git@gitlab.data.meditwin-project.eu:$DEPLOY.git && git push -u origin main
```

`--dry-run=server` sends the objects to the cluster, which checks them and your permissions,
and changes **nothing**. It is not a deployment. The other members cannot run it, because the
namespace is the host's.
:::

:::{dropdown} Hidden files: `Chart.yaml`, `values.yaml`, `fleet.yaml`
`__APP__`, `__BUCKET__`, `__APP_HOST__` and `__NS__` are replaced by the `sed` above.

```bash
mkdir -p ~/model-api-deploy/helm/templates ~/model-api-deploy/helm/files && cd ~/model-api-deploy/helm
cat > Chart.yaml <<'EOF'
apiVersion: v2
name: model-api
version: 0.1.0
appVersion: "1.0.0"
EOF
cat > values.yaml <<'EOF'
image:
  repository: gitlab.data.meditwin-project.eu:6443/__APP__
  tag: ""                        # set in fleet.yaml
  pullSecret: model-api-registry
model:
  bucket: __BUCKET__
  key: model.joblib
host: __APP_HOST__
resources:
  requests: {cpu: 100m, memory: 256Mi}
  limits: {memory: 512Mi}
autoscaling: {minReplicas: 2, maxReplicas: 5, cpuPercent: 60}
EOF
cat > fleet.yaml <<'EOF'
defaultNamespace: __NS__
helm:
  releaseName: model-api
  values:
    image:
      tag: v1.0.0
EOF
```
:::

:::{dropdown} Hidden files: `templates/configmap.yaml`, `deployment.yaml`, `service.yaml`
```bash
cd ~/model-api-deploy/helm/templates
cat > configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: model-api
data:
  MODEL_BUCKET: {{ .Values.model.bucket | quote }}
  MODEL_KEY: {{ .Values.model.key | quote }}
  AWS_ENDPOINT_URL: https://s3.training.meditwin-project.eu
  AWS_DEFAULT_REGION: us-east-1
EOF
cat > deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-api
  labels: {app: model-api}
spec:
  selector:
    matchLabels: {app: model-api}
  template:
    metadata:
      labels: {app: model-api}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      automountServiceAccountToken: false
      imagePullSecrets:
        - name: {{ .Values.image.pullSecret }}
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile: {type: RuntimeDefault}
      containers:
        - name: model-api
          image: "{{ .Values.image.repository }}:{{ required "image.tag" .Values.image.tag }}"
          ports:
            - {name: http, containerPort: 8000}
          envFrom:
            - configMapRef: {name: model-api}
            - secretRef: {name: model-api-s3}
            - secretRef: {name: model-api-auth}
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: {drop: [ALL]}
          volumeMounts:
            - {name: tmp, mountPath: /tmp}
          startupProbe:
            httpGet: {path: /health, port: http}
            periodSeconds: 3
            failureThreshold: 40
          readinessProbe:
            httpGet: {path: /health, port: http}
          livenessProbe:
            httpGet: {path: /health, port: http}
            periodSeconds: 20
          resources: {{- toYaml .Values.resources | nindent 12 }}
      volumes:
        - {name: tmp, emptyDir: {}}
EOF
cat > service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: model-api
  labels: {app: model-api}
spec:
  selector: {app: model-api}
  ports:
    - {name: http, port: 80, targetPort: http}
EOF
```

`checksum/config` changes when the ConfigMap changes, so the pods restart and read the new
values.
:::

:::{dropdown} Hidden files: `templates/ingress.yaml`, `hpa.yaml`
```bash
cd ~/model-api-deploy/helm/templates
cat > ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: model-api
  annotations:
    cert-manager.io/cluster-issuer: ecmwf-meditwin-training-issuer
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
    - hosts: [{{ .Values.host | quote }}]
      secretName: model-api-tls
  rules:
    - host: {{ .Values.host | quote }}
      http:
        paths:
          {{- range list "/predict" "/health" }}
          - path: {{ . }}
            pathType: Exact
            backend:
              service:
                name: model-api
                port: {name: http}
          {{- end }}
EOF
cat > hpa.yaml <<'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: model-api
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: model-api}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: {type: Utilization, averageUtilization: {{ .Values.autoscaling.cpuPercent }}}
  behavior:
    scaleDown: {stabilizationWindowSeconds: 60}
EOF
```

- `router.entrypoints: websecure`: Traefik serves the routes on port 443 only. On port 80
  they answer `404`. cert-manager's HTTP-01 check still works: it uses its own temporary
  Ingress.
- `scaleDown.stabilizationWindowSeconds: 60`: the pods go back down one minute after the load
  stops, instead of five. You see both directions in the demonstration.
:::

:::{dropdown} Hidden files: `templates/servicemonitor.yaml`, `dashboard.yaml`, `files/dashboard.json`
The dashboard is a plain JSON file. Helm does **not** template it: its legends contain `{{ }}`,
which Helm would try to execute. The template only replaces `__NS__` with the namespace.

```bash
cd ~/model-api-deploy/helm
cat > templates/servicemonitor.yaml <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: model-api
spec:
  selector:
    matchLabels: {app: model-api}
  endpoints:
    - {port: http, path: /metrics, interval: 15s}
EOF
cat > templates/dashboard.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: model-api-dashboard
  labels:
    grafana_dashboard: "1"
data:
  model-api-{{ .Release.Namespace }}.json: |-
{{ .Files.Get "files/dashboard.json" | replace "__NS__" .Release.Namespace | indent 4 }}
EOF
cat > files/dashboard.json <<'EOF'
{
 "uid": "model-api-__NS__",
 "title": "Model API - __NS__",
 "schemaVersion": 39,
 "refresh": "10s",
 "time": {
  "from": "now-30m",
  "to": "now"
 },
 "panels": [
  {
   "id": 1,
   "type": "timeseries",
   "title": "Requests per second, by status",
   "datasource": {
    "type": "prometheus",
    "uid": "prometheus"
   },
   "gridPos": {
    "x": 0,
    "y": 0,
    "w": 12,
    "h": 8
   },
   "fieldConfig": {
    "defaults": {
     "unit": "reqps"
    },
    "overrides": []
   },
   "targets": [
    {
     "refId": "A",
     "datasource": {
      "type": "prometheus",
      "uid": "prometheus"
     },
     "expr": "sum by (status) (rate(model_api_requests_total{namespace=\"__NS__\", path=\"/predict\"}[1m]))",
     "legendFormat": "{{status}}"
    }
   ]
  },
  {
   "id": 2,
   "type": "timeseries",
   "title": "Prediction latency (p95)",
   "datasource": {
    "type": "prometheus",
    "uid": "prometheus"
   },
   "gridPos": {
    "x": 12,
    "y": 0,
    "w": 12,
    "h": 8
   },
   "fieldConfig": {
    "defaults": {
     "unit": "s"
    },
    "overrides": []
   },
   "targets": [
    {
     "refId": "A",
     "datasource": {
      "type": "prometheus",
      "uid": "prometheus"
     },
     "expr": "histogram_quantile(0.95, sum by (le) (rate(model_api_request_duration_seconds_bucket{namespace=\"__NS__\", path=\"/predict\"}[1m])))",
     "legendFormat": "p95"
    }
   ]
  },
  {
   "id": 3,
   "type": "timeseries",
   "title": "Errors per second",
   "datasource": {
    "type": "prometheus",
    "uid": "prometheus"
   },
   "gridPos": {
    "x": 0,
    "y": 8,
    "w": 12,
    "h": 8
   },
   "fieldConfig": {
    "defaults": {
     "unit": "reqps"
    },
    "overrides": []
   },
   "targets": [
    {
     "refId": "A",
     "datasource": {
      "type": "prometheus",
      "uid": "prometheus"
     },
     "expr": "sum(rate(model_api_requests_total{namespace=\"__NS__\", status=~\"5..\"}[1m]))",
     "legendFormat": "5xx"
    },
    {
     "refId": "B",
     "datasource": {
      "type": "prometheus",
      "uid": "prometheus"
     },
     "expr": "sum(rate(model_api_auth_failures_total{namespace=\"__NS__\"}[1m]))",
     "legendFormat": "401"
    }
   ]
  },
  {
   "id": 4,
   "type": "timeseries",
   "title": "Pods (HPA)",
   "datasource": {
    "type": "prometheus",
    "uid": "prometheus"
   },
   "gridPos": {
    "x": 12,
    "y": 8,
    "w": 12,
    "h": 8
   },
   "fieldConfig": {
    "defaults": {
     "unit": "short"
    },
    "overrides": []
   },
   "targets": [
    {
     "refId": "A",
     "datasource": {
      "type": "prometheus",
      "uid": "prometheus"
     },
     "expr": "max(kube_horizontalpodautoscaler_status_current_replicas{namespace=\"__NS__\", horizontalpodautoscaler=\"model-api\"})",
     "legendFormat": "replicas"
    }
   ]
  }
 ]
}
EOF
```
:::

(gp-step-7)=
## Step 7. Deploy with Fleet

**Host only.** The pods need three Secrets that must not be in Git. Create them once, with
`kubectl`; these are the only manual changes to the cluster.

| Secret | Contents |
| --- | --- |
| `model-api-s3` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`: your S3 keys |
| `model-api-auth` | `API_USER`, `API_PASSWORD`: the service's credentials |
| `model-api-registry` | a deploy token of `model-api-<group>` with scope `read_registry` |

The registry Secret has its **own name**. If you reuse `gitlab-registry` from Day 4, you replace
the token your Day 4 deployment uses, and that deployment can no longer pull its image.

Then register the deploy project as a GitRepo in **your** Fleet workspace ([Day 4, section 7](../day04/07-fleet.md)).

:::{dropdown} Hidden instructions: the three Secrets
```bash
source ~/day5.env
kubectl -n $NS create secret generic model-api-s3 \
  --from-literal=AWS_ACCESS_KEY_ID="$(awk -F' = ' '/^access_key/{print $2}' ~/.s3cfg)" \
  --from-literal=AWS_SECRET_ACCESS_KEY="$(awk -F' = ' '/^secret_key/{print $2}' ~/.s3cfg)"
kubectl -n $NS create secret generic model-api-auth \
  --from-literal=API_USER=$GROUP --from-literal=API_PASSWORD="$(openssl rand -base64 18)"
```

For the registry token, open `model-api-<group>` → **Settings → Repository → Deploy tokens →
Add token**: name `cluster-pull`, scope **`read_registry`**. Then:

```bash
read -rp 'Deploy token username: ' u; read -rsp 'Deploy token: ' t; echo
kubectl -n $NS create secret docker-registry model-api-registry \
  --docker-server=$REG --docker-username="$u" --docker-password="$t"
unset t
```

Give the API password to the members of your group (not in Git, not in a merge request):

```bash
kubectl -n $NS get secret model-api-auth -o jsonpath='{.data.API_PASSWORD}' | base64 -d; echo
```
:::

:::{dropdown} Hidden instructions: the GitRepo
1. In `model-api-<group>-deploy`, open **Settings → Repository → Deploy tokens → Add token**:
   name `fleet`, scope **`read_repository`**.
2. Rancher: **☰** (top left) → **Continuous Delivery**; workspace `fleet-<host>` in the drop-down at the top right;
   left menu **Resources → Git Repos** → **Add Repository**. In the form, as in Day 4 Hands-on 7:
   - **Metadata**: Name `model-api`.
   - **Repository details**: Repository URL
     `https://gitlab.data.meditwin-project.eu/<host>/model-api-<group>-deploy.git`, branch `main`
     (replace `master`), **Add Path**, then `helm` in **Root directory**.
   - **Target details**: leave *All Clusters in the workspace*.
   - **Advanced**: Git Authentication, *Create an HTTP Basic Auth Secret* with the token's username
     and value; tick **Enable self-healing**. Then **Create**, in the form: not after **Edit as
     YAML**, which drops the token.
3. Wait one to three minutes, then check:

```bash
kubectl -n $NS get deploy,pods,hpa,ingress,servicemonitor
kubectl -n $NS logs deploy/model-api | head      # downloading s3://… then: model 2026.11-1 loaded
kubectl -n $NS get certificate                    # model-api-tls  READY True (1–2 min)
```
:::

(gp-step-8)=
## Step 8. Check the security

**Everyone.** Show each of these with a command. Load the API password first (from the host):
`read -rsp 'API password: ' API_PASSWORD; export API_PASSWORD; export API_USER=$GROUP`.

| Check | Expected |
| --- | --- |
| `/health` over HTTPS, no password | `200` |
| `/predict` without, or with a wrong password | `401` |
| `/predict` with the password and `sample-input.json` | five labels |
| `/metrics` from outside | `404` (not routed) |
| `http://` (port 80) | `404`: there is no route in clear |
| The certificate | issued by Let's Encrypt, valid |
| Git history | no password, no key, no model |

:::{dropdown} Hidden instructions: step 8
```bash
curl -s https://$APP_HOST/health
curl -s -o /dev/null -w '%{http_code}\n' -X POST https://$APP_HOST/predict
curl -s -o /dev/null -w '%{http_code}\n' -u $API_USER:wrong -X POST https://$APP_HOST/predict
curl -s -u "$API_USER:$API_PASSWORD" -H 'Content-Type: application/json' \
     -d @$HOME/day5/sample-input.json https://$APP_HOST/predict
curl -s -o /dev/null -w '%{http_code}\n' https://$APP_HOST/metrics
curl -s -o /dev/null -w '%{http_code}\n' http://$APP_HOST/predict
echo | openssl s_client -connect $APP_HOST:443 -servername $APP_HOST 2>/dev/null | openssl x509 -noout -issuer -dates
git -C ~/model-api log --all --name-only --format= | grep -c '\.joblib$'     # 0: no model ever committed
git -C ~/model-api log -p --all | grep -ciE 'secret_key|aws_secret|api_password='   # 0
```
:::

(gp-step-9)=
## Step 9. The dashboard

**Everyone.** Open `https://grafana.training.meditwin-project.eu` and choose **Sign in with
GitLab**: your training GitLab account. You are a *Viewer*. You can explore and look, but not
save.

1. In **Explore**, check that Prometheus scrapes your two pods, and query the request rate.
2. Open the dashboard **Model API - `<namespace>`**, which the chart created (step 6). It
   shows the request rate by status, the p95 latency, the errors (5xx and 401) and the number
   of pods.

:::{dropdown} Hidden instructions: queries for Explore
Replace `<ns>` with the host's account.

```promql
up{namespace="<ns>", service="model-api"}
sum by (status) (rate(model_api_requests_total{namespace="<ns>", path="/predict"}[1m]))
histogram_quantile(0.95, sum by (le) (rate(model_api_request_duration_seconds_bucket{namespace="<ns>", path="/predict"}[1m])))
```

`up` must show 2 series with value 1. Run step 8's `curl` commands again, and the `401` line
appears in the errors panel.
:::

(gp-step-10)=
## Step 10. Load and autoscaling

**Everyone.** Send a constant load to `/predict` for five minutes. The HPA must add pods: more
than 60 % of the requested CPU, on average, triggers it. Watch it in Grafana; the host can also
watch it with `kubectl`. After the load stops, the HPA goes back to 2 pods within about a
minute.

:::{dropdown} Hidden instructions: step 10
20 requests in parallel, 30 000 in total. At the end, the count of each status code is printed:

```bash
seq 30000 | xargs -P 20 -I{} curl -s -o /dev/null -w '%{http_code}\n' \
  -u "$API_USER:$API_PASSWORD" -H 'Content-Type: application/json' \
  -d @$HOME/day5/sample-input.json https://$APP_HOST/predict | sort | uniq -c
```

The host, in a second terminal:

```bash
watch -n5 "kubectl -n $NS get hpa model-api; kubectl -n $NS top pods"
```

Expect the HPA's CPU to go above 60 % and the pods to go from 2 toward 5 within one to three
minutes. Then the request rate goes up and the latency goes down, as the new pods take traffic.
:::

## 3.3 The demonstration

Each group shows, in this order:

1. **A prediction over HTTPS**: the `401` without the password, then the five labels with it.
2. **A change from a commit to production.**
   1. A `feat:` merge request in the application.
   2. Its merge.
   3. The pipeline and the new release `vX.Y.Z` with its notes.
   4. A merge request in the deploy project that changes `tag:` in `fleet.yaml`.
   5. Fleet rolls the pods onto the new image, and what the `feat:` changed answers with `curl`.
3. **A rollback with `git revert`** of that deploy commit. Fleet goes back to the previous image.
4. **The dashboard during the load test** of step 10.

:::{dropdown} Hidden instructions: change and rollback in the deploy project
```bash
cd ~/model-api-deploy && git switch main && git pull
git switch -c deploy/v1.1.0
sed -i 's/tag: v1.0.0/tag: v1.1.0/' helm/fleet.yaml
git commit -am "chore: deploy model-api v1.1.0" && git push -u origin deploy/v1.1.0
# merge request → review → merge; then:
kubectl -n $NS rollout status deploy/model-api     # the host
kubectl -n $NS get deploy model-api -o jsonpath='{..image}{"\n"}'   # ...:v1.1.0

# rollback, also through a merge request. The merge created a merge commit: -m 1 reverts it
# relative to main as it was before (Day 4, Hands-on 9).
git switch main && git pull
git switch -c revert/v1.1.0
git revert -m 1 --no-edit HEAD && git push -u origin revert/v1.1.0
# merge request → review → merge
```
:::

## 3.4 Evaluation checklist

- [ ] The model is not in the Git repository or in the image.
- [ ] No secret is in the Git repository.
- [ ] The container does not run as root.
- [ ] The pipeline stops on a fixable critical vulnerability.
- [ ] Each release has a version and release notes.
- [ ] Fleet does the deployment. Nobody changed the cluster manually, except the three Secrets.
- [ ] The endpoint uses HTTPS only and `/predict` needs authentication.
- [ ] The dashboard shows the metrics of the service.
- [ ] The HPA scales the service under load, and back down after it.
- [ ] After the first push, every change reached `main` through a reviewed merge request, apart from
  semantic-release's own `chore(release)` commits.

## 3.5 Problems and solutions

:::{dropdown} `release` fails: `ENOGLTOKEN No GitLab token specified`, then `403 You are not allowed to push code`
The job has no `GITLAB_TOKEN`. Either the variable does not exist, or it is *Protected* and
`main` is not a protected branch. Protect `main` (step 4), then run a **new** pipeline on
`main`: left sidebar **Build → Pipelines**, **New pipeline** (top right), branch `main`, **New pipeline**.
:::

:::{dropdown} The release `vX.Y.Z` exists, but the registry has no image `:vX.Y.Z`
Open `tag-release-image` in the `main` pipeline that made the release (as in Day 4, Hands-on 6):

- `No release in this pipeline`: `RELEASE_TAG` arrived empty. Check the `echo "RELEASE_TAG=…"`
  line and the `dotenv` report in `release`, and `needs: [release]` in `tag-release-image`.
- `MANIFEST_UNKNOWN`: the `<short-sha>` image does not exist: `build-container-image` did not
  push it in this pipeline.

Do not wait for a pipeline on the tag: the release commit carries `[skip ci]`, so it is skipped.
:::

:::{dropdown} The pods stay in `ImagePullBackOff`
`kubectl -n $NS describe pod -l app=model-api | grep -A3 Events`. Usually one of these:

- `model-api-registry` is missing.
- Its deploy token belongs to the other project, or lacks `read_registry`.
- `tag:` in `fleet.yaml` names a version that `tag-release-image` has not tagged yet.
:::

:::{dropdown} The pods restart (`CrashLoopBackOff`) and the log shows an S3 error
`kubectl -n $NS logs deploy/model-api --previous`.

| Error | Cause |
| --- | --- |
| `InvalidAccessKeyId` | `AWS_ENDPOINT_URL` is missing: boto3 went to Amazon |
| `AccessDenied` / `403` | the keys do not belong to the bucket's owner, the host |
| `404` / `NoSuchKey` | `model.joblib` is not in the bucket, or `model.key` is wrong |
| `InconsistentVersionWarning`, then an error | scikit-learn in `requirements.txt` is not the version in `model-card.json` |
:::

:::{dropdown} The Day 4 deployment stopped working after step 7
The Day 4 deployment pulls with the Secret `gitlab-registry`. If you replaced it with this
project's token, create it again with a deploy token of `climate-telemetry-<you>`, as on Day 4.
:::

:::{dropdown} Grafana: no "Sign in with GitLab", or the login is refused
Sign in with your **training** GitLab account (`traineeNN`). Other GitLab accounts are not
admitted. If GitLab asks you to authorise the application, accept.
:::
