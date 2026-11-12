# 3. GitLab CI hands-on

Everything in this section goes in **one file** at the root of the repository,
called `.gitlab-ci.yml`. Commit it, push, and GitLab runs it. There is nothing else to install.

**Slides:** the slides used in class are in one file, {download}`day04-git-cicd-gitops.pdf`.

## 3.1 ⌨️ Your first pipeline

⏱ **Type along with the instructor**, about 5 minutes.

Use the scratch project made for you: `training/autumn-2026/hello-ci-traineeNN`, empty and private.
Clone it (the URL is under the blue **Code** button), and add this file:

`.gitlab-ci.yml`

```yaml
hello:
  script:
    - echo "Hello from a runner!"
    - whoami
    - pwd
```

```bash
git add .gitlab-ci.yml
git commit -m "ci: add first pipeline"
git push
```

Go to your project in GitLab → **Build → Pipelines**. There is a pipeline, with one job called
`hello`.

:::{figure} img/ui-gl-build.png
:alt: The GitLab left sidebar with the Build group open and Pipelines marked.
**Build → Pipelines.** ① Open the group. ② Click the page.
:::

:::{figure} img/gitlab-pipelines-list.png
:alt: The GitLab Pipelines list, showing one row per pipeline with its status, branch and commit.
**Build → Pipelines.** One row per pipeline. The status on the left is the whole pipeline's
result, and the commit it ran against is on the right, so a red row always names the change
that broke it.
:::

Click it and read the log:

```text
Running with gitlab-runner 17.x
Preparing the "docker" executor
Using Docker executor with image ruby:3.1 ...
$ echo "Hello from a runner!"
Hello from a runner!
$ whoami
root
$ pwd
/builds/you/my-project
Job succeeded
```

Committing a file caused code to run on a build server. Two things to notice:

- Your repository was already checked out at `/builds/you/my-project`. GitLab does that for you
  before every job.
- It picked some default image (`ruby:3.1` above). You should always choose your own.

## 3.2 Choosing the image

Each job runs inside a container. Say which one:

```yaml
hello:
  image: python:3.12-slim
  script:
    - python --version
```

```text
$ python --version
Python 3.12.8
```

:::{tip} Pick the smallest image that has what you need
`python:3.12-slim` is ~150 MB; `python:3.12` is ~1 GB. The runner downloads it on every job, so this
is real time. If you only need `curl` and `jq`, use `alpine:3.22` and `apk add`.
:::

## 3.3 Stages: making order explicit

Without `stages`, every job runs at once. Declare the order:

`.gitlab-ci.yml`

```yaml
stages:
  - lint
  - test
  - build

check-format:
  stage: lint
  image: python:3.12-slim
  script:
    - pip install ruff
    - ruff format --check .

unit-tests:
  stage: test
  image: python:3.12-slim
  script:
    - pip install -r requirements.txt
    - pytest -v

build-package:
  stage: build
  image: python:3.12-slim
  script:
    - pip install build
    - python -m build
```

Now `check-format` must pass before `unit-tests` starts, which must pass before `build-package`
starts. Add a second job to `test` and the two run side by side:

```yaml
type-check:
  stage: test
  image: python:3.12-slim
  script:
    - pip install mypy
    - mypy src/
```

:::{warning} A job with no `stage` goes into `test`
That is GitLab's default and it surprises people. Always write `stage:` explicitly.
:::

(d4-3-4)=
## 3.4 Artifacts: keeping what a job produced

A runner is **disposable**. Every job starts in a fresh, empty environment, and that environment is
destroyed when the job ends. A file the job made is gone with it, unless you tell GitLab to keep it.

**Two jobs, one file.** The first job makes a bundle. The second job, in a later stage, checks it:

```yaml
stages: [build, check]

bundle:
  stage: build
  image: python:3.12-slim
  script:
    - tar czf climate-service.tar.gz app.py requirements.txt

check-bundle:
  stage: check
  image: python:3.12-slim
  script:
    - tar tzf climate-service.tar.gz      # list what is inside
```

`check-bundle` fails with `climate-service.tar.gz: Cannot open: No such file or directory`.
`bundle` made the file, then its runner was destroyed, and `check-bundle` runs in a new, empty
one. Declare the file as an **artifact**, and GitLab uploads it when `bundle` ends and downloads it
into every later job:

```yaml
bundle:                     # check-bundle is unchanged
  stage: build
  image: python:3.12-slim
  script:
    - tar czf climate-service.tar.gz app.py requirements.txt
  artifacts:
    paths:
      - climate-service.tar.gz
```

Now `check-bundle` passes, and the bundle is also downloadable from the job's page.

