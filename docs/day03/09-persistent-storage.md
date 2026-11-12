# 9. Persistent storage

Section 8 kept working on `climate-telemetry`, a service with no data of its own. Databases are different: their data must survive when a Pod is replaced. This section runs the `db` service of the [Day 2 `climate-stack`](../day02/06-docker-compose.md#id-6-3-the-compose-specification-compose-yaml) — PostgreSQL — on persistent storage, and repeats the Day 2 persistence experiment. Section 10 then adds the other two services of that stack.

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

## 9.1 Storage concepts for users

Before looking at resources, start from one practical question:

**What happens to my data when a Pod is replaced?**

- If data is stored only in the container filesystem, it is **ephemeral** — the `ephemeral-redis` experiment of [Day 2, section 4.1](../day02/04-persisting-data-and-web-ui.md#id-4-1-state-volatility-the-problem-in-action).
- If data is stored on a volume backed by a PVC, it can be **persistent** across Pod replacement — the `climate_db_data` named volume of [the same section](../day02/04-persisting-data-and-web-ui.md#id-4-3-named-volume-management-and-database-durability), and `postgres_data` in the [section 6 `compose.yaml`](../day02/06-docker-compose.md#id-6-3-the-compose-specification-compose-yaml).

From the user perspective, persistence is a chain: the Pod mounts a volume, the volume references a claim, and the claim is bound to a real disk.

```{mermaid}
flowchart LR
    POD[Pod] -->|volumeMounts:<br/>a folder in the container| VOL[volume]
    VOL -->|persistentVolumeClaim| PVC[PVC<br/>your request:<br/>size, access mode]
    PVC -->|bound to| PV[PV<br/>the real disk]
    SC[StorageClass] -.->|creates on demand| PV
```

Read it from left to right: you write the first three; the StorageClass of the platform creates the PV when the claim needs one.

- **PersistentVolumeClaim (PVC)**: your request for storage (size, access mode, class). This is the `docker volume create` of [Day 2](../day02/04-persisting-data-and-web-ui.md#volume-management-api), with a size.
- **PersistentVolume (PV)**: storage object that satisfies the claim.
- You usually define **PVC**; platform automation (a StorageClass with dynamic provisioning) or admins provide PVs.

:::{note} Jargon: *StorageClass*, *dynamic provisioning*, *access mode*
A **StorageClass** is a kind of storage offered by the cluster (for example OpenStack block volumes). With **dynamic provisioning**, the StorageClass creates a new PV automatically for each PVC. Many classes wait until a Pod uses the claim before they create the volume (`WaitForFirstConsumer`), so that it is created next to that Pod. The **access mode** says how the volume can be mounted: `ReadWriteOnce` (RWO) means by one node at a time, which is the usual case for block storage.
:::

The key user question is:

> **How do I request persistent storage for my workload?**

The key user test is:

> **If I delete the Pod, does my data remain available in the new Pod?**

## 9.2 ⌨️ Hands-on 8: the Day 2 persistence experiment, on Kubernetes

On [Day 2, section 4.3](../day02/04-persisting-data-and-web-ui.md#postgresql-state-durability-verification), you started PostgreSQL on a named volume, inserted weather stations, destroyed the container with `docker rm -f`, started a new one on the same volume, and found the rows still there. Repeat the same experiment here with the `db` service of the [section 6 `compose.yaml`](../day02/06-docker-compose.md#id-6-3-the-compose-specification-compose-yaml); only the vocabulary changes.

### Step 1: a new folder, and the database Secret

This is a different application from `climate-telemetry`, so its manifests go in their own folder (`~/climate-service-k8s` keeps only the files of `climate-telemetry`, which section 12 turns into a Helm chart):

```bash
mkdir -p ~/climate-stack-k8s && cd ~/climate-stack-k8s
```

Create `db-secret.yaml`, the Secret of section 8.2, with the credentials of the Day 2 `db` service:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: climate-db
type: Opaque
stringData:
  POSTGRES_DB: climate_db
  POSTGRES_USER: ecmwf_admin
  POSTGRES_PASSWORD: EcmwfPassphrase123!
```

```bash
kubectl apply -f db-secret.yaml
kubectl get secret climate-db
```

In a real project this file never goes into Git with the others (section 8.2): it is here only to keep the exercise short.

### Step 2: create a PVC

Create `db-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: climate-db-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply and inspect:

```bash
kubectl apply -f db-pvc.yaml
kubectl get pvc
kubectl describe pvc climate-db-data
```

On this training cluster (OpenStack Cinder CSI), `STATUS` should usually move from `Pending` to `Bound` shortly after creation.
If it stays `Pending` for too long, read the events at the end of the `describe` above.

### Step 3: the init script becomes a ConfigMap

In Compose, `./init.sql:/docker-entrypoint-initdb.d/init.sql:ro` bind-mounted a file from your laptop. Nodes have no access to your laptop, so the file travels inside the API: a ConfigMap, mounted as a **file** this time.

Create `db-init-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: climate-db-init
data:
  init.sql: |
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

Apply it and inspect:

```bash
kubectl apply -f db-init-configmap.yaml
kubectl get configmap climate-db-init -o yaml
```

### Step 4: run PostgreSQL on the PVC

Create `db.yaml`, using the Secret of Step 1 for the credentials and a Service so that the API can find it by name:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: climate-db
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: climate-db
  template:
    metadata:
      labels:
        app: climate-db
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          envFrom:
            - secretRef:
                name: climate-db
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - name: postgres
              containerPort: 5432
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "ecmwf_admin", "-d", "climate_db"]
            periodSeconds: 5
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: init
              mountPath: /docker-entrypoint-initdb.d
              readOnly: true
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: climate-db-data
        - name: init
          configMap:
            name: climate-db-init
---
apiVersion: v1
kind: Service
metadata:
  name: climate-db
spec:
  selector:
    app: climate-db
  ports:
    - name: postgres
      port: 5432
      targetPort: postgres
```

Line by line against the [Day 2 `db` service](../day02/06-docker-compose.md#id-6-3-the-compose-specification-compose-yaml):

| Part of `db.yaml` | What it does | Compose twin (`db` service) |
|---|---|---|
| `envFrom: secretRef: climate-db` | Database name, user and password, from the Secret of Step 1 | `environment:` |
| `env: PGDATA` | Puts the data in a subfolder of the volume: the root of a new volume often contains `lost+found`, and PostgreSQL refuses a non-empty folder | — |
| `volumes: data` → `persistentVolumeClaim` | The data folder lives on the PVC of Step 2 | `postgres_data:/var/lib/postgresql/data` |
| `volumes: init` → `configMap` | `init.sql` is mounted as a file from the ConfigMap of Step 3 | `./init.sql:/docker-entrypoint-initdb.d/init.sql:ro` |
| `readinessProbe: pg_isready` | The Pod receives traffic only when PostgreSQL accepts connections | `healthcheck:` with `pg_isready` |
| `strategy: Recreate` | **New**: the old Pod stops before the new one starts. With a `ReadWriteOnce` volume, a rolling update could wait for ever | — |
| The `Service` `climate-db` (second document) | Gives the database the name the API uses | The service name `db` on the Compose network |

:::{note} Deployment or StatefulSet?
A single-replica database on a PVC works fine as a Deployment. The moment you need several replicas with one volume *each*, with stable names (`db-0`, `db-1`) and ordered start-up, you want a **StatefulSet**. The persistence mechanism (PVC) is the same.
:::

Apply it and wait for readiness:

```bash
kubectl apply -f db.yaml
kubectl rollout status deployment/climate-db
kubectl get pods -l app=climate-db
kubectl get pvc
```

### Step 5: check the seed data and add a row

`init.sql` ran at first start, as in Compose. Check, then add a station of your own (the [Day 2, section 4.3](../day02/04-persisting-data-and-web-ui.md#postgresql-state-durability-verification) experiment).

1. Check the seed data:

```bash
POD=$(kubectl get pod -l app=climate-db -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$POD" -- psql -U ecmwf_admin -d climate_db -c "SELECT * FROM climate_telemetry;"
```

Expected output:

```text
 id |     station_id      | temperature_celsius |         timestamp
----+---------------------+---------------------+----------------------------
  1 | STATION-ATHENS-01   |               37.40 | 2026-10-23 13:03:58.585827
  2 | STATION-BOLOGNA-02  |               29.80 | 2026-10-23 13:03:58.585827
  3 | STATION-VALENCIA-03 |               34.10 | 2026-10-23 13:03:58.585827
(3 rows)
```

2. Add one more row:

```bash
kubectl exec "$POD" -- psql -U ecmwf_admin -d climate_db -c \
  "INSERT INTO climate_telemetry (station_id, temperature_celsius) VALUES ('STATION-MARSEILLE-04', 31.6);"
```

Expected output:

```text
INSERT 0 1
```

### Step 6: destroy the Pod and let the Deployment recreate it

This is `docker rm -f climate-db` followed by a new `docker run` on the same volume — except that nobody has to run the second command.

**Before you run it, predict:** how many rows will the new Pod show? Will `init.sql` run again?

Delete the Pod, and wait until the Deployment has a new one ready:

```bash
kubectl delete pod "$POD"
kubectl rollout status deployment/climate-db
```

Find the name of the new Pod, and compare it with the old one:

```bash
NEW_POD=$(kubectl get pod -l app=climate-db -o jsonpath='{.items[0].metadata.name}')
echo "$POD -> $NEW_POD"
```

Read the table from the new Pod:

```bash
kubectl exec "$NEW_POD" -- psql -U ecmwf_admin -d climate_db -c "SELECT * FROM climate_telemetry;"
```

:::{dropdown} What happened, and why
Four rows, new Pod — and `init.sql` did **not** run again, because the data directory on the PVC already existed, exactly as with the [named volume on Day 2](../day02/04-persisting-data-and-web-ui.md#postgresql-state-durability-verification).

Note the extra guarantee compared with Day 2: `kubectl get pod -o wide` may show the new Pod on a *different node*, and the data followed it — a named Docker volume never leaves its host.
:::

:::{dropdown} 📖 Optional: prove the opposite
Comment out the `data` entries under `volumeMounts` and `volumes` in `db.yaml`, `kubectl apply`, and query the table again: three rows, Marseille is gone (`init.sql` ran again on an empty, ephemeral directory). Restore the lines and apply once more: four rows, because the PVC was never deleted. **Check that the count is 4 again before you go on: section 10 expects 4.** Like `docker compose down` without `-v`, deleting workloads never deletes claims.
:::

**Checkpoint 8** — the PVC is `Bound`, and a **new** PostgreSQL Pod still has four rows.

```bash
kubectl get pvc climate-db-data
kubectl exec deploy/climate-db -- psql -U ecmwf_admin -d climate_db -tAc "SELECT count(*) FROM climate_telemetry;"
```

```text
NAME              STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
climate-db-data   Bound    pvc-3f2a…       1Gi        RWO            <default>      15m
4
```

:::{dropdown} Not there?
- PVC `Pending` and Pod `Pending`: no default StorageClass, or the class cannot provision. `kubectl describe pvc climate-db-data` shows the event; ask the instructor which StorageClass to use.
- Pod in `CrashLoopBackOff` with `directory ... exists but is not empty`: `PGDATA` is missing, so PostgreSQL sees `lost+found`. Check the `env` block of `db.yaml`.
- Count `3`: the `INSERT` of Step 5 did not run (or ran in a Pod without the volume).
- `role "ecmwf_admin" does not exist`: the Secret was created after the first start with other values. Delete the Deployment **and** the PVC, then apply both again.
:::

:::{dropdown} If you stop here (you skip section 10)
Section 10 builds on this database. If you do not continue with it, remove the stack now:

```bash
kubectl delete -f ~/climate-stack-k8s/ --ignore-not-found
kubectl delete pvc climate-db-data --ignore-not-found
```
:::

## 9.3 Core takeaway

> **Data that must survive a Pod goes on a PersistentVolumeClaim; the claim outlives every Pod that uses it.**

- A **PVC** is your request for storage: the named volume of Day 2, with a size, satisfied by a PersistentVolume that the platform provides.
- Data on a PVC survives Pod replacement, and can follow the Pod to another node.
- A file that Compose bind-mounted (`init.sql`) becomes a **ConfigMap mounted as a file**.
- Deleting workloads never deletes claims: PVCs are removed explicitly.

## 9.4 📖 Check yourself

1. Your PVC stays `Pending` and no Pod uses it yet. Is that a problem?
2. Why does the database Deployment use `strategy: Recreate`?
3. You delete the Deployment `climate-db` and apply it again. Are the rows still there? Why?
4. Why is `PGDATA` set to a subfolder of the mount path?
5. When would you use a StatefulSet instead of this Deployment?

:::{dropdown} Solutions
**1.** Not necessarily: many StorageClasses create the volume only when the first Pod uses the claim (`WaitForFirstConsumer`). It is a problem only if it stays `Pending` after the Pod is scheduled: then `kubectl describe pvc` shows why.

**2.** The volume is `ReadWriteOnce`: one node at a time. A rolling update would start the new Pod while the old one still holds the volume, and the new Pod could wait for ever. `Recreate` stops the old Pod first.

**3.** Yes. Deleting workloads never deletes claims: the PVC and its data stay, and the new Pod mounts them again.

**4.** The root of a new volume often contains `lost+found`. PostgreSQL refuses to initialise a data directory that is not empty; a subfolder is empty.

**5.** With several database replicas, each with its own volume and a stable name (`db-0`, `db-1`), and ordered start-up.
:::

Continue with: [10. From Compose to Kubernetes](10-from-compose-to-kubernetes.md).
