# Day 4 - Git, CI/CD and GitOps

Welcome to Day 4. Today is about **automation**: how a change you type on your laptop ends up
running in a real Kubernetes cluster, safely, without anybody logging into a server.

## The one story this day tells

Everything today hangs off one sentence:

> **A change is committed to Git. An automated system detects it, checks it, builds it, and
> deploys it. If the change is faulty, reverting the commit restores the previous state.**

```{mermaid}
flowchart TB
    You([You]) -->|1. push| Git[(Git repository)]
    Git -->|2. triggers| CI[CI pipeline]
    CI -->|3. pushes the image| Reg[(Container registry)]
    Git -->|4. watched by| Fleet[GitOps agent]
    Fleet -->|5. applies| K8s[(Kubernetes cluster)]
    Reg -.->|image pulled by| K8s
```

Steps 1–3 are **CI/CD**. Steps 4–5 are **GitOps**. They are different things, and one of today's
goals is that you never confuse them again.

You do all of it on one application: the **climate telemetry service** you containerised on Day 2.
Each section explains one part of the chain and then has you build that part, so by the end of the
day the service is tested, scanned, released and deployed by merging.

## What you will be able to do by the end

- Use Git confidently for everyday work: commit, branch, push, open a merge request, fix a conflict.
- Read and write a `.gitlab-ci.yml` file, and explain what each part does.
- Build and publish a container image from a pipeline, automatically.
- Keep secrets out of your repository, and make scanners fail the pipeline before a reviewer has to.
- Let your commit messages produce version numbers, changelogs and releases.
- Explain what GitOps is, and why it is not the same as "my pipeline runs `kubectl apply`".
- Deploy an application to Kubernetes with **Rancher Fleet** by merging a Git change, and roll it
  back with `git revert`.
- Choose between plain manifests and a **Helm chart** for a Fleet deployment — or, on the advanced
  path, generate projects and charts with **Cookiecutter** and run pipeline steps locally with
  **Dagger**.

**Focus for the day**: every change reaches the cluster through Git, and only through Git.

## Prerequisites

You need what you built on Days 2 and 3:

- The `~/climate-service` project from [Day 2, section 5](../day02/05-application-containerisation.md):
  `app.py` and its `Dockerfile`.
- The GitLab project `training/autumn-2026/climate-telemetry-traineeNN`, created for you and used in
  [Day 2, section 5.7](../day02/05-application-containerisation.md), with your image in its registry.
- The Ingress, TLS and Helm basics of Day 3.

Your VM is already set up with the rest:

| What | Where | Log in with |
|---|---|---|
| GitLab account `traineeNN`, Maintainer of your three projects in **training/autumn-2026** (`climate-telemetry-traineeNN`, `…-deploy`, `hello-ci-traineeNN`); other trainees' projects are invisible to you | `https://gitlab.data.meditwin-project.eu` | `traineeNN` + your claim code |
| Rancher account `traineeNN`, for Fleet's *Continuous Delivery* | `https://rancher-training.apps.meditwin-project.eu` | `traineeNN` + your claim code |
| Your namespace `traineeNN` on the training cluster, as `kubectl`'s default | `~/.kube/config` | — |
| Your SSH key registered on your GitLab account | `~/.ssh/gitlab_training` | — |
| `git`, `docker`, `kubectl`, `helm`, `jq`, `trivy`, `dagger`, `cookiecutter` | your `PATH` | — |

Quick checks:

```bash
ls ~/climate-service                          # app.py  Dockerfile  requirements.txt
kubectl get pods                              # "No resources found in traineeNN namespace."
ssh -T git@gitlab.data.meditwin-project.eu    # "Welcome to GitLab, @traineeNN!"
docker --version && helm version --short && trivy --version | head -1
```

The values the hands-on sections use throughout:

| Value | Yours |
|---|---|
| App project | `training/autumn-2026/climate-telemetry-traineeNN` |
| Config project | `training/autumn-2026/climate-telemetry-traineeNN-deploy` |
| Namespace | `traineeNN` |
| Hostname | `climate-traineeNN.training.meditwin-project.eu` |
| Ingress class / certificate issuer | `traefik` / `ecmwf-meditwin-training-issuer` |

## How today works

Every hands-on and type-along section is marked ⌨️ **your turn**: you
type every command yourself, on your training VM, in your GitLab projects, in Rancher and in your
namespace. The two short **investigative exercises** (2.6 and 6.8) have no mark: you read and
discuss, you do not type.

The slides used in class, with the theory of each section, are in one file: {download}`day04-git-cicd-gitops.pdf`.

:::{tip} Every command can be run directly
If something does not work, that is a bug in the material or in the environment: say so. Do not
assume you did it wrong.
:::

## Sections

| # | Section | What it covers |
|---|---|---|
| 1 | [Git from zero](01-git.md) | Commits, branches, remotes, merge requests |
| 2 | [What CI/CD actually is](02-ci-cd.md) | The mental model, before any YAML |
| 3 | [GitLab CI hands-on](03-gitlab-ci.md) | Writing real pipelines: stages, artifacts, rules, needs, Kaniko |
| 4 | [Security in pipelines](04-pipeline-security.md) | Secrets, scanning, Trivy as a gate, supply chain |
| 5 | [Conventional commits and automated releases](05-releases.md) | Commit messages that produce versions and changelogs |
| 6 | [What GitOps is](06-gitops.md) | Why "push to deploy" is not the best idea |
| 7 | [Rancher Fleet](07-fleet.md) | Deploying, reconciling and rolling back by merging |
| 8 | [Fleet with plain manifests or Helm charts](08-fleet-manifests-vs-helm.md) **or** | Local charts, values per deployment, upstream charts through Fleet |
| 8B | [Cookiecutter and Dagger (advanced path)](08b-cookiecutter-and-dagger.md) | Templates for projects and charts; pipeline steps you run locally and on GitLab's Dagger runner |
| 9 | [Day 4 cheat sheet](09-cheatsheet.md) | Commands and troubleshooting |

Sections [1](01-git.md) to [8](08-fleet-manifests-vs-helm.md) each alternate explanation and hands-on, and each hands-on adds one link to the
chain: by the end of [section 7](07-fleet.md) your service goes from a commit to the cluster, and back with
`git revert`. For [section 8](08-fleet-manifests-vs-helm.md), pick **one**: 8 goes deeper into Helm with Fleet, building on Day 3;
8B, the advanced path, is for people who already look after several services and pipelines.

## Clicking in GitLab and Rancher

The material writes a click path as *A → B → C*. The first time a section asks you to click
somewhere in GitLab or Rancher, it shows a screenshot with numbered red marks, and the numbers in the
text under it say what each mark is. GitLab is version 19, Rancher version 2.13.

## A word on jargon

This field uses a large amount of vocabulary, most of which is simpler than it sounds.
Whenever a new word appears today it is defined in a box like this one:

:::{note} Jargon: *repository*
A **repository** (or *repo*) is a folder whose entire history Git records. Nothing
more exotic than that.
:::

If a word gets used without a box, that is an oversight: ask.