:::{figure} img/gitlab-job-artifacts.png
:alt: A GitLab job page: the log ends by uploading trivy-report.txt, and the right-hand panel shows Job artifacts with Keep, Download and Browse buttons.

**A job that declared an artifact.** This one is the `security-audit` job of a Day 5 project, which
keeps its scan report; `bundle` looks the same with its `.tar.gz`. ① At the end of the log the runner
uploads the files listed under `paths:`. ② **Download** gets them as a zip. ③ **Browse** opens them in
the browser, one file at a time. Open a job from **Build → Jobs**, or click it in a pipeline.
:::

The same applies to a test report, which is a result you want to read after the job:

```yaml
unit-tests:
  stage: test
  image: python:3.12-slim
  script:
    - pip install -r requirements.txt pytest-cov
    - pytest --junitxml=report.xml --cov=src --cov-report=term
  artifacts:
    when: always            # keep the report even when tests fail, which is when it is needed
    expire_in: 1 week
    paths:
      - report.xml
    reports:
      junit: report.xml     # makes GitLab show a test summary in the merge request
```

`paths:` makes the file downloadable and passes it to later stages. `reports:` is different — it tells
GitLab to *understand* the file, so failed tests appear directly in the merge request instead of
hiding in a log.

:::{figure} img/gitlab-job-test-summary.png
:alt: A GitLab job page for run-unit-tests: the log shows report.xml uploaded as junit, and the right-hand panel shows a Test summary link with the number 12.

**A job with a `reports: junit:` artifact.** ① The runner uploads `report.xml` as a `junit` report. ② The
job's panel now has a **Test summary**: the number is how many tests GitLab read from the file, and it
links to the report below. This job declared only `reports:`, no `paths:`, so it has no Download button.
:::

:::{figure} img/gitlab-test-report.png
:alt: The Tests tab of a GitLab pipeline, opened on the run-unit-tests job, listing each test with its suite, name, status, duration and a View details button.

**The test report.** ① Every pipeline that has a `junit` report gets a **Tests** tab, next to **Pipeline**
and **Jobs**. ② Click a job to see its tests. ③ Each test is a row, by name; a failed one is marked
red, and **View details** shows its error. This is what appears in a merge request too, instead of
line 400 of a log.
:::

:::{important} `when: always` on test artifacts
The default is to keep artifacts only on success, which is exactly backwards for a test report. When
you most want the report is when the job went red.
:::

Typical artifacts: `report.xml`, `coverage.xml`, `trivy-report.txt`, a built package
(`dist/*.whl`, `*.tar.gz`). Each is a **result**: if it disappeared, you would lose information, not
just time.

## 3.5 ⌨️ Hands-on 3: your service's first pipeline

⏱ **About 15 minutes.**

The pipeline you read in [section 2.6](02-ci-cd.md) runs `pip install -r requirements.txt` and
`pytest tests/`, but there is no `tests/` yet, and the project's `requirements.txt` from Day 2
lists only what the service needs to run (Flask and gunicorn), not the test runner. Its first job
would fail at once. Add both, then the pipeline, on a branch.

```bash
cd ~/climate-service
git switch main && git pull
git switch -c test/health-endpoint
mkdir -p tests
```

Add the test runner as a third line of `requirements.txt`, under the two from Day 2:

`requirements.txt`

```text
flask==3.1.3
gunicorn==26.2.0
pytest==8.3.4
```

(pytest then ends up in the image as well. Larger projects keep test tools in a separate
`requirements-dev.txt`; one file keeps this course simpler.)

`pytest.ini`

```ini
[pytest]
# Lets the files in tests/ import app.py from the project root.
pythonpath = .
```

`tests/test_health.py`

```python
import pytest

from app import app


@pytest.fixture
def client():
    # Flask's test client calls the app directly: no server, no port.
    return app.test_client()


def test_health_reports_healthy(client):
    response = client.get("/health")
    assert response.status_code == 200
    body = response.get_json()
    assert body["status"] == "healthy"


def test_unknown_path_is_404(client):
    assert client.get("/nope").status_code == 404
```

Run the tests locally, **in the same image the pipeline uses**, before pushing anything:

```bash
docker run --rm -v "$PWD":/src -w /src python:3.12-slim-bookworm \
  sh -c "pip install -q -r requirements.txt && pytest -v tests/"
```

```text
tests/test_health.py::test_health_reports_healthy PASSED
tests/test_health.py::test_unknown_path_is_404 PASSED
```

