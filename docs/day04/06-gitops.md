# 6. What GitOps is

In [section 2](02-ci-cd.md) we described a deployment job that runs `kubectl apply` with cluster credentials,
and listed four problems with it. This section fixes all four with one structural change.

**Slides:** the slides used in class are in one file, {download}`day04-git-cicd-gitops.pdf`.

## 6.1 The change: turn the arrow around

**Push-based CD** — what we had:

```{mermaid}
flowchart TB
    Git[(GitLab)] --> CI[CI runner]
    CI -->|holds cluster credentials| K8s[(Cluster)]
    style CI fill:#ffe6e6,stroke:#c00,color:#5c0a0a
```

**GitOps** — pull-based:

```{mermaid}
flowchart TB
    Git[(GitLab)]
    subgraph cluster["Kubernetes cluster"]
        Agent[GitOps agent] --> Res[Your resources]
    end
    Agent -->|reads every minute| Git
    style Agent fill:#e6ffe6,stroke:#0a0,color:#0a3320
```

An agent runs **inside** the cluster. It reads your Git repository on a loop, compares what Git says
to what is actually running, and fixes any difference.

Nothing outside the cluster has credentials to it. The credential flow is reversed: instead of CI
holding the keys to the cluster, the cluster holds a read-only key to Git.

(d4-6-2)=
## 6.2 The four properties

GitOps is usually defined by four principles. They are worth taking one at a time, because each does
real work.

### 1. Declarative

Git holds the **end state** you want, not the steps to get there.

Take a deployment that should run 3 replicas. There are two ways to write that down:

- **The end state:** "there are 3 replicas".

  ```yaml
  spec:
    replicas: 3
  ```

- **A step:** "add one replica", starting from however many there are now.

  ```bash
  kubectl scale deployment my-app \
    --replicas=$(( $(kubectl get deploy my-app -o jsonpath='{.spec.replicas}') + 1 ))
  ```

Now imagine the agent applies each of them every minute, starting from 2 replicas:

| Minute | The end state (`replicas: 3`) | The step (+1) |
|---|---|---|
| 1 | 3 | 3 |
| 2 | 3 | 4 |
| 3 | 3 | 5 |

The end state can be applied a thousand times and the result is always 3. If somebody changes the
number by hand, the next application puts it back. The step adds one more replica each time. An agent
that runs every minute can only work with the first kind.

:::{note} Jargon: *declarative*, *imperative*, *idempotent*
**Declarative**: a description of the end state, kept in a file ("there are 3 replicas"). It can be
applied again at any time. Kubernetes manifests are declarative.

