# 7. Rancher Fleet

Fleet is a GitOps agent. It is bundled with Rancher, so if you have
Rancher you already have Fleet.

**Slides:** the slides used in class are in one file, {download}`day04-git-cicd-gitops.pdf`.

## 7.1 Three objects, and that is all

```{mermaid}
flowchart TB
    G["GitRepo: one repository"] --> B1["Bundle: dev"]
    G --> B2["Bundle: prod"]
    B1 --> BD1["BundleDeployment"]
    B2 --> BD2["BundleDeployment"]
    BD1 --> K1[(Cluster A)]
    BD2 --> K2[(Cluster B)]
```

| Object | Means | You create it… |
|---|---|---|
| **GitRepo** | "Watch this repository and branch." | Once, in Rancher |
| **Bundle** | One deployable unit, discovered from a folder | Automatically, by adding a `fleet.yaml` |
| **BundleDeployment** | That bundle, on one specific cluster | Automatically, from targeting rules |

The rule that makes everything click:

:::{important} Every folder containing a `fleet.yaml` becomes one Bundle
Fleet scans the repository for files named `fleet.yaml`. Each one it finds turns that folder into an
independently deployed, independently reported unit. Adding a folder is how you add a deployment; you
do not register anything.
:::

## 7.2 A repository layout

```text
my-app-gitops/
├── chart/                    # a Helm chart, shared by all environments
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       └── ingress.yaml
├── dev/
│   └── fleet.yaml            # → Bundle "dev"
└── prod/
    └── fleet.yaml            # → Bundle "prod"
```

Two bundles, one chart. `dev` and `prod` deploy the same templates; each `fleet.yaml` only sets the
few values that differ, such as the number of replicas or the host name. The alternative is a full
copy of the manifests in each folder. Then a fix made in `dev/` has to be repeated by hand in `prod/`,
somebody eventually forgets, and the two environments stop running the same application.

(d4-7-3)=
## 7.3 `fleet.yaml`, three flavours

A `fleet.yaml` answers two questions: **what** to deploy from this folder, and **where**.

### Flavour 1: raw manifests

Drop `.yaml` files in the folder and give it a nearly empty `fleet.yaml`:

`apps/hello/fleet.yaml`

```yaml
defaultNamespace: hello
```

Fleet applies every manifest in the folder into the `hello` namespace. Simplest possible thing, and
often enough.

### Flavour 2: a Helm chart

`dev/fleet.yaml`

```yaml
defaultNamespace: my-app-dev

helm:
  # Fleet clones the repo to /workspace/source, so this is a repo-relative path
  chart: /workspace/source/chart
  releaseName: my-app-dev
  values:                      # overrides chart/values.yaml, for this environment only
    app:
      image:
        repository: gitlab.example.com:6443/team/my-app
        tag: abc123def456      # a SHA, never "latest"
      replicaCount: 1
      ingress:
        host: my-app-dev.internal.meditwin-project.eu
```

The chart can also come from outside the repository — a public Helm repo, or an OCI registry:

`fleet.yaml` — deploying JupyterHub from its published chart

```yaml
defaultNamespace: jupyterhub
helm:
  releaseName: jhub
  repo: https://hub.jupyter.org/helm-chart/
  chart: jupyterhub
  version: 4.4.4                     # the version of Day 3
  values:
    proxy:
      service:
        type: ClusterIP
    ingress:
      enabled: true
      annotations:
        cert-manager.io/cluster-issuer: ecmwf-meditwin-training-issuer
      ingressClassName: traefik
      hosts:
        - jupyterhub.training.meditwin-project.eu
      tls:
        - hosts: [jupyterhub.training.meditwin-project.eu]
          secretName: jupyterhub-tls
```

That is the entire deployment of JupyterHub in this training platform. Nine lines of intent and a
values block — no `helm install`, no `kubectl`, nobody's laptop involved.

### Flavour 3: kustomize

If your folder has a `kustomization.yaml`, Fleet runs kustomize:

```text
my-app-gitops/
├── fleet.yaml
├── kustomization.yaml         # → resources: [overlays/prod]
├── base/
│   ├── kustomization.yaml     # → resources: [deployment.yaml, service.yaml, ingress.yaml]
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── overlays/
    └── prod/
        ├── kustomization.yaml # → resources: [../../base] + patches
        └── deployment.yaml    # a patch: just the fields that differ
```

:::{note} Jargon: *kustomize*, *base*, *overlay*
**kustomize** builds Kubernetes YAML by *patching*. A **base** holds the complete manifests; an
**overlay** holds only the differences for one environment. So `base/deployment.yaml` has 40 lines,
and `overlays/prod/deployment.yaml` has the 3 lines that change — no copy-paste, no drift between
near-identical files.

