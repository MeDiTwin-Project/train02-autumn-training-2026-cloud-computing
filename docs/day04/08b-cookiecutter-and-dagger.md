# 8B. Templates and pipelines as code: Cookiecutter and Dagger (advanced path)

By now your service has a `Dockerfile`, a `.gitlab-ci.yml`, Kubernetes manifests and a `fleet.yaml`.
Two kinds of repetition are left, and this section removes one of each:

- **Files that every new service repeats.** The next service needs almost the same files with a
  different name in twenty places. **Cookiecutter** turns a working project into a template.
- **A pipeline you can only test by pushing.** Every change to `.gitlab-ci.yml` so far was checked by
  committing and waiting for a runner. **Dagger** makes pipeline steps a program you run locally.

:::{note} Advanced path
This section is the **advanced path**, for people who already look after several services and
pipelines. It is taken *instead of* [8. Fleet with plain manifests or Helm charts](08-fleet-manifests-vs-helm.md),
which stays with Kubernetes and Helm as you met them on Day 3. Take one.
:::

**Slides:** the slides used in class are in one file, {download}`day04-git-cicd-gitops.pdf`.

## 8B.1 Cookiecutter: what it is

:::{note} Jargon: *Cookiecutter*
**Cookiecutter** is a command-line tool that generates a directory of files from a template. You give
it a folder full of files containing placeholders, plus answers to some questions, and it produces a
real project with the placeholders filled in.

It is written in Python and knows nothing about Kubernetes, Helm or CI. It copies files and substitutes
variables. That is all it does. The capability comes from what it is pointed at.
:::

```bash
cookiecutter --version     # preinstalled on the training VM, in the `course` environment
# elsewhere: pip install cookiecutter pyyaml
```

## 8B.2 The smallest possible template

Two things make a template: a `cookiecutter.json` listing the questions, and **a directory whose name
is a placeholder**.

```text
mini-template/
├── cookiecutter.json
└── {{ cookiecutter.project_name }}/
    └── README.md
```

`mini-template/cookiecutter.json`

```json
{
  "project_name": "my-project",
  "author": "Anonymous"
}
```

`mini-template/{{ cookiecutter.project_name }}/README.md`

```markdown
# {{ cookiecutter.project_name }}

Written by {{ cookiecutter.author }}.
```

Run it:

```bash
cookiecutter ./mini-template
```

```text
  [1/2] project_name (my-project): weather-api
  [2/2] author (Anonymous): Federico
```

You now have:

```text
weather-api/
└── README.md
```

```markdown
# weather-api

Written by Federico.
```

Three things happened, and each is worth naming:

1. **The values in `cookiecutter.json` are the defaults**, shown in brackets. Press Enter to accept.
2. **The directory name was a placeholder too**, so the output folder is named from your answer.
3. **`{{ cookiecutter.x }}` was substituted everywhere** — in file *contents* and in file *names*.

:::{note} Jargon: *Jinja2*
The `{{ ... }}` syntax is **Jinja2**, Python's standard templating language. It does more than
substitution — `{% if %}`, `{% for %}`, and filters like `{{ name | lower }}` all work. Cookiecutter
runs your template files through Jinja2.
:::

## 8B.3 Choices, no prompts, and hooks

### Choices

A **list** value becomes a multiple-choice question, and the first entry is the default:

```json
{
  "project_name": "my-project",
  "language": ["python", "go", "rust"],
  "include_dockerfile": ["yes", "no"]
}
```

```text
  [2/3] Select language
    1 - python
    2 - go
    3 - rust
    Choose from [1/2/3] (1): 1
```

:::{warning} There are no booleans
`"include_dockerfile": ["yes", "no"]` gives you the *strings* `"yes"` and `"no"`. So the test in your
templates is `{% if cookiecutter.include_dockerfile == 'yes' %}` — comparing to a string, not
`{% if cookiecutter.include_dockerfile %}`. That second form is **always true**, because `"no"` is a
non-empty string. This is a common first mistake.
:::

### Running with no human present

For CI you need it non-interactive. Two flags:

```bash
# Take every default, prompt for nothing
cookiecutter --no-input ./mini-template

# Take defaults but override some on the command line
cookiecutter --no-input ./mini-template project_name=weather-api author=Federico

# Or read the answers from a file
cookiecutter --no-input --config-file answers.yaml ./mini-template
```

The config file uses a `default_context:` block:

`answers.yaml`

```yaml
default_context:
  project_name: "weather-api"
  author: "Federico"
  include_dockerfile: "yes"
```

