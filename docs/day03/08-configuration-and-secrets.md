# 8. Configuration and Secrets

Your `climate-telemetry` Deployment still has its settings written inside the manifest. This section moves them out, into two kinds of objects:

- **ConfigMap**: configuration that changes across environments.
- **Secret**: sensitive values you do not want in images or in shared files.

Both were in the `compose.yaml` of [Day 2, section 6](../day02/06-docker-compose.md#id-6-3-the-compose-specification-compose-yaml), mixed in the same `environment:` block: the database password (`POSTGRES_PASSWORD=EcmwfPassphrase123!`) sits in clear next to harmless settings such as `DB_HOST=db`. Kubernetes separates them.

You keep working in `~/climate-service-k8s`, on `climate-telemetry`.

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

## 8.1 ConfigMap

A **ConfigMap** is a Kubernetes object used to keep application configuration outside the container image.
It stores non-sensitive key-value data (for example feature flags, hostnames, log levels, or app versions) and Pods can consume it as environment variables, files, or command arguments.
Use a ConfigMap when you want the same image to run in different environments with different settings; use a Secret instead for sensitive values.

### ⌨️ Hands-on 7: move `APP_VERSION` to a ConfigMap

Take `APP_VERSION`, which the [Day 2 app](../day02/05-application-containerisation.md#id-5-1-dockerfile-specification-and-directives) reads and reports in `/health`. In Hands-on 2 it is a literal `value:` in the Deployment; move it to a ConfigMap.

#### Step 1: create the ConfigMap

Go to the folder of the `climate-telemetry` manifests:

```bash
cd ~/climate-service-k8s
```

Create `configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: climate-config
data:
  APP_VERSION: "1.0.0"
  LOG_LEVEL: info
```

#### Step 2: read it from the Deployment

In `deployment.yaml`, replace the existing literal block:

```yaml
          env:
            - name: APP_VERSION
              value: "1.0.0"
```

with one of these two patterns (do not keep both the old literal and the new ConfigMap-based definition):

Option A, explicit keys with `env`:

```yaml
          env:
            - name: APP_VERSION
              valueFrom:
                configMapKeyRef:
                  name: climate-config
                  key: APP_VERSION
```

(`LOG_LEVEL` stays in the ConfigMap as an extra key, but this app does not read it.)

Option B, import all keys at once with `envFrom`:

```yaml
          envFrom:
            - configMapRef:
                name: climate-config
```

`envFrom` is the closest equivalent of Compose's `env_file:`.

#### Step 3: apply

Apply the ConfigMap and the updated Deployment, and wait for the rollout:

```bash
kubectl apply -f configmap.yaml -f deployment.yaml
kubectl rollout status deployment/climate-telemetry
```

Look at the ConfigMap as the cluster stores it:

```bash
kubectl get configmap climate-config -o yaml
```

**Checkpoint 7** — the Deployment reads `APP_VERSION` from the ConfigMap, and the service still works.

```bash
kubectl exec deploy/climate-telemetry -- printenv APP_VERSION
kubectl get deployment climate-telemetry
```

```text
1.0.0
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
climate-telemetry   3/3     3            3           40m
```

:::{dropdown} Not there?
- Pods in `CreateContainerConfigError`: the ConfigMap does not exist or has another name. `kubectl get configmap climate-config`, and check `name:` under `configMapKeyRef` / `configMapRef`.
- `printenv APP_VERSION` prints nothing: the Deployment is still using literal `value:` or references the wrong key/name. Re-apply `deployment.yaml` and run `kubectl rollout restart deployment/climate-telemetry`.
:::

:::{note} Changing a ConfigMap does not restart Pods
Environment variables are read at container start. After editing the ConfigMap, run `kubectl rollout restart deployment/climate-telemetry` to pick up the new values. (A ConfigMap mounted as a *file* is updated in place, but the application must re-read it.)
:::

## 8.2 Secret

A **Secret** stores sensitive data and is consumed like a ConfigMap. You already have one: in Hands-on 2, step 1, the registry credentials went into the Secret `gitlab-registry`, of type `docker-registry`. The generic kind is `Opaque`. For example, the database credentials of the [Day 2 `climate-stack`](../day02/06-docker-compose.md#id-6-3-the-compose-specification-compose-yaml) look like this (you create this exact file in section 9, where the database needs it):

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

A container uses it as environment variables, exactly like a ConfigMap — one key with `env`:

```yaml
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: climate-db
                  key: POSTGRES_PASSWORD
```

or all keys at once with `envFrom: [{secretRef: {name: climate-db}}]`.

### `env` vs `envFrom`: differences and when to use each

- Use **`env`** when you want explicit, reviewable mapping of selected variables.
- Use **`envFrom`** when you want to inject a full bundle of keys with less YAML.
- Prefer **`env`** for sensitive or high-impact variables where explicitness matters.
- Prefer **`envFrom`** when many variables belong together and naming is already consistent.
- Avoid key collisions across sources; if collisions are possible, prefer explicit `env`.

### ⌨️ Try it: a Secret is encoded, not encrypted

Create a throw-away Secret from the command line, read it back, and delete it:

```bash
kubectl create secret generic demo-secret --from-literal=password=not-a-real-password
kubectl get secret demo-secret -o yaml
kubectl get secret demo-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl delete secret demo-secret
```

The `-o yaml` output shows `bm90LWEtcmVhbC1wYXNzd29yZA==`: that is **base64**, an encoding, not encryption. One `base64 -d` gives the password back. So:

> **Kubernetes Secret does not automatically mean encrypted and secure secret management.**

Anyone who can read the Secret object can read the password. Security depends on cluster setup and policy:

- **RBAC** controls who can read or modify Secret resources (your namespace is yours; other trainees cannot read your Secrets).
- **Encryption at rest** in etcd must be explicitly configured by cluster operators.
- External secret managers (for example Vault or cloud secret stores) are common in production. Day 4 shows one.

For this course, the practical user question is: how to inject sensitive values safely into workloads while respecting namespace boundaries and RBAC. Rule: keep the file of a Secret apart from the other manifests, and never share or publish it. Day 4 shows why.

### ⌨️ Cleanup: `climate-telemetry` is done for now

Sections 9 to 11 work on other applications. Remove the `climate-telemetry` objects from the cluster, but **keep the files**: section 12 turns them into a Helm chart and deploys the service again.

```bash
cd ~/climate-service-k8s
kubectl delete -f . --ignore-not-found
kubectl get deploy,svc,ingress
```

```text
No resources found in traineeNN namespace.
```

The `gitlab-registry` Secret stays: you created it with a command, not from a file, and section 10 needs it to pull the `climate-stats-api` image. The TLS Secret `climate-telemetry-tls` stays too: in section 12 cert-manager reuses it.

## 8.3 Core takeaway

> **Settings go in a ConfigMap, passwords in a Secret: the image stays the same in every environment.**

- **ConfigMap** for non-sensitive configuration: the harmless part of the `environment:` block, versioned with the other manifests.
- **Secret** for sensitive values: the passwords of the `environment:` block, *not* versioned. Base64 is not encryption: protection comes from RBAC and the platform.

- `env` maps keys one by one; `envFrom` imports all the keys of a ConfigMap or Secret.
- Environment variables are read at start: after a change, `kubectl rollout restart`.

## 8.4 📖 Check yourself

1. You change `APP_VERSION` in the ConfigMap and apply it. `/health` still reports the old version. Why, and what do you run?
2. A colleague says: "the password is safe, it is in a Secret". What do you answer?
3. When do you prefer `env` with `secretKeyRef` to `envFrom`?
4. The Pods of a Deployment are in `CreateContainerConfigError`. What do you check?

:::{dropdown} Solutions
**1.** Environment variables are read when the container starts. Run `kubectl rollout restart deployment/climate-telemetry`.

**2.** A Secret is base64-*encoded*, not encrypted: anyone who can read it decodes it with `base64 -d`. Its protection is RBAC (who can read Secrets), encryption at rest configured by the cluster operators, and keeping the file out of Git.

**3.** When you want to see in the manifest exactly which sensitive values a container receives, or when key names could collide with other sources.

**4.** That the ConfigMap or Secret it references exists in the namespace, with that exact name and key: `kubectl describe pod` names the missing one.
:::

Continue with: [9. Persistent storage](09-persistent-storage.md).