Helm and kustomize solve the same problem differently: Helm renders *templates*
(`{{ .Values.x }}`), kustomize *patches* finished YAML. Both are fine. Helm suits things
you ship to others; kustomize suits your own manifests with a couple of environment
differences.
:::

An overlay patch is just the fields you want to change:

`overlays/prod/kustomization.yaml`

```yaml
resources:
  - ../../base
patches:
  - path: deployment.yaml
    target:
      kind: Deployment
      name: my-app
```

`overlays/prod/deployment.yaml` — only the fields that differ

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
```

Preview exactly what Fleet will apply, before committing:

```bash
kubectl kustomize overlays/prod
```

## 7.4 ⌨️ Hands-on 7: deploy your service with Fleet, from plain manifests

⏱ **About 35 minutes.**

You deploy the image your pipeline built in Hands-on 3, using flavour 1: plain manifests. The
manifests go in a **separate config repository**, for the reasons in
[section 6.5](06-gitops.md#gitops-is-not).

| Value | Yours |
|---|---|
| App project (Hands-on 2) | `climate-telemetry-traineeNN` |
| Config project (already created for you) | `climate-telemetry-traineeNN-deploy` |
| Namespace | `traineeNN` |
| Image | copy it from *Deploy → Container Registry* in the app project, with the 8-character SHA tag |
| Hostname | `climate-traineeNN.training.meditwin-project.eu` — DNS records under `training.meditwin-project.eu` are created automatically from the Ingress |

### Step 1: the config repository

Your config repository already exists in GitLab, empty and private:
`training/autumn-2026/climate-telemetry-traineeNN-deploy`. Then:

```bash
mkdir -p ~/climate-deploy/manifests && cd ~/climate-deploy
git init -b main
```

`manifests/fleet.yaml`

```yaml
defaultNamespace: traineeNN
```

`manifests/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: climate-telemetry
  labels:
    app: climate-telemetry
spec:
  replicas: 2
  selector:
    matchLabels:
      app: climate-telemetry
  template:
    metadata:
      labels:
        app: climate-telemetry
    spec:
      imagePullSecrets:
        - name: gitlab-registry       # created in step 2
      containers:
        - name: app
          # Paste the image path from the registry page, and your SHA from Checkpoint 3.
          image: gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:<short-sha>
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

`manifests/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: climate-telemetry
spec:
  selector:
    app: climate-telemetry
  ports:
    - name: http
      port: 80
      targetPort: http
```

`manifests/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: climate-telemetry
  annotations:
    cert-manager.io/cluster-issuer: ecmwf-meditwin-training-issuer
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - climate-traineeNN.training.meditwin-project.eu
      secretName: climate-telemetry-tls
  rules:
    - host: climate-traineeNN.training.meditwin-project.eu
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: climate-telemetry
                port:
                  number: 80
```

These are the same three objects you wrote by hand on Day 3. The only new file is `fleet.yaml`, and
it says one thing: put everything in `traineeNN`.

```bash
git add manifests
git commit -m "feat: deploy climate-telemetry from plain manifests"
git remote add origin git@gitlab.data.meditwin-project.eu:training/autumn-2026/climate-telemetry-traineeNN-deploy.git
git push -u origin main
```

:::{warning} Do not add a `Namespace` manifest to the folder
Fleet installs each bundle as a Helm release, and Helm refuses to take over an object it did not
create. If `traineeNN` already exists and the bundle also contains a `Namespace` called `traineeNN`,
the bundle fails with *`Namespace "traineeNN" … cannot be imported into the current release`*.
Your namespace already exists, created for you with your accounts, and `defaultNamespace` is all
Fleet needs to use it.
:::

### Step 2: let the cluster pull a private image

The registry is private, so the cluster needs a credential to pull from it. In the **app** project,
create a deploy token: *Settings → Repository → Deploy tokens → Add token*, name `cluster-pull`, scope
**`read_registry`** only. GitLab shows the token once; keep the page open.

```bash
read -rp  "Deploy token username: " DT_USER
read -rsp "Deploy token: " DT_TOKEN; echo
kubectl -n traineeNN create secret docker-registry gitlab-registry \
  --docker-server=gitlab.data.meditwin-project.eu:6443 \
  --docker-username="$DT_USER" \
  --docker-password="$DT_TOKEN"
unset DT_TOKEN
```

`read -s` keeps the token out of your shell history and off the screen.