:::{important} This file is the "sparse manifest"
Remember this shape. A small YAML file describing *what is different about this service*, fed to a
generator that produces all the boilerplate. [Section 8B.7](#d4-8b-7) shows what that buys at scale.
:::

### Hooks: running code during generation

Sometimes substitution is not enough. **Hooks** are Python scripts Cookiecutter runs for you:

```text
mini-template/
├── cookiecutter.json
├── hooks/
│   ├── pre_gen_project.py     # runs BEFORE generating — use it to validate
│   └── post_gen_project.py    # runs AFTER — use it to tidy up or compute
└── {{ cookiecutter.project_name }}/
```

Validation, so bad input fails immediately instead of producing a broken project:

`hooks/pre_gen_project.py`

```python
import re, sys

name = "{{ cookiecutter.project_name }}"

if not re.match(r"^[a-z][a-z0-9-]*$", name):
    print(f"ERROR: '{name}' must be lowercase letters, digits and hyphens.")
    sys.exit(1)          # a non-zero exit aborts generation
```

Post-generation clean-up:

`hooks/post_gen_project.py`

```python
import os

if "{{ cookiecutter.include_dockerfile }}" == "no":
    os.remove("Dockerfile")
```

:::{important} The trick that makes hooks powerful
**Hooks are themselves Jinja-rendered before Python runs them.** That is why
`"{{ cookiecutter.project_name }}"` above is a plain string by the time Python sees it.

Which means you can hand Python the *entire* answer set in one line:

```python
import json
context = json.loads(r"""{{ cookiecutter | jsonify }}""")
print(context["project_name"])
```

Now your hook is a normal Python program with a dictionary of every answer. It can compute anything —
and the chart generator below is built on it.
:::

Also note: `post_gen_project.py` runs **inside the generated directory**, so `os.remove("Dockerfile")`
refers to the new project's file, not the template's.


(helm-chart-generator)=
## 8B.4 Building a Helm chart generator

The following template has seven files, and every step can be run directly.

```text
mini-helm-template/
├── cookiecutter.json
├── hooks/
│   └── post_gen_project.py
└── {{ cookiecutter.chart_name }}/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        └── ingress.yaml
```

### The questions

`mini-helm-template/cookiecutter.json`

```json
{
  "chart_name": "my-app",
  "chart_namespace": "default",
  "chart_version": "0.1.0",

  "include_deployment": ["yes", "no"],
  "deployment_image_repo": "nginx",
  "deployment_image_tag": "1.27",
  "deployment_replicas": "1",
  "deployment_port": "80",

  "include_service": ["yes", "no"],
  "include_ingress": ["yes", "no"],
  "ingress_host": "my-app.internal.meditwin-project.eu",

  "_copy_without_render": ["templates/*"]
}
```

That last line is the most important line in the file.

:::{danger} `_copy_without_render` — the collision you must know about
**Helm and Jinja2 both use `{{ }}`.**

A Helm template contains `{{ .Values.replicaCount }}`. Jinja2 will try to evaluate that as *its own*
expression, fail to find a variable called `.Values`, and either crash or silently produce empty
output.

`"_copy_without_render": ["templates/*"]` tells Cookiecutter: *copy everything under `templates/`
byte-for-byte, do not touch it.* So:

- `Chart.yaml` and `values.yaml` → **Jinja's**, rendered at generation time.
- `templates/*` → **Helm's**, left alone, rendered later by `helm install`.

Two templating engines, one directory, cleanly separated. Forget this line and your generated chart
will be mysteriously empty.
:::

### `Chart.yaml` — rendered by Jinja

`mini-helm-template/{{ cookiecutter.chart_name }}/Chart.yaml`

```yaml
apiVersion: v2
name: {{ cookiecutter.chart_name }}
description: A Helm chart for {{ cookiecutter.chart_name }}
type: application
version: {{ cookiecutter.chart_version }}
```

### `values.yaml` — empty lists, on purpose

`mini-helm-template/{{ cookiecutter.chart_name }}/values.yaml`

```yaml
# Filled in by hooks/post_gen_project.py
deployments: []
services: []
ingresses: []
```

That looks useless. It is deliberate: it is **valid YAML that Python can load and append to**. The hook
does the real work.

### The Helm templates — copied verbatim, loop-driven

Because `values.yaml` holds *lists*, the templates iterate:

`templates/deployment.yaml`

```yaml
{{- range .Values.deployments }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .name }}
  namespace: {{ .namespace }}
spec:
  replicas: {{ .replicaCount | default 1 }}
  selector:
    matchLabels:
      app: {{ .name }}
  template:
    metadata:
      labels:
        app: {{ .name }}
    spec:
      containers:
        - name: {{ .name }}
          image: "{{ .image.repository }}:{{ .image.tag }}"
          ports:
            - name: http
              containerPort: {{ .port }}
          readinessProbe:
            tcpSocket:
              port: http
            initialDelaySeconds: 5
{{- end }}
```

`templates/service.yaml`

```yaml
{{- range .Values.services }}
---
apiVersion: v1
kind: Service
metadata:
  name: {{ .name }}
  namespace: {{ .namespace }}
spec:
  type: ClusterIP
  selector:
    app: {{ .name }}
  ports:
    - name: http
      port: 80
      targetPort: {{ .targetPort }}
{{- end }}
```

`templates/ingress.yaml`

```yaml
{{- range .Values.ingresses }}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .name }}
  namespace: {{ .namespace }}
  annotations:
    cert-manager.io/cluster-issuer: {{ .clusterIssuer }}
    external-dns.alpha.kubernetes.io/hostname: {{ .host | quote }}
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [{{ .host | quote }}]
      secretName: {{ .name }}-tls
  rules:
    - host: {{ .host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .name }}
                port:
                  number: 80
{{- end }}
```

Notice: **the templates never mention `my-app`.** They are completely generic. Everything specific
lives in `values.yaml`, which is generated.

### The hook that assembles `values.yaml`

`mini-helm-template/hooks/post_gen_project.py`

```python
"""Turn the flat Cookiecutter answers into the nested lists values.yaml needs."""
import json
import yaml

# Hooks are Jinja-rendered first, so this becomes a literal JSON object.
context = json.loads(r"""{{ cookiecutter | jsonify }}""")


def main() -> None:
    with open("values.yaml") as f:
        values = yaml.safe_load(f)

    name = context["chart_name"]
    namespace = context["chart_namespace"]

    if context["include_deployment"] == "yes":
        values["deployments"].append({
            "name": name,
            "namespace": namespace,
            "replicaCount": int(context["deployment_replicas"]),
            "port": int(context["deployment_port"]),
            "image": {
                "repository": context["deployment_image_repo"],
                "tag": context["deployment_image_tag"],
            },
        })

    if context["include_service"] == "yes":
        values["services"].append({
            "name": name,
            "namespace": namespace,
            "targetPort": int(context["deployment_port"]),
        })

    if context["include_ingress"] == "yes":
        values["ingresses"].append({
            "name": name,
            "namespace": namespace,
            "host": context["ingress_host"],
            "clusterIssuer": "ecmwf-meditwin-issuer",
        })

    with open("values.yaml", "w") as f:
        yaml.dump(values, f, default_flow_style=False, sort_keys=True)


if __name__ == "__main__":
    main()
```

### Run it

`weather-api.yaml` — the sparse manifest

```yaml
default_context:
  chart_name: "weather-api"
  chart_namespace: "weather"
  chart_version: "0.1.0"
  include_deployment: "yes"
  deployment_image_repo: "gitlab.example.com:6443/team/weather-api"
  deployment_image_tag: "abc123def456"
  deployment_replicas: "2"
  deployment_port: "8080"
  include_service: "yes"
  include_ingress: "yes"
  ingress_host: "weather-api.internal.meditwin-project.eu"
```

```bash
cookiecutter --no-input --config-file weather-api.yaml ./mini-helm-template
```

Out comes a chart:

`weather-api/values.yaml` — generated

```yaml
deployments:
- image:
    repository: gitlab.example.com:6443/team/weather-api
    tag: abc123def456
  name: weather-api
  namespace: weather
  port: 8080
  replicaCount: 2
ingresses:
- clusterIssuer: ecmwf-meditwin-issuer
  host: weather-api.internal.meditwin-project.eu
  name: weather-api
  namespace: weather
services:
- name: weather-api
  namespace: weather
  targetPort: 8080
```

And it is a real chart:

```bash
helm template ./weather-api
```

A Deployment, a Service and an Ingress — about 90 lines of Kubernetes YAML — from **13 lines of
input**.

:::{tip} Try deleting things
Set `include_ingress: "no"` and regenerate. `ingresses` stays `[]`, the `{{- range }}` loop in
`templates/ingress.yaml` iterates zero times, and no Ingress is produced — with no `if` statement
anywhere. That is why the empty-list design is worth the initial strangeness.
:::

## 8B.5 ⌨️ Hands-on 10B: turn your service into a template

⏱ **About 14 minutes.**

Your `~/climate-service` now has an app, a Dockerfile, tests and a pipeline that works. The next
service should start from all of that, not from an empty folder.

```bash
mkdir -p ~/templates/service-template && cd ~/templates/service-template
```

`cookiecutter.json`

```json
{
  "service_name": "ocean-telemetry",
  "port": "8000",
  "app_version": "1.0.0"
}
```

Copy the working project in as the template body:

```bash
mkdir '{{ cookiecutter.service_name }}'
cd '{{ cookiecutter.service_name }}'
cp -r ~/climate-service/{app.py,Dockerfile,requirements.txt,pytest.ini,tests,.gitignore,.gitlab-ci.yml,.releaserc.json} .
```

Now replace what differs between services with placeholders:

```bash
sed -i 's/ecmwf-climate-telemetry/ecmwf-{{ cookiecutter.service_name }}/' app.py
sed -i 's/8000/{{ cookiecutter.port }}/' app.py Dockerfile
sed -i 's/APP_VERSION=1.0.0/APP_VERSION={{ cookiecutter.app_version }}/' Dockerfile
grep -rn 'cookiecutter' .
```

The `grep` should list five lines: the service name and the port in `app.py`, the port twice in the
`Dockerfile` (`EXPOSE` and gunicorn's `--bind`), and the version. Anything you missed stays hard-coded in every service generated
afterwards.

:::{note} Why `.gitlab-ci.yml` needs no placeholder
It never mentions the service by name: the image path comes from `$CI_REGISTRY_IMAGE`, which GitLab
fills in from whichever project runs the pipeline. A file that already takes its specifics from its
environment does not need templating. `.releaserc.json` is copied for the same reason: the `release`
job needs it. What the template cannot copy is the `GITLAB_TOKEN` variable of Hands-on 6: each new
project needs its own token.
:::

Generate a new service, overriding two answers on the command line:

```bash
cd ~
cookiecutter --no-input ~/templates/service-template service_name=ocean-telemetry port=8080
cd ~/ocean-telemetry
docker run --rm -v "$PWD":/src -w /src python:3.12-slim-bookworm \
  sh -c "pip install -q -r requirements.txt && pytest -q tests/"
docker build -t ocean-telemetry:dev .
docker run --rm -d -p 8080:8080 --name ocean ocean-telemetry:dev
curl -s localhost:8080/health
docker rm -f ocean
```

```json
{"status":"healthy","service":"ecmwf-ocean-telemetry","version":"1.0.0","node":"…","region":"unknown"}
```

**Checkpoint 10B** — a second service, with its own name and port, passes the same tests and builds
with the same Dockerfile, and you typed none of it.

Think about one thing before moving on: next month you fix a bug in the template's Dockerfile. What
happens to `ocean-telemetry`?

:::{dropdown} Answer
Nothing. Cookiecutter generates once and forgets; projects made from the old template keep the old
Dockerfile. [Section 8B.7](#d4-8b-7) deals with this by regenerating from a pipeline. For services that people
edit by hand afterwards, a tool such as `cruft` records which template version a project came from
and can apply later template changes as a diff.
:::

## 8B.6 ⌨️ Hands-on 11B: generate the chart for your service and let Fleet deploy it

⏱ **About 14 minutes.**

You now replace the hand-written manifests of Hands-on 7 with a chart generated by the
`mini-helm-template` of [section 8B.4](#helm-chart-generator). If you did not build the template while reading, create its seven
files now in `~/templates/mini-helm-template`.

### Step 1: make the template fit this cluster

Generate once with a sparse manifest for your service and look at the result before trusting it.

`~/climate-deploy/climate-telemetry.yaml`

```yaml
default_context:
  chart_name: "climate-telemetry"
  chart_namespace: "traineeNN"
  chart_version: "0.1.0"
  include_deployment: "yes"
  deployment_image_repo: "gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN"
  deployment_image_tag: "<short-sha>"   # the tag running now: the output of the git log command in Hands-on 8
  deployment_replicas: "2"
  deployment_port: "8000"
  include_service: "yes"
  include_ingress: "yes"
  ingress_host: "climate-traineeNN.training.meditwin-project.eu"
```

```bash
cd ~/climate-deploy
cookiecutter --no-input --config-file climate-telemetry.yaml ~/templates/mini-helm-template
helm template ./climate-telemetry | grep -nE 'image:|cluster-issuer|imagePullSecrets|ingressClassName'
```

Three things are wrong for this cluster:

- `cluster-issuer: ecmwf-meditwin-issuer` is hard-coded in the hook. This cluster's issuer is
  `ecmwf-meditwin-training-issuer` (`kubectl get clusterissuer`).
- The Ingress asks for class `nginx` and carries an nginx-only annotation. This cluster's default class
  is `traefik` (`kubectl get ingressclass`).
- There is no `imagePullSecrets`, so the cluster could not pull from your private registry.

That is the usual life of a template: the first real user finds what it assumed. Fix the template,
not the output, so every chart generated afterwards is right:

1. In `cookiecutter.json`, add two questions, above `_copy_without_render` (each line ends with a comma,
   so it must not become the last entry):
   ```json
     "cluster_issuer": "ecmwf-meditwin-training-issuer",
     "image_pull_secret": "gitlab-registry",
   ```
2. In `hooks/post_gen_project.py`, replace `"clusterIssuer": "ecmwf-meditwin-issuer"` with
   `"clusterIssuer": context["cluster_issuer"]`, and add `"imagePullSecret":
   context["image_pull_secret"],` to the dictionary appended to `values["deployments"]`.
3. In `templates/deployment.yaml`, under the Pod `spec:` and above `containers:`, add:
   ```yaml
         {{- with .imagePullSecret }}
         imagePullSecrets:
           - name: {{ . }}
         {{- end }}
   ```
4. In `templates/ingress.yaml`, change `ingressClassName: nginx` to `ingressClassName: traefik` and
   delete the `nginx.ingress.kubernetes.io/ssl-redirect` annotation. (Making the class a question in
   `cookiecutter.json` too is the better fix, if you have time.)

Regenerate and check again:

```bash
rm -rf climate-telemetry
cookiecutter --no-input --config-file climate-telemetry.yaml ~/templates/mini-helm-template
helm lint ./climate-telemetry
helm template ./climate-telemetry | grep -nE 'image:|cluster-issuer|imagePullSecrets|ingressClassName' -A1
```

### Step 2: make it a Fleet bundle

Fleet treats a folder containing `Chart.yaml` as a Helm bundle. The `fleet.yaml` next to it only adds
the namespace and a readable release name:

`~/climate-deploy/climate-telemetry/fleet.yaml`

```yaml
defaultNamespace: traineeNN
helm:
  releaseName: climate-telemetry
```

Regenerating deletes the folder, and with it this file. So put `fleet.yaml` in the template instead,
next to `Chart.yaml`, and let Jinja fill it in:

`mini-helm-template/{{ cookiecutter.chart_name }}/fleet.yaml`

```yaml
defaultNamespace: {{ cookiecutter.chart_namespace }}
helm:
  releaseName: {{ cookiecutter.chart_name }}
```

Regenerate once more. The generator now produces a complete bundle, not just a chart.

### Step 3: switch Fleet from the manifests to the chart

```bash
git switch -c deploy/generated-chart
git add climate-telemetry.yaml climate-telemetry
git commit -m "deploy: climate-telemetry from a generated Helm chart"
git push -u origin deploy/generated-chart
```

Merge it. Nothing changes yet, because the GitRepo still points at `manifests`. Then switch Fleet over to it. This takes three steps, because of a race you will see once and
should recognise afterwards:

1. In Rancher (**☰ → Continuous Delivery → Resources → Git Repos**), open the **⋮** menu at the end of your
   GitRepo's row → **Edit Config**. In the wizard's step **Repository details**, change **Paths** from
   `manifests` to `climate-telemetry`,
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

Then remove the unused folder through a merge request, so the repository only describes what is
deployed:

```bash
git switch main && git pull
git switch -c chore/remove-manifests
git rm -r manifests
git commit -m "chore: remove the manifests replaced by the chart"
git push -u origin chore/remove-manifests
```

Open the merge request and merge it.

**Checkpoint 11B** — the bundle for `climate-telemetry` is *Ready*, `helm -n traineeNN list` shows a
release called `climate-telemetry`, and `/health` answers over HTTPS again.

Look closely at the answer: `region` is back to `unknown`. The generated chart has no way to set
environment variables, so the setting from Hands-on 8 was lost in the switch. Nobody reviewing the
chart would have noticed, because the diff did not show a line being removed — it showed a whole new
folder. Adding `env` support to the template is a good exercise if you have time.

(d4-8b-7)=
## 8B.7 Cookiecutter at scale

The generator above is the small version of a system that runs on a production platform, producing
tens of charts. Three ideas carry over, and one problem appears that the small version does not have.

- **The sparse manifest lives with the code.** Each application repository holds its own answers
  file. Its pipeline writes the new image tag into it, runs `cookiecutter --no-input`, and publishes
  the generated chart to a separate chart repository, which GitOps deploys.
- **The output is never edited by hand.** The pipeline deletes the old chart and copies in the new
  one, every time. A hand edit to the generated chart is destroyed on the next run, which is exactly
  what keeps the answers file the single source of truth.
- **One declaration, several correct outputs.** A secret declared once in the answers file becomes an
  `ExternalSecret` entry, a container `env` entry and the Secret name that wires them together. The
  saving is not keystrokes; it is three places that can no longer disagree.

The new problem is **template variants**. A web service, a batch job and a model server need different
sets of templates. One template full of `{% if %}` flags becomes hard to read; one template per
variant (that system uses a Git branch per component, selected with `cookiecutter --checkout`) turns a
fix to `deployment.yaml` into thirty merges. Neither is free, and choosing between them is the real
work of templating once more than a few teams use a template.

## 8B.8 Dagger: the problem

Consider the image build from [section 3](03-gitlab-ci.md). A mistake in it produces
this sequence:

1. Edit `.gitlab-ci.yml`.
2. Commit and push.
3. Wait for a runner.
4. Wait for the job.
5. Read the log, find the next mistake.
6. Return to step 1.

Each iteration costs minutes, and the pipeline cannot be run locally because it is
data interpreted by GitLab. The usual result is a series of commits with messages
such as "fix pipeline", "fix pipeline again", "try once more". That history is a
reliable sign of this problem, and it appears in almost every repository that has a
non-trivial pipeline.

:::{note} Jargon: *Dagger*
**Dagger** is a portable execution engine for CI pipelines. A pipeline is written as a
program, in a general-purpose language, which calls an API to run steps in containers.
The engine executes the same program identically on a laptop and on a CI runner.
:::

## 8B.9 The model

Three ideas carry the whole design.

**Every step runs in a container.** There is no ambient machine state, so a step
behaves the same wherever it runs. This is the same reasoning as the fresh runner in
[section 2](02-ci-cd.md), applied to each individual step.

**Directories and files are values.** A directory is a value that can be passed to a
function, returned from one, and modified to produce a new directory. Nothing is
mutated on disk.

**Results are cached by content.** A step whose inputs have not changed is not
re-executed. The unit of caching is a step rather than a job, so editing a test does
not rebuild dependencies.

```{mermaid}
flowchart TB
    P["Pipeline program"] --> E["Dagger engine"]
    E --> S1["Step in a container"]
    E --> S2["Step in a container"]
    S1 --> C[("Content-addressed cache")]
    S2 --> C
```

(d4-8b-10)=
## 8B.10 A worked example

The test and build steps of your climate service's pipeline from [section 3](03-gitlab-ci.md), written
as a Dagger module in Python, plus a third function that publishes the image.

`.dagger/src/climate_telemetry/main.py`

```python
import dagger
from dagger import dag, function, object_type


@object_type
class ClimateTelemetry:
    @function
    def test_env(self, source: dagger.Directory) -> dagger.Container:
        """The container the tests run in, with the dependencies installed."""
        return (
            dag.container()
            .from_("python:3.12-slim-bookworm")       # the same image as run-unit-tests
            # A cache volume, so dependencies are not downloaded on every run.
            .with_mounted_cache("/root/.cache/pip", dag.cache_volume("pip"))
            .with_directory("/src", source, exclude=[".dagger", ".venv", ".git"])
            .with_workdir("/src")
            .with_exec(["pip", "install", "-r", "requirements.txt"])
        )

    @function
    async def test(self, source: dagger.Directory) -> str:
        """Run the test suite and return its output."""
        return await self.test_env(source).with_exec(["pytest", "-v", "tests/"]).stdout()

    @function
    def build(self, source: dagger.Directory) -> dagger.Container:
        """Build the image from the project's own Dockerfile."""
        return source.docker_build()

    @function
    async def publish(
        self,
        source: dagger.Directory,
        registry: str,
        image: str,
        username: str,
        password: dagger.Secret,
    ) -> str:
        """Test, then build, then push. Returns the published reference."""
        await self.test(source)
        return await (
            self.build(source)
            .with_registry_auth(registry, username, password)
            .publish(image)
        )
```

Four things in that file are worth naming.

`publish` calls `self.test` and `self.build` as ordinary function calls. Expressing
"do not publish something that failed its tests" needs no `needs:` graph, because it
is the control flow of a program.

`password` is typed as `dagger.Secret`. The engine keeps the value out of logs and out
of the cache, so it cannot be accidentally printed or cached into an image layer.

`source` is a `Directory` parameter rather than a path. The function does not know or
care whether the directory came from a local checkout or a Git clone.

`test_env` returns a *container* and `test` returns a *string*. That split is what lets you open a
shell in the test environment: you can only step inside a container.

(d4-8b-11)=
## 8B.11 Running it

The same functions run locally:

```bash
# What can this pipeline do?
dagger functions

# Run the tests against the working tree, before committing anything.
dagger call test --source=.

# Build the image and open a shell inside it.
dagger call build --source=. terminal
```

That third command is the point of the exercise. A step can be inspected
interactively, in the exact container the pipeline uses, without a commit.

Calling the same functions from GitLab CI is then a thin wrapper, because this GitLab has a runner
built for it: the one tagged **`dagger`**. Every job it runs gets a Dagger engine that is already
running on the node, reached through a socket the runner mounts into the job and announces in
`_EXPERIMENTAL_DAGGER_RUNNER_HOST`. So the job needs no Docker-in-Docker and no engine of its own,
only the `dagger` CLI, at the engine's version:

`.gitlab-ci.yml`

```yaml
.dagger:
  image:
    name: ghcr.io/purpleclay/dagger-cli:0.18.10   # the CLI only, same version as the engine
    entrypoint: [""]                              # the image's entrypoint is `dagger` itself
  tags: [dagger]                                  # the runner with an engine on its nodes
  variables:
    DAGGER_NO_NAG: "1"

dagger-test:
  extends: .dagger
  stage: test
  script:
    - dagger call test --source=.

dagger-publish:
  extends: .dagger
  stage: build
  needs: [dagger-test]
  script:
    - dagger call publish
        --source=.
        --registry="$CI_REGISTRY"
        --image="$CI_REGISTRY_IMAGE:dagger-$CI_COMMIT_SHORT_SHA"
        --username="$CI_REGISTRY_USER"
        --password=env:CI_REGISTRY_PASSWORD
```

`tags: [dagger]` is the only thing that ties these jobs to this platform. The `script:` lines are
exactly what you type on your own machine: the logic lives in the module, not in the YAML, so moving to
another CI system means rewriting these few lines, not the pipeline.

Note `--password=env:CI_REGISTRY_PASSWORD`. The `env:` prefix passes a *reference* to the variable
rather than its value, so the secret is never expanded onto a command line where `ps` could read it,
and the engine never writes it to its cache.

The engine on the runner keeps its cache between jobs and between pipelines. Run the same step twice
on unchanged inputs and the second run reports its steps as `CACHED`: the property you see locally,
now on the shared runner.

## 8B.12 ⌨️ Hands-on 12B: one pipeline, on your machine and on GitLab

⏱ **About 20 minutes. Optional:** do it if you finish early, or after the course.

Every pipeline change so far was checked the same way: push, wait for a runner, read the log. Here you
turn the test and build steps of `~/climate-service` into the module of [section 8B.10](#d4-8b-10), run them on your
own machine, and then run **the same functions** in GitLab CI on the `dagger` runner, and compare.

```bash
cd ~/climate-service
git switch main && git pull
git switch -c ci/dagger

dagger version                                  # preinstalled, same version as the CI engine
docker ps > /dev/null && echo "docker ok"     # the engine runs as a container
```

Create the module:

```bash
dagger init --sdk=python --source=.dagger --name=climate-telemetry
```

Before anything else, pin one of the module's own dependencies, then let Dagger lock it:

```bash
cat >> .dagger/pyproject.toml <<'EOF'

[tool.uv]
# gql 4 asks the engine a question engine v0.18.10 does not understand.
constraint-dependencies = ["gql<4"]
EOF
dagger develop
grep -A1 '^name = "gql"' .dagger/uv.lock         # version = "3.5.x"
```

The module talks to the engine through the Python library `gql`. The Dagger SDK accepts any `gql`
from 3.5 upwards, so `dagger init` locks the newest release, and gql 4 sends a schema query the
v0.18.10 engine rejects: every `dagger call` then fails with *`Unknown argument "includeDeprecated"
on field "__Field.args"`*. Pinning it below 4 matches the library to the engine. This is the same
lesson as the engine version itself: a tool you pin is only pinned if everything it pulls in is
pinned too, which is what the lock file `.dagger/uv.lock` is for, and why it is committed.

`dagger init` generates a Python package under `.dagger/src/climate_telemetry/`. Replace the
contents of the file that defines the class (`main.py` in current versions, `__init__.py` in older
ones) with the module from [section 8B.10](#d4-8b-10).

Run the tests without committing anything, twice:

```bash
dagger call test --source=.
dagger call test --source=.
```

The first run downloads the engine and the base image. The second finishes in seconds, because
nothing it depends on changed. Every command ends with *A new release of dagger is available*:
ignore it. The CLI must stay at the engine's version, v0.18.10, or the two refuse to talk.

Now break a test on purpose — change `"healthy"` to `"sick"` in `tests/test_health.py` — and run it
again. You get the same failure the pipeline would give you, without a push. Undo the change.

Then use what CI cannot give you at all, a shell **inside** the step:

```bash
dagger call test-env --source=. terminal
```

Run `pytest -v tests/` by hand, look around, and `exit`. Do the same with the image the pipeline
builds:

```bash
dagger call build --source=. terminal
```

Inside, `ls /app` shows exactly what the Dockerfile copied: `app.py` and `requirements.txt`, nothing
from `tests/`.

**Checkpoint 12B, part 1** — `dagger call test --source=.` passes locally, a broken test fails locally,
and you have opened a shell in both the test environment and the built image.

### On GitLab: the same functions on the `dagger` runner

Now the part a local tool alone cannot show: the pipeline running the very same code. Add the
`.dagger` template and the two jobs of [section 8B.11](#d4-8b-11) to `.gitlab-ci.yml`, next to the jobs you
already have. Keep `run-unit-tests` and `build-container-image`: comparing the two is the point.

`.gitlab-ci.yml` — add at the end

```yaml
.dagger:
  image:
    name: ghcr.io/purpleclay/dagger-cli:0.18.10
    entrypoint: [""]
  tags: [dagger]
  variables:
    DAGGER_NO_NAG: "1"

dagger-test:
  extends: .dagger
  stage: test
  script:
    - dagger call test --source=.
  rules:
    - if: $CI_COMMIT_BRANCH

dagger-publish:
  extends: .dagger
  stage: build
  needs: [dagger-test]
  script:
    - dagger call publish
        --source=.
        --registry="$CI_REGISTRY"
        --image="$CI_REGISTRY_IMAGE:dagger-$CI_COMMIT_SHORT_SHA"
        --username="$CI_REGISTRY_USER"
        --password=env:CI_REGISTRY_PASSWORD
  rules:
    - if: $CI_COMMIT_BRANCH
```

The image is tagged `dagger-<sha>`, so it never replaces the image that `build-container-image`
publishes and that Fleet deploys.

Commit the module and the jobs together, and open a merge request:

```bash
git add .dagger dagger.json .gitlab-ci.yml
git commit -m "ci: run the test and build steps as Dagger functions, locally and in CI"
git push -u origin ci/dagger
```

(`.dagger/sdk/` is not committed: `dagger init` lists it in `.dagger/.gitignore`, and the engine
regenerates it. `.dagger/uv.lock` is committed: it is what keeps `gql` below 4 on the runner too.)

In the merge request's pipeline, open the jobs side by side:

1. **`run-unit-tests` and `dagger-test`.** Both print the same pytest result. `dagger-test` printed
   exactly what you saw on your own machine an hour ago, because it ran the same function in the
   same container.
2. **`dagger-publish`.** Under *Deploy → Container Registry* a new tag `dagger-<sha>` appears next to
   the one Kaniko pushed. In the job log, `test` runs again inside `publish` before anything is
   pushed: the "test before publish" rule is a line of Python, not a `needs:`.
3. **Retry `dagger-test`** (the ↻ button on the job). It finishes far faster, and the log shows the
   steps as `CACHED`: the engine on the runner remembered them, as yours did.
4. **Break the test again** (`"healthy"` → `"sick"`), commit, push. `run-unit-tests` and `dagger-test`
   both fail, with the same failure you got locally before pushing. Revert the commit and push.

**Checkpoint 12B, part 2** — in one pipeline, `dagger-test` and `dagger-publish` are green on the
`dagger` runner, the registry has the `dagger-<sha>` image, and a retried `dagger-test` shows `CACHED`
steps.

Merge the request once the pipeline is green. Whether `dagger-test` and `dagger-publish` should then
*replace* `run-unit-tests` and `build-container-image` is a real decision, with the costs of
[section 8B.13](#d4-8b-13) on one side and one place for the build logic on the other. In this course you keep
both.

:::{dropdown} The Dagger jobs stay *pending*: "no runner with the tag `dagger`"
The `dagger` runner must be available to the `training/autumn-2026` group. Check *Settings → CI/CD →
Runners* in your project: it should be listed under the runners available to it. If it is not, tell
the instructor; there is nothing to fix in your pipeline.
:::

:::{dropdown} The job fails at once with *`unknown command "sh"`* or *`unknown flag`*
The `entrypoint: [""]` line is missing. The image's own entrypoint is `dagger`, so GitLab's script is
handed to `dagger` as arguments instead of to a shell.
:::

:::{dropdown} `dagger-test` fails on the runner with *`Unknown argument "includeDeprecated"`* but works locally
`.dagger/uv.lock` was not committed, or was committed before the `gql<4` constraint. Commit the current
lock file.
:::

:::{dropdown} `dagger: command not found`
The VM normally has it in `/usr/local/bin`. If yours does not, install the same version into your
home directory:
```bash
curl -fsSL https://dl.dagger.io/dagger/install.sh | DAGGER_VERSION=0.18.10 BIN_DIR=$HOME/.local/bin sh
export PATH="$HOME/.local/bin:$PATH"
```
:::

:::{dropdown} The first run hangs at "connecting to engine"
Dagger starts its engine as a container, so `docker ps` must work. If it gives a permission error,
your user is not in the `docker` group in this shell session: log out and back in.
:::

:::{dropdown} `Failed to build schema from introspection query: Unknown argument "includeDeprecated"`
The module's lock file has gql 4. Add the `[tool.uv]` constraint shown after `dagger init` to
`.dagger/pyproject.toml`, run `dagger develop`, and check that `.dagger/uv.lock` now has gql 3.5.x.
:::

:::{dropdown} `ModuleNotFoundError` or "no object found"
The class name must match the module name: `--name=climate-telemetry` expects `ClimateTelemetry`. Run
`dagger develop` to regenerate the SDK files, and check which file the generated code imports the
class from.
:::

(d4-8b-13)=
## 8B.13 What it costs

Dagger is not free, and the costs are the reason many projects reasonably decline it.

**Another engine to run.** The engine is a container that needs somewhere to run, with
a cache volume, both locally and on the runners. That is real operational work: the `dagger` runner
of this course exists because somebody runs an engine on every node it uses, keeps it at the version
every module pins, and shares it, and its cache, between every project that uses the runner.

**Another language in the repository.** A pipeline in Python needs the same review,
dependency management and upkeep as application code. For a project whose pipeline is
twenty lines of YAML, this is a poor trade.

**A less familiar failure mode.** When the engine misbehaves, the error is about the
engine rather than about the build, and fewer people can read it. Cache reuse in
particular can preserve state that a run should not have inherited, which produces
failures that disappear when the cache is cleared.

:::{important} When the trade is worth making
Dagger earns its place when a pipeline is long enough that iterating on it is
expensive, when the same steps have to run in more than one place, or when the
pipeline needs real control flow such as loops and conditionals. A short, linear,
single-system pipeline is better left as YAML.
:::

## 8B.14 Core takeaway

> **Cookiecutter makes the next project start from one that already works; Dagger makes a pipeline
> step something you can run, and step inside, before you push, and then runs that same step in CI.**

- Cookiecutter is file copying plus Jinja2. `cookiecutter.json` holds the questions; `--no-input
  --config-file` is how a pipeline runs it, and that answers file is the *sparse manifest*.
- `_copy_without_render: ["templates/*"]` is mandatory for Helm charts, because Helm and Jinja both use
  `{{ }}`. There are no booleans, only the strings `"yes"` and `"no"`.
- Generated files are never edited by hand, or the generator becomes decorative.
- Dagger runs each step in a container from a program, the same way on a laptop and on a runner, and
  caches by content. `dagger call … terminal` opens a shell inside a step. In GitLab the job is the
  CLI image, `tags: [dagger]` and the same `dagger call` line: the runner provides the engine.
- Both are worth their cost with many services or long pipelines, and not before.

## 8B.15 Check yourself

1. Your generated chart's `templates/` files come out empty, or Cookiecutter crashes with
   `'.Values' is undefined`. What is wrong?
2. Your template has `"include_metrics": ["yes", "no"]` and `{% if cookiecutter.include_metrics %}` in a
   file. The section always appears. Why?
3. A colleague says Dagger is "just a nicer syntax for the same YAML". What is the substantive
   difference?
4. When would you advise a team *not* to adopt Dagger?

:::{dropdown} Solutions
**1.** You are missing `"_copy_without_render": ["templates/*"]` in `cookiecutter.json`. Jinja2 is
processing the Helm templates and trying to evaluate `{{ .Values.replicaCount }}` as a Jinja
expression. Since Jinja has no variable `.Values`, it either raises an error or renders empty strings —
producing a chart whose templates are blank. Add the line and Cookiecutter will copy those files
untouched, leaving `{{ }}` for Helm.

**2.** Because `cookiecutter.include_metrics` is the **string** `"no"`, and a non-empty string is truthy
in Jinja2. `{% if "no" %}` is true. You need an explicit comparison:
```jinja
{% if cookiecutter.include_metrics == 'yes' %}
```
This is the single most common Cookiecutter bug. If you want real booleans, use a `pre_gen` hook to
normalize, or a JSON schema. Comparing to `'yes'` is the idiomatic fix, and what the template above
does throughout.

**3.** The substantive difference is where the pipeline runs. YAML is data interpreted
by one CI system, so the only way to execute it is to commit and let that system
interpret it. A Dagger pipeline is a program executed by an engine that runs anywhere,
so it can be executed against the working tree before committing, and a step can be
entered interactively. Two consequences follow: iterating no longer costs a commit and
a runner wait, and the pipeline is portable, because moving CI system means rewriting
the wrapper rather than the pipeline.

**4.** When the pipeline is short, linear and runs in one place. A twenty-line YAML
file that builds and pushes an image is cheaper to read and maintain than an
equivalent program plus an engine plus a cache volume, and it needs no additional
skill from whoever is on call. Also when nobody on the team is comfortable maintaining
the pipeline's language: a pipeline that only one person can debug is worse than a
verbose one that everybody can read.
:::

---

Continue with: [9. Day 4 cheat sheet](09-cheatsheet.md).