**Imperative**: a command that acts once. `kubectl scale deployment my-app --replicas=3` is
imperative too, even though it names the end number: once it has run, nothing records that anybody
wanted 3, so there is nothing to apply again ([section 6.8](#d4-6-8) shows what follows).

An operation is **idempotent** if doing it many times has the same effect as doing it once. "Set the
temperature to 20°" is idempotent; "turn the dial up two notches" is not. Applying an end state is
idempotent, which is what lets GitOps re-apply Git every minute.
:::

### 2. Versioned and immutable

The desired state lives in Git. So you get, for free and without designing anything:

- Every change ever made, with author, timestamp and reason.
- Code review on infrastructure changes, via merge requests.
- The exact state of production at any past moment (`git checkout` that commit).
- An answer to "who changed this and why" that is not "let me ask around".

### 3. Pulled automatically

The agent pulls. Nobody pushes. This is the security win: **no external system needs cluster
credentials.**

### 4. Continuously reconciled

This is the one people miss, and it is the most valuable.

The agent does not just apply once when a commit lands. It keeps checking — every minute, forever —
and corrects anything that has drifted.

```bash
# Someone changes production by hand
kubectl scale deployment my-app --replicas=10
```

A minute later the agent notices the cluster says 10 and Git says 3, and puts it back to 3.

:::{important} Reconciliation is the actual product
A pipeline that runs `kubectl apply` gives you *convergence at deploy time*. GitOps gives you
*convergence all the time*. The difference is that with GitOps, "what does Git say?" is always a
truthful answer to "what is running?" — which means Git is a real inventory, not a hopeful record.

This is also why manual `kubectl edit` stops being a thing you do. It is not forbidden by policy; it
has no lasting effect. The agent reverts it. To change the cluster, change Git.
:::

## 6.3 The four problems, solved

| Problem from [section 2](02-ci-cd.md) | How GitOps solves it |
|---|---|
| CI holds cluster admin credentials | It no longer needs any. The agent is inside, and only reads Git. |
| Nothing notices manual changes | Reconciliation reverts them within a minute. |
| Cannot answer "what is running?" | Git is the answer, and it is enforced. |
| A rollback leaves objects behind | The agent applies the whole folder: what is no longer in Git is deleted. |

That last row deserves emphasis.

## 6.4 Rollback is `git revert`

Deployment went wrong. The command is the same in both models:

```bash
git revert HEAD          # make a new commit that undoes the bad one
git push
```

What differs is what happens next. In the push model, the revert goes through the whole pipeline
(tests, build, scan) before its deploy job applies the old manifests, and anything the bad commit
added stays in the cluster. In GitOps the agent sees the new commit and, within a minute, makes the
cluster match it: the old manifests are back, and objects that are no longer in Git are deleted.
And the rollback is itself a reviewable commit in the history, so *the recovery is documented
too*.

This is why [section 1](01-git.md#undoing-things) insisted on the difference between `revert` and
`reset`. `reset` rewrites history and needs a force-push; `revert` moves forward. GitOps rollbacks are
`revert`.

(gitops-is-not)=
## 6.5 What GitOps is *not*

Three common misconceptions:

**"We have GitOps, our pipeline runs `kubectl apply`."** No — that is push-based CD with extra steps.
The test is: *if someone edits the cluster by hand, does anything notice?* If no, it is not GitOps.

**"GitOps replaces CI."** No. They do different halves and you need both:

```{mermaid}
flowchart TB
    subgraph ci["CI builds the artifact"]
        A[Test] --> B[Build the image]
        B --> C[Push to the registry]
    end
    subgraph gitops["GitOps runs the artifact"]
        D[Read Git] --> E[Apply the manifests]
        E --> F[Reconcile continuously]
    end
    C -->|image tag| D
```

CI produces an artifact and proves it works. GitOps decides what runs. The handover is the image tag.

**"GitOps means one repository."** No, and in practice you usually want two:

| Repository | Contains | Who changes it |
|---|---|---|
| **Application repo** | Source code, `Dockerfile`, `.gitlab-ci.yml` | Developers, constantly |
| **Config repo** | Manifests, Helm values, `fleet.yaml` (Fleet's settings for a folder, section 7) | Anyone deploying; reviewed carefully |

Why separate them? Because a change to a comment in the README should not trigger a reconciliation,
and because "who may change production" is a different list of people from "who may change the code".
It also stops the loop where CI commits to the repo that CI watches.

:::{note} You can do it in one repo, and it is fine to start there
For a small project, one repository is simpler and you will not notice the difference at first. The
hands-on sections today use two — `climate-telemetry-traineeNN` for the code and
`climate-telemetry-traineeNN-deploy` for what runs — so that you see the split working before you
decide whether you need it.
:::

(image-tag-updates)=
## 6.6 How the image tag gets updated

This is the join between the two halves, and the most intricate part of GitOps.

CI builds `my-app:abc123`. Something must change the manifest in the config repo from `abc123` to the
new SHA. Three options, all real:

**1. CI commits the new tag.** The pipeline edits the manifest and pushes.

```yaml
update-manifest:
  stage: deploy
  script:
    - git clone https://…/config-repo.git && cd config-repo
    # Quoted: an unquoted line containing ": " would be read as a YAML key.
    - 'sed -i "s|image: .*|image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA|" dev/deployment.yaml'
    - 'git commit -am "chore: deploy $CI_COMMIT_SHORT_SHA" && git push'
```

Simple and explicit. It needs a token that can write to the config repo, and nothing more: the push
goes to **another** repository, so it does not start the application's pipeline again.

The loop only exists with **one** repository for both code and manifests. Then the pipeline pushes to
the repository it runs in, the push starts a new pipeline, which builds, bumps the tag and pushes
again, for ever. With one repository you need a guard: push with `git push -o ci.skip`, or put
`[skip ci]` in the commit message. Two repositories avoid the problem by design, which is one more
reason for the split.

**2. The agent watches the registry.** GitOps systems have controllers that poll for new image tags matching a
policy and commit the bump themselves. Powerful; more machinery.

**3. Nothing — deploy by digest, manually.** Perfectly reasonable for production: a human decides
when, in a reviewed merge request. "Manual" here does not mean "worse".

## 6.7 The complete picture

```{mermaid}
%%{init: {"sequence": {"actorMargin": 12, "width": 130, "diagramMarginX": 12}}}%%
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant CI as GitLab CI
    participant Git as Config repo
    participant Agent as Agent in cluster

    Dev->>CI: Merge to main
    CI->>CI: Test, scan, build
    CI->>Git: Commit the new tag
    Agent->>Git: Poll every minute
    Agent->>Agent: Apply that state
    loop Continuously
        Agent->>Git: Compare with the cluster
    end
```

Follow the credentials in that diagram, because it is the whole argument:

- CI can write to the **registry** and to the **config repo**. It cannot touch the cluster.
- The agent can read the **config repo**. It cannot write anything to Git.
- Nobody outside the cluster has cluster credentials.

Compromise the CI runner and you can publish a bad image and propose a bad commit — both of which
leave a reviewable trail — but you cannot reach production directly.

(d4-6-8)=
## 6.8 Small investigative exercise

⏱ **About 7 minutes.**

No typing: this one is about reasoning. Discuss it with your neighbour.

**The situation.** An application, `web`, runs on a cluster with **3 replicas**. Below are five
different ways that `web` could have been set to 3 replicas, all of which you have used or seen on
Day 3 or today. In each case the cluster has only what the row describes: in rows 1 to 4 there is
**no** GitOps agent; in row 5 there is Fleet, with drift correction on, as you will set it up in
[section 7](07-fleet.md).

1. Somebody ran `kubectl scale deployment web --replicas=3`.
2. Somebody ran `kubectl apply -f web.yaml` from their laptop. `web.yaml` says `replicas: 3`.
3. Somebody ran `helm upgrade --install web ./chart -f values.yaml` from their laptop.
   `values.yaml` says `replicaCount: 3`.
4. A CI pipeline runs `kubectl apply -f k8s/` after every merge to `main`. `k8s/web.yaml` in Git
   says `replicas: 3`.
5. A Fleet `GitRepo` points at a folder in Git whose `web.yaml` says `replicas: 3`.

**Then**, an hour later, a colleague changes the cluster by hand:
`kubectl scale deployment web --replicas=1`.

**For each row, answer three questions:**

- **a.** Is the *content* that set 3 replicas **declarative** (a description of the desired state) or
  **imperative** (a command that does one thing)?
- **b.** **Who or what** changes the cluster, and **when**: a person, once? a pipeline, when something
  is merged? an agent, all the time?
- **c.** After the colleague's change, **how many replicas** does `web` have, and **until when**?
  Does anything bring it back to 3 by itself?

:::{dropdown} Answers
| # | a. Content | b. Who changes the cluster, and when | c. After `--replicas=1` |
|---|---|---|---|
| 1 | Imperative: "make it 3" | A person, once | **1, for ever.** Nothing records that anybody wanted 3 |
| 2 | Declarative file | A person, once, from a laptop | **1**, until somebody runs `kubectl apply -f web.yaml` again |
| 3 | Declarative (chart + values) | A person, once, from a laptop | **1**, until the next `helm upgrade` |
| 4 | Declarative, and in Git | The pipeline, push-based, only when something is merged | **1**, until the next merge to `main`, which may be days away |
| 5 | Declarative, and in Git | The Fleet agent, pulling, all the time | **3 again within seconds**: Fleet sees the drift and corrects it. Without drift correction, Fleet only reports the deployment as *Modified* (section 7 names Fleet's objects) |

The point is in the middle column. Rows 2 to 4 are declarative *content* delivered by a one-off
*push*: the file in Git describes what you wanted at the moment somebody ran the command. Only row 5
keeps checking. That is property 4 of [section 6.2](#d4-6-2), continuous reconciliation, and it is what
separates GitOps from "we keep our YAML in Git".
:::

## 6.9 Core takeaway

> **In GitOps the cluster pulls its desired state from Git and keeps converging to it: a deployment
> is a commit, and a rollback is a revert.**

- GitOps = **declarative** state, **versioned** in Git, **pulled** by an in-cluster agent,
  **continuously reconciled**.
- The arrow points *from* the cluster *to* Git. No external system holds cluster credentials.
- **Reconciliation is the point.** Without it you have push-based CD with a Git flavour.
- **Rollback is `git revert`.**
- CI and GitOps are complementary: CI builds and proves the image, GitOps decides what runs. The
  handover is the **immutable image tag** — never `latest`.
- Two repositories (app + config) is the usual shape, for review and blast-radius reasons.

## 6.10 Check yourself

1. A colleague says: "our GitLab pipeline runs `helm upgrade` on merge to `main`, so we do GitOps."
   What single question settles it, and what would each answer mean?
2. Your manifest says `image: my-app:latest`. Someone reports a bug in production. Name three things
   you now cannot do.
3. With reconciliation running every minute, how do you legitimately make an emergency change during
   an incident?

:::{dropdown} Solutions
**1.** Ask: **"If I edit a Deployment by hand with `kubectl`, does anything change it back?"**
- *No* → push-based CD. The cluster converges only at deploy time, drift accumulates silently, and the
  pipeline holds cluster credentials.
- *Yes, within a minute, without anyone re-running the pipeline* → GitOps.

A useful follow-up: "where are the cluster credentials stored?" If the answer is "in CI variables",
that settles it too.

**2.** You cannot:
- **Identify the code.** "Production runs `latest`" does not tell you which commit, so you cannot read
  the source that produced the bug or reproduce it.
- **Roll back.** `git revert` reverts your manifest, but the manifest still says `latest`, so the
  cluster keeps whatever `latest` currently points at.
- **Trust reconciliation.** The agent compares the manifest to the cluster; both say `latest`, so it
  reports "in sync" whether or not the right code is running.

Bonus fourth: you cannot guarantee your replicas are even running the *same* image, since pods started
at different times may have pulled different `latest` builds.

**3.** Three legitimate answers, in order of preference:

1. **Commit to Git anyway.** A one-line change, self-merged with the incident number in the message,
   takes about a minute — usually faster than debating alternatives. The change is then recorded,
   which is exactly what you want during an incident.
2. **Suspend reconciliation for that one resource**, do the manual change, then reconcile Git
   afterwards. Fleet supports this: pause the bundle. Suspending
   is honest — the tool reports "not reconciling" rather than silently lying about being in sync.
3. **Scale/patch by hand knowing it will revert**, purely to buy 60 seconds while you write the real
   commit.

What you must *not* do is disable the agent entirely and forget. That is how you end up with a cluster
nobody can describe — the exact situation GitOps was adopted to escape.
:::

---

Continue with: [7. Rancher Fleet](07-fleet.md).
