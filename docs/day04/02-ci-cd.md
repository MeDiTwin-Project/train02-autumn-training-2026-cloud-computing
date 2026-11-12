# 2. What CI/CD actually is

The goal here is that when you see a pipeline file, you already know
what it is trying to achieve.

**Slides:** the slides used in class are in one file, {download}`day04-git-cicd-gitops.pdf`.

## 2.1 "It works on my laptop"

You wrote code. It runs. You push it. Your colleague pulls it and it explodes.

Why? Pick any of these — all of them happen constantly:

- You have Python 3.12, they have 3.9.
- You installed a library months ago and forgot to add it to `requirements.txt`.
- You have a file on your Desktop that the code reads. They do not.
- You have an environment variable set in your shell profile. They do not.
- The test you "ran" was the one you ran three days ago, before the last four changes.

Every one of these is the same bug: **the only proof that the code works is that it worked once, on
one machine, that nobody else can inspect.**

The fix is not discipline. The fix is to make a *clean, identical, disposable* machine build and test
the code every single time, automatically. That is Continuous Integration.

## 2.2 The two halves

:::{note} Jargon: *CI* — Continuous Integration
Every time anybody pushes a change, an automated system fetches it into a **fresh** environment,
builds it, and runs the tests. The word "integration" is about merging everyone's work together
often — daily, not quarterly — so problems surface while they are still small.
:::

:::{note} Jargon: *CD* — Continuous Delivery / Deployment
**Delivery**: after CI passes, the system automatically produces the finished, deployable artifact
(a container image, a package) and puts it somewhere ready to go. A human decides when to release.

**Deployment**: the same, but the release happens automatically too, with no human in the loop.

People say "CD" for both and rarely bother to distinguish. When it matters, ask which one they mean.
:::

```{mermaid}
flowchart TB
    subgraph CI["Continuous Integration"]
        A[Fetch the code] --> B[Install dependencies]
        B --> C[Run tests]
        C --> D[Build the artifact]
    end
    subgraph CD["Continuous Delivery"]
        E[Publish the artifact] --> F[Deploy it]
    end
    D --> E
```

(d4-2-3)=
## 2.3 The parts, with their real names

You need five words. Learn them in this order.

### Runner

A **runner** is the machine (usually a container) that executes your automation. It is created empty,
does the work, and is thrown away. That disposability is the entire point: nothing can silently carry
over from the last run.

:::{important} This is why CI catches "works on my laptop"
Your laptop has years of accumulated state. A runner has none. If your code needs something, the
code must *say so* — in `requirements.txt`, in a `Dockerfile`, in the pipeline file — or it will
fail. Being forced to declare your dependencies is a feature, not an annoyance.
:::

### Job

A **job** is one unit of work with a name and a list of shell commands: "run the unit tests", "build
the image", "check the formatting". Each job runs on a fresh runner.

### Stage

A **stage** is a named group of jobs that run **at the same time as each other**, and stages run
**one after another**. Classic set-up:

```{mermaid}
flowchart TB
    subgraph s1["Stage: test"]
        t1[Unit tests]
        t2[Linting]
        t3[Type checks]
    end
    subgraph s2["Stage: build"]
        b1[Build the image]
    end
    subgraph s3["Stage: deploy"]
        d1[Deploy]
    end
    s1 --> s2 --> s3
```

The three test jobs run in parallel — three runners at once, so the stage takes as long as the
slowest job, not the sum. Nothing in `build` starts until *every* job in `test` has passed. That
ordering is the safety property: a failing test cannot produce an image.

### Pipeline

A **pipeline** is the whole run: all the stages, for one specific commit. Each push creates a new
pipeline. It ends green ✅ or red ❌.

### Artifact

An **artifact** is a file a job produces that needs to outlive it — a test report, a coverage
summary, a built package. Because the runner is destroyed, anything you want to keep or hand to a
later job must be declared as an artifact.

:::{note} Jargon: *cache* vs *artifact*
Both save files from a runner that is about to be destroyed, for opposite reasons.

- **Artifact**: "this file is a *result* of my job." A test report, a coverage file, a built
  package. GitLab stores it: you can download it, and later jobs receive it.
- **Cache**: "this folder is *slow to recreate*, but it is not a result." Downloaded libraries,
  such as `.venv/` or `node_modules/`. It only saves time, and it may or may not be there.