Now add the pipeline: create `.gitlab-ci.yml` at the project root with the file from
[section 2.6](02-ci-cd.md). Then apply what [section 3.4](#d4-3-4) just covered: change the test job so it
publishes a test report. The build job uses Kaniko, which [section 3.12](#d4-3-12) explains; for now it is enough
to know that it builds the `Dockerfile` and pushes the image to the project's registry.

`.gitlab-ci.yml` — the test job, updated

```yaml
run-unit-tests:
  stage: test
  image: python:3.12-slim-bookworm
  script:
    - pip install --no-cache-dir -r requirements.txt
    - pytest tests/ --junitxml=report.xml
  artifacts:
    when: always          # keep the report when tests fail too: that is when you need it
    expire_in: 1 week
    reports:
      junit: report.xml
  rules:
    - if: $CI_COMMIT_BRANCH
```

Before pushing, **predict**: which of the three jobs will run on this branch?

```bash
git add requirements.txt pytest.ini tests .gitlab-ci.yml
git commit -m "test: cover the /health endpoint and run it in CI"
git push -u origin test/health-endpoint
```

Open the merge request and watch its pipeline. Only `run-unit-tests` runs, because the other two jobs
are restricted to `main`. Open the pipeline (from the merge request's pipeline widget, or left sidebar **Build →
Pipelines**), then its **Tests** tab: the two tests are listed by name, read
from `report.xml`.

Merge. On `main` all three jobs run, and the second one pushes your image.

:::{figure} img/gitlab-pipeline-success.png
:alt: A GitLab pipeline graph in which every stage has completed successfully.
**What a green pipeline looks like.** Every stage completed, so every later stage was allowed to
start. The failed pipeline later in this section looks different: the first failing stage stops
everything behind it.
:::

**Checkpoint 3** — under *Deploy → Container Registry*, the project's image has the tags `latest` and
an 8-character commit SHA, next to the `1.0.0` you pushed on Day 2. A second repository, `…/cache`,
holds layers Kaniko keeps to build faster next time (`--cache=true`): ignore it. Note the SHA down; [section 7](07-fleet.md) deploys it.

```bash
git switch main && git pull
git rev-parse --short=8 HEAD     # the same 8 characters as the image tag
```

:::{dropdown} `ModuleNotFoundError: No module named 'app'`
`pytest.ini` is missing or not at the project root. `pythonpath = .` is what lets `tests/` import
`app.py`; it needs pytest 7 or newer, which `requirements.txt` pins.
:::

:::{dropdown} The pipeline stays *pending* and never starts
No runner picked the job up. Check *Settings → CI/CD → Runners*: instance runners must be enabled for
the project. If none are listed, ask the instructors.
:::

:::{dropdown} `build-container-image` fails with `UNAUTHORIZED` or `denied`
The container registry may be switched off for the project: *Settings → General → Visibility,
project features, permissions → Container registry*. The job itself needs no token from you:
`$CI_REGISTRY_USER` and `$CI_REGISTRY_PASSWORD` are created by GitLab for each job.
:::

(d4-3-6)=
## 3.6 Caching: making it fast

`pip install` on every job, every push, is wasteful. Cache the download directory:

```yaml
variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

cache:
  key:
    files:
      - requirements.txt     # new cache only when dependencies actually change
  paths:
    - .cache/pip
```

Put that at the top level and every job inherits it. Every job still runs `pip install`. With the
cache, pip finds the packages already downloaded; without it, pip downloads them again. Either way
the job passes:

| | `pip install` | Job result |
|---|---|---|
| Cache hit | about 20 s, from `.cache/pip` | passes |
| Cache miss (first run, another runner, cleared cache, changed `requirements.txt`) | about 2 min, from the internet | passes |

That is a correct use of a cache: it changes the time, never the result.

**A cache used as if it were an artifact.** Here one job creates a virtual environment, and the
next job uses it without creating it:

```yaml
cache:
  paths: [.venv/]

install-deps:
  stage: .pre
  script:
    - python -m venv .venv
    - .venv/bin/pip install -r requirements.txt

unit-tests:
  stage: test
  script:
    - .venv/bin/pytest            # assumes .venv/ is already there
```

On a cache hit it works. On a cache miss, `unit-tests` fails with
`.venv/bin/pytest: No such file or directory`.

But `install-deps` runs first and saves `.venv/` to the cache: why would `unit-tests` miss it? Because
the cache is not kept by GitLab. It is kept by the **runner** that ran the job, and only that runner
can give it back. The log of any job on this course's GitLab says so:

```text
No URL provided, cache will not be uploaded to shared cache server. Cache will be stored only locally.
```

"Locally" means on that runner's own disk, and separately for each job slot when a runner runs
several jobs at once. So if `install-deps` and `unit-tests` run on different runners, or in different
slots, `unit-tests` finds an empty cache. Which runner takes a job is not up to you: the same commit
passes or fails depending on where its jobs landed. 

The fix is to make every job correct on an **empty** runner, and to use the cache only to make it
faster: each job installs what it needs, as above. If a job really needs a file that another job
*made*, that file is an **artifact** (3.4), not a cache.

:::{warning} Never rely on the cache for correctness
A pipeline must pass on a completely empty runner. The cache may only make it faster. If a job
passes with a cache hit and fails with a miss, the pipeline uses a cache where it needs an artifact.
:::

**Artifact or cache?** Ask: *if this file disappeared, would I lose a result, or only time?*

| File | Lose a result or time? | Use |
|---|---|---|
| `report.xml`, `coverage.xml`, `trivy-report.txt` | a result | artifact |
| `dist/app.tar.gz`, a file the next job needs | a result | artifact |
| `.cache/pip`, `.venv/`, `node_modules/`, `~/.m2` | only time | cache |

## 3.7 Variables

Three ways to set them, from most to least local:

```yaml
variables:                  # 1. pipeline-wide
  APP_NAME: my-app
  LOG_LEVEL: info

deploy-dev:
  stage: deploy
  variables:                # 2. job-level, overrides the above
    LOG_LEVEL: debug
  script:
    - echo "Deploying $APP_NAME with log level $LOG_LEVEL"
```

3. **In the GitLab UI**, under *Settings → CI/CD → Variables*. This is where anything secret goes —
never in the file. We cover that properly on [section 4](04-pipeline-security.md).

GitLab also predefines a large number of variables. The ones in regular use:

| Variable | Meaning |
|---|---|
| `CI_COMMIT_SHA` | The commit's **SHA** (or *hash*): the 40-character hexadecimal ID Git gives every commit, the one `git log` shows |
| `CI_COMMIT_SHORT_SHA` | Its first 8 characters: the image tag used in this course |
| `CI_COMMIT_REF_NAME` | Branch or tag name |
| `CI_COMMIT_BRANCH` | Branch name (empty on tags) |
| `CI_DEFAULT_BRANCH` | Usually `main` |
| `CI_PROJECT_DIR` | Where your code is checked out |
| `CI_REGISTRY` | Hostname of this GitLab's container registry |
| `CI_REGISTRY_IMAGE` | Full image path for this project |
| `CI_REGISTRY_USER` / `CI_REGISTRY_PASSWORD` | Auto-generated, job-scoped registry credentials |
| `CI_PIPELINE_SOURCE` | Why the pipeline ran: `push`, `merge_request_event`, `schedule`, `web` |

:::{tip} `CI_REGISTRY_PASSWORD` is a short-lived token
GitLab creates it for the job and revokes it when the job ends. You never store it anywhere. This is
the model to aim for with all credentials — see [section 4](04-pipeline-security.md).
:::

(d4-3-8)=
## 3.8 `rules`: running jobs only when they should

This is the part people get wrong most often, so slow down here.

By default every job runs on every pipeline. Usually you do not want that.

```yaml
deploy-prod:
  stage: deploy
  script:
    - ./deploy.sh production
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

`deploy-prod` now runs *only* on `main`.

`rules` is a list evaluated top to bottom; **the first match wins and the rest are ignored.** Each
entry can carry a `when:`:

```yaml
build-image:
  stage: build
  script:
    - ./build.sh
  rules:
    # 1. Never build for documentation-only changes
    - if: $CI_COMMIT_BRANCH
      changes:
        - "**/*.md"
      when: never
    # 2. On main: build automatically
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: on_success
    # 3. On a merge request: build automatically
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: on_success
    # 4. Anywhere else: only if a human clicks the button
    - when: manual
      allow_failure: true
```

The `when:` values you need:

| `when:` | Effect |
|---|---|
| `on_success` | Run if all earlier stages passed (the default) |
| `never` | Do not create this job at all |
| `manual` | Create it, but wait for someone to press ▶ |
| `always` | Run even if an earlier stage failed — for cleanup and notifications |
| `on_failure` | Run *only* if an earlier stage failed |

### Manual jobs: an optional button, or a real gate

`when: manual` creates the job and waits for someone to press ▶. What the **pipeline** does in the
meantime depends on `allow_failure`.

**A real gate** (`allow_failure: false`):

```yaml
deploy-prod:
  stage: deploy
  script: [./deploy.sh production]
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
      allow_failure: false
```

```text
build    ✅  passed
test     ✅  passed
deploy   ⏸️  manual        →  pipeline: blocked
```

The pipeline stops and waits. Its status is *blocked*, not *passed*. Later stages do not run, and a
merge request that needs a successful pipeline cannot merge. It ends only when someone presses ▶ on
`deploy-prod`, or cancels the pipeline. This is a production approval.

**An optional button** (`allow_failure: true`):

```yaml
build-preview:
  stage: deploy
  script: [./build-preview.sh]
  rules:
    - when: manual
      allow_failure: true
```

```text
build    ✅  passed
test     ✅  passed
deploy   ⏸️  manual        →  pipeline: passed
```

The pipeline is green at once. The job stays there as an action anybody may run, or not.

| You want | Write |
|---|---|
| A step a human must approve before anything continues | `when: manual` + `allow_failure: false` |
| A button people may press if they want | `when: manual` + `allow_failure: true` |

:::{important} Get it backwards, and the gate is fake, or the pipeline never ends
- A production deployment with `allow_failure: true`: the pipeline goes green without anybody
  approving. Merge requests merge, and the "gate" gates nothing.
- An optional job with `allow_failure: false`: every pipeline stays *blocked*, waiting for an
  approval nobody intended to give.

**Write `allow_failure` explicitly.** Its default differs: `when: manual` **inside `rules:`**
defaults to `allow_failure: false` (a gate), but `when: manual` set **directly on the job** defaults
to `allow_failure: true` (a button). The same two words, `when: manual`, behave in opposite ways
depending on where you write them.
:::

:::{note} `changes:` is a trap on the first push to a branch
`changes:` compares against the previous commit. On a brand-new branch there may be nothing sensible
to compare with, and GitLab errs on the side of "yes, it changed". Do not use `changes:` as a
*security* control — only as a speed optimisation.
:::

(d4-3-9)=
## 3.9 `needs`: escaping the stage lockstep

Stages are simple but blunt: `build` waits for *all* of `test`, even the slow lint job it does not
care about. `needs:` lets you declare the real dependencies and build a graph instead:

```yaml
stages: [lint, test, build, publish]

lint:        { stage: lint,  script: ["ruff check ."] }
unit-tests:  { stage: test,  script: ["pytest"] }
slow-e2e:    { stage: test,  script: ["./e2e.sh"] }

build-image:
  stage: build
  needs: ["unit-tests"]      # do NOT wait for lint or slow-e2e
  script: ["./build.sh"]

publish:
  stage: publish
  needs: ["build-image", "slow-e2e"]
  script: ["./publish.sh"]
```

```{mermaid}
flowchart LR
    subgraph S1["stage: lint"]
        lint[lint]
    end
    subgraph S2["stage: test"]
        direction TB
        unit[unit-tests]
        e2e[slow-e2e]
    end
    subgraph S3["stage: build"]
        build[build-image]
    end
    subgraph S4["stage: publish"]
        pub[publish]
    end
    unit -->|needs| build
    build -->|needs| pub
    e2e -->|needs| pub
```

Each box is a stage, each arrow a `needs:`. `lint` has no arrow: nothing waits for it, and it does
not wait for anything. `build-image` now starts the moment `unit-tests` goes green. On a real project this routinely halves
the wall-clock time.

:::{tip} `needs` also passes artifacts
A job with `needs:` receives artifacts **only** from the jobs it lists — not from every earlier
stage. If a job suddenly cannot find a file it used to find, a newly-added `needs:` is the usual
culprit.
:::

## 3.10 ⌨️ Hands-on 4: variables, caching, `rules` and `needs` on your pipeline

⏱ **About 15 minutes.**

Your pipeline works, but it repeats the image name in two jobs, downloads pytest on every run, and
leaves the job order to stages alone. Fix all three on a branch.

```bash
cd ~/climate-service
git switch main && git pull
git switch -c ci/tidy-pipeline
```

Replace `.gitlab-ci.yml` with this version. Every change is marked with a comment and maps to one of
sections [3.6](#d4-3-6) to [3.9](#d4-3-9).

`.gitlab-ci.yml`

```yaml
stages:
  - test
  - build
  - scan

variables:
  # 3.7: one definition of "the image this pipeline builds", used by two jobs
  IMAGE: "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"
  PIP_CACHE_DIR: "${CI_PROJECT_DIR}/.cache/pip"

run-unit-tests:
  stage: test
  image: python:3.12-slim-bookworm
  cache:                              # 3.6: keep downloaded packages between pipelines
    key:
      files: [requirements.txt]       # a new cache only when the dependencies change
    paths: [.cache/pip]
  script:
    - pip install -r requirements.txt
    - pytest tests/ --junitxml=report.xml
  artifacts:
    when: always
    expire_in: 1 week
    reports:
      junit: report.xml
  rules:
    - if: $CI_COMMIT_BRANCH

build-container-image:
  stage: build
  needs: [run-unit-tests]             # 3.9: the real dependency, stated
  image:
    name: gcr.io/kaniko-project/executor:v1.23.0-debug
    entrypoint: [""]
  before_script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "${CI_REGISTRY_USER}" "${CI_REGISTRY_PASSWORD}" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json
  script:
    - >-
      /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --destination "${IMAGE}"
      --destination "${CI_REGISTRY_IMAGE}:latest"
      --cache=true
  rules:                              # 3.8: no hard-coded branch name
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

security-audit:
  stage: scan
  needs: [build-container-image]
  image:
    name: aquasec/trivy:0.69.3              # a pinned version: section 4.5 explains why
    entrypoint: [""]
  variables:
    GIT_STRATEGY: none
    TRIVY_USERNAME: $CI_REGISTRY_USER
    TRIVY_PASSWORD: $CI_REGISTRY_PASSWORD
  script:
    - trivy image --no-progress --severity HIGH,CRITICAL "${IMAGE}"
  allow_failure: true
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Before committing, paste the file into *Build → Pipeline editor* and check that GitLab reports
*Pipeline syntax is correct*. A YAML mistake found there costs nothing; the same mistake found by
pushing costs a pipeline.

Then answer, **before** pushing:

1. On this branch, which jobs run?
2. After the merge, on `main`, can `security-audit` start before `run-unit-tests` has finished?
3. In the second pipeline on this branch, what should be faster than in the first?

```bash
git commit -am "ci: name the image once, cache pip, declare job dependencies"
git push -u origin ci/tidy-pipeline
```

Push a second, empty commit to see the cache at work:

```bash
git commit --allow-empty -m "ci: check the pip cache"
git push
```

In the job log of the second pipeline, look for `Successfully extracted cache` near the top. If it says
instead that no cache was found, the job ran on another runner slot, which keeps its own cache
([section 3.6](#d4-3-6)): push once more.

Merge. On `main`, open the pipeline and switch the graph to **Job dependencies**: the arrows are your
`needs:` lines.

:::{dropdown} Answers
**1.** Only `run-unit-tests`. The rules did not change, only how they are written.

**2.** No. `security-audit` needs `build-container-image`, which needs `run-unit-tests`. `needs:` can
only *remove* waiting that is not required; it cannot skip a dependency you declared.

**3.** `pip install`. The packages are restored from the cache instead of downloaded. With one small
dependency the saving is a second or two; on a real project with large dependencies it is minutes.
:::

**Checkpoint 4** — the merged pipeline on `main` is green, the *Job dependencies* view shows
`run-unit-tests → build-container-image → security-audit`, and the registry has a new SHA tag.

## 3.11 Not repeating yourself

### Hidden jobs and `extends`

A job whose name starts with a dot is a **template**: it is never run, only inherited.

```yaml
.python:
  image: python:3.12-slim
  before_script:
    - pip install --quiet -r requirements.txt

unit-tests:
  extends: .python
  stage: test
  script: ["pytest -v"]

type-check:
  extends: .python
  stage: test
  script: ["mypy src/"]
```

`before_script` runs before `script` in every job that inherits it. There is also `after_script`,
which runs even when the job fails — good for cleanup.

### `include`: reusing across projects

```yaml
include:
  # A file from another project in this GitLab
  - project: 'infrastructure/ci-templates'
    ref: main
    file: '/templates/python.gitlab-ci.yml'
  # One of GitLab's built-in templates
  - template: Security/SAST.gitlab-ci.yml
  # A file in this repository
  - local: '/ci/deploy.yml'
```

That second form is how you switch on GitLab's security scanners with one line.

(d4-3-12)=
## 3.12 The real thing: build and push a container image

This is the job you will actually use for the rest of the course. Two ways to do it.

### Option A: Docker-in-Docker

The obvious approach — run a Docker daemon inside the job:

```yaml
build-image:
  stage: build
  image: docker:27
  services:
    - docker:27-dind          # a second container running the Docker daemon
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
    # Also tag as :latest, but only on the default branch
    - |
      if [ "$CI_COMMIT_BRANCH" = "$CI_DEFAULT_BRANCH" ]; then
        docker tag  "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" "$CI_REGISTRY_IMAGE:latest"
        docker push "$CI_REGISTRY_IMAGE:latest"
      fi
```

It works, but it needs a **privileged** runner. A privileged container can essentially
do anything to its host, so a malicious `docker build` can escape onto the runner
machine. Many platforms do not allow it.

### Option B: Kaniko

**Kaniko** builds an image from a `Dockerfile` without a Docker daemon and without privileges:

`.gitlab-ci.yml` — building and pushing with Kaniko

```yaml
build-image:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.23.0-debug
    entrypoint: [""]          # the image's own entrypoint would bypass our script
  script:
    # Hand Kaniko the registry credentials in Docker's config format
    - mkdir -p /kaniko/.docker
    - |
      echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "$CI_REGISTRY_USER" "$CI_REGISTRY_PASSWORD" | base64 | tr -d '\n')\"}}}" \
        > /kaniko/.docker/config.json
    - /kaniko/executor
        --context "$CI_PROJECT_DIR"
        --dockerfile "$CI_PROJECT_DIR/Dockerfile"
        --destination "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
  rules:
    - exists:
        - Dockerfile          # only run where there is something to build
```

This is what stages look like once a pipeline has more than one of them:

:::{figure} img/gitlab-pipeline-failed.png
:alt: A GitLab pipeline graph with a failed lint stage and skipped test and build stages.
**A failed pipeline.** `manifests` failed in the `lint` stage and `ruff` passed. Because
`test` and `build` come after `lint`, none of their jobs ever ran: the double-arrow icon
means *skipped*, not *failed*. One broken job at the front of the pipeline stops everything
behind it, which is exactly what stages are for.
:::

Two details are easy to miss:

- **`entrypoint: [""]`** is mandatory. The Kaniko image's default entrypoint is the executor itself,
  which would ignore your `script:` entirely.
- Only the **`-debug`** tags of the Kaniko image contain a shell, so only those work in GitLab CI.

That second point has a distinctive failure. GitLab runs your `script:` with `sh`, so an image
with no shell in it cannot start the job at all, and the error arrives from the container runtime
rather than from your code:

:::{figure} img/gitlab-job-log.png
:alt: The end of a GitLab job log, showing a container init error because sh was not found.
**Read a job log from the bottom.** Everything above is the runner waiting; the two coloured
lines at the end are the actual result. `exec: "sh": executable file not found in $PATH` means
the job image has no shell, so the job failed before running a single line of your `script:`.
:::

:::{tip} Tag images with the commit SHA, not `latest`
The commit SHA (`$CI_COMMIT_SHORT_SHA`, its first 8 characters, is enough) names exactly one build of
exactly one commit, forever. `latest` means "whatever was
pushed most recently", which makes "what is running in production?" unanswerable and rollbacks
guesswork. Push `latest` as a *convenience alias* if you like, but always push the SHA tag too, and
deploy the SHA. This becomes critical on [section 6](06-gitops.md).
:::

## 3.13 A complete, realistic file

Putting the whole file together:

`.gitlab-ci.yml` — a complete pipeline

```yaml
stages:
  - lint
  - test
  - build
  - deploy

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
  IMAGE: "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"   # the image, tagged with the commit

cache:
  key:
    files: [requirements.txt]
  paths: [.cache/pip]

.python:
  image: python:3.12-slim
  before_script:
    - pip install --quiet -r requirements.txt

# ---------- lint ----------
format:
  extends: .python
  stage: lint
  script:
    - pip install --quiet ruff
    - ruff check .
    - ruff format --check .

# ---------- test ----------
unit-tests:
  extends: .python
  stage: test
  script:
    - pip install --quiet pytest pytest-cov
    - pytest --junitxml=report.xml --cov=app
  artifacts:
    when: always
    expire_in: 1 week
    reports:
      junit: report.xml

# ---------- build ----------
build-image:
  stage: build
  needs: ["unit-tests"]
  image:
    name: gcr.io/kaniko-project/executor:v1.23.0-debug
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    - |
      echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "$CI_REGISTRY_USER" "$CI_REGISTRY_PASSWORD" | base64 | tr -d '\n')\"}}}" \
        > /kaniko/.docker/config.json
    - /kaniko/executor
        --context "$CI_PROJECT_DIR"
        --dockerfile "$CI_PROJECT_DIR/Dockerfile"
        --destination "$IMAGE"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# ---------- deploy ----------
deploy-dev:
  stage: deploy
  needs: ["build-image"]
  image: alpine:3.22
  script:
    - echo "Deploying $IMAGE to dev"
  environment:
    name: dev
    url: https://my-app-dev.internal.meditwin-project.eu
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-prod:
  stage: deploy
  needs: ["deploy-dev"]
  image: alpine:3.22
  script:
    - echo "Deploying $IMAGE to production"
  environment:
    name: production
    url: https://my-app.internal.meditwin-project.eu
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
      allow_failure: false     # a real gate: the pipeline waits for a human
```

:::{note} Jargon: *environment*
`environment:` tells GitLab that this job deploys somewhere real. In return you get a
**Deployments** page showing what is running where, when it got there, and a rollback button. It is
free bookkeeping — always fill it in.
:::

## 3.14 Debugging pipelines

You will spend more time on this than on writing them. In order of usefulness:

**1. Lint the file before pushing.** *Build → Pipeline editor* has a **Validate** tab that catches
YAML and schema errors instantly. Much faster than push-and-wait.

**2. Read the first error, not the last.** Failures cascade; the top of the failure is the cause.

**3. Print the state.** When a variable is not what you expect:

```yaml
script:
  - echo "branch=$CI_COMMIT_BRANCH source=$CI_PIPELINE_SOURCE"
  - ls -la
  - env | sort | grep -v -i -E 'token|password|secret|key'   # never dump raw env
```

**4. Reproduce locally.** The job is just a container and some commands:

```bash
docker run --rm -it -v "$PWD:/src" -w /src python:3.12-slim bash
# now paste your before_script and script lines
```

**5. When a job "does not exist", it is `rules`.** By far the most common confusion. A job absent
from the pipeline was not *skipped*, it was never *created* — some rule evaluated to `never`, or no
rule matched. Read them top to bottom and remember the first match wins.

Common errors and what they mean:

| Message | Cause |
|---|---|
| `yaml invalid` | Indentation. Lists need `- `, and tabs are illegal in YAML. |
| `job: needs 'x' is not defined` | Typo in a `needs:` entry, or the job it names was excluded by its own `rules`. |
| `no such file or directory` in a later stage | The file was never declared as an `artifact`, or a `needs:` excludes the job that made it. |
| `denied: access forbidden` on push | Registry login failed. Check `CI_REGISTRY_USER`/`CI_REGISTRY_PASSWORD` are used and the project's registry is enabled. |
| `error building image: ... no such file` | Kaniko `--context` or `--dockerfile` path is wrong. |
| `This job is stuck` | No runner matches the job's `tags:`. Check *Settings → CI/CD → Runners*. |

## 3.15 Core takeaway

> **A pipeline is a file in the repository: jobs grouped in stages, run by runners, and controlled
> by `rules` and `needs`.**

- One file, `.gitlab-ci.yml`, at the repository root. Commit it and it runs.
- **`stages`** give order; jobs in a stage run in parallel; **`needs`** replaces the lockstep with a
  real dependency graph.
- **`artifacts`** keep results (use `when: always` for test reports); **`cache`** only makes things
  faster.
- **`rules`** decide whether a job exists at all — first match wins. Most missing-job puzzles are
  here.
- **`extends`** and **`include`** avoid repetition, and are how production pipelines are
  structured.
- Build images with **Kaniko** (no privileged runner) and tag them with the commit SHA (**`$CI_COMMIT_SHORT_SHA`**).

## 3.16 Check yourself

1. Write a job that runs the test suite, but only for merge requests, and never when the only
   changed files are Markdown.
2. `build-image` has `needs: ["unit-tests"]`. It fails with `report.xml: no such file`, even though
   `unit-tests` produced it. Why, and what are two fixes?
3. Why is `--destination "$IMAGE:latest"` alone a bad idea, when it makes deployment configuration so
   much simpler?

:::{dropdown} Solutions
**1.**
```yaml
unit-tests:
  stage: test
  image: python:3.12-slim
  script:
    - pip install -r requirements.txt pytest
    - pytest -v
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - "**/*.md"
      when: never
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```
Order matters: the exclusion must come **first**, because the first matching rule wins. Swap the two
and the docs-only exclusion is dead code.

**2.** Because `unit-tests` declared `report.xml` only under `reports:`, not under `paths:`.
`reports:` hands the file to GitLab's UI for parsing; it does not reliably pass it to other *jobs*.
Fixes: (a) add it to `paths:` as well —
```yaml
  artifacts:
    paths: [report.xml]
    reports:
      junit: report.xml
```
or (b) if `build-image` does not actually need the report, stop asking for it. A third, subtler cause
worth knowing: adding `needs:` to a job **restricts** it to artifacts from the listed jobs only, so a
job that previously received everything from earlier stages now receives almost nothing.

**3.** Because `latest` is a *moving* label, and that breaks three things at once. **Reproducibility:**
"production runs `latest`" does not identify any particular code, so you cannot recreate a bug
report. **Rollback:** to go back you must rebuild and re-push an older commit, racing whoever pushes
next — whereas with SHA tags you just point at the previous SHA. **GitOps:** as you will see on
[section 6](06-gitops.md), the whole model rests on a Git commit describing an exact desired state. If the manifest says
`latest`, two identical commits can produce different running software, and drift detection cannot
tell whether the cluster is correct. Push `latest` as an alias for humans if you want; deploy the SHA.
:::

---

Continue with: [4. Security in pipelines](04-pipeline-security.md).
