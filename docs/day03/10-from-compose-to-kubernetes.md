# 10. From Compose to Kubernetes

On Day 2 the `climate-stack` was one `compose.yaml` with three services. Section 9 brought its `db` service to Kubernetes; this section brings the other two, `cache` and `api`, and the stack runs again — this time on the cluster.

You do it twice: first **by hand** (Hands-on 9), translating each block of the `compose.yaml` yourself; then, optionally, **with a tool**, `kompose`, which converts the same file automatically. Comparing the two shows what a converter can and cannot do for you.

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

## 10.1 ⌨️ Hands-on 9: the whole `climate-stack` on Kubernetes

The database of section 9 is up; now put the [Day 2 `api` and `cache` services](../day02/06-docker-compose.md#id-6-3-the-compose-specification-compose-yaml) in front of it. This is where ConfigMap, Secret and service discovery meet. Keep working in the same folder:

```bash
cd ~/climate-stack-k8s
```

### Step 1: push the API image

The `climate-stats-api:1.0.0` image of [Day 2, section 6](../day02/06-docker-compose.md#id-6-6-operational-lifecycle-management) was only built locally by `docker compose`, so it is not in the registry. Push it now, into the same GitLab project, but as a **separate image** named `climate-stats-api` (see the table in the [Day 3 introduction](index.md)). It is a different application from `climate-telemetry` (it adds the `/stats` endpoint, Redis and PostgreSQL), so it gets its own name and its own versions; `climate-telemetry-traineeNN:1.0.0` stays untouched:

Log in to the registry (if you are not logged in already from Day 2):

```bash
docker login gitlab.data.meditwin-project.eu:6443 -u traineeNN
```

Tag and push the image:

```bash
docker tag climate-stats-api:1.0.0 \
  gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN/climate-stats-api:1.0.0
docker push gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN/climate-stats-api:1.0.0
```

If the local image is missing, rebuild it first: `cd ~/climate-stack && docker compose build`.

### Step 2: Redis

Create `cache.yaml` — no volume, no secret, no probe beyond the TCP port; this is the `cache` service of the `compose.yaml`, almost line for line:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: climate-cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: climate-cache
  template:
    metadata:
      labels:
        app: climate-cache
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - name: redis
              containerPort: 6379
          readinessProbe:
            tcpSocket:
              port: redis
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: climate-cache
spec:
  selector:
    app: climate-cache
  ports:
    - name: redis
      port: 6379
      targetPort: redis
```

### Step 3: the API

Create `api.yaml`. Compare its `env` section with the `environment:` block of the Compose `api` service: same five variables, but the three sensitive ones come from the Secret, and the two hostnames are Service names:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: climate-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: climate-api
  template:
    metadata:
      labels:
        app: climate-api
    spec:
      imagePullSecrets:
        - name: gitlab-registry
      containers:
        - name: app
          image: gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN/climate-stats-api:1.0.0
          env:
            - name: DB_HOST
              value: climate-db
            - name: REDIS_HOST
              value: climate-cache
          envFrom:
            - secretRef:
                name: climate-db
          ports:
            - name: http
              containerPort: 8000
          readinessProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 5
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: climate-api
spec:
  selector:
    app: climate-api
  ports:
    - name: http
      port: 80
      targetPort: http
```

| Compose `api` service (Day 2) | `api.yaml` | What changes |
|---|---|---|
| `build:` and `image: climate-stats-api:1.0.0` | `image: …/climate-stats-api:1.0.0` + `imagePullSecrets` | Kubernetes never builds: the image comes from the registry (Step 1) |
| `DB_HOST=db`, `REDIS_HOST=cache` | `env: DB_HOST=climate-db`, `REDIS_HOST=climate-cache` | The hostnames are the **Service** names |
| `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | `envFrom: secretRef: climate-db` | The sensitive values come from the Secret, not from the file |
| `ports: "8000:8000"` | `containerPort: 8000` + the Service `climate-api` (port 80) | Publishing is a separate object; nothing is bound on a host |
| `depends_on: condition: service_healthy` | — | No start order: readiness and retries (below) |
| `restart: unless-stopped` | — | The Deployment always replaces failed Pods |
| one container | `replicas: 2` | Two identical Pods behind one Service |

There is no `depends_on`: if the API Pods start before the database is ready, `/stats` fails with a connection error until it is, and the readiness probe on `/health` keeps the Pod in the Service anyway. That is the honest translation of the "Unsynchronized Dependency" case of [Day 2, section 6.5](../day02/06-docker-compose.md#unsynchronized-dependency-depends-on-without-conditions) — and why real applications retry.

Apply the two files:

```bash
kubectl apply -f cache.yaml -f api.yaml
```

Wait until the API is ready:

```bash
kubectl rollout status deployment/climate-api
```

List the three services of the stack:

```bash
kubectl get deploy,po,svc -l 'app in (climate-api, climate-db, climate-cache)'
```

### Step 4: the Day 2 test, through the Service

You use two terminals, as in section 4.5.

**Terminal 1: open a tunnel to the Service** and leave it open:

```bash
kubectl port-forward service/climate-api 8080:80
```

**Terminal 2: call `/stats` twice:**

```bash
curl -s http://localhost:8080/stats; echo
curl -s http://localhost:8080/stats; echo
```

```json
{"cache_access_count":1,"database_record_count":4,"service_state":"synchronized"}
{"cache_access_count":2,"database_record_count":4,"service_state":"synchronized"}
```

`database_record_count` is 4 because the Marseille row from Hands-on 8 is there; `cache_access_count` grows in Redis. The API found both backends by name, like `DB_HOST=db` on [Day 2](../day02/06-docker-compose.md#id-6-4-service-discovery-and-embedded-dns-resolution) — only the names are Service names now (`climate-db`, `climate-cache`).

**Terminal 2: delete the cache Pod.** Before you run the commands, **predict** the two numbers that `/stats` will return:

```bash
kubectl delete pod -l app=climate-cache
kubectl rollout status deployment/climate-cache
curl -s http://localhost:8080/stats; echo
```

:::{dropdown} What happened, and why
- `cache_access_count` restarts from `1`: Redis has no PVC, so its data is ephemeral, like `ephemeral-redis` on [Day 2](../day02/04-persisting-data-and-web-ui.md#id-4-1-state-volatility-the-problem-in-action).
- `database_record_count` stays `4`: the PostgreSQL data is on a PVC.

Same stack, two kinds of storage: the choice is made service by service.
:::

**Checkpoint 9** — three Deployments are ready, `/stats` answers through the Service, and the database count survives a cache restart.

```bash
kubectl get deploy climate-api climate-db climate-cache
curl -s http://localhost:8080/stats; echo      # in terminal 2, with the port-forward of terminal 1 running
```

```text
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
climate-api     2/2     2            2           5m
climate-db      1/1     1            1           20m
climate-cache   1/1     1            1           5m
{"cache_access_count":2,"database_record_count":4,"service_state":"synchronized"}
```

(`cache_access_count` depends on how many calls you made since the cache Pod was deleted: it restarted from `1` then. `database_record_count` must be `4`.)

:::{dropdown} Not there?
- `climate-api` in `ImagePullBackOff`: the `climate-stats-api:1.0.0` image was not pushed (Step 1), or the path in `api.yaml` is different.
- `/stats` returns `"error": "... could not translate host name"`: a Service name is wrong. `DB_HOST` must be `climate-db` and `REDIS_HOST` `climate-cache`, as in `kubectl get svc`.
- `/stats` returns `password authentication failed`: the API and the database read different credentials. Both must use the `climate-db` Secret.
- `/stats` returns `relation "climate_telemetry" does not exist`: `init.sql` never ran (the PVC was created before the ConfigMap). Delete `db.yaml` and the PVC, then apply them again.
:::

## 10.2 ⌨️ Optional: the same stack with `kompose`

[`kompose`](https://kompose.io/) converts a `compose.yaml` into Kubernetes manifests. Run it on the **same** Day 2 file that you have just ported by hand, and compare:

```bash
cd ~/climate-stack
mkdir -p k8s-kompose
kompose convert -f compose.yaml -o k8s-kompose/
ls k8s-kompose/
```

The output is a first draft, with one or more files per service (Deployments, a Service for `api`, a PersistentVolumeClaim for the `postgres_data` volume, a ConfigMap for the `init.sql` bind mount). **Do not apply it**: read it next to your files in `~/climate-stack-k8s`, and look for these differences:

| Check | Your manifests (Hands-on 8–9) | `kompose` output |
|---|---|---|
| Services for `db` and `cache` | `climate-db`, `climate-cache` | Missing: Compose declared no `ports:` for them, so `kompose` creates no Service, and `DB_HOST=db` would not resolve |
| Password | In a Secret, read with `envFrom` | In clear text in the Deployments, copied from `environment:` |
| API image | From the registry, with `imagePullSecrets` | `climate-stats-api:1.0.0`, the local tag: no node can pull it |
| Health | Readiness probes | The `healthcheck:` of `db` becomes a `livenessProbe` (no readiness); `api` and `cache` have none |
| Resources | Requests and limits on the API | None |
| Database update strategy | `Recreate`, because of the `ReadWriteOnce` volume | `Recreate` too: `kompose` sets it on every service that mounts a volume. One thing it gets right |
| `PGDATA` | Set to a subfolder, to avoid `lost+found` | Not set |
| Start order | No `depends_on`: readiness and retries | `depends_on` is dropped silently |

`kompose` saves typing, but each line of its output must be checked against what you know: it translates syntax, not intent. It is useful to start a large port; it does not replace understanding the objects.

## 10.3 Cleanup

The stack is not used in the rest of the day. Remove it, and its data:

```bash
cd ~/climate-stack-k8s
kubectl delete -f .
kubectl get deploy,svc,pvc
```

```text
No resources found in traineeNN namespace.
```

`kubectl delete -f .` also removed the PVC, because `db-pvc.yaml` is in the folder. No Deployments, Services or PVCs are left: `climate-telemetry` was removed at the end of section 8, and comes back in section 12 as a Helm chart. The Secrets `gitlab-registry` and `climate-telemetry-tls` stay on purpose. (If you skipped the cleanup of section 8, you still see the Deployment and Service of `climate-telemetry`: that is fine.) In terminal 1, stop the port-forward with **Ctrl+C** if it is still running.

## 10.4 Core takeaway

> **A Compose stack ports to Kubernetes service by service: one Deployment and one Service per service, Secrets for the passwords, PVCs for the data.**

- **Service names** are the `DB_HOST=db` of Compose: a multi-service stack ports almost line by line.
- There is no `depends_on`: readiness probes decide when a Pod receives traffic, and applications retry their dependencies.
- A converter such as `kompose` gives a draft, not a result: Services, Secrets, probes, resources and the image path still need you.

## 10.5 📖 Check yourself

1. The API logs `could not translate host name "climate-db"`. The database Pod is `Running`. What is missing?
2. In Compose, `depends_on: condition: service_healthy` made the API wait for the database. What replaces it on Kubernetes?
3. Why must the image of the API come from the registry, even though `docker compose` built it on the same VM?
4. `kompose` produced a Deployment with `POSTGRES_PASSWORD` in clear text. What do you change before you commit it to Git?

:::{dropdown} Solutions
**1.** A **Service** named `climate-db` that selects the database Pod. Without it, the name does not exist in the cluster DNS.

**2.** Nothing starts in order. Readiness probes keep a Pod out of the Service until it is ready, and the application retries its dependencies (`/stats` fails until the database answers).

**3.** The Pods may run on any node, and a node never sees the build cache of your VM. It pulls the image from a registry, with `imagePullSecrets` for a private one.

**4.** Move the values to a **Secret**, keep its file out of Git, and read it with `envFrom` or `secretKeyRef`.
:::

## 10.6 🏠 Homework (advanced): port the Day 2 capstone application

The [Day 2 capstone lab](../day02/07-lab.md) has a larger stack: a FastAPI ingestion API, PostgreSQL, Redis and Portainer. Port it to Kubernetes on your own, with what you learned in sections 7–10.

1. **Push the API image.** The local image `climate-ingestion-api:1.0.0` is not in the registry. Push it as a separate image in your project:

   ```bash
   cd ~/meditwin-climate-capstone
   docker build -t climate-ingestion-api:1.0.0 .
   docker login gitlab.data.meditwin-project.eu:6443 -u traineeNN
   docker tag climate-ingestion-api:1.0.0 \
     gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN/climate-ingestion-api:1.0.0
   docker push gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN/climate-ingestion-api:1.0.0
   ```

2. **Generate a draft**: `mkdir -p k8s && kompose convert -f compose.yaml -o k8s/`.
3. **Fix it**, as in the table of section 10.2: registry image and `imagePullSecrets`, a Secret for the `POSTGRES_*` values, Services for `db` and `cache`, readiness on `/health`, requests and limits, `PGDATA` for the database. Check that the database keeps `strategy: Recreate`.
4. **Leave Portainer out**: it needs the Docker socket of a host, which has no equivalent on a Kubernetes cluster.
5. **Test persistence** as in the Day 2 lab: send an observation, delete the database Pod, flush Redis (`kubectl exec deploy/<cache> -- redis-cli FLUSHALL`), and check that the API answers with `"source": "database"`.
6. **Expose the API** with an Ingress on `climate-api-traineeNN.training.meditwin-project.eu`.
7. **Clean up** when you have finished: `kubectl delete -f k8s/` and delete the PVCs.

A corrected set of manifests, to compare with yours, is in the course repository, folder `meditwin-climate-capstone/k8s-gold/` (see its `README.md`). Ask the instructor for the repository link.

Continue with: [11. GPU workloads and scheduling](11-gpu-workloads-and-scheduling.md).