:::{note} Why this one step is not in Git
It is a credential, and [section 4.1](04-pipeline-security.md#secrets-rule) applies: credentials never go in a repository, private or not.
Creating it with `kubectl` once is the honest workaround for a training exercise. On a real platform
the External Secrets Operator creates it from Vault, as shown in [section 7.7](#secrets), and nothing
is typed by hand.
:::

### Step 3: register the config repository with Fleet

Fleet needs read access to the config repository too. In the **config** project, create another
deploy token, scope **`read_repository`** only.

Log in to Rancher at `https://rancher-training.apps.meditwin-project.eu` as `traineeNN`, with your
claim code as the password (**Log in with Local User**). This is the first time you use Rancher
today, so here is where things are:

:::{figure} img/ui-r-home.png
:alt: The Rancher home page with the menu button, the cluster tile, the cluster link and the Continuous Delivery icon marked.
**The home page.** On the left is a narrow bar of icons. ① **☰** opens it in full, with names.
② The **EMG** tile, and ③ the link **ecmwf-meditwin-training** in the Clusters table: both open the
training cluster (Pods, Deployments, Services…). ④ The sailboat icon: **Continuous Delivery**, which
is Fleet.
:::

:::{figure} img/ui-r-menu.png
:alt: The Rancher main menu opened in full, with Continuous Delivery marked under Global Apps.
**☰ opened.** ① The button that opens and closes the menu. ② **Continuous Delivery**, under
**Global Apps**.
:::

:::{figure} img/ui-r-cd.png
:alt: The Rancher Continuous Delivery page, Resources then Git Repos open, with the workspace drop-down and Create Git Repo marked.
**Continuous Delivery → Resources → Git Repos.** ① The **workspace** drop-down, at the top right: it
must show `fleet-traineeNN`. ② **Resources** in the left menu opens a list of Fleet's objects;
③ **Git Repos** is yours. ④ **Create Git Repo**, while the list is empty (**Add Repository**, top
right, does the same).
:::

On a list, the **⋮** at the end of a row opens the actions on that object, such as **Edit Config**
and **Force Update**.

Now create the GitRepo:

1. Open **Continuous Delivery**: the sailboat icon in the bar on the left, or **☰ → Continuous
   Delivery**.
2. Check the **workspace** drop-down at the top right: it must show **fleet-traineeNN**, a Fleet
   workspace of your own, which nobody else sees.
3. In the left menu, open **Resources** and choose **Git Repos**. Press **Create Git Repo** (or
   **Add Repository**, top right). A form opens with four steps; **Next** (bottom right) moves to the
   next one. Rancher titles the page *App Bundle: Create*: since Rancher 2.13, *App Bundle* is its
   common name for a GitRepo and for a Helm-based deployment.

**Step 1, Metadata.** **Name**: `climate-telemetry-traineeNN`.

:::{figure} img/ui-r-wizard-1.png
:alt: Step 1 of the Create Git Repo form, with the Name field and Next marked.
**Step 1.** ① The name. ② **Next**.
:::

**Step 2, Repository details.**

- **Repository URL**: the **HTTPS** clone URL of the config project. In GitLab, open the config
  project, press the **Code** button and copy **Clone with HTTPS**.
- **Branch Name**: it starts as `master`. **Replace it with `main`**, or Fleet finds no branch.
- **Add Path**, then type `manifests` in **Root directory**.

:::{figure} img/ui-r-wizard-2.png
:alt: Step 2 of the form with the repository URL, the branch main, Add Path and the path marked.
**Step 2.** ① Repository URL. ② Branch Name: `main`. ③ **Add Path**. ④ The path: `manifests`.
:::

**Step 3, Target details.** Leave **All Clusters in the workspace** and press **Next**. The box
*Selected clusters* says *No clusters in the workspace*: that is expected. Your workspace holds no
clusters of its own, but it is allowed to deploy to the training cluster, and "all clusters" includes
it.

:::{figure} img/ui-r-wizard-3.png
:alt: Step 3 of the form, Deploy To, with All Clusters in the workspace selected.
**Step 3.** ① Leave it as it is. ② **Next**.
:::

**Step 4, Advanced.** Open **Git Authentication** and choose **Create an HTTP Basic Auth Secret**;
type the **username** and the **token** of the config project's deploy token.

:::{figure} img/ui-r-wizard-auth.png
:alt: The Git Authentication drop-down open, with Create an HTTP Basic Auth Secret marked.
**Git Authentication.** ① Create an HTTP Basic Auth Secret.
:::

:::{figure} img/ui-r-wizard-4.png
:alt: Step 4 of the form with Git Authentication and the username and password fields marked.
**Step 4.** ① Git Authentication. ② The deploy token's username and token.
:::

Then scroll down to **Resource Handling**, tick **Enable self-healing**, and press **Create**.
Self-healing is Fleet's **drift correction** (`correctDrift`), which Hands-on 9 depends on: if
somebody changes a deployed object by hand, Fleet puts it back as Git says.

:::{figure} img/ui-r-wizard-heal.png
:alt: Step 4 scrolled down, with Enable self-healing ticked and the Create button marked.
**Further down in step 4.** ① **Enable self-healing**. ② **Create**.
:::

:::{warning} Press Create in the form, not after Edit as YAML
**Edit as YAML** switches the form to its text, and the deploy token typed in step 4 does not reach
it: the GitRepo is created without credentials and fails with *Authentication required: HTTP Basic:
Access denied*. If that happened, delete the GitRepo (**⋮ → Delete**) and create it again with the
form.
:::

The result is an object like this, which you could equally keep in Git and apply:

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: climate-telemetry-traineeNN
  namespace: fleet-traineeNN
spec:
  repo: https://gitlab.data.meditwin-project.eu/training/autumn-2026/climate-telemetry-traineeNN-deploy.git
  branch: main
  paths:
    - manifests
  clientSecretName: auth-xxxxx    # the secret Rancher created from the deploy token
  correctDrift:
    enabled: true                 # Enable self-healing
  targets:                        # All Clusters in the workspace
    - clusterSelector:
        matchExpressions:
          - key: provider.cattle.io
            operator: NotIn
            values: [harvester]
```

:::{note} What your GitRepo may deploy
Every GitRepo in your workspace is deployed by a service account of your own, which is an
administrator of your namespace `traineeNN` and of nothing else. Deployments, Services, Ingresses,
Secrets and Helm releases in `traineeNN` work as usual. Another trainee's namespace, a new
`Namespace`, a `ClusterRole`, a CRD: all refused, and the bundle shows the error. Without that
restriction a GitRepo would deploy with cluster-admin rights, and anybody allowed to create one could
take over the cluster.
:::

### Step 4: check it

Within a minute, the new row in **Git Repos** becomes **Active**, and **Clusters Ready** shows `1/1`.
Click its name. The **Resources** box says how many objects Fleet created and how many are ready;
press the arrow next to *Resources ready* to see them counted by state.

:::{figure} img/rancher-bundle-detail.png
:alt: The GitRepo page with the Resources box open, listing Ready, Not Ready, Modified and the other states with their counts.
**The GitRepo's objects, by state.** ① The arrow opens the list. ② **Ready**: everything Fleet
deployed and found healthy, here the Deployment, the Service and the Ingress. ③ **Modified**: objects that somebody changed in the cluster, so they no
longer match Git (Hands-on 9). The **Resources** tab below the boxes stays empty for a trainee
account: listing the objects one by one needs rights on Fleet's own namespaces.
:::

```bash
kubectl -n traineeNN get deploy,pods,svc,ingress,certificate
curl -s https://climate-traineeNN.training.meditwin-project.eu/health
```

```json
{"status":"healthy","service":"ecmwf-climate-telemetry","version":"1.0.0","node":"climate-telemetry-6d9f…"}
```

Run the `curl` a few times. `node` changes between the two Pods: the Service is load-balancing.

**Checkpoint 7** — the GitRepo is *Active* in Rancher and `/health` answers over HTTPS. You did not
run `kubectl apply`.

:::{dropdown} Pods stuck in `ImagePullBackOff`
`kubectl -n traineeNN describe pod <pod>` shows the pull error. Usually: the image path or tag does
not match the registry page exactly; the secret is not called `gitlab-registry`; the
`--docker-server` does not match the host in the image path; or the deploy token lacks
`read_registry`.
:::

:::{dropdown} The GitRepo says `forbidden` … `in the namespace "default"`
Fleet did not read your `fleet.yaml`, so it had no `defaultNamespace` and used `default`, where you
have no rights. The file must be called exactly `fleet.yaml` (not `fleet.yml`, `Fleet.yaml` or
`ffleet.yaml`), inside `manifests/`:

```bash
cd ~/climate-deploy && ls manifests/
git mv manifests/<wrong-name> manifests/fleet.yaml
git commit -m "fix: name fleet.yaml correctly" && git push
```
:::

:::{dropdown} The GitRepo shows an authentication error
The repository URL must be the HTTPS one, and the credential must be the config project's deploy
token with `read_repository`. A token from the app project cannot read the config project. If you
pressed **Create** after **Edit as YAML**, the token never reached the GitRepo: delete it and create
it again from the form.
:::

:::{dropdown} The certificate stays not ready, or DNS does not resolve
Both are asynchronous, exactly as on Day 3: allow a couple of minutes, then
`kubectl -n traineeNN describe certificate climate-telemetry-tls`. Meanwhile, bypass DNS:
```bash
kubectl -n traineeNN port-forward svc/climate-telemetry 8080:80
curl -s localhost:8080/health
```
:::

## 7.5 Ordering with `dependsOn`

Bundles deploy in parallel by default. When one requires another first, such as a Secret before the
Deployment that mounts it — say so:

`dev/fleet.yaml`

```yaml
defaultNamespace: my-app-dev

dependsOn:
  - name: my-app-dev-secrets

helm:
  chart: /workspace/source/chart
  releaseName: my-app-dev
```

And the bundle being depended on names itself:

`secrets/dev/fleet.yaml`

```yaml
defaultNamespace: my-app-dev
name: my-app-dev-secrets
```

Now the app bundle waits until the secrets bundle reports Ready. Without this you get a race: the pod
starts, cannot find its Secret, and sits in `CreateContainerConfigError` until something retries.

## 7.6 Targeting clusters

This is Fleet's strongest feature. One `GitRepo` can deploy to many clusters, chosen by **label**:

`gitrepo.yaml` — applied once by an administrator

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app-gitops
  namespace: fleet-default
spec:
  repo: https://gitlab.example.com/team/my-app-gitops.git
  branch: main
  paths:                       # omit to auto-discover every fleet.yaml
    - dev
    - secrets/dev
  clientSecretName: git-credentials   # for a private repo
  targets:
    - clusterSelector:
        matchLabels:
          environment: production
```

Add a fifth production cluster, label it `environment: production`, and it gets everything — with no
change to the repository at all. Hold on to this: selecting clusters by label, rather than by where a
file sits, is what makes Fleet's fan-out cheap.

Your GitRepo from Hands-on 7 uses a selector too. "All Clusters in the workspace" is written as a
`clusterSelector` that matches every cluster except Harvester ones (the YAML at the end of 7.4), so it
reaches the training cluster without naming it.

(secrets)=
## 7.7 Secrets: the External Secrets Operator

We established on [section 4](04-pipeline-security.md) that a Kubernetes `Secret` in Git is a plaintext
secret in Git. So how does a GitOps repository — where *everything* is in Git — handle secrets?

**You commit a pointer, not a value.**

```{mermaid}
flowchart TB
    Git[("Git: an ExternalSecret")] -->|Fleet applies it| ESO[External Secrets Operator]
    Vault[("Vault: the values")] -->|read by| ESO
    ESO -->|creates| S["Secret, in the cluster only"]
    S --> Pod[Your Pod]
```

The committed file names *where* the secret lives, never what it is:

`secrets/dev/my-app-secrets.yaml` — safe to commit

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-applications      # provided by the platform team
    kind: ClusterSecretStore
  target:
    name: my-app-secret           # the Kubernetes Secret that will be created
    creationPolicy: Owner
  dataFrom:
    - extract:
        key: applications/my-app  # the path in Vault
```

The MediTwin cluster already has this wired up — `ClusterSecretStore/vault-applications`, backed by
Vault at `https://vault.apps.meditwin-project.eu/`. You reference it; you do not build it.

Your Deployment then consumes the Secret normally, and the value existed only in Vault and in the
cluster's memory:

```yaml
env:
  - name: DATABASE_URI
    valueFrom:
      secretKeyRef:
        name: my-app-secret
        key: DATABASE_URI
```

Same trick for pulling images from a private registry:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: registry-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-applications
    kind: ClusterSecretStore
  target:
    name: registry-credentials
    creationPolicy: Owner
    template:
      type: kubernetes.io/dockerconfigjson
  data:
    - secretKey: .dockerconfigjson
      remoteRef:
        key: applications/my-app/registry
        property: .dockerconfigjson
```

```yaml
# ...then in the Deployment
      imagePullSecrets:
        - name: registry-credentials
```

Public images need none of this — leave `imagePullSecrets` out entirely.

## 7.8 When Fleet fights the cluster: `diff.comparePatches`

Sometimes a controller legitimately edits the resources you deployed — adding a generated field,
filling in a default. Fleet sees Git and the cluster disagree, re-applies, the controller edits again,
and the bundle flaps between `Ready` and `Modified` forever.

Tell Fleet to ignore those specific fields:

`fleet.yaml` — ignoring a field owned by a controller

```yaml
defaultNamespace: gpu-operator

diff:
  comparePatches:
    - apiVersion: nvidia.com/v1
      kind: ClusterPolicy
      name: cluster-policy
      operations:
        - {"op": "remove", "path": "/spec/driver/licensingConfig/secretName"}

helm:
  releaseName: gpu-operator
  repo: https://helm.ngc.nvidia.com/nvidia
  chart: gpu-operator
  version: 25.10.0
```

Reach for this only when you have identified the exact field a controller owns. Broad ignore rules
silently switch off drift detection, which is the feature you came for.

## 7.9 The daily workflow

Once the `GitRepo` exists, deploying is entirely Git:

```bash
git switch -c bump-image
sed -i "s|tag: .*|tag: def456abc789|" dev/fleet.yaml
git commit -am "chore: deploy def456a to dev"
git push
# open the merge request, get it reviewed, merge
```

Fleet notices within a minute (or instantly with a webhook) and reconciles. Watch it in Rancher:
**☰ → Continuous Delivery → Resources → Git Repos**, then click the GitRepo.

:::{figure} img/rancher-gitrepo-list.png
:alt: The Rancher Git Repos list with one GitRepo, Active, and the commit it applied.
**The list of GitRepos.** Each row is one `GitRepo`. ① Its state. ② Under the repository URL,
`main @ <commit>`: the exact commit currently applied to the cluster. When you merge, that commit
changes and the row goes briefly out of **Active** while Fleet catches up. ③ The **⋮** menu: **Edit
Config**, **Force Update**, **Delete**.
:::

Clicking a repository shows what it produced:

:::{figure} img/rancher-gitrepo-detail.png
:alt: The GitRepo page with its state, the Bundles tab and its one bundle marked.
**What the GitRepo produced.** ① Its state. ② The **Bundles** tab: ③ one bundle, because the GitRepo
has one path. A GitRepo with two paths shows two bundles, reported separately. **Deployments** counts
the clusters the bundle reached. This is the `GitRepo → Bundle → BundleDeployment` chain from the table
in 7.1, as Rancher shows it.
:::

The deployed objects themselves, from the terminal, with your own kubeconfig:

```bash
kubectl -n my-app-dev get pods,deploy,svc,ingress
```

Rollback:

```bash
git revert HEAD && git push
```

## 7.10 ⌨️ Hands-on 8: deploy by merging

⏱ **About 20 minutes**, including one pipeline run.

This is the main point of the day: change what runs in production **without touching the cluster**.
It takes two merge requests, one per repository.

### The code change, in the app repository

Make the service report which site it runs in, read from an environment variable:

```bash
cd ~/climate-service
git switch main && git pull
git switch -c feat/report-region
```

In `app.py`, add one line to the `payload` dictionary, after the `"node"` line. That line has no
comma at its end yet: add one, or Python stops with `SyntaxError`:

```python
        "node": socket.gethostname(),
        "region": os.getenv("REGION", "unknown"),
```

and one line at the end of `test_health_reports_healthy` in `tests/test_health.py`:

```python
    assert "region" in body
```

```bash
git commit -am "feat: report the deployment region on /health"
git push -u origin feat/report-region
```

Open the merge request, wait for the green pipeline, merge. When the pipeline on `main` has finished,
get the tag of the new image. Your merge also produced a release (`feat:` → v1.1.0), so the newest
commit is the release commit, which has no image of its own; use the commit before it, as in
[section 5](05-releases.md):

```bash
git switch main && git pull
git log -1 --invert-grep --grep='^chore(release)' --format=%h --abbrev=8
```

Nothing has changed in the cluster yet, and that is correct. A new image in the registry is not a
deployment.

### The deployment, in the config repository

```bash
cd ~/climate-deploy
git switch -c deploy/report-region
```

In `manifests/deployment.yaml`, change the image tag to the new SHA, and give the container its
environment variable:

```yaml
        - name: app
          image: gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:<new-short-sha>
          env:
            - name: REGION
              value: bologna
```

```bash
git commit -am "deploy: climate-telemetry <new-short-sha> with REGION=bologna"
git push -u origin deploy/report-region
```

Open the merge request and **read the diff**. Two lines: a new tag and a new setting. That diff is
the whole deployment: it is reviewable, it records who made it and when, and it can be reverted.

Merge it, then watch the rollout:

```bash
kubectl -n traineeNN rollout status deployment/climate-telemetry
curl -s https://climate-traineeNN.training.meditwin-project.eu/health
```

```json
{"status":"healthy","service":"ecmwf-climate-telemetry","version":"1.0.0","node":"climate-telemetry-7c4b…","region":"bologna"}
```

In Rancher (**Resources → Git Repos**), the GitRepo went briefly out of **Active** while Fleet applied the
new commit, and is **Active** again.

**Checkpoint 8** — `/health` reports `"region": "bologna"`, and you changed production with two
merges and no `kubectl apply`.

## 7.11 ⌨️ Hands-on 9: reconciliation and rollback

⏱ **About 12 minutes.**

### Change the cluster by hand, and watch Fleet undo it

Do it in the Rancher web UI, so that you see both sides at once. Open two browser tabs on Rancher.

**Tab 1, the deployment.**
1. Open the training cluster: the **EMG** tile in the bar on the left (or the
   **ecmwf-meditwin-training** link on the home page).
2. In the **top bar**, open the namespace drop-down (it says *Only User Namespaces*) and choose
   **traineeNN**.
3. In the **left menu**: **Workloads → Deployments**, then click **climate-telemetry**. At the top
   right is **Scale**, with **−** and **+**; below it, the list of its **Pods**: two, *Running*.

:::{figure} img/ui-r-explore.png
:alt: The Rancher cluster explorer with the cluster tile, the namespace drop-down and Workloads, Deployments marked.
**The training cluster.** ① The **EMG** tile opens it. ② The **namespace** drop-down at the top:
choose `traineeNN` (it starts at *Only User Namespaces*). ③ **Workloads → Deployments** in the left
menu.
:::

**Tab 2, Fleet.** **☰ → Continuous Delivery → Resources → Git Repos**, click
**climate-telemetry-traineeNN**, and open the list under *Resources ready* (as in Hands-on 7, step 4).

Now change the cluster by hand: in tab 1, press **+** three times, from 2 to 5 replicas. Then watch:

- **Tab 1**: the count jumps to **5** and three new Pods appear.
- Within seconds, Fleet puts it back: the count returns to **2** and the three extra Pods
  terminate. On the training cluster this took about 6 seconds.
- **Tab 2**: Fleet is usually faster than the page, so you may never see **Modified** go to 1;
  after the correction every object is **Ready** again. Tab 1 going back to 2 is the proof.

The same from the terminal, if you prefer:

```bash
kubectl -n traineeNN scale deployment climate-telemetry --replicas=5
kubectl -n traineeNN get deployment climate-telemetry -w      # 5 wanted, back to 2 within seconds
```

A manual change to a resource managed by GitOps is not forbidden. It just does not last. If you want
five replicas, change `replicas:` in Git.

:::{dropdown} The deployment stays at 5
Self-healing is off. In **Resources → Git Repos**, open the **⋮** menu at the end of the GitRepo's row →
**Edit Config** → step **Advanced** → tick **Enable self-healing** → **Save**. Without it, Fleet
reports the drift as *Modified* and leaves it alone. **Force Update**, in the same **⋮** menu,
redeploys from Git at once.
:::

:::{dropdown} There is no **EMG** tile
Your account sees the cluster only through your project. If it is not there, use the `kubectl`
commands above for tab 1, and tell the instructor.
:::

### Deploy something broken, then roll back with `git revert`

```bash
cd ~/climate-deploy
git switch main && git pull
git switch -c deploy/bad-tag
sed -i 's|\(climate-telemetry-traineeNN:\).*|\1this-tag-does-not-exist|' manifests/deployment.yaml
git commit -am "deploy: a tag that does not exist"
git push -u origin deploy/bad-tag
```

Merge it, then look at the Pods in tab 1, or:

```bash
kubectl -n traineeNN get pods
```

```text
NAME                                 READY   STATUS             RESTARTS
climate-telemetry-5b7d9…-x2k9p       0/1     ImagePullBackOff   0
climate-telemetry-7c4b8…-qw8rt       1/1     Running            0
climate-telemetry-7c4b8…-zt6mn       1/1     Running            0
```

:::{tip} Notice what did *not* happen
The old Pods are **still running and still serving**. A rolling update does not remove a healthy Pod
until its replacement passes the `readinessProbe`, and the replacement never started. The bad deploy
did not cause an outage. `curl` the service to confirm.
:::

Roll back, the GitOps way: a new commit that undoes the bad one.

```bash
git switch main && git pull
git revert -m 1 HEAD --no-edit
git push
```

`HEAD` is the merge commit GitLab created for the bad merge request. A merge commit has two parents,
and `-m 1` says "undo it relative to the first one", which is `main` as it was before the merge. (If
your project merges by fast-forward instead, there is no merge commit: drop `-m 1`.)

GitLab's **Revert** button on the merged merge request does the same thing through a new merge
request, which is the better habit on a team.

```bash
kubectl -n traineeNN get pods -w
```

The `ImagePullBackOff` Pod disappears and the service carries on as before.

**Checkpoint 9** — a manual scale was reverted without anybody acting, and a broken deployment was
rolled back by a commit that is now part of the history.

## 7.12 Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| Bundle stuck `Modified` / `NotReady` | Rendered manifests differ from the cluster, or apply failed | Open the bundle in Rancher; read the per-resource message |
| `ImagePullBackOff` | Wrong image path/tag, or missing pull secret | `kubectl describe pod`; verify `gitlab-registry` exists in your namespace |
| `CreateContainerConfigError` | A referenced Secret does not exist yet | Did the ExternalSecret sync? Does the app bundle `dependsOn` the secrets bundle? |
| Ingress has no address | Ingress controller / external-dns issue | `kubectl describe ingress` |
| TLS certificate never Ready | Wrong `cluster-issuer`, or the challenge is failing | `kubectl describe certificate`, then `certificaterequest` |
| Fleet ignores your change | Wrong branch, or `paths:` excludes your folder | Check `branch` and `paths` on the GitRepo |
| `forbidden` … `in the namespace "default"` | `fleet.yaml` misnamed (`fleet.yml`, `ffleet.yaml`…), so no `defaultNamespace` | Rename it to exactly `fleet.yaml` in the folder |
| Values ignored | Your `values:` keys do not match the chart's | Compare against the chart's `values.yaml` — a typo'd key is silently ignored |
| `Forbidden` from kubectl | Your Rancher role lacks access to that namespace | Ask the platform team |

The last-but-one row is the sneakiest: Helm does not error on unknown values. `replicasCount: 3` is
accepted, ignored, and you get one replica. Render the chart locally to be sure:

```bash
helm template ./chart -f my-values.yaml | less
```

## 7.13 Core takeaway

> **With Fleet, deploying is merging: a GitRepo points at a folder, and the cluster follows whatever
> that folder says.**

- **GitRepo → Bundle → BundleDeployment.** You write `fleet.yaml` files; Fleet creates the rest.
- **Every folder with a `fleet.yaml` is a Bundle.** That is how you add a deployment.
- `fleet.yaml` deploys **raw manifests**, a **Helm chart**, or a **kustomize** tree.
- **`dependsOn`** orders bundles — use it so secrets exist before the pods that need them.
- **`targets.clusterSelector`** deploys one repository to many clusters by label. This is Fleet's
  most distinctive capability.
- **Never commit a Secret.** Commit an `ExternalSecret` pointing at Vault, via the existing
  `vault-applications` store.
- **`diff.comparePatches`** stops flapping when a controller owns a field — use it narrowly.

## 7.14 Check yourself

1. You add a folder `staging/` with a `fleet.yaml` and push to `main`. Nothing deploys. Give two
   plausible causes.
2. Why put an `ExternalSecret` in its own bundle with `dependsOn`, rather than in the same folder as
   the app?
3. You need one bundle on all 5 clusters, and a second bundle on only the 2 GPU clusters. Sketch it.

:::{dropdown} Solutions
**1.** Most likely:
- **The GitRepo has an explicit `paths:` list** that names `dev` and `prod` but not `staging`. With
  `paths:` set, Fleet stops auto-discovering — new folders are invisible until listed. (Remove `paths:`
  entirely to auto-discover every `fleet.yaml`.)
- **The GitRepo tracks a different branch.** Check `spec.branch`; pushing to `main` does nothing if
  Fleet watches `master` or a release branch.

Also possible: the `fleet.yaml` is malformed (Fleet reports a bundle error rather than silently
skipping — look at the GitRepo status), or the folder's targeting excludes every cluster.

**2.** Because bundles are the unit of ordering *and* of reporting.
- **Ordering:** `dependsOn` works bundle-to-bundle. In one folder, everything applies together, so the
  Deployment can start before the ExternalSecret has fetched from Vault — pod in
  `CreateContainerConfigError` until a retry rescues it. Two bundles make the ordering explicit.
- **Reporting:** a separate bundle turns "Vault is unreachable" into a clearly failing *secrets*
  bundle, instead of a confusing failure inside the app bundle.
- **Lifecycle:** secrets change on a different rhythm from application manifests, often with different
  reviewers.

**3.** Label the clusters (`gpu: "true"` on the two), then either use two `GitRepo` objects, or one
with `targetCustomizations`. Two GitRepos is clearer:

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: platform-everywhere
  namespace: fleet-default
spec:
  repo: https://gitlab.example.com/team/gitops.git
  branch: main
  paths: [platform]
  targets:
    - clusterSelector: {}                 # empty selector = every cluster in the workspace
---
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: gpu-only
  namespace: fleet-default
spec:
  repo: https://gitlab.example.com/team/gitops.git
  branch: main
  paths: [gpu-workloads]
  targets:
    - clusterSelector:
        matchLabels:
          gpu: "true"
```

The thing to notice: **the repository does not know how many clusters exist.** Adding a sixth cluster
is one label, zero commits.
:::

---

Continue with one of the two paths:

- [8. Fleet with plain manifests or Helm charts](08-fleet-manifests-vs-helm.md), or
- [8B. Cookiecutter and Dagger](08b-cookiecutter-and-dagger.md), the advanced path.
