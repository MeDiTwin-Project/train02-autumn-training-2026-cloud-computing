# 3. Deploying an application

In [Hands-on 1](02-interacting-with-a-cluster.md#hands-on-1) you created a Pod by hand: `hello` ran once, and then nothing looked after it. That is fine for a test, not for a service that must run all the time. A bare Pod is not recreated if it crashes, is deleted, or its node fails; you cannot ask for three copies of it; and to change its image you must delete it and create it again, with a gap in service.

A **Deployment** solves this. You no longer create Pods: you describe them once (a *template*) and say how many you want. Kubernetes creates them, replaces any that disappear, and updates them one by one when you change the template. In this section you deploy the image you built on Day 2, `climate-telemetry` ([Day 2, section 5](../day02/05-application-containerisation.md)), this way.

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

## 3.1 Deployment flow

```{mermaid}
flowchart TB
    IMG[Image in the registry]
    DEP[Deployment]
    RS[ReplicaSet]
    PODS[Pods]
    SVC[Service]

    IMG -->|used in pod template| DEP
    DEP -->|creates and updates| RS
    RS -->|maintains desired replicas| PODS
    SVC -->|selects matching labels| PODS
```

This view highlights two key points:

- **Deployment** manages lifecycle and updates through a **ReplicaSet**.
- **Service** does not create Pods; it selects existing Pods via labels.

### 3.1.1 What are Deployment and ReplicaSet?

- **ReplicaSet** is the controller that keeps a target number of Pod replicas running (for example `3` Pods). If one Pod crashes or is deleted, the ReplicaSet creates a replacement.
- **Deployment** is a higher-level controller that manages ReplicaSets over time. It adds rollout strategy, rollout history, and rollback.

In practice:

- You usually create and edit a **Deployment**.
- Kubernetes creates and updates the underlying **ReplicaSet** for you.
- You inspect ReplicaSets mainly for troubleshooting rollout behavior.

:::{note} Compare with Day 2
[`docker run`](../day02/03-running-containers.md#id-3-2-container-lifecycle-states) was one command that pulled, created and started a container on *this* host. Here the Deployment is the desired state, and the scheduler decides on which node the Pods run. That is also why the image must come from a registry: the node that runs your Pod has never seen the `docker build` cache of your VM.
:::

## 3.2 ⌨️ First: imperative workflow

Start quickly with imperative commands to see immediate results. Create a Deployment from the `nginx` image:

```bash
kubectl create deployment nginx --image=nginx:1.27
```

List the Pods: the Deployment created one for you. Its name is the name of the Deployment plus a random suffix:

```bash
kubectl get pods
```

```text
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7c5ddbdf54-x2kqp   1/1     Running   0          10s
```

Look at the Deployment itself: how many Pods it wants, and how many are ready:

```bash
kubectl get deployment
```

```text
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           15s
```

Delete the Deployment: its Pods go with it.

```bash
kubectl delete deployment nginx
```

This is the `docker run -d nginx` of Kubernetes. It is useful for quick tests, but it becomes hard to track and reproduce changes over time — the same reason [Day 2](../day02/06-docker-compose.md) moved from `docker run` flags to `compose.yaml`.

## 3.3 Why YAML is usually preferable

The previous step used an imperative command:

```bash
kubectl create deployment nginx --image=nginx:1.27
```

Now we can express the **same workload** with YAML, so the desired state is explicit, reviewable, and reproducible. This is the same step as going from `docker create` or `docker run` flags to `compose.yaml` on [Day 2](../day02/06-docker-compose.md#id-6-3-the-compose-specification-compose-yaml): the file *is* the documentation of what runs.

Create `nginx-deployment.yaml` in your home directory (`cd ~`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - name: http
          containerPort: 80
```

Apply it:

```bash
kubectl apply -f nginx-deployment.yaml
```

You now get the same functional result as `kubectl create deployment ...`, but with one major operational difference: the state is in a file you can version, review, and re-apply.

### 3.3.1 Anatomy of a manifest

Every Kubernetes object, of every kind, has the same top-level fields: you already met them in the `-o yaml` of the Pod `hello` ([Hands-on 1, Step 6](02-interacting-with-a-cluster.md#hands-on-1-step-6)). There, Kubernetes showed them to you; in a manifest, you write the first four yourself. Learn them once, and you can read every manifest of the day:

| Field | What it says | In the `nginx` Deployment |
|---|---|---|
| `apiVersion` | Which API defines this kind of object: a **group** and a **version** | `apps/v1`: group `apps`, version `v1`. Pods, Services, ConfigMaps and Secrets use the core group, written only `v1` |
| `kind` | The type of object | `Deployment` |
| `metadata` | Who the object is: its `name`, its `namespace`, its `labels` and `annotations` | `name: nginx` |
| `spec` | The **desired state**: what you want. You write it | `replicas`, `selector`, `template` |
| `status` | The **actual state**: what is true now. Kubernetes writes it; it is never in your file | See it with `kubectl get deployment nginx -o yaml` (section 2.3) |

A Deployment contains a second object: `spec.template` is the **model of the Pods** that it creates, with its own `metadata` and `spec`. Two places must agree:

- `spec.selector.matchLabels` says which Pods belong to the Deployment;
- `spec.template.metadata.labels` are the labels that each new Pod receives.

If they differ, the API server refuses the Deployment. The same idea — a selector that must match labels — comes back with Services (section 4).

You do not need to remember every field. Ask the cluster:

```bash
kubectl explain deployment.spec                      # the fields of spec, with a description
kubectl explain deployment.spec.template.spec.containers
kubectl api-resources | head -20                     # every kind, its short name and its API group
```

`kubectl explain` answers from the cluster itself, so it always matches the version you are using.

:::{tip} The official documentation: concepts, and examples to start from
`kubectl explain` tells you *which* fields exist; the [Kubernetes documentation](https://kubernetes.io/docs/home/) tells you *why* and *how* to use them. It is detailed and well maintained, and most pages include a complete manifest that you can copy and adapt:

- [Concepts](https://kubernetes.io/docs/concepts/): one page per object, for example [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), with an example manifest at the top.
- [Tasks](https://kubernetes.io/docs/tasks/): step-by-step guides, for example configuring probes or using a ConfigMap.
- [kubectl reference](https://kubernetes.io/docs/reference/kubectl/) and [API reference](https://kubernetes.io/docs/reference/kubernetes-api/): every command and every field.

The site describes the latest Kubernetes release. The training cluster runs 1.33 (Hands-on 1): almost everything used today is the same, but when a field or a feature is missing on the cluster, `kubectl explain` is the reference.
:::

You do not need `nginx` any more. Remove it:

```bash
kubectl delete -f ~/nginx-deployment.yaml
```

:::{note} Jargon: *CRD* (CustomResourceDefinition)
Kubernetes can be **extended** with new kinds of objects. A **CRD** (CustomResourceDefinition) teaches the API server a new `kind`, in its own API group; a controller installed with it does the work. You meet several of them in sections 4 and 6: a `Middleware` of Traefik (`traefik.io/v1alpha1`, section 4.7.1), a `Certificate` and a `ClusterIssuer` of cert-manager (`cert-manager.io/v1`, section 6). They have the same shape and you apply them with the same `kubectl apply`. Only the `apiVersion` tells you that they are not built in.
:::

Why this approach scales better:

- You can keep the YAML files with the code, and review every change to them (Day 4 shows how).
- You can apply the same manifest in multiple environments.
- You can reason about current state vs desired state more clearly.

:::{note} Tag immutability reminder
Use tag `1.27`, not `latest`. The [Day 2 rule](../day02/05-application-containerisation.md#id-5-6-image-tagging-taxonomies-and-versioning) — *no `latest` in production* — matters even more here, because `kubectl apply` with an unchanged `image:` line changes **nothing**: Kubernetes compares desired state with desired state, and `latest` looks identical to `latest` even when the registry content changed.
:::

## 3.4 `HEALTHCHECK` becomes probes

On Day 2 the [capstone Dockerfile](../day02/07-lab.md#phase-2-multi-stage-image-compilation) declared `HEALTHCHECK ... curl -f http://localhost:8000/health`, and `compose.yaml` used `healthcheck:` + `depends_on: condition: service_healthy` ([Day 2, section 6.5](../day02/06-docker-compose.md#id-6-5-synchronization-and-healthcheck-gated-dependencies)). **Kubernetes ignores the Dockerfile `HEALTHCHECK`** entirely. You declare health in the Pod template, as two probes with two different consequences:

| Probe | Question | If it fails |
|---|---|---|
| `readinessProbe` | Can this Pod receive traffic *right now*? | The Pod is removed from the Service endpoints; nothing is restarted |
| `livenessProbe` | Is this process still alive? | The container is **restarted** |

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: http
  periodSeconds: 5
livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 10
```

`port: http` works only if the container declares a port named `http` (for example `ports: [{name: http, containerPort: 8000}]`).

Never check a dependency (a database, another service) in the liveness probe: if the dependency is down, Kubernetes restarts your healthy Pod for nothing. Dependencies belong, at most, in the readiness probe.

The readiness probe is also what replaces `depends_on`: there is no start ordering in Kubernetes. A Pod that is not ready simply receives no traffic, and the application is expected to retry its own dependencies.

## 3.5 ⌨️ Hands-on 2: deploy the Day 2 image

### Step 0: find your image in the registry

The cluster nodes pull images from the GitLab Container Registry, never from a local `docker build` cache. On [Day 2, section 5.7](../day02/05-application-containerisation.md#id-5-7-oci-container-distribution-registries) you pushed `climate-telemetry:1.0.0` there. Check it is listed: open your `climate-telemetry-traineeNN` project in GitLab, *Deploy → Container Registry*, tag `1.0.0`. The full image path is:

```text
gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:1.0.0
```

If it is missing, repeat the `docker login` / `docker tag` / `docker push` steps of [Day 2, section 5.7](../day02/05-application-containerisation.md#id-5-7-oci-container-distribution-registries).

### Step 1: registry credentials (required)

Your project image is private, so cluster nodes need registry credentials to pull it.

For this hands-on, to keep the flow simple, reuse the Docker login config already present on the training VM:

```bash
kubectl create secret generic gitlab-registry \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
```

:::{dropdown} 📖 Best practices for production
If `~/.docker/config.json` contains `"credsStore"` instead of `"auths"`, the credentials are in a helper, not in the file, and the Secret is empty: ask the instructor.

- Use a **dedicated access token** for image pulls, not personal credentials from a workstation or VM.
- Grant **least privilege**: for pull-only workloads, use only `read_registry` (no write/delete scopes).
- Scope the token to the specific project or group required by the workload.
- Rotate tokens regularly and immediately on suspected exposure.
- Store the token only in Kubernetes Secrets and restrict who can read those Secrets with RBAC.
- Prefer attaching `imagePullSecrets` to a dedicated ServiceAccount instead of repeating them in many manifests.
:::

Reference that secret in the Pod template:

```yaml
    spec:
      imagePullSecrets:
        - name: gitlab-registry
      containers:
        - ...
```

A private image without this gives `ErrImagePull` with `unauthorized` in `kubectl describe pod`, which is the first thing to check when a pull fails.

### Step 2: create the Deployment manifest

Create a working folder for today and the first manifest:

```bash
mkdir -p ~/climate-service-k8s && cd ~/climate-service-k8s
```

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: climate-telemetry
  labels:
    app: climate-telemetry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: climate-telemetry
  template:
    metadata:
      labels:
        app: climate-telemetry
    spec:
      imagePullSecrets:
        - name: gitlab-registry
      containers:
        - name: app
          image: gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:1.0.0
          env:
            - name: APP_VERSION
              value: "1.0.0"
          ports:
            - name: http
              containerPort: 8000
          readinessProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              memory: 128Mi
```

Read it against the [Day 2 Dockerfile](../day02/05-application-containerisation.md#id-5-2-anatomy-of-a-production-grade-dockerfile) and `compose.yaml`:

| Part of the manifest | What it does | Day 2 twin |
|---|---|---|
| `replicas: 1` | How many identical Pods the Deployment keeps running | One `docker run` |
| `selector.matchLabels` and `template.metadata.labels` | The Deployment finds *its* Pods by the label `app: climate-telemetry`; the two must match | `--name` (Kubernetes uses labels, not names) |
| `imagePullSecrets` | The credentials the node uses to pull a private image (Step 1) | `docker login` |
| `image` | The tag you pushed to the registry | `docker run <image>` |
| `env` | The `APP_VERSION` variable that the app reads | `-e APP_VERSION=…` / `environment:` |
| `ports` (`name: http`, `8000`) | Documents the port and gives it a name that probes and Services can use | `EXPOSE 8000` |
| `readinessProbe` / `livenessProbe` | Ready for traffic? Still alive? (section 3.4) | `HEALTHCHECK` / `healthcheck:` |
| `resources.requests` / `limits` | What the scheduler reserves / what the kernel allows (section 7) | `docker run -m` (cgroups) |

:::{note} `USER 10001` and `runAsNonRoot`
The [Day 2 Dockerfile](../day02/05-application-containerisation.md#id-5-2-anatomy-of-a-production-grade-dockerfile) drops privileges with `USER 10001`. Kubernetes can *enforce* that with `securityContext: {runAsNonRoot: true}`: the kubelet checks that the user ID of the image is not 0. It can check only a **number**: with a name such as `USER appuser`, the Pod fails with `container has runAsNonRoot and image has non-numeric user`. That is why Day 2 uses a fixed numeric UID. Day 5 comes back to this.
:::

### Step 3: apply the manifest

```bash
kubectl apply -f deployment.yaml
```

### Step 4: check status

```bash
kubectl get deployment climate-telemetry
kubectl rollout status deployment/climate-telemetry
kubectl get pods -l app=climate-telemetry -o wide
```

If the Pod is stuck in `ErrImagePull` or `ImagePullBackOff`, `kubectl describe pod <pod>` tells you whether the image path or tag is wrong (or, for a private image, that credentials are missing). If the Pod is `Running` but `0/1` ready, the readiness probe is failing: `kubectl logs <pod>`.

Confirm the app answers, from inside the Pod, exactly like `docker exec` on [Day 2](../day02/03-running-containers.md#id-4-enter-container-for-live-diagnostics-docker-exec):

Find the name of the Pod, by its label:

```bash
POD=$(kubectl get pod -l app=climate-telemetry -o jsonpath='{.items[0].metadata.name}')
```

Open a shell in it:

```bash
kubectl exec -it "$POD" -- sh
```

Inside the Pod, call `/health` (the image has Python, not `curl`):

```bash
python3 -c "import urllib.request;print(urllib.request.urlopen('http://localhost:8000/health').read().decode())"
```

Leave the Pod:

```bash
exit
```

The `node` field in the answer comes from `socket.gethostname()` in the [Day 2 `app.py`](../day02/05-application-containerisation.md#id-5-1-dockerfile-specification-and-directives), i.e. the hostname seen from inside the container. With Docker that was the container ID; here it is the **Pod name** (despite the field's name, it is not the Kubernetes node). It is what will let you see, in section 4, which replica answered each request.

### Step 5: roll out a change, and roll it back

Change the image to a tag that does not exist. **Before you run the commands, predict:** the Deployment has one working Pod. Will the application stop answering? What will `kubectl get pods` show?

```bash
kubectl set image deployment/climate-telemetry \
  app=gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:9.9.9
```

```bash
kubectl rollout status deployment/climate-telemetry --timeout=15s
```

```text
Waiting for deployment "climate-telemetry" rollout to finish: 1 old replicas are pending termination...
error: timed out waiting for the condition
```

This error is expected: the new Pod cannot pull the image, so the rollout never finishes. Look at the Pods:

```bash
kubectl get pods -l app=climate-telemetry
```

Compare with your prediction, then open the solution.

:::{dropdown} What happened, and why
The rollout never completes: one new Pod is in `ImagePullBackOff` (the tag does not exist), but the old Pod is still `Running` and still serving. That is the rolling update strategy at work: the Deployment never takes down a working Pod before its replacement is ready. The application did not stop.
:::

Now roll back:

```bash
kubectl rollout history deployment/climate-telemetry
kubectl rollout undo deployment/climate-telemetry
kubectl rollout status deployment/climate-telemetry
```

Check `kubectl get deployment climate-telemetry -o jsonpath='{.spec.template.spec.containers[0].image}'`: you are back on `1.0.0`. Note that `kubectl set image` changed the cluster but not `deployment.yaml`: in real life you edit the file and `apply` it, so that the file and the cluster agree. Day 4 makes that automatic.

**Checkpoint 2** — `climate-telemetry` runs from your registry image, version `1.0.0`, after a failed rollout and a rollback.

The Deployment and its image:

```bash
kubectl get deployment climate-telemetry
kubectl get deployment climate-telemetry -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

```text
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
climate-telemetry   1/1     1            1           5m
gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:1.0.0
```

The application, from inside the Pod:

```bash
POD=$(kubectl get pod -l app=climate-telemetry -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it "$POD" -- sh
```

```bash
python3 -c "import urllib.request;print(urllib.request.urlopen('http://localhost:8000/health').read().decode())"
exit
```

```text
{"status":"healthy","service":"ecmwf-climate-telemetry","version":"1.0.0","node":"climate-telemetry-…"}
```

:::{dropdown} Not there?
- `ErrImagePull` / `ImagePullBackOff` with `unauthorized`: the `gitlab-registry` Secret is missing or wrong (Step 1).
- `ErrImagePull` with `not found`: wrong image path or tag. Compare with the registry page in GitLab (Step 0).
- `InvalidImageName`: `traineeNN` was not replaced in `image:`. Use your real project path (for example `climate-telemetry-trainee07`).
- `READY 0/1` and the Pod is `Running`: the readiness probe fails. `kubectl describe pod` shows the probe error; `kubectl logs` shows the app.
- Image still `:9.9.9`: the rollback did not run. `kubectl rollout undo deployment/climate-telemetry`.
- Probe error `strconv.Atoi: parsing "http": invalid syntax`: the probe uses `port: http`, but no container port is named `http`. Add `name: http` under `ports:`.
:::

## 3.6 Core takeaway

> **Kubernetes does not just execute commands; it continuously maintains desired state.**

- A **Deployment** manages a **ReplicaSet**, which keeps the right number of **Pods** running.
- Write the desired state in **YAML** and `kubectl apply` it: the file is the documentation of what runs.
- The image must come from a **registry**; a private registry needs an `imagePullSecret`.
- **Readiness** decides who receives traffic; **liveness** decides who is restarted. The Dockerfile `HEALTHCHECK` is ignored.
- A rolling update never removes a working Pod before its replacement is ready; `kubectl rollout undo` goes back.

## 3.7 📖 Check yourself

1. You push a new image with the same tag `latest` and run `kubectl apply -f deployment.yaml` again. Why does nothing change?
2. A Pod is `Running` but `READY 0/1`. Which probe fails, and what are the consequences for traffic and for restarts?
3. A Pod is in `ImagePullBackOff`. Give two possible causes, and the command that tells you which one.
4. You used `kubectl set image` and then `kubectl rollout undo`. Do the cluster and `deployment.yaml` agree now? What did the cluster go through that the file never recorded?
5. Why must `port: http` in a probe match a `name:` under `ports:`?

:::{dropdown} Solutions
**1.** Kubernetes compares the desired state with the desired state: the `image:` line did not change, so there is nothing to roll out. A new build needs a new tag.

**2.** The **readiness** probe. The Pod is removed from the Service endpoints, so it receives no traffic, but it is **not** restarted. A failing **liveness** probe would restart it.

**3.** A wrong image path or tag (`not found`), or missing or wrong registry credentials (`unauthorized`). `kubectl describe pod <pod>` shows the message in the events.

**4.** After `rollout undo`, the cluster runs `1.0.0` again, as the file says, so the image agrees. But the cluster changed twice without the file: in real work you edit the file and `apply` it. Day 4 makes the file in Git the only way to change the cluster.

**5.** The kubelet resolves the name to a container port. Without a port called `http`, it reports an error and the probe never succeeds.
:::

Continue with: [4. Exposing an application](04-exposing-an-application.md).
