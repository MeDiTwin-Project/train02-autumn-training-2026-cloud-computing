# 8. Fleet with plain manifests or Helm charts

This section stays with tools you already know: the Kubernetes manifests of Hands-on 7 and the Helm
charts of [Day 3, section 12](../day03/12-helm-jupyterhub.md). If you already look after several
services and pipelines, the **advanced path**, [8B. Cookiecutter and Dagger](08b-cookiecutter-and-dagger.md),
can be taken instead. Both end the day.

On Day 3 you installed JupyterHub with `helm upgrade --install` from your terminal. In [section 7](07-fleet.md) you
deployed your own service from plain manifests, through Git. This section brings the two together:
when should Fleet deploy plain manifests, and when a chart?

**Slides:** the slides used in class are in one file, {download}`day04-git-cicd-gitops.pdf`.

## 8.1 Two ways to describe the same deployment

With **plain manifests**, the files in Git are exactly what reaches the cluster. With a **Helm
chart**, Git holds templates plus values, and the objects only exist after rendering.

```{mermaid}
flowchart TB
    subgraph A["Plain manifests"]
        M["deployment.yaml<br/>service.yaml<br/>ingress.yaml"]
    end
    subgraph B["Helm chart"]
        T["templates/"] --> R["helm template"]
        V["values.yaml +<br/>fleet.yaml values"] --> R
    end
    M --> F["Fleet bundle"]
    R --> F
    F --> K[("Objects in the cluster")]
```

Fleet deploys both in the same way. Internally it wraps even plain manifests in a Helm release, which
you can see for yourself:

```bash
helm -n traineeNN list
```

The list has a release named after the GitRepo and the path, `climate-telemetry-traineeNN-manifests`,
and its `REVISION` has gone up with every deployment you merged in [section 7](07-fleet.md).

So the choice between the two is not about what Fleet can do. It is about **what the people changing and
reviewing the deployment read**.

## 8.2 When to choose which

