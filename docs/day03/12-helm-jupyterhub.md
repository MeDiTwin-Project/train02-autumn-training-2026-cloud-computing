# 12. Helm: installing charts and writing your own

Helm is the package manager for Kubernetes. It lets you install and update complex applications using reusable charts and parameterized values.

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

## 12.1 Why Helm

Without Helm, operating multi-resource applications means maintaining many YAML files by hand. The `climate-telemetry` service alone needs five (`configmap.yaml`, `deployment.yaml`, `service.yaml`, `ingress.yaml`, `middleware-redirect-https.yaml`); JupyterHub needs several dozen.

**Helm gives you**:

- Templating and values overrides.
- Versioned chart releases.
- Repeatable upgrades and rollbacks.

Think of the analogy with [Day 2](../day02/05-application-containerisation.md): a **Dockerfile** packages one application into an image that anyone can run with a few parameters; a **chart** packages a whole deployment (Deployments, Services, Ingress, ConfigMaps, PVCs…) that anyone can install with a `values.yaml`. Pulling `postgres:16-alpine` instead of writing your own database image is the same decision as installing the JupyterHub chart instead of writing its manifests.

## 12.2 Helm concepts

:::{note} Jargon: *chart*, *release*, *values*
A **chart** is a package: Kubernetes manifests written as templates, plus their default settings. A **release** is one installation of a chart in a namespace, with a name and a numbered history of revisions. **Values** are the settings that fill the templates for one release.
:::

The three words have Docker twins:

| Helm | What it is | Docker twin |
|---|---|---|
| **Chart** | Templates + default values + metadata (`Chart.yaml`) | An image |
| **Release** | One installation of the chart, with a name (`jhub`, `climate-telemetry`) | A container started from the image |
| **Values** | The settings for that installation (`-f values.yaml`, `--set`) | The `-e`, `-p`, `-v` flags of `docker run` |
| **Repository** | Where charts are published, with versions | A registry, with tags |

## 12.3 Repository setup

```bash
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update
helm search repo jupyterhub/jupyterhub --versions | head
```

This is the `docker pull` side of Helm: a repository of charts, each with versions, like image tags.

## 12.4 ⌨️ Hands-on 10: deploy JupyterHub