To decide, ask: *if this file disappeared, would I lose a result, or only time?* A result: artifact.
Only time: cache. [Section 3](03-gitlab-ci.md) shows both on a real pipeline ([3.4](#d4-3-4) and [3.6](#d4-3-6)), including how a cache
misused as an artifact breaks a pipeline.
:::

## 2.4 What a pipeline usually does, in order

A realistic sequence for a containerised application:

| Stage | Job | Question it answers |
|---|---|---|
| `lint` | format, style | Is it written consistently? A **linter** checks this without running the code |
| `test` | unit tests | Does the logic do what we claimed? |
| `test` | secret detection | Did anyone commit a token? |
| `test` | dependency scan | Are our libraries known-vulnerable? |
| `build` | build image | Does it actually assemble? |
| `scan` | container scan | Is the resulting image vulnerable? |
| `publish` | push image | Make it available to the cluster |
| `deploy` | deploy to dev | Does it run for real? |
| `deploy` | deploy to prod (manual) | Ship it — when a human says so |

Two principles worth stating explicitly:

**Cheap and likely-to-fail things go first.** Linting takes 5 seconds; an integration test takes 5
minutes. Fail fast on the cheap check so nobody waits for the expensive one to tell them about a
missing comma.

**Every stage is a gate.** The point is not to run lots of jobs; it is that *nothing progresses past
a red gate*. A pipeline where failures are routinely ignored has no value at all.

(d4-2-5)=
## 2.5 Push-based CD, and its problem

Look carefully at that last stage. `deploy to prod` means the runner needs to be able to change
production. Concretely, for Kubernetes, the runner needs cluster credentials so it can run
`kubectl apply`.

```{mermaid}
flowchart TB
    Git[(GitLab)] --> CI[CI runner]
    CI -->|holds cluster credentials| K8s[(Cluster)]
    style CI fill:#ffe6e6,stroke:#c00,color:#5c0a0a
```

This works, and it is what many teams do. But notice what you have built:

1. **The cluster's admin credentials live in your CI system.** Anyone who can edit the pipeline file
   — or exploit any job in it — can reach production. And a pipeline file is editable by anyone who
   can open a merge request.
2. **Nothing watches the cluster afterwards.** If somebody runs `kubectl edit deployment` by hand,
   the cluster differs from the repository and nothing reports it. Git states one thing,
   reality says another.
3. **You cannot easily answer "what is running right now?"** You can only answer "what did the last
   green pipeline push". Those diverge the first time a deploy half-fails.
4. **A rollback leaves things behind.** You roll back the way you deploy: `git revert`, and the
   pipeline applies the previous manifests. But `kubectl apply` only creates and updates: an object
   that the bad commit *added* is still in the cluster afterwards, and nothing tells you (point 2).

:::{note} Jargon: *configuration drift*
**Drift** is the gap that opens between what your repository says should be running and what is
actually running. Manual fixes, half-failed deployments and expired resources all cause it. Drift is
the thing GitOps is designed to eliminate.
:::

Hold that thought. On [section 6](06-gitops.md) we invert the arrow: instead of CI pushing into the
cluster, an agent *inside* the cluster pulls from Git. That change fixes all four problems at once.

First, though, [section 3](03-gitlab-ci.md) builds the CI half, which is required either
way. CI and GitOps are not alternatives: CI tests and builds the image, GitOps deploys
it.

## 2.6 Small investigative exercise

⏱ **About 5 minutes.**

No typing yet: this one is about reading. Below is the pipeline your climate telemetry service will
use from [section 3](03-gitlab-ci.md) on. It tests the code, builds the image, pushes it to the project's container
registry and scans it. Do not worry about the details of each line yet; [section 3](03-gitlab-ci.md) explains them,
and the scanner it uses, **Trivy**, which looks for known vulnerabilities in the image, is
[section 4.5](#d4-4-5).

`.gitlab-ci.yml` — the climate service's pipeline

```yaml
stages:
  - test
  - build
  - scan

# Stage 1: run the unit tests
run-unit-tests:
  stage: test
  image: python:3.12-slim-bookworm
  script:
    - pip install --no-cache-dir -r requirements.txt
    - pytest tests/
  rules:
    - if: $CI_COMMIT_BRANCH

# Stage 2: build the image and push it to the project's registry
build-container-image:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.23.0-debug
    entrypoint: [""]
  before_script:
    # Log Kaniko in to the GitLab registry with the job's own credentials
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "${CI_REGISTRY_USER}" "${CI_REGISTRY_PASSWORD}" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json
  script:
    - >-
      /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"
      --destination "${CI_REGISTRY_IMAGE}:latest"
      --cache=true
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# Stage 3: scan the image that was just pushed
security-audit:
  stage: scan
  image:
    name: aquasec/trivy:0.69.3
    entrypoint: [""]
  variables:
    GIT_STRATEGY: none
    # The registry is private: Trivy logs in with the job's own credentials,
    # exactly as Kaniko did to push.
    TRIVY_USERNAME: $CI_REGISTRY_USER
    TRIVY_PASSWORD: $CI_REGISTRY_PASSWORD
  script:
    - trivy image --no-progress --severity HIGH,CRITICAL "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"
  allow_failure: true
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

Using only the vocabulary from [section 2.3](#d4-2-3), answer:

1. How many **stages** are there, and how many **jobs**? Which job belongs to which stage?
2. Which container image does each job run in? What does that tell you about where the **runner**
   executes the commands?
3. Which job produces something that outlives the pipeline? Is it a GitLab **artifact**, or
   something else?
4. On a push to a branch other than `main`, which jobs run?
5. If you wanted this pipeline to *deploy* as well, push-style, what would the new job need to
   hold, and why is that a problem?

:::{dropdown} Answers
**1.** Three stages (`test`, `build`, `scan`) and three jobs, one per stage: `run-unit-tests`,
`build-container-image`, `security-audit`.

**2.** `python:3.12-slim-bookworm`, the Kaniko executor image, and the Trivy image. Each job runs in a
fresh container started from its own image, so a job only has the tools that image ships. The runner
is the machine that starts those containers; the job does not run "on GitLab".

**3.** `build-container-image`. Its output is a container image pushed to the registry, which is not a
GitLab `artifacts:` entry: it is stored in the registry and kept until somebody deletes it. In the
broader sense of [section 2.3](#d4-2-3) it is the most important artifact the pipeline makes.

**4.** Only `run-unit-tests`. The other two have `rules: - if: $CI_COMMIT_BRANCH == "main"`, so a
branch gets its tests checked without publishing an image.

**5.** Credentials that can change the cluster — a kubeconfig or a token with write access. Every
job that can read that variable, and every person who can change `.gitlab-ci.yml`, can then change
production. [Section 2.5](#d4-2-5) calls this the push-based problem, and [section 6](06-gitops.md) removes it.
:::

## 2.7 Core takeaway

> **CI checks every change automatically, on a machine nobody configured by hand; CD delivers it.
> Delivering by pushing from the pipeline works, but puts cluster credentials in the pipeline.**

- CI exists because "it worked on my machine" is not evidence. A **fresh runner** every time is.
- **Job** = commands. **Stage** = jobs in parallel. **Pipeline** = all stages for one commit.
- Stages are **gates**: nothing proceeds past a failure. Cheaper checks first.
- **Artifacts** are results you keep; **caches** are just speed. Never depend on a cache for
  correctness.
- Push-based CD hands cluster credentials to CI and does not notice drift. Remember these two
  problems.

## 2.8 Check yourself

1. Your pipeline has a `test` stage with 4 jobs (30s, 45s, 2min, 40s) and a `build` stage with 1 job
   (3min). Roughly how long is the pipeline, and why?
2. A colleague says: "the tests pass locally, the CI failure must be a CI problem." What is the most
   likely actual explanation, and how would you check?
3. Why is it a problem that a push-based deploy job holds cluster admin credentials, given that only
   trusted colleagues work on the repository?

:::{dropdown} Solutions
**1.** About **5 minutes**. The four test jobs run in parallel, so the `test` stage takes as long as
its slowest job (2 min), then `build` runs afterwards (3 min). 2 + 3 = 5. Note what this tells you:
adding a fifth 30-second test job costs you nothing, but making the 2-minute job faster helps
everyone.

**2.** Almost always: **an undeclared dependency**. Something exists on their laptop that is not in
the repository — an installed package missing from `requirements.txt`, a file, an environment
variable, a service running on `localhost`, or a different language version. Read the CI log for the
first error rather than the last, and try reproducing in a clean container:
```bash
docker run --rm -it -v "$PWD:/src" -w /src python:3.12 bash
# then run the install and test commands by hand
```
If it fails there too, it was never a CI problem. (Second most likely: the tests depend on each
other and only pass in a particular order, which parallel CI exposes.)

**3.** Because the credential is available to **code**, not to people. Every job in the pipeline runs
with access to the CI variables, so the blast radius includes: any merge request that edits the
pipeline file (a `script:` line can just print the secret); any third-party dependency or CI image
your jobs pull, which now executes in a context holding production credentials; and any bug that
leaks a job log. Trust in colleagues is not the relevant question — the question is how many
*machines and packages* you have implicitly trusted with production. GitOps removes the credential
from CI entirely, so there is nothing to steal.
:::

---

Continue with: [3. GitLab CI hands-on](03-gitlab-ci.md).
