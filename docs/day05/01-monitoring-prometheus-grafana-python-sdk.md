# 1. Monitoring with Prometheus, Grafana, and the Python SDK

Your service runs in the cluster. The probes of Day 3 tell Kubernetes *whether* a pod is alive.
They do not tell **you** how the service behaves. This section answers one question, all the
time:

> **Is my application healthy right now, and how is that changing over time?**

The tools are **Prometheus** and **Grafana**, two open-source projects that are used together very
often, and are the usual monitoring stack on Kubernetes. Prometheus, like Kubernetes, is a
project of the Cloud Native Computing Foundation (CNCF).

You add metrics to a Python service, let Prometheus collect them, and look at them in Grafana.
You use the same tools in the group project (step 9 and step 10).

:::{important} What this section is not
It is not a full observability course. It covers **metrics** only: numbers that change over time.
Logs and traces are mentioned in 1.1, but you do not set them up. **Alerting** is not covered
either: in production, Prometheus also checks rules all the time (for example "more than 5 % of
the requests fail for 5 minutes"), and **Alertmanager** sends a message by e-mail, chat or pager
when a rule is true. The PromQL you learn here is the same language those rules use.
:::

**Symbols in this section:** ⌨️ you type the commands · 📖 reference material: read it after the
lesson, or when you need it. The lesson does not cover it in class.

**Slides:** the slides used in class are in one file, {download}`day05-monitoring-slides.pdf`.

## 1.1 Why monitoring is necessary

Without metrics, you find problems when a user tells you, and you investigate by guessing. With
metrics, you can see:

- **Traffic**: how many requests arrive.
- **Errors**: how many of them fail.
- **Latency**: how long they take.
- **Saturation**: how full the resources are (CPU, memory, connections).

These four are the *golden signals*. For a service that answers requests, start with three of them,
the **RED method**:

| Letter | Signal | Example question |
| --- | --- | --- |
| **R**ate | Requests per second | "Is anybody using `/stats`?" |
| **E**rrors | Failed requests per second, or the fraction that fails | "Did the last release break something?" |
| **D**uration | The distribution of response times | "Is the slowest 5 % of requests too slow?" |

:::{note} Jargon: *observability* and its three pillars
**Observability** is how well you can understand the inside of a system from what it shows outside.
It has three usual sources of data:

- **Metrics**: numbers, collected at regular intervals. They are cheap and good for trends and
  alerts. *This section.*
- **Logs**: one line of text for each event. They are good for "what exactly happened". You used
  `kubectl logs` on Day 3.
- **Traces**: the path of one request through many services. They are good for "where is the time
  spent". The usual tool is OpenTelemetry.
:::

### Probes and monitoring compared

| | Probes (Day 3) | Monitoring (this section) |
| --- | --- | --- |
| Who asks | The kubelet | Prometheus |
| Question | "Is this pod alive? Is it ready?" | "How many, how fast, how many failures, since when?" |
| Result | Yes or no, now | A number, stored over time |
| Action | Restart the pod, or remove it from the Service | A graph for you, or an alert |

A pod can pass every probe and still answer 30 % of the requests with `500`. Only monitoring
shows that.

**Further reading:** Cindy Sridharan,
[*Monitoring in the time of Cloud Native*](https://copyconstruct.medium.com/monitoring-in-the-time-of-cloud-native-c87c7a5bfa3e)
(Velocity New York, 2017). It explains why a platform such as Kubernetes makes monitoring your own
application more important, not less, and how monitoring differs from observability.

## 1.2 How Prometheus works

**Prometheus** collects metrics and stores them. **Grafana** shows them in graphs and dashboards.

Prometheus uses a **pull** model. Your application does not send its metrics anywhere. It shows
them as text on an HTTP path, usually `/metrics`. Every few seconds, Prometheus reads that path
from each pod. This read is called a **scrape**.

```{mermaid}
flowchart LR
    U([You]) -->|asks| G[Grafana<br/>dashboards, Explore]
    G -->|PromQL| P[Prometheus<br/>stores the time series]
    P -->|"GET /metrics, every 15 s"| P1[pod 1]
    P -->|"GET /metrics, every 15 s"| P2[pod 2]
    P -->|"GET /metrics, every 15 s"| K[kube-state-metrics]
    SM[ServiceMonitor<br/>in your namespace] -.->|what to scrape| P
```

Read the arrows from left to right: you ask Grafana, Grafana asks Prometheus, and Prometheus asks
the pods. The pods never send anything by themselves.

The opposite model, **push**, also exists, but it is the exception. A short batch job can end
before the next scrape, so nobody could read its metrics. Such a job pushes them to a
**Pushgateway**, and Prometheus scrapes the Pushgateway. For a service that runs all the time,
such as an API, use pull.

On the training cluster:

- The platform team installed **Prometheus**, **Grafana** and **kube-state-metrics** in a
  namespace of its own. These three are often installed together, for example with the Helm chart
  `kube-prometheus-stack`. You use them; you do not install them. Prometheus watches **every**
  namespace.
- You tell Prometheus what to scrape with a **ServiceMonitor**: a small object in your namespace
  that selects your Service by label (1.7).
- **kube-state-metrics** turns the state of Kubernetes objects into metrics: the number of
  replicas, the restarts of each container, the requests and limits. Your own application does
  not need to export these values: you query them in 1.11.

:::{note} Jargon: *target*, *scrape*, *time series*
A **target** is one endpoint that Prometheus scrapes: in Kubernetes, one pod. A **scrape** is one
read of its `/metrics`. Each value that Prometheus stores belongs to a **time series**: a metric
name plus a set of labels, with one value for each scrape.
:::

### Metric name and labels

A time series is identified by its **name** and its **labels**:

```text
climate_api_http_requests_total{path="/stats", status="200", namespace="trainee07", pod="climate-api-7d9…"}  1532
└──────────── name ───────────┘└───────────────────────────── labels ──────────────────────────────┘  value
```

Some labels come from your code (`path`, `status`). Prometheus adds others when it scrapes, to say
where the value came from:

| Label added by Prometheus | Value |
| --- | --- |
| `job`, `service` | The Service name |
| `namespace`, `pod`, `container` | Where the pod runs |
| `instance` | The pod's IP address and port |
| `endpoint` | The name of the Service port (for example `http`) |
| `up` | Not a label: a metric, `1` if the last scrape worked, `0` if not |

:::{warning} Do not use these names for your own labels
If your code also writes a label called `endpoint`, `pod`, `namespace` or `instance`, Prometheus
keeps its own value and renames yours to `exported_endpoint`, `exported_pod`, and so on. Your
queries then find nothing. Use names such as `path`, `route` or `handler`.
:::

## 1.3 The four types of metric

The Python SDK, `prometheus_client`, has four types. Choose the type from the question you want to
answer.

| Type | It can | Use it for | Example |
| --- | --- | --- | --- |
| **Counter** | Only go up (and go back to 0 when the process restarts) | Things that happen: requests, errors, refused logins | `http_requests_total` |
| **Gauge** | Go up and down | A current state: a queue length, rows in a table, temperature | `queue_length` |
| **Histogram** | Count values in buckets | A distribution: latency, size of a request | `http_request_duration_seconds` |
| **Summary** | Calculate quantiles in the process | Rarely: its quantiles cannot be combined across pods | — |

Prefer a **histogram** to a summary for latency. Prometheus can add histograms from many pods
together and then calculate the 95th percentile of the whole service. It cannot do that with the
quantiles of a summary.

### Naming rules

- Put the **unit** in the name, in base units: `_seconds`, `_bytes`, not `_ms` or `_kb`.
- End a counter with **`_total`**.
- Start with the name of your application: `climate_api_…`, `model_api_…`.
- Never change a name between releases. Dashboards and alerts use it.

### What `/metrics` looks like

```text
# HELP demo_http_requests_total HTTP requests
# TYPE demo_http_requests_total counter
demo_http_requests_total{method="GET",path="/",status="200"} 100.0
# HELP demo_http_request_duration_seconds Request duration
# TYPE demo_http_request_duration_seconds histogram
demo_http_request_duration_seconds_bucket{path="/",le="0.005"} 97.0
demo_http_request_duration_seconds_bucket{path="/",le="0.01"} 100.0
…
demo_http_request_duration_seconds_bucket{path="/",le="+Inf"} 100.0
demo_http_request_duration_seconds_count{path="/"} 100.0
demo_http_request_duration_seconds_sum{path="/"} 0.162
```

One histogram makes several series:

- `_bucket{le="0.01"}`: how many requests took **0.01 s or less**. The buckets are cumulative.
- `_count`: how many requests in total.
- `_sum`: the total time of all of them. `_sum / _count` is the mean.

## 1.4 Labels and cardinality

In 1.2 you saw that a time series is a name plus labels. Each different combination of label
values is a separate time series. Prometheus keeps every series in memory. The number of series is
the **cardinality**.

```text
method (2) × path (4) × status (5) × pods (2)              =     80 series   good
method (2) × raw URL (every scanner probe, for ever) × …   = 100 000 series  Prometheus stops
```

A histogram multiplies the series again. With the default buckets of `prometheus_client`, each
combination of labels makes 15 `_bucket` series, plus `_count` and `_sum` (1.3). This is why a
histogram usually has fewer labels than the counter next to it: in Hands-on 1, the counter has
`method`, `path` and `status`, the histogram only `path`.

Rules:

- A label value must come from a **small, fixed set**: a route name, a status code, a method.
- **Never** put these in a label: a user ID, an e-mail address, a request ID, a raw URL with its
  query string, a timestamp.
- Map unknown values to one value, such as `other`. In Hands-on 1, the set `KNOWN_PATHS` does this
  for the `path` label.

## 1.5 ⌨️ Hands-on 1: metrics on your VM

Before you use the cluster, look at metrics on your own VM. You need no image and no Prometheus:
only `curl`.

### Step 1: check the Python environment

The `course` Python environment has every package of this section, `flask` and
`prometheus-client` included. It is normally active when you log in: the prompt starts with
`(course)`. Check it:

```bash
which python                 # ~/venvs/course/bin/python
python -c "import flask, prometheus_client; print('ok')"
```

If `which python` shows another path, or the import fails, activate the environment:

```bash
source ~/venvs/course/bin/activate
```

Do the same in every new terminal that runs Python.

### Step 2: create a working folder

```bash
mkdir -p ~/day5-metrics
cd ~/day5-metrics
```

### Step 3: write the service

Open a new file `demo.py` in this folder with your editor: VS Code (Remote - SSH, Day 1
section 0) or `nano demo.py`. Write the code below. Read each part before you go on: the table
after the code explains it.

```python
"""A small Flask service with Prometheus metrics: /, /slow, /fail and /metrics."""
import random
import time

from flask import Flask, Response, g, request
from prometheus_client import CONTENT_TYPE_LATEST, Counter, Histogram, generate_latest

REQUESTS = Counter("demo_http_requests_total", "HTTP requests", ["method", "path", "status"])
LATENCY = Histogram("demo_http_request_duration_seconds", "Request duration", ["path"])
KNOWN_PATHS = {"/", "/slow", "/fail", "/metrics"}     # anything else is counted as "other"

app = Flask(__name__)


@app.before_request
def start_timer():
    g.t0 = time.perf_counter()


@app.after_request
def count(resp):
    path = request.path if request.path in KNOWN_PATHS else "other"
    REQUESTS.labels(request.method, path, str(resp.status_code)).inc()
    LATENCY.labels(path).observe(time.perf_counter() - g.t0)
    return resp


@app.get("/")
def home():
    return "hello from the metrics demo\n"


@app.get("/slow")
def slow():
    time.sleep(random.uniform(0.05, 0.5))
    return "that took a while\n"


@app.get("/fail")
def fail():
    return "something broke\n", 500


@app.get("/metrics")
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)


if __name__ == "__main__":
    app.run(host="127.0.0.1", port=8080)
```

| Part of the code | What it does |
| --- | --- |
| `REQUESTS = Counter(…, ["method", "path", "status"])` | A counter with three labels: one time series for each combination of method, path and status |
| `LATENCY = Histogram(…, ["path"])` | A histogram of the duration, one for each path. It uses the default buckets, from 5 ms to 10 s |
| `KNOWN_PATHS` | The only values that the `path` label can have. Any other path is counted as `other` (see 1.4) |
| `@app.before_request` | Runs before **every** request: it notes the start time in `g`, the object of the current request |
| `@app.after_request` | Runs after **every** request, also a `404` or a `500`: it increases the counter and records the duration |
| `/`, `/slow`, `/fail` | Three routes to make traffic: fast, slow (50 to 500 ms) and failed (`500`) |
| `/metrics` | `generate_latest()` writes all metrics in the text format of 1.3. This is what Prometheus reads |

The metrics are counted in the two hooks, not in each route. Every route is counted the same
way, also the routes you add later.

:::{note} You do not always write the standard metrics yourself
Here you write the HTTP metrics by hand, to see how they work. In a real project, much of this
already exists:

- **Libraries for web frameworks** add the RED metrics with one or two lines of code: for example
  `prometheus-flask-exporter` (Flask), `prometheus-fastapi-instrumentator` (FastAPI) and
  `django-prometheus` (Django).
- **Exporters** collect metrics from software that you cannot change, and show them on
  `/metrics`: for example `postgres_exporter`, `redis_exporter`, `node_exporter` (the metrics of a
  host) and `blackbox_exporter` (checks an endpoint from outside). An exporter usually runs as a
  separate container, often next to the application in the same pod (a *sidecar*).
- **Many cloud-native tools have `/metrics` already**: Traefik, cert-manager, Grafana, and
  Kubernetes itself. You only need a ServiceMonitor.
- **`prometheus_client` itself** adds `process_…` metrics (CPU, memory, open files) and `python_…`
  metrics (garbage collector). Look for them in the output of Step 5.

So you usually write only the **domain metrics**: the values that only your application knows,
such as the predictions made, the files processed, or the rows in a table. Before you use a
library or an exporter, read the names and labels of its metrics: your dashboards and alerts
depend on them.
:::

### Step 4: start the service

Start it in the foreground, in this terminal:

```bash
python demo.py
```

```text
 * Running on http://127.0.0.1:8080
```

The terminal now belongs to the service: it shows one line for each request. Leave it open. Open
a **second terminal** on the VM for the next steps.

### Step 5: generate traffic and read the raw metrics

In the second terminal, send some requests:

```bash
for i in $(seq 1 100); do curl -s localhost:8080/ > /dev/null; done
for i in $(seq 1 20);  do curl -s localhost:8080/slow > /dev/null; done
for i in $(seq 1 5);   do curl -s localhost:8080/fail > /dev/null; done
for i in $(seq 1 3);   do curl -s localhost:8080/wp-login.php > /dev/null; done   # a scanner
```

Then read the metrics, as Prometheus would:

```bash
curl -s localhost:8080/metrics | grep '^demo_http_requests_total'
curl -s localhost:8080/metrics | grep '^demo_http_request_duration_seconds.*path="/slow"'
```

```text
demo_http_requests_total{method="GET",path="/",status="200"} 100.0
demo_http_requests_total{method="GET",path="/slow",status="200"} 20.0
demo_http_requests_total{method="GET",path="/fail",status="500"} 5.0
demo_http_requests_total{method="GET",path="other",status="404"} 3.0
…
```

Answer these questions from the output:

1. How many requests to `/slow` took 0.25 s or less? (Look at `le="0.25"`.)
2. What was the mean duration of `/slow`? (Divide `_sum` by `_count`.)
3. Why is there no series with `path="/wp-login.php"`?

### Step 6: restart the process

In the first terminal, stop the service with **Ctrl+C**, then start it again:

```bash
python demo.py
```

In the second terminal, read the counters again:

```bash
curl -s localhost:8080/metrics | grep '^demo_http_requests_total'
```

The counters are gone: they live in the memory of the process. A restart sets every counter back
to 0. This is normal. In 1.6 you see that the PromQL function `rate()` detects this reset and
ignores it.

Stop the service with **Ctrl+C** when you finish.

:::{dropdown} Solutions
1. The value of `…_bucket{path="/slow",le="0.25"}`. With `uniform(0.05, 0.5)`, it is a little
   less than half of the 20: about 9.
2. About 0.27 s: the middle of 0.05 and 0.5, plus a little overhead.
3. `KNOWN_PATHS`. Every unknown path is counted as `other`. If the raw path were a label, each
   scanner probe (`/wp-login.php`, `/.env`, `/admin`, …) would create a new time series in
   Prometheus, for ever. See 1.4.
:::

## 1.6 PromQL: five patterns that answer most questions

PromQL is the query language of Prometheus. You type it in Grafana (**Explore**, or a panel). These
five patterns cover the RED method.

**1. Select series by label.** `=` is equal, `!=` not equal, `=~` matches a regular expression.

```promql
climate_api_http_requests_total{namespace="traineeNN", path="/stats", status=~"5.."}
```

**2. `rate()`: from a counter to a speed.** The raw value of a counter is only "how many since the
process started". You almost never want it. `rate(x[1m])` gives the mean increase **per second**
over the last minute, and handles counter resets.

```promql
rate(climate_api_http_requests_total{namespace="traineeNN", path="/stats"}[1m])
```

**3. `sum by (…)`: combine series.** The query above gives one series for each pod and each status.
`sum` adds them. `by (status)` keeps one series for each status.

```promql
sum by (status) (rate(climate_api_http_requests_total{namespace="traineeNN", path="/stats"}[1m]))
```

**4. A ratio: the fraction of errors.**

```promql
sum(rate(climate_api_http_requests_total{namespace="traineeNN", path="/stats", status=~"5.."}[5m]))
/
sum(rate(climate_api_http_requests_total{namespace="traineeNN", path="/stats"}[5m]))
```

**5. `histogram_quantile()`: a percentile from the buckets.** First `rate` on the buckets, then
`sum by (le)` to combine all pods, then the quantile. `le` must stay in the `by`: it is the bucket
limit.

```promql
histogram_quantile(0.95,
  sum by (le) (rate(climate_api_http_request_duration_seconds_bucket{namespace="traineeNN", path="/stats"}[5m])))
```

:::{tip} How long should the range `[…]` be?
At least **four times the scrape interval**. With a scrape every 15 s, use `[1m]` or more. With a
shorter range, `rate()` often has fewer than two points and gives no result. In Grafana panels,
use `[$__rate_interval]`: Grafana calculates a correct range for you.
:::

:::{note} Jargon: *percentile* (or *quantile*)
The **95th percentile** (p95) of the latency is the time below which 95 % of the requests finish.
The mean hides the slow requests: 95 requests of 10 ms and 5 of 2 s have a mean of 110 ms, but
5 users in 100 wait 2 s. The p95 shows them.
:::

## 1.7 Integration with Kubernetes

For Prometheus to scrape your pods, three things must be true:

1. The pods show `/metrics` on a port.
2. A **Service** selects the pods, and that port has a **name** (for example `http`).
3. A **ServiceMonitor** in your namespace selects the Service by label, and names the port and the
   path.

:::{note} Jargon: *ServiceMonitor*
A **ServiceMonitor** is not a built-in Kubernetes object. It is a **CRD** (Day 3, section 3) of the
**Prometheus Operator** (`apiVersion: monitoring.coreos.com/v1`), like the `Middleware` of Traefik
or the `Certificate` of cert-manager. It says *"scrape the pods behind this Service, on this port
and path, every 15 s"*. The operator reads every ServiceMonitor and writes the scrape
configuration of Prometheus for you. So you never edit the configuration file of Prometheus: you
apply a small object in your own namespace, as you do for a Deployment.
:::

```{mermaid}
flowchart LR
    SM["ServiceMonitor<br/>selector: app=climate-api<br/>port: http, path: /metrics"] -->|"matches labels of"| S["Service climate-api<br/>labels: app=climate-api<br/>port name: http"]
    S -->|"selects"| P["Pods app=climate-api<br/>containerPort 8000, name http"]
```

The ServiceMonitor selects the **labels of the Service** (`metadata.labels`), not the labels of the
pods. A Service with a correct `selector` but no `labels` is invisible to the ServiceMonitor. This
is the most usual mistake.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: climate-api
spec:
  selector:
    matchLabels:
      app: climate-api          # the labels of the Service
  endpoints:
    - port: http                # the name of the Service port, not a number
      path: /metrics
      interval: 15s
```

On the training cluster, Prometheus reads ServiceMonitors in every namespace. It needs no extra
label such as `release: prometheus`.

:::{note} One process, one set of counters
`prometheus_client` keeps its values in the memory of the process. A server with **several worker
processes** (for example `gunicorn --workers 4`) has four sets of counters, and each scrape reads
the counters of one worker only, chosen at random. The graphs jump up and down. Use one process
for each pod, with threads if necessary, and scale with **pods**. The `climate-stats-api` of Day 2
and the group project both run `gunicorn --workers 1 --threads 4` for this reason.
:::

## 1.8 ⌨️ Hands-on 2: monitor the `climate-stats-api` on Kubernetes

You add metrics to the `climate-stats-api` of Day 2 (section 6), which you ran on Kubernetes on
Day 3 (Hands-on 8 and 9): `climate-api`, `climate-db` and `climate-cache`. Then Prometheus scrapes
it, and you look at it in Grafana.

### Step 0: start the stack again

Day 3, section 10.3, removed the stack and its PVC. The files are still in `~/climate-stack-k8s`.
Apply them again, in this order.

**1. Go to the folder of the stack.**

```bash
cd ~/climate-stack-k8s
```

**2. Create what the database needs before it starts:** the Secret with its credentials, the PVC
for its data, and the ConfigMap with `init.sql`. The ConfigMap must exist **before** the database
starts for the first time, otherwise the table is never created.

```bash
kubectl apply -f db-secret.yaml -f db-pvc.yaml -f db-init-configmap.yaml
```

**3. Start the database, and wait until it is ready.**

```bash
kubectl apply -f db.yaml
kubectl rollout status deployment/climate-db
```

**4. Start Redis and the API, and wait for the API.**

```bash
kubectl apply -f cache.yaml -f api.yaml
kubectl rollout status deployment/climate-api
```

**5. Check that the three Deployments are ready.**

```bash
kubectl get deploy climate-api climate-db climate-cache
```

```text
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
climate-api     2/2     2            2           40s
climate-db      1/1     1            1           1m
climate-cache   1/1     1            1           40s
```

The database is new, so it has the three rows of `init.sql`, not four.

:::{dropdown} Not there?
- `climate-api` in `ImagePullBackOff`: the Secret `gitlab-registry` is missing or was replaced.
  Create it again as on Day 4, section 7.
- The files are missing: copy them again from Day 3, sections 9.2 and 10.1.
- `/stats` returns `relation "climate_telemetry" does not exist`: the database started before the
  ConfigMap existed. Delete `db.yaml` **and** the PVC, then apply both again.
:::

### Step 1: add the SDK

In `~/climate-stack/requirements.txt`, add one line:

```text
prometheus-client==0.26.0
```

This is the same version as the `course` environment and the group project.

### Step 2: add metrics to `app.py`

Replace `~/climate-stack/app.py` with the version below. It uses the same two hooks as the demo of
Hands-on 1. The changes, compared with Day 2:

| Change | Why |
| --- | --- |
| A `Counter` for requests, by `method`, `path` and `status` | Rate and errors (R and E) |
| A `Histogram` for the duration, by `path` | Duration (D) |
| Two `Gauge`s: the Redis counter and the number of rows | The current state of the two backends |
| `KNOWN_PATHS` | Low cardinality (1.4). `request.path` has no query string |
| `@app.before_request` and `@app.after_request`, as in Hands-on 1 | Every request is counted, also a `404` and a `500` |
| A new route, `/metrics` | For Prometheus |
| No default value for the database password | Fail closed: without the Secret, the service does not start (section 2) |


Full file: `~/climate-stack/app.py`

```python
import os
import time

import psycopg2
import redis
from flask import Flask, Response, g, request
from prometheus_client import CONTENT_TYPE_LATEST, Counter, Gauge, Histogram, generate_latest

DB_HOST = os.getenv("DB_HOST", "db")
DB_NAME = os.getenv("POSTGRES_DB", "climate_db")
DB_USER = os.getenv("POSTGRES_USER", "ecmwf_admin")
DB_PASS = os.environ["POSTGRES_PASSWORD"]          # no default: fail closed
REDIS_HOST = os.getenv("REDIS_HOST", "cache")

REQUESTS = Counter(
    "climate_api_http_requests_total",
    "HTTP requests handled by climate-stats-api",
    ["method", "path", "status"],
)
LATENCY = Histogram(
    "climate_api_http_request_duration_seconds",
    "HTTP request duration of climate-stats-api",
    ["path"],
)
CACHE_ACCESS_COUNT = Gauge(
    "climate_api_cache_access_count",
    "Current value of the access counter stored in Redis",
)
DATABASE_RECORD_COUNT = Gauge(
    "climate_api_database_record_count",
    "Current number of rows in the climate_telemetry table",
)
KNOWN_PATHS = {"/health", "/stats", "/metrics"}    # anything else is counted as "other"

app = Flask(__name__)


@app.before_request
def start_timer():
    g.t0 = time.perf_counter()


@app.after_request
def count(resp):
    path = request.path if request.path in KNOWN_PATHS else "other"
    REQUESTS.labels(request.method, path, str(resp.status_code)).inc()
    LATENCY.labels(path).observe(time.perf_counter() - g.t0)
    return resp


@app.get("/health")
def health():
    return {"status": "healthy"}


@app.get("/stats")
def stats():
    try:
        r = redis.Redis(host=REDIS_HOST, port=6379, socket_connect_timeout=2)
        access_count = r.incr("telemetry:access_count")

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

    CACHE_ACCESS_COUNT.set(access_count)
    DATABASE_RECORD_COUNT.set(total_records)

    return {
        "cache_access_count": access_count,
        "database_record_count": total_records,
        "service_state": "synchronized",
    }


@app.get("/metrics")
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

Why a `Gauge` for the two backend values: they are the **current state** of Redis and PostgreSQL,
read from outside. They are not events that this process counts. The row count can also go down.

:::{note} The Docker Compose stack of Day 2 still works
`POSTGRES_PASSWORD` has no default now, but `compose.yaml` sets it, and so does the `climate-db`
Secret on Kubernetes. If you start the file without the variable, Python stops at once with
`KeyError: 'POSTGRES_PASSWORD'`. That is the intention.
:::

### Step 3: test the image on the VM

Before you push anything, test the new code with the Docker Compose stack of Day 2.

**1. Rebuild the image and start the stack.** `--build` builds the image again with the new
`app.py` and `requirements.txt`.

```bash
cd ~/climate-stack
docker compose up --build -d
```

**2. Call `/stats` once**, so that there is something to count:

```bash
curl -s localhost:8000/stats; echo
```

**3. Read the metrics.**

```bash
curl -s localhost:8000/metrics | grep '^climate_api_'
```

You see the requests counter, the histogram and the two gauges.

**4. Stop the stack.**

```bash
docker compose down
```

### Step 4: build and push version 1.1.0

**1. Build the image with the new tag.** Run it in `~/climate-stack`: the `.` at the end is the
folder with the `Dockerfile`. The tag is the full name in the registry, with the version `1.1.0`.

```bash
docker build -t gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN/climate-stats-api:1.1.0 .
```

**2. Push it to the registry.**

```bash
docker push gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN/climate-stats-api:1.1.0
```

Changed code gets a new tag. This is a best practice: do not push changed code with a tag that
already exists. If you reuse a tag:

- A node may not download the new image. With the default `imagePullPolicy: IfNotPresent` (the
  default for every tag except `:latest`), a node that already has the tag in its cache uses its
  old copy. Only `imagePullPolicy: Always` makes the kubelet ask the registry again each time a
  container starts.
- `kubectl apply` starts no rollout, because the manifest did not change.
- Pods started at different times can run different code under the same name, and you cannot
  roll back to "the old `1.0.0`": it does not exist any more.

A new tag avoids all three: the manifest changes, Kubernetes starts a rollout, and each version
has its own name.

### Step 5: roll out, and add a label to the Service

You change two things in `~/climate-stack-k8s/api.yaml`, then apply it.

**1. Change the image tag** from `1.0.0` to `1.1.0`, and check the result:

```bash
cd ~/climate-stack-k8s
sed -i 's|climate-stats-api:1.0.0|climate-stats-api:1.1.0|' api.yaml
grep 'image:' api.yaml
```

**2. Add `labels` to the Service.** Open `api.yaml` with your editor. In the **Service** part, add
`labels` under `metadata`, so that the ServiceMonitor can find it. The Service part becomes:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: climate-api
  labels:
    app: climate-api          # new: the ServiceMonitor selects this
spec:
  selector:
    app: climate-api
  ports:
    - name: http              # the ServiceMonitor uses this name
      port: 80
      targetPort: http
```

**3. Apply the file, and wait for the new pods.** The tag changed, so Kubernetes starts a rollout:

```bash
kubectl apply -f api.yaml
kubectl rollout status deployment/climate-api
```

**4. Check the label of the Service.**

```bash
kubectl get svc climate-api --show-labels
```

### Step 6: the ServiceMonitor

First check that you may create one:

```bash
kubectl auth can-i create servicemonitors.monitoring.coreos.com     # yes
```

In `~/climate-stack-k8s`, create the file `servicemonitor.yaml` with your editor, and write the
ServiceMonitor of 1.7:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: climate-api
spec:
  selector:
    matchLabels:
      app: climate-api          # must be a label of the Service (Step 5)
  endpoints:
    - port: http                # must be the name of the Service port (Step 5)
      path: /metrics
      interval: 15s
```

Before you apply it, compare the two commented lines with the Service:

```bash
kubectl get svc climate-api --show-labels
kubectl get svc climate-api -o jsonpath='{.spec.ports[*].name}'; echo
```

Then apply the ServiceMonitor and check that it exists:

```bash
kubectl apply -f ~/climate-stack-k8s/servicemonitor.yaml
kubectl get servicemonitor
```

### Step 7: generate traffic

You use two terminals on the VM, as in Hands-on 1.

**Terminal 1: open a tunnel to the Service.** Start the port-forward in the foreground and leave
it open:

```bash
kubectl port-forward service/climate-api 8080:80
```

```text
Forwarding from 127.0.0.1:8080 -> 8000
```

**Terminal 2: check that the new image answers.** Call `/stats` once, then read the metrics of
the pod:

```bash
curl -s localhost:8080/stats; echo
curl -s localhost:8080/metrics | grep '^climate_api_'
```

You see the counter, the histogram and the two gauges. If `/metrics` returns `404`, the pod still
runs the image `1.0.0`: check Step 5.

**Terminal 2: send a constant load.** Five requests per second, for one minute:

```bash
for i in $(seq 1 300); do curl -s localhost:8080/stats > /dev/null; sleep 0.2; done
```

The loop keeps the terminal busy until it finishes. Meanwhile, go on with Step 8 in the browser.
Prometheus needs one or two scrapes, that is 15 to 30 s, before the first values arrive.

:::{note} The port-forward goes to one pod
`kubectl port-forward service/…` chooses **one** pod and sends all traffic to it. In Grafana, one
pod receives the requests and the other receives none. That is a property of port-forward, not of
the Service. Traffic through the Ingress, as in the group project, is shared between the pods.
:::

### Step 8: Grafana

Open `https://grafana.training.meditwin-project.eu` and choose **Sign in with GitLab**, with your
training GitLab account. You are a **Viewer**: you can use **Explore** and open dashboards, but you
cannot save a dashboard. In Step 9 you use a dashboard that the platform team prepared.

In **Explore**, choose the data source **Prometheus**, and run these queries one at a time.
Replace `traineeNN` with your namespace.

#### Query 1: does Prometheus scrape the pods?

```promql
up{namespace="traineeNN", service="climate-api"}
```

Two series, value `1`: Prometheus scrapes both pods. If you see nothing, look at the table in 1.12.

#### Query 2: requests per second, by status

```promql
sum by (status) (rate(climate_api_http_requests_total{namespace="traineeNN", path="/stats"}[1m]))
```

Read a PromQL query **from the inside out**. Build this one in three steps in Explore, and look
at the result after each step:

| Step | Type this | You get |
| --- | --- | --- |
| a | `climate_api_http_requests_total{namespace="traineeNN", path="/stats"}` | The **raw counters**: one series for each pod and each status **that has received requests**. Each value only goes up: it is the number of requests since the pod started |
| b | `rate( a [1m])` | For each series, the **mean increase per second** over the last minute: requests per second. Still one series for each pod and status |
| c | `sum by (status) ( b )` | The series are **added together**, and only the label `status` is kept: one line for each status, for the whole service |

Without `rate`, the graph is a line that only goes up and tells you nothing about *now*. Without
`sum by (status)`, you see one line for each pod and status.

:::{note} Why do I see only one series at step a?
A counter with labels has no series until its first `inc()` with those label values. With
port-forward, all requests to `/stats` go to **one** pod (Step 7). The other pod has never counted
a `/stats` request, so it has no series for `path="/stats"`. Remove the filter on `path`: both pods
have series for `/health` (the probes) and `/metrics` (the scrapes). After a database failure, a
series with `status="500"` also appears. `sum` gives the right total in all these cases.
:::

During the load of Step 7 (five requests per second), there is only one line, `200`, close to `5`.
The service has no errors yet: a `500` line appears in Hands-on 3, when you stop the database.

#### Query 3: the 95th percentile of the latency

```promql
histogram_quantile(0.95,
  sum by (le) (rate(climate_api_http_request_duration_seconds_bucket{namespace="traineeNN", path="/stats"}[1m])))
```

This query works on the **buckets** of the histogram (1.3). Build it in four steps:

| Step | Type this | You get |
| --- | --- | --- |
| a | `climate_api_http_request_duration_seconds_bucket{namespace="traineeNN", path="/stats"}` | One series for each **bucket limit** `le`, for each pod that has received requests (`0.005`, `0.01`, … `+Inf`). Each value counts the requests that took `le` seconds or less, since the pod started |
| b | `rate( a [1m])` | For each bucket: how many requests per second, in the last minute, took `le` seconds or less |
| c | `sum by (le) ( b )` | The buckets of all pods added together, bucket by bucket. `le` **must** stay: without it, the bucket limits are lost and no percentile can be calculated |
| d | `histogram_quantile(0.95, c )` | Prometheus finds the bucket where 95 % of the requests are reached, and estimates the value inside it. The result is in seconds |

The result means: *in the last minute, 95 % of the requests to `/stats` took this time or less*.

:::{note} The result is an estimate
Prometheus knows only how many requests are in each bucket, not their exact durations. It assumes
that the requests are spread evenly inside a bucket. If p95 falls in the bucket from 0.025 s to
0.05 s, the answer is somewhere between the two, never more precise than the bucket. When the
precision matters, choose the bucket limits yourself in the `Histogram(…, buckets=[…])`, close to
the latencies that you expect.

An example with 100 requests in the last minute:

| `le` (seconds) | 0.05 | 0.1 | 0.25 | 0.5 | 1 |
| --- | --- | --- | --- | --- | --- |
| Requests that took `le` or less | 40 | 80 | 93 | 99 | 100 |

93 requests took 0.25 s or less, and 99 took 0.5 s or less. So the 95th request is in the bucket
from 0.25 s to 0.5 s. That bucket has 6 requests (99 − 93), and the 95th is the 2nd of them.
Prometheus answers 0.25 + 2/6 × 0.25 ≈ 0.33 s.
:::

#### Query 4: a gauge from several pods

```promql
max(climate_api_database_record_count{namespace="traineeNN"})
```

A gauge without labels exists from the start, with the value `0`. Each pod sets it only when it
answers a `/stats` request. So the pod that receives the traffic reports the real number of rows,
and the other pod still reports `0`. `max` gives the real value. `avg` would give half of it, and
`sum` would be wrong as soon as both pods receive traffic. Remove `max( )` to see the two series.

### Step 9: the dashboard

The platform team prepared one dashboard for all participants: **Climate API Monitoring (Trainee
Namespaces)**.

1. In Grafana, open **Dashboards** and choose **Climate API Monitoring (Trainee Namespaces)**.
2. At the top, in the **Namespace** list, choose your namespace.
3. Find the panels. They follow the RED order: rate and errors at the top, duration in the middle,
   the state of the service at the bottom.

| Panel | What it shows | Query pattern (1.6) |
| --- | --- | --- |
| Requests per second /stats, by status | Rate and errors: one line for each status. Only `200` until Hands-on 3 | 2 and 3: `rate`, then `sum by (status)` (Query 2 of Step 8) |
| Success ratio /stats | The fraction of requests answered with `2xx`. The dashed red line is the 95 % threshold. A flat line at 100 % until Hands-on 3, and no line when there is no traffic | 4: a ratio |
| Latency /stats (p50 and p95) | Half of the requests are faster than p50; 95 % are faster than p95 | 5: `histogram_quantile` (Query 3 of Step 8) |
| Pods scraped | How many pods Prometheus reads: the sum of `up` | 1: a selector (Query 1 of Step 8) |
| Rows in PostgreSQL | The gauge of the row count | A gauge, with `max` over the pods (Query 4 of Step 8) |
| Redis access counter | The gauge of the Redis counter | A gauge, with `max` over the pods |

Point at the **i** next to a panel title: a short description says what the panel shows and which
pattern it uses. With no traffic, **Success ratio** shows *No data*, not 0 %: there is nothing to
divide.

To see the query of a panel, open the panel menu (top right of the panel) and choose
**Explore**.

One dashboard serves every namespace because its queries use the **variable** `$namespace`
(`namespace=~"$namespace"`). Grafana replaces it with the value you choose in the list.

When you finish, keep everything as it is: the stack, the port-forward in terminal 1, and the
dashboard in the browser. Hands-on 3 uses them.

## 1.9 ⌨️ Hands-on 3: break something, and see it

The purpose of monitoring is to see problems. Make one: stop the database while the API receives
traffic. The API pods keep running, and `/health` does not check the database. So the question is
the one of 1.1: **who notices, Kubernetes or the metrics?**

You need the stack of Hands-on 2, with its ServiceMonitor, and three terminals on the VM:

| Terminal | What runs in it |
| --- | --- |
| 1 | The port-forward of Hands-on 2, Step 7. If you closed it, start it again: `kubectl port-forward service/climate-api 8080:80` |
| 2 | The load |
| 3 | `kubectl` commands |

Open the dashboard of Hands-on 2, Step 9 in the browser, with your namespace.

Before you start, read the four questions of Step 5 and write down your answers. Then check them
on the dashboard.

### Step 1: start a load

**Terminal 2.** This loop sends five requests per second for about two minutes:

```bash
for i in $(seq 1 600); do curl -s localhost:8080/stats > /dev/null; sleep 0.2; done
```

### Step 2: stop the database

**Terminal 3.** Note the time, then scale the database to zero pods:

```bash
date +%T
kubectl scale deployment/climate-db --replicas=0
kubectl get pods -l app=climate-db        # no pods
```

### Step 3: watch the dashboard

**Browser.** Watch the dashboard for two minutes. Do not change anything. Look at each panel.

### Step 4: start the database again

**Terminal 3.**

```bash
kubectl scale deployment/climate-db --replicas=1
kubectl rollout status deployment/climate-db
```

Watch the dashboard go back to normal. The data is still there: it is on the PVC.

### Step 5: answer from the dashboard

1. Which panel showed the problem first, and how long after the `scale`?
2. What happened to the p95 latency while the database was down? Why?
3. Did **Pods scraped** change? Did the readiness probes remove the API pods from the Service?
4. Did **Rows in PostgreSQL** change? What does that tell you about a gauge that is updated only
   when a request succeeds?

:::{dropdown} Solutions
1. The `500` line of **Requests per second** and the **Success ratio**, after one or two scrapes
   (15 to 30 s), plus the range of `rate()`.
2. The **p95 latency may go up or down while the database is down**. The API records the duration
   of every `/stats` request, including failed requests. If the connection is refused immediately,
   failures are fast and the p95 may go down. If connecting takes longer, latency rises; the API
   uses `connect_timeout=3` seconds, so the database connection attempt can wait up to about 3 seconds.
   Always read latency together with the error ratio.
3. No. **Pods scraped** stayed at 2. `/health` does not check the database, so the API pods stayed
   ready and up. From the point of view of Kubernetes, everything was healthy. Only the metrics
   showed the problem. This is the difference between probes and monitoring in 1.1.
4. No. The gauge kept its last value, because nothing set it while `/stats` failed. A gauge that is
   updated only on success can hide an outage. Look at it together with the error ratio.
:::

When you finish, stop the loop (if it still runs) in terminal 2 and the port-forward in
terminal 1, each with **Ctrl+C**.

### Step 6: cleanup

Remove the stack. This deletes every object of the files in `~/climate-stack-k8s`, including the
ServiceMonitor and the PVC:

```bash
kubectl delete -f ~/climate-stack-k8s/
kubectl get deploy,svc,pvc,servicemonitor        # nothing of the stack is left
```

## 1.10 📖 Grafana in short

| Place | Use it for |
| --- | --- |
| **Explore** | Try queries; investigate a problem now |
| **Dashboards** | Fixed graphs, for everyone, that you open often |
| **Variables** (`$namespace`, `$pod`) | One dashboard for many namespaces or pods |
| **Time picker** (top right) | Look at a period; zoom with the mouse on a graph |

A good dashboard for a service:

- Starts with the RED signals at the top, then the resources (CPU, memory, restarts).
- Has a unit on each panel (`reqps`, `s`, `percentunit`, `bytes`).
- Shows the p95 or p99, not only the mean.
- Is in Git, as JSON, not only in the Grafana database.

## 1.11 📖 What to monitor first

Start with a small set. Add more after you have seen real incidents.

| Signal | Query (replace `traineeNN`) | Source |
| --- | --- | --- |
| Request rate | `sum(rate(climate_api_http_requests_total{namespace="traineeNN"}[5m]))` | Your code |
| Error ratio | Pattern 4 in 1.6 | Your code |
| p95 latency | Pattern 5 in 1.6 | Your code |
| Container restarts | `sum by (pod) (increase(kube_pod_container_status_restarts_total{namespace="traineeNN"}[1h]))` | kube-state-metrics |
| CPU used | `sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="traineeNN", container!=""}[5m]))` | kubelet (cAdvisor) |
| Memory used | `sum by (pod) (container_memory_working_set_bytes{namespace="traineeNN", container!=""})` | kubelet (cAdvisor) |
| Memory limit | `sum by (pod) (kube_pod_container_resource_limits{namespace="traineeNN", resource="memory"})` | kube-state-metrics |

Put the last two queries in one panel: you see how close each pod is to its limit. On Day 3 (section 7), a pod over its
memory limit was stopped with `OOMKilled`, exit code 137. With this graph, you see it coming.

## 1.12 📖 Problems and solutions

| Symptom | Usual cause | Check |
| --- | --- | --- |
| `up{…}` returns nothing | The ServiceMonitor does not match the Service | `kubectl get svc climate-api --show-labels`: the label of the ServiceMonitor `selector` must be there |
| `up{…}` returns nothing, labels are correct | The `port:` of the ServiceMonitor is not the **name** of the Service port | `kubectl get svc climate-api -o jsonpath='{.spec.ports[*].name}'` |
| `up` is `0` | Prometheus reaches the pod, but `/metrics` fails | `kubectl port-forward` and `curl localhost:8080/metrics` |
| `rate(…)` returns nothing, the raw counter works | The range is shorter than two scrapes | Use `[1m]` or more |
| Your label is called `exported_endpoint` | A label name is the same as a Prometheus target label | Rename it (1.2) |
| The graphs jump up and down | Several worker processes in one pod | One process for each pod (1.7) |
| Prometheus becomes slow; very many series | A label with many values | Look for raw paths or IDs in labels (1.4) |
| A gauge is flat during an outage | It is set only when a request succeeds | Look at it with the error ratio (Hands-on 3) |
| Your namespace is not in the **Namespace** list of the dashboard | The list shows only namespaces where Prometheus already has `climate_api_…` metrics | Check the ServiceMonitor; run Query 1 of Step 8; wait one scrape and reload |
| The dashboard is empty for your namespace | No traffic yet, or another namespace is chosen | Check the **Namespace** list; send traffic (Step 7) |

## 1.13 Core takeaway


> **Instrumentation without visualization is incomplete; visualization without instrumentation is
> blind.**

- Probes tell Kubernetes whether a pod is alive. Metrics tell **you** how the service behaves.
- Start with **RED**: rate, errors, duration. Then restarts, CPU and memory.
- Prometheus **pulls**: the application shows `/metrics`, and a ServiceMonitor says where to find
  it. The ServiceMonitor selects the **labels of the Service** and the **name** of its port.
- **Counter** for events, **Gauge** for a current state, **Histogram** for latency.
- Always `rate()` a counter. Combine with `sum by (…)`. Calculate percentiles with
  `histogram_quantile()` and keep `le`.
- Keep label values few and fixed. Never a raw URL, never a user ID.
- One process for each pod. Scale with pods, not with workers.

## 1.14 📖 Check yourself

1. Your service has handled 1 000 000 requests since it started. Is that a useful number for a
   dashboard? What do you show instead?
2. Which type of metric do you use for: (a) the number of predictions made; (b) the number of
   files waiting in a queue; (c) the time to load the model?
3. A colleague adds the label `user_id` to the requests counter "to see who uses the API". What do
   you answer?
4. You have three pods. Somebody calculates the p95 latency of each pod, then takes the mean of the
   three values. Why is that wrong, and what is the correct query?
5. You created a ServiceMonitor, but `up{namespace="traineeNN"}` shows nothing for your service.
   Give two causes to check.
6. In Hands-on 3, the readiness probes did not remove the pods, but the users got errors. What would
   you change in the application, and what is the risk of that change?

:::{dropdown} Solutions
**1.** No. A counter since the start depends on when the process started, and goes back to 0 at
each restart. Show the **rate**: `sum(rate(…_total[5m]))`, requests per second.

**2.** (a) Counter: predictions only happen, they never "un-happen". (b) Gauge: the queue goes up
and down. (c) Histogram, if it happens often; a Gauge with the last duration is enough if it
happens once at start.

**3.** Each user creates new time series, for ever: high cardinality (1.4), which can stop
Prometheus. The user ID also becomes visible to everybody who can read the metrics. "Who uses the
API" is a question for **logs**, not for metrics.

**4.** A mean of percentiles is not a percentile. If one pod is slow and receives most of the
traffic, the mean hides it. Add the buckets of all pods first, then calculate the quantile:
`histogram_quantile(0.95, sum by (le) (rate(…_bucket[5m])))`.

**5.** Any two of: the Service has no `metadata.labels` that match the ServiceMonitor `selector`;
the `port:` of the ServiceMonitor is not the **name** of a Service port; the ServiceMonitor is in
another namespace; `/metrics` does not answer (then `up` exists but is `0`).

**6.** Make `/health` (or a separate readiness path) check the database. Then Kubernetes removes the
pods from the Service when the database is down. The risk: if the database is down, **every** API
pod becomes not ready, the Service has no endpoints, and users get connection errors instead of a
clear `500`. Liveness probes must **never** check a dependency: Kubernetes would restart all pods,
which does not repair the database. This is a design decision. Monitoring shows the problem in both
cases.
:::

---

Continue with: [2. Security fundamentals for cloud-native workloads](02-security-fundamentals.md).