JupyterHub goes in your own namespace, `traineeNN`: do **not** create a new namespace. Your context already points at `traineeNN` ([Hands-on 1](02-interacting-with-a-cluster.md#hands-on-1)), so no command below needs `-n`. The chart's objects have their own names (`hub`, `proxy`, `proxy-public`, …), so they do not collide with yours.

:::{important} A minimal installation, not a production one
This JupyterHub is a **teaching example**: it shows how one `helm` command installs and manages a complex application (a hub, a proxy, an Ingress, a database on a PVC, one Pod per user, network policies, RBAC). It is **not production ready**. A real deployment would at least use a proper authenticator (for example OAuth/OIDC with your organisation's accounts) instead of a shared password, size the storage and the per-user resources for the real workload, plan backups of the hub database, enable the components switched off below where the cluster allows it, and follow the [Zero to JupyterHub](https://z2jh.jupyter.org/en/stable/) guide for security and scaling.
:::

JupyterHub is a third application, so its files get their own folder, next to `~/climate-service-k8s` and `~/climate-stack-k8s`:

```bash
mkdir -p ~/jhub && cd ~/jhub
```

Run all the commands of this section from there.

### 12.4.1 Where the values come from

A chart has **default values** for every setting. Your values file only lists what you want to change; everything else keeps the default. Before writing it, look at the defaults:

- **From the command line**, for exactly the chart version you install:

  ```bash
  helm show values jupyterhub/jupyterhub --version 4.4.4 > jhub-default-values.yaml
  less jhub-default-values.yaml
  ```

- **On the web**:
  - the same file in the chart's source, at the same version: [`jupyterhub/values.yaml` at tag 4.4.4](https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/4.4.4/jupyterhub/values.yaml);
  - the [configuration reference](https://z2jh.jupyter.org/en/stable/resources/reference.html), which explains each key.

Two examples of defaults that the file below changes: `proxy.service.type` is `LoadBalancer` by default (we use `ClusterIP`, because traffic arrives through the Ingress), and `ingress.enabled` is `false`.

### 12.4.2 Install

Create a minimal values file named `values-jhub.yaml`. The Ingress block is the one you wrote by hand in Hands-on 4 (without the HTTP → HTTPS redirect), in the chart's own vocabulary:

```yaml
hub:
  config:
    JupyterHub:
      authenticator_class: dummy
    Authenticator:
      allow_all: true
    DummyAuthenticator:
      password: "<choose-a-password>"   # shared password for this training hub

proxy:
  service:
    type: ClusterIP

ingress:
  enabled: true
  ingressClassName: traefik
  hosts:
    - jhub-traineeNN.training.meditwin-project.eu
  tls:
    - hosts:
        - jhub-traineeNN.training.meditwin-project.eu
      secretName: jhub-tls
  annotations:
    cert-manager.io/cluster-issuer: ecmwf-meditwin-training-issuer

singleuser:
  image:
    name: quay.io/jupyter/minimal-notebook
    tag: "2024-10-07"
  cpu:
    guarantee: 0.5
    limit: 2
  memory:
    guarantee: 1G
    limit: 4G
  storage:
    type: none          # no PVC per user: the home directory is temporary (see the note below)

# Components that need cluster-wide permissions, or that run on every node:
# not allowed (and not needed) in a shared training namespace.
scheduling:
  userScheduler:
    enabled: false
  userPlaceholder:
    enabled: false
prePuller:
  hook:
    enabled: false
  continuous:
    enabled: false
```

`singleuser.cpu.guarantee` / `limit` are `resources.requests` / `limits` from section 7, applied to every user's notebook Pod. The image tag is pinned, for the [Day 2 reason](../day02/05-application-containerisation.md#id-5-6-image-tagging-taxonomies-and-versioning).

The last two blocks switch off parts of the chart that a normal user of a shared cluster cannot, and should not, install:

- `scheduling.userScheduler` is an extra Kubernetes scheduler that packs notebooks onto as few nodes as possible. It needs a **ClusterRole**, an object that is valid for the whole cluster, outside any namespace. Your account can only manage objects inside `traineeNN`, so the install fails with *`cannot get resource "clusterroles" ... at the cluster scope`*. Without it, the normal scheduler places the notebook Pods.
- `prePuller` runs a DaemonSet that downloads the notebook image on **every node** in advance, and makes `helm install` wait until all nodes have it. The image cache belongs to the node, not to the namespace: once a node has the image, every trainee's notebook on that node reuses it. Twenty pre-pullers (one per trainee) would download nothing more, but would add twenty Pods per node and slow down every install. Without them, the first notebook started on a node waits for the download; the next ones start at once.

This is a common situation with upstream charts: the defaults are written for a team that owns the whole cluster. Reading the defaults (12.4.1) is how you find what to switch off.

:::{note} A shortcut: no persistent storage for users
`singleuser.storage.type: none` gives each notebook server a **temporary** home directory: when the server stops, its files are lost. It is a shortcut for this short demo: no volume to create at start-up, and nothing to clean up afterwards.

By default (`type: dynamic`), the hub creates one PVC per user, `claim-<username>` (10 GiB by default, `singleuser.storage.capacity`), so that notebooks survive a restart: that is what a real deployment needs. Those PVCs are created by the **hub** when a user starts a server, not by Helm. So `helm uninstall` does **not** delete them: you must remove them by hand (see 12.6.1).
:::

The `hub.config` block sets the login. The *dummy* authenticator accepts any username, but only with the password you choose here. Without a password, anybody on the internet who finds the URL could start a notebook — that is, run code on the cluster.

**Install** — with the chart version pinned, for the same reason as the image tag:

```bash
helm upgrade --install jhub jupyterhub/jupyterhub \
  --version 4.4.4 \
  --values values-jhub.yaml
```

`upgrade --install` is idempotent, like `kubectl apply` and `docker compose up`: run it again with a changed values file and only the difference is applied.

To see which values a release uses:

```bash
helm get values jhub          # only what you set (values-jhub.yaml)
helm get values jhub --all    # your values merged with the chart defaults
```

## 12.5 Observe and validate

List the releases in your namespace:

```bash
helm list
```

List the objects that the release created:

```bash
helm get manifest jhub | kubectl get -f -
```

`helm get manifest` prints the YAML of the release; piped into `kubectl get -f -`, it lists exactly the objects the chart created. They are ordinary Kubernetes objects: Deployments, Services, a PVC, an Ingress, exactly like the ones you wrote by hand. Look at the Ingress, as you did in section 4:

```bash
kubectl describe ingress jupyterhub
```

To see the rendered YAML without installing anything:

```bash
helm template jhub jupyterhub/jupyterhub --version 4.4.4 --values values-jhub.yaml | less
```

Once the certificate is ready, open `https://jhub-traineeNN.training.meditwin-project.eu`, log in with any username and the password from `values-jhub.yaml`, and start a server: that is a Pod being scheduled with the resources from your values file. See it in your namespace:

```bash
kubectl get pods -l component=singleuser-server
```

```text
NAME            READY   STATUS    RESTARTS   AGE
jupyter-alice   1/1     Running   0          40s
```

The Pod is called `jupyter-<username>`; Helm did not create it, the hub did, when you clicked *Start*. Look at its resources:

```bash
kubectl get pod jupyter-<username> -o jsonpath='{.spec.containers[0].resources}'; echo
```

`singleuser.cpu.guarantee` and `singleuser.memory.guarantee` became `requests`; `singleuser.cpu.limit` and `singleuser.memory.limit` became `limits` — the same fields as in section 7.

## 12.6 Upgrade and rollback

Change `singleuser.memory.limit` to `2G` in `values-jhub.yaml`, then apply the new values:

```bash
helm upgrade jhub jupyterhub/jupyterhub --version 4.4.4 --values values-jhub.yaml
```

List the revisions of the release: the install is revision 1, the upgrade revision 2:

```bash
helm history jhub
```

Go back to revision 1:

```bash
helm rollback jhub 1
```

Compare with `kubectl rollout undo` in Hands-on 2: that rolled back one Deployment; `helm rollback` rolls back every object of the release together.

`helm rollback` does not change your files: `values-jhub.yaml` still says `2G`, and the next `helm upgrade` would bring `2G` back. Set it back to `4G` in the file, so that the file and the release agree.

**Checkpoint 10** — JupyterHub is installed from a pinned chart, you logged in over HTTPS, and the release history shows the upgrade and the rollback.

```bash
helm list
helm history jhub
```

```text
NAME   NAMESPACE   REVISION   STATUS     CHART              APP VERSION
jhub   traineeNN   3          deployed   jupyterhub-4.4.4   5.5.4
REVISION   STATUS       CHART              DESCRIPTION
1          superseded   jupyterhub-4.4.4   Install complete
2          superseded   jupyterhub-4.4.4   Upgrade complete
3          deployed     jupyterhub-4.4.4   Rollback to 1
```

(Columns shortened: the real output also shows `UPDATED`.)

:::{dropdown} Not there?
- *`clusterroles ... is forbidden`* (or another resource *at the cluster scope*): the `scheduling` and `prePuller` blocks are missing from `values-jhub.yaml`. Add them and run the same `helm upgrade --install` again.
- Pods stay `Pending` or `ContainerCreating` for a few minutes: the first install pulls large images. Wait, and watch with `kubectl get pods`.
- Login page does not load: check `kubectl get certificate jhub-tls` (TLS) and the host in `values-jhub.yaml`.
- Login refused: the password is the one in `hub.config.DummyAuthenticator.password`. After a change, run `helm upgrade` again.
- Your server does not start (`Pending`): the cluster has no room for the `guarantee` values. Lower `singleuser.memory.guarantee` and upgrade.
:::

### 12.6.1 Remove JupyterHub

Uninstall the release when you have finished, so that its Pods and storage do not keep resources on the shared cluster:

```bash
helm uninstall jhub
kubectl delete pod -l component=singleuser-server --ignore-not-found
kubectl get pvc
```

`helm uninstall` removes every object the chart created, in one command, as `helm install` created them — including the PVC of the hub database (`hub-db-dir`), which is part of the chart: its data is gone with the release. What Helm did not create stays:

- **Notebook Pods** (`jupyter-<username>`): the hub started them, not Helm, so Helm does not know them. That is the `delete pod` line.
- **User PVCs** (`claim-<username>`): with `singleuser.storage.type: none` there are none, and `kubectl get pvc` shows nothing from JupyterHub. If you install with user storage enabled (the default), the hub creates one per user, and they outlive the release, like a named volume after `docker compose down` without `-v`. Delete them by hand, using the names that `kubectl get pvc` shows:

  ```bash
  kubectl delete pvc claim-<username>
  ```

`helm list` must no longer show `jhub`.

## 12.7 ⌨️ Hands-on 11: a chart for your own application

Installing somebody else's chart is half of Helm. The other half is packaging *your* application, and you have everything you need in `~/climate-service-k8s`: five manifests (`configmap.yaml`, `deployment.yaml`, `service.yaml`, `ingress.yaml`, `middleware-redirect-https.yaml`) that differ between deployments only in a handful of values — image tag, replica count, hostname. A chart is exactly that: the manifests, with those values pulled out.

### 12.7.1 What a chart is made of

```text
climate-telemetry/          # the chart directory; its name is the chart name
├── Chart.yaml              # metadata: name, version, description
├── values.yaml             # default values
└── templates/              # your manifests, with {{ placeholders }}
    ├── configmap.yaml
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── middleware-redirect-https.yaml
```

Templates are Go templates: `{{ .Values.replicaCount }}` is replaced by the `replicaCount` key of the values, `{{ .Release.Name }}` by the release name, `{{ .Chart.Version }}` by the chart version. For today, `.Values` is all you need.

:::{note} `helm create`
`helm create mychart` generates a complete starter chart, with helpers, a service account, probes, HPA and tests already templated. It is the right starting point for a new project, but it is forty lines of template to read before you understand one. Here you start from your own manifests instead, so that every line is one you wrote; have a look at `helm create` afterwards.
:::

### 12.7.2 Step 1: start from the manifests

Helm will not take over objects that `kubectl apply` created. You removed them at the end of section 8; run the same command again, to be sure (the files stay):

```bash
kubectl delete -f ~/climate-service-k8s/ --ignore-not-found
```

If it prints nothing, there was nothing left to remove.

If `~/climate-service-k8s` also contains Gateway API manifests, keep those out of this chart exercise. Hands-on 11 packages the five app exposure manifests: `configmap.yaml`, `deployment.yaml`, `service.yaml`, `ingress.yaml`, and `middleware-redirect-https.yaml`.

Then lay out the chart from them:

```bash
cd ~/climate-service-k8s
mkdir -p ~/climate-service-chart/templates
cp configmap.yaml deployment.yaml service.yaml ingress.yaml middleware-redirect-https.yaml ~/climate-service-chart/templates/
cd ~/climate-service-chart
```


`Chart.yaml` is the chart identity card: Helm reads it to know the chart name, version, type, and app version metadata.

Create the file `Chart.yaml`:

```yaml
apiVersion: v2
name: climate-telemetry
description: The MeDiTwin climate telemetry service
type: application
version: 0.1.0        # version of the chart (packaging)
appVersion: "1.0.0"   # version of the application it deploys (the ConfigMap uses it)
```

### 12.7.3 Step 2: pull the values out

`values.yaml` — one key per thing that differs between deployments, with the defaults:

```yaml
replicaCount: 3

image:
  repository: gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN
  tag: "1.0.0"

ingress:
  host: climate-traineeNN.training.meditwin-project.eu
  clusterIssuer: ecmwf-meditwin-training-issuer
  className: traefik
```

Now edit the templates, replacing the hardcoded values with placeholders. Only the lines that change are shown.

`templates/deployment.yaml`:

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  …
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

`templates/configmap.yaml`:

```yaml
data:
  APP_VERSION: {{ .Chart.AppVersion | quote }}
```

`APP_VERSION` comes from `appVersion` in `Chart.yaml`, not from `values.yaml`: the version of the application is written in one place only.

`templates/ingress.yaml`:

```yaml
  annotations:
    cert-manager.io/cluster-issuer: {{ .Values.ingress.clusterIssuer }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: climate-telemetry-tls
  rules:
    - host: {{ .Values.ingress.host }}
```

`templates/service.yaml` and `templates/middleware-redirect-https.yaml` need no change: nothing in them varies.

Follow one value from the file to the cluster:

| Step | Where | What you see |
|---|---|---|
| a | `values.yaml` | `replicaCount: 3` |
| b | `templates/deployment.yaml` | `replicas: {{ .Values.replicaCount }}` |
| c | `helm template` (rendered YAML) | `replicas: 3`, as in your `deployment.yaml` |
| d | `helm upgrade … --set replicaCount=5` | `--set` overrides the default: `replicas: 5` |
| e | The cluster | `kubectl get deployment climate-telemetry` shows `5/5` |

`| quote` is a template *function*: it wraps the value in double quotes, so that `1.0.0` stays a string in the rendered YAML. You will see `quote`, `default` and `toYaml` in almost every chart; the full list is in the [template function reference](https://helm.sh/docs/chart_template_guide/function_list/).

:::{note}
You may also see `"{{ .Chart.AppVersion }}"`. It works for simple values, but `| quote` is preferred because Helm escapes special characters safely and keeps YAML string rendering consistent.
:::

### 12.7.4 Step 3: render, lint, install

Before touching the cluster, look at what the chart produces:

```bash
helm lint .
```

Expected output:

```
==> Linting .
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

Now render the full manifests, then test a quick values override (`replicaCount`) at render time:

```bash
helm template climate-telemetry . | less
```

```bash
helm template climate-telemetry . --set replicaCount=5 | grep replicas
```

Expected output (at least one matching line):

```text
  replicas: 5
```

`helm template` renders the YAML locally — compare it with the original files in `~/climate-service-k8s`: it must be identical, except for the values you overrode. This is the cheapest place to catch a templating mistake. Then install and check:

```bash
helm install climate-telemetry .
```

Expected output:

```text
NAME: climate-telemetry
LAST DEPLOYED: Fri Oct 23 14:27:27 2026
NAMESPACE: traineeNN
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

```bash
helm list
```

Expected output (after first install):

```text
NAME                NAMESPACE   REVISION   UPDATED                                 STATUS     CHART                     APP VERSION
climate-telemetry   traineeNN   1          2026-10-23 14:27:27.459321636 +0000 UTC deployed   climate-telemetry-0.1.0   1.0.0
```

```bash
kubectl get deploy,po,svc,ing
```

Expected result: Deployment, Pods, Service, and Ingress for `climate-telemetry` are listed.

```bash
curl https://climate-traineeNN.training.meditwin-project.eu/health
```

Expected output:

```json
{"status":"healthy","service":"ecmwf-climate-telemetry","version":"1.0.0","node":"climate-telemetry-..."}
```

:::{important} Key point
- Release name: `climate-telemetry`.
- Chart reference: `.` (a local directory; it could also be a `.tgz` package or `repo/name`).
- Same Kubernetes objects as before, but now Helm owns them as one release.
- Ownership check:

  ```bash
  kubectl get deployment climate-telemetry -o yaml | grep -A3 annotations
  ```

  You should see `meta.helm.sh/release-name`.
:::

### 12.7.5 Step 4: upgrade through values, roll back

Change the deployment without editing a template:

```bash
helm upgrade climate-telemetry . --set replicaCount=5
```

Expected output:

```text
Release "climate-telemetry" has been upgraded. Happy Helming!
NAME: climate-telemetry
LAST DEPLOYED: Fri Oct 23 14:35:23 2026
NAMESPACE: traineeNN
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

```bash
kubectl get pods -l app=climate-telemetry
```

Expected output (five replicas running: two new, three from the install):

```text
NAME                                 READY   STATUS    RESTARTS   AGE
climate-telemetry-7bf5ccbdc8-49wkg   1/1     Running   0          9s
climate-telemetry-7bf5ccbdc8-k2xqd   1/1     Running   0          9s
climate-telemetry-7bf5ccbdc8-6v87p   1/1     Running   0          8m5s
climate-telemetry-7bf5ccbdc8-sn5cn   1/1     Running   0          8m5s
climate-telemetry-7bf5ccbdc8-zt8wm   1/1     Running   0          8m5s
```

```bash
helm history climate-telemetry
```

Expected output:

```text
REVISION   UPDATED                  STATUS       CHART                     APP VERSION   DESCRIPTION
1          Fri Oct 23 14:27:27 2026 superseded   climate-telemetry-0.1.0   1.0.0         Install complete
2          Fri Oct 23 14:35:23 2026 deployed     climate-telemetry-0.1.0   1.0.0         Upgrade complete
```

```bash
helm rollback climate-telemetry 1
```

Expected output:

```text
Rollback was a success! Happy Helming!
```

```bash
helm history climate-telemetry
```

Expected output:

```text
REVISION   UPDATED                  STATUS       CHART                     APP VERSION   DESCRIPTION
1          Fri Oct 23 14:27:27 2026 superseded   climate-telemetry-0.1.0   1.0.0         Install complete
2          Fri Oct 23 14:35:23 2026 superseded   climate-telemetry-0.1.0   1.0.0         Upgrade complete
3          Fri Oct 23 14:36:20 2026 deployed     climate-telemetry-0.1.0   1.0.0         Rollback to 1
```

:::{important} Key points
- `helm upgrade ... --set replicaCount=5` created revision `2` and rolled out 5 replicas.
- `helm history` records each revision and its status.
- `helm rollback climate-telemetry 1` did not delete history: it created revision `3`, pointing back to the state of revision `1`.
- `--set` is useful for quick experiments; for real environment differences, use values files (for example `-f values-prod.yaml`) and verify applied overrides with `helm get values climate-telemetry`.
:::


**Checkpoint 11** — `climate-telemetry` runs from **your chart**, answers over HTTPS, and its history shows an upgrade and a rollback.

```bash
helm list
helm history climate-telemetry
kubectl get deployment climate-telemetry -o jsonpath='{.metadata.annotations.meta\.helm\.sh/release-name}'; echo
curl https://climate-traineeNN.training.meditwin-project.eu/health
```

```text
NAME                NAMESPACE   REVISION   STATUS     CHART                     APP VERSION
climate-telemetry   traineeNN   3          deployed   climate-telemetry-0.1.0   1.0.0
REVISION   STATUS       CHART                     DESCRIPTION
1          superseded   climate-telemetry-0.1.0   Install complete
2          superseded   climate-telemetry-0.1.0   Upgrade complete
3          deployed     climate-telemetry-0.1.0   Rollback to 1
climate-telemetry
{"status":"healthy","service":"ecmwf-climate-telemetry","version":"1.0.0","node":"climate-telemetry-…"}
```

(Columns shortened: the real output of `helm list` and `helm history` also shows `UPDATED`.)

:::{dropdown} Not there?
- `helm install` fails with *`invalid ownership metadata`* / *`cannot be imported into the current release`*: an object created with `kubectl apply` still exists. Run the `kubectl delete -f` of Step 1 and install again.
- `helm lint` or `helm template` errors: usually indentation in a template, or a `{{ .Values.x }}` key that is not in `values.yaml` (it renders as empty). Compare `helm template` with the original file in `~/climate-service-k8s`.
- `Could not resolve host` or HTTP `404`: the Ingress host in `values.yaml` is not your `climate-traineeNN` name.
:::

### 12.7.6 Step 5 (optional): package it

Package the chart into one file, named after the `name` and `version` of `Chart.yaml`:

```bash
helm package .
```

```text
Successfully packaged chart and saved it to: /home/…/climate-service-chart/climate-telemetry-0.1.0.tgz
```

Render the templates from the package, as you did from the folder:

```bash
helm template test-release ./climate-telemetry-0.1.0.tgz | less
```

The `.tgz` is the chart as an artifact — what a chart repository (or an OCI registry such as the GitLab one, `helm push`) stores and what `helm repo add` + `helm install jupyterhub/jupyterhub` fetched in section 12.3. Your chart could be published the same way.

### End of day: cleanup

The cluster is shared: at the end of the day, leave your namespace `traineeNN` empty. Keep your **files** (`~/climate-service-k8s`, `~/climate-service-chart`, `~/jhub`) as a reference: on Day 4 you write the same Deployment, Service and Ingress again in a Git repository, and turn them into a chart as you did here.

Keep only one object in the cluster: the TLS Secret `climate-telemetry-tls`. On Day 4 the Ingress uses the same name and reuses the certificate, so no new certificate is requested (see the tip in section 4.7).

**Step 1 — Helm releases.** List them:

```bash
helm list
```

Uninstall each one (`jhub` only if you did not already do it in 12.6.1):

```bash
helm uninstall climate-telemetry --ignore-not-found
helm uninstall jhub --ignore-not-found
```

Remove the notebook Pods, which the hub started and Helm does not know about:

```bash
kubectl delete pod -l component=singleuser-server --ignore-not-found
```

**Step 2 — objects created from files.** Everything you applied from `~/climate-service-k8s` (Hands-on 2–4, the ConfigMap, the Middleware, the HPA, and the Gateway API files if you created them there), and the stack of sections 9–10 if you did not remove it in section 10.3:

```bash
kubectl delete -f ~/climate-service-k8s/ --ignore-not-found
kubectl delete -f ~/climate-stack-k8s/ --ignore-not-found
```

**Step 3 — objects created from the command line**, which no file describes:

Pods started with `kubectl run`, and the OOM demo:

```bash
kubectl delete pod hello loadgen oom-demo --ignore-not-found
```

The HPA, if `hpa.yaml` is not in `~/climate-service-k8s`:

```bash
kubectl delete hpa climate-telemetry-hpa --ignore-not-found
```

The Gateway API objects of section 5, if you created them and their files are not in `~/climate-service-k8s`:

```bash
kubectl delete httproute climate-route --ignore-not-found
kubectl delete gateway climate-gateway --ignore-not-found
kubectl delete certificate climate-gw-tls --ignore-not-found
```

The Secrets: the registry credential (Day 4 creates it again with a deploy token) and the TLS Secrets you no longer need:

```bash
kubectl delete secret gitlab-registry climate-gw-tls jhub-tls --ignore-not-found
```

**Step 4 — storage.** No `delete` command above removes a PVC: delete them explicitly, like `docker compose down -v`:

```bash
kubectl delete pvc --all
```

**Checkpoint 12** — no Helm release, no workload, no storage and no configuration of yours is left in `traineeNN`; only the TLS Secret and the objects Kubernetes creates in every namespace remain.

```bash
helm list
kubectl get all,ingress,configmap,secret,pvc,hpa,certificate
kubectl get middleware.traefik.io
```

```text
NAME   NAMESPACE   REVISION   UPDATED   STATUS   CHART   APP VERSION
NAME                         DATA   AGE
configmap/kube-root-ca.crt   1      2d

NAME                           TYPE                DATA   AGE
secret/climate-telemetry-tls   kubernetes.io/tls   2      8h
No resources found in traineeNN namespace.
```

- `helm list` shows only the header: no release.
- `kubectl get all` shows no Pods, Deployments, Services, ReplicaSets or Jobs (the `all` group does not include Ingresses, ConfigMaps, Secrets or PVCs, which is why the command lists them explicitly).
- `kube-root-ca.crt` is created by Kubernetes in every namespace: do not delete it.
- `climate-telemetry-tls` is kept on purpose.

:::{dropdown} Complete check: every type of object in the namespace
`kubectl get all` covers only a few types. To list **every** type you are allowed to read in your namespace:

```bash
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n1 kubectl get --show-kind --ignore-not-found 2>/dev/null
```

Besides `kube-root-ca.crt` and `climate-telemetry-tls`, you may see objects that you did not create and must not delete: the `default` ServiceAccount, the RoleBindings that give you access to the namespace, and recent Events (they expire after about an hour).
:::

:::{dropdown} Not there?
- `helm list` still shows a release: `helm uninstall <name>`. On Day 4, Fleet creates objects with the same names, and a leftover release blocks it.
- A PVC stays in `Terminating`: a Pod still uses it. `kubectl get pods`, delete that Pod, and the PVC goes away.
- `error: the server doesn't have a resource type "middleware"`: Traefik's CRDs are not visible to you; skip that line.
- `secret/gitlab-registry` is still listed: run the `delete secret` line of Step 3 again. On Day 4 the `kubectl create secret … gitlab-registry` step fails if it already exists.
:::

### 📖 Where to read more

- [Charts](https://helm.sh/docs/topics/charts/) — the structure of a chart, `Chart.yaml` fields, dependencies.
- [Chart template guide](https://helm.sh/docs/chart_template_guide/getting_started/) — templates step by step, from `.Values` to control structures and named templates (`_helpers.tpl`).
- [Values files](https://helm.sh/docs/chart_template_guide/values_files/) and [built-in objects](https://helm.sh/docs/chart_template_guide/builtin_objects/) (`.Release`, `.Chart`, `.Capabilities`).
- [Chart best practices](https://helm.sh/docs/chart_best_practices/) — naming, labels, values layout.
- [Helm commands](https://helm.sh/docs/helm/) — the reference for `helm template`, `lint`, `package`, `push`.

## 12.8 Core takeaway

> **A chart is your manifests with the variable parts pulled out; a release is one installation of it with one set of values.**

- Installing an upstream chart and writing your own are the same mechanism seen from the two sides.
- **Values** override the chart defaults: read the defaults (`helm show values`) before you write yours.
- Pin the **chart version**, as you pin an image tag.
- `helm upgrade --install` is idempotent; `helm rollback` restores **every** object of a release together.
- `helm template` and `helm lint` show the result before anything reaches the cluster.
- `helm uninstall` removes what the chart created, and only that: objects created by the application itself (notebook Pods, user PVCs) stay.

## 12.9 📖 Check yourself

1. What is the difference between a chart, a release and values?
2. You install the JupyterHub chart without `--version`. What can go wrong next month?
3. `helm install` fails with *`clusterroles ... is forbidden`*. Do you ask for more permissions? What else can you do?
4. In your chart, `replicaCount` is 3 in `values.yaml`. You run `helm upgrade climate-telemetry . --set replicaCount=5`. Where does the value 5 live afterwards, and how do you see it?
5. `helm install` of your own chart fails with *`cannot be imported into the current release`*. Why?

:::{dropdown} Solutions
**1.** A **chart** is the package (templates and default values). A **release** is one installation of a chart in a namespace, with a name and a history of revisions. **Values** are the inputs that fill the templates for that release.

**2.** Helm installs the newest version available that day. A new major version can change defaults or keys, and the same command gives a different installation without any change on your side.

**3.** Read the defaults: a component of the chart needs cluster-wide permissions. Switch it off in your values if you do not need it (here: `scheduling.userScheduler`, `prePuller`). Granting cluster-wide rights to every user of a shared cluster is a security risk.

**4.** In the release, not in any file: `helm get values climate-telemetry` shows it. Keep such settings in a values file under version control instead of `--set`.

**5.** An object with the same name already exists and was not created by this release (for example, created earlier with `kubectl apply`). Helm does not take over objects it did not create: delete them first, as in Step 1.
:::

Continue with: [13. Day 3 cheat sheet](13-cheatsheet.md).