| Question | Plain manifests | Helm chart |
|---|---|---|
| How many copies of this deployment exist? | One | Several: environments, clusters, teams |
| What does a reviewer see in a merge request? | The exact YAML that will run | A values diff; the result needs `helm template` |
| Who wrote it? | You | You, or upstream (JupyterHub, cert-manager, podinfo) |
| Logic needed? | None | Defaults, conditionals, loops |
| How do copies differ? | Overlays (kustomize, [section 7.3](#d4-7-3)) | Values, per target (`targetCustomizations`) |
| Typical failure | Copy-paste drift between copies | Templates and values nobody fully understands |

A rule of thumb that holds up well:

- **Start with plain manifests** for your own service. They are the easiest thing to review.
- **Move to a chart** when the *same* thing is deployed a second time with differences.
- **Use the upstream chart** for third-party software. Never rewrite someone else's chart as
  manifests: you would inherit the job of keeping up with every release.

:::{note} Jargon: *values*
**Values** are the inputs of a chart: a YAML document whose keys the templates read as
`.Values.<key>`. The chart's own `values.yaml` holds the defaults; anything given at install time —
`-f values.yaml` on Day 3, the `helm.values` block of a `fleet.yaml` here — overrides them.
:::

## 8.3 ⌨️ Hands-on 10: turn your manifests into a chart

⏱ **About 15 minutes.**

You convert the three manifests from Hands-on 7 into a small chart, in the same config repository. You
did the same on Day 3, in [Hands-on 11](../day03/12-helm-jupyterhub.md), with `helm install` from your
terminal, and the keys are the same (`replicaCount`, `image.repository`, `image.tag`). The difference
today: the chart lives in Git, and Fleet installs it.

```bash
cd ~/climate-deploy
git switch main && git pull
git switch -c deploy/helm-chart
mkdir -p helm/templates
```

`helm/Chart.yaml`

```yaml
apiVersion: v2
name: climate-telemetry
description: The climate telemetry service
version: 0.1.0
```

`helm/values.yaml` — the defaults, one key per thing that differs between deployments

```yaml
replicaCount: 2

image:
  repository: gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN
  tag: "<short-sha currently deployed>"
  pullSecret: gitlab-registry

region: unknown

ingress:
  host: climate-traineeNN.training.meditwin-project.eu
  clusterIssuer: ecmwf-meditwin-training-issuer
  className: traefik
```

Copy the manifests in as templates, then replace the specifics with values:

```bash
cp manifests/deployment.yaml manifests/service.yaml manifests/ingress.yaml helm/templates/
```

`helm/templates/deployment.yaml` — only the lines that change are shown

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  …
    spec:
      imagePullSecrets:
        - name: {{ .Values.image.pullSecret }}
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          env:
            - name: REGION
              value: {{ .Values.region | quote }}
```

`helm/templates/ingress.yaml` — likewise

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

`service.yaml` needs no change.

Finally, the bundle. Because `Chart.yaml` sits in the same folder, Fleet treats the folder as a chart
and needs no `chart:` field:

`helm/fleet.yaml`

```yaml
defaultNamespace: traineeNN
helm:
  releaseName: climate-telemetry
  values:
    region: bologna            # this deployment's own settings; section 8.4 explains why here
```

Render it locally and compare the result with what is running, **before** committing:

```bash
helm lint ./helm
helm template climate-telemetry ./helm --set region=bologna \
  | kubectl -n traineeNN diff --server-side --force-conflicts -f -
```

`kubectl diff` asks the cluster what *would* change if the rendered objects were applied, without
applying anything. (`--server-side --force-conflicts` lets it compare objects that Fleet, not
`kubectl`, created.) No output means the chart produces exactly what the manifests produced. Any `+` or `-`
line is either a change you meant or a templating mistake, and this is the cheapest moment to find
it.

```bash
git add helm
git commit -m "deploy: climate-telemetry as a local Helm chart"
git push -u origin deploy/helm-chart
```

Merge it. Then switch Fleet over to it. This takes three steps, because of a race you will see once and
should recognise afterwards:

1. In Rancher (**☰ → Continuous Delivery → Resources → Git Repos**), open the **⋮** menu at the end of your
   GitRepo's row → **Edit Config**. In the wizard's step **Repository details**, change **Paths** from
   `manifests` to `helm`,
   and save.
2. Fleet now does two things at once: it removes the old bundle, which uninstalls its release and
   deletes the Deployment, Service and Ingress, and it installs the new bundle. The install normally runs
   while the old objects still exist, so it fails: the new bundle shows *Not Ready* with
   *`Service "climate-telemetry" … exists and cannot be imported into the current release`*.
   Wait until the old objects are gone:

   ```bash
   kubectl -n traineeNN get deploy,svc,ingress      # "No resources found"
   ```

3. In Rancher, press **Force Update** in the same **⋮** menu of the GitRepo's row. Fleet does not retry a
   failed install by itself, however long you wait; this tells it to. Within a minute the bundle is
   *Ready*.

Your service is down between step 2 and step 3, usually a minute or two.

:::{dropdown} The bundle still says *`… cannot be imported into the current release`* after a Force Update
The old objects are still there. `kubectl -n traineeNN get svc climate-telemetry -o
jsonpath='{.metadata.annotations.meta\.helm\.sh/release-name}'` names the release that owns them.
If it is the old one (`…-manifests`), check that **Paths** no longer lists `manifests`: with both
paths listed, two bundles claim the same objects and one of them always fails. **Two bundles must
never own the same object.**
:::

Remove the folder nothing uses any more:

```bash
git switch main && git pull
git switch -c chore/remove-manifests
git rm -r manifests
git commit -m "chore: remove the manifests replaced by the chart"
git push -u origin chore/remove-manifests
```

Open the merge request and merge it.

**Checkpoint 10** — `helm -n traineeNN list` shows a release called `climate-telemetry`, the bundle is
*Ready*, and `/health` still reports `"region": "bologna"`.

## 8.4 Values: the knob you turn

A chart has two places to put a value, and choosing well keeps merge requests small:

- **`values.yaml` in the chart** — defaults that are true for *every* deployment of the chart.
- **`helm.values` in `fleet.yaml`** — what is specific to *this* deployment: its tag, its replica
  count, its settings.

With several clusters, `targetCustomizations` adds a third level: values that apply only to clusters
matching a selector.

`fleet.yaml` — one chart, different settings per cluster (for reading: the training has one cluster)

```yaml
defaultNamespace: traineeNN
helm:
  releaseName: climate-telemetry
  values:
    region: bologna
targetCustomizations:
  - name: production
    clusterSelector:
      matchLabels:
        env: prod
    helm:
      values:
        replicaCount: 3
  - name: staging
    clusterSelector:
      matchLabels:
        env: staging
    helm:
      values:
        replicaCount: 1
```

Fleet uses the **first** entry whose selector matches a cluster, so order them from most to least
specific.

## 8.5 ⌨️ Hands-on 11: change the deployment through values only

⏱ **About 13 minutes.**

Deploy a new image and scale up, without touching the chart.

1. In `~/climate-service`, after `git pull`, find the tag of the latest image on `main`, skipping the
   release commit as in [section 5](05-releases.md):
   `git log -1 --invert-grep --grep='^chore(release)' --format=%h --abbrev=8`. If you have merged
   nothing since Hands-on 8, it is the image already running; change only `replicaCount` then.
2. In `~/climate-deploy`, on a branch, edit only `helm/fleet.yaml`:

   ```yaml
   helm:
     releaseName: climate-telemetry
     values:
       region: bologna
       replicaCount: 3
       image:
         tag: "<new-short-sha>"
   ```

3. Commit, push, open the merge request, and look at the diff. Then merge.

```bash
kubectl -n traineeNN get deployment climate-telemetry -w
helm -n traineeNN history climate-telemetry
```

`helm history` lists each deployment Fleet made as a numbered revision.

Compare this diff with the one in Hands-on 8. There, the merge request showed the very lines of the
Deployment that changed. Here, it shows inputs to a template: a reviewer has to know the chart to
know what `replicaCount` turns into. For one deployment that is a cost with no benefit. For ten
deployments of the same chart it is what keeps them identical.

**Checkpoint 11** — three Pods run, with the new image if there was one, and the merge request changed
`fleet.yaml` alone.

:::{dropdown} The values were ignored
`values:` must sit under `helm:` in `fleet.yaml`. At the top level Fleet does not read it, and does
not complain either. Check indentation first.
:::

## 8.6 ⌨️ Hands-on 12: an upstream chart, deployed by Fleet

⏱ **About 15 minutes. Optional:** do it if you finish early, or after the course.

On Day 3 you installed JupyterHub from its published chart with `helm upgrade --install`. The same
chart could be deployed by Fleet, with the version and values reviewed in Git. JupyterHub is too
heavy to run once per trainee, so use a small, well-known chart instead:
[podinfo](https://github.com/stefanprodan/podinfo), a demo web service.

:::{note} What podinfo is
A tiny web application written in Go, made to demonstrate Kubernetes deployments, and used in many
Kubernetes tutorials. It does nothing useful, but it shows what you need to
see. `curl` on `/` answers with JSON describing the running Pod: its `hostname` (the Pod name) and
the podinfo `version`. A browser gets a small web page with the same data. It also has `/healthz`
and `/readyz` for probes, and `/metrics` for Prometheus.

Its Helm chart is published by the author at `https://stefanprodan.github.io/podinfo`, and the chart
version is the application version: chart `6.7.1` runs podinfo `6.7.1`. So when you upgrade the
chart in Git, the `version` in the JSON changes, and you can see the upgrade from the outside.
:::

Discover what the chart can be configured with, exactly as you would for JupyterHub:

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm search repo podinfo/podinfo --versions | head -5
helm show values podinfo/podinfo --version 6.7.1 | grep -A20 '^ingress:'
```

Add a second folder to the config repository:

`podinfo/fleet.yaml`

```yaml
defaultNamespace: traineeNN
helm:
  repo: https://stefanprodan.github.io/podinfo
  chart: podinfo
  version: 6.7.1               # pinned: an upgrade is a reviewed change, never a surprise
  releaseName: podinfo
  values:
    replicaCount: 1
    ingress:
      enabled: true
      className: traefik
      annotations:
        cert-manager.io/cluster-issuer: ecmwf-meditwin-training-issuer
      hosts:
        - host: podinfo-traineeNN.training.meditwin-project.eu
          paths:
            - path: /
              pathType: ImplementationSpecific
      tls:
        - secretName: podinfo-tls
          hosts:
            - podinfo-traineeNN.training.meditwin-project.eu
```

There is no `Chart.yaml` here and no template: the chart comes from its repository, and the folder
holds only your decisions about it.

Commit it through a merge request, then add `podinfo` as a **second path** in the GitRepo next to
`helm`: **⋮ → Edit Config** on the GitRepo's row in **Resources → Git Repos**, step **Repository details**,
**Add Path**. Each path becomes its own bundle.

```bash
kubectl -n traineeNN get pods -l app.kubernetes.io/name=podinfo
curl -s https://podinfo-traineeNN.training.meditwin-project.eu | jq '{hostname, version}'
```

Now upgrade it the way you would upgrade JupyterHub in production: pick a newer version from the
`helm search` output, change `version:` in a merge request, read the diff, merge. `helm -n traineeNN
history podinfo` shows the upgrade as a new revision, `curl` shows the new `version`, and `git log`
shows who merged it.

**Checkpoint 12** — podinfo answers over HTTPS, and its upgrade was a one-line merge request.

:::{dropdown} Why pin `version`?
Without it, Fleet installs whatever version is newest when it renders the bundle. An upstream release
then reaches your cluster with no merge request, no review and no record, which is exactly what
GitOps was supposed to stop. It is the same reason why Day 3 pinned `--version` in `helm upgrade`.
:::

## 8.7 Core takeaway

> **Fleet deploys manifests and charts the same way. Choose by who reads the change: plain manifests
> show reviewers exactly what will run; a chart trades that for one definition deployed many times.**

- Every Fleet bundle is a Helm release, whether you wrote a chart or not.
- Your own single service: plain manifests. The same thing deployed more than once: a chart.
- Third-party software: the upstream chart, pinned to a version, configured through `helm.values`.
- Chart defaults go in `values.yaml`; deployment-specific settings go in `fleet.yaml`.
- Two bundles must never own the same object.

## 8.8 Check yourself

1. A colleague proposes converting the three manifests of a service deployed once, to one cluster,
   into a chart "to be ready for the future". What do you answer?
2. You deploy the same chart to staging and production. Where does the image tag go, and why not in
   the chart's `values.yaml`?
3. A merge request changes `version: 6.7.1` to `version: 6.9.0` in `podinfo/fleet.yaml`. What should
   the reviewer check before approving?

:::{dropdown} Solutions
**1.** Not yet. A chart adds a layer between the merge request and what runs, and for a single
deployment it buys nothing in exchange. Convert when the second deployment arrives; with the diff
check from Hands-on 10 it takes minutes.

**2.** In each bundle's `fleet.yaml` (or a `targetCustomizations` entry), because the two
environments run *different* tags while one is being promoted. A tag in the chart's `values.yaml`
would be shared by both, so changing it would deploy to both at once.

**3.** The upstream release notes between the two versions, especially renamed or removed values,
which silently stop applying. Then render both versions with the same values
(`helm template … --version`) and compare the output.
:::

---

Continue with: [9. Day 4 cheat sheet](09-cheatsheet.md).
