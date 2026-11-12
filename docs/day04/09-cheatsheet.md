# 9. Day 4 cheat sheet

A command reference, followed by a symptom to cause to fix table.

**Slides:** the slides used in class are in one file, {download}`day04-git-cicd-gitops.pdf`.

## Git

### Everyday loop

```bash
git switch main && git pull              # start from up to date
git switch -c feat/my-change             # branch
git add -p                               # stage interactively (review as you go)
git commit -m "feat: add the thing"
git push -u origin feat/my-change        # push; GitLab prints an MR link
```

### Looking around

```bash
git status                               # what is going on right now
git log --oneline -20                    # recent history
git log --oneline --graph --all          # history with branches
git log -p path/to/file                  # every change to one file, with diffs
git log -S "some_function"               # commits that added/removed that string
git blame path/to/file                   # who last touched each line
git diff                                 # unstaged changes
git diff --staged                        # staged changes
git diff main..my-branch                 # difference between branches
git show <sha>                            # one commit in full
```

### Undoing

| Situation | Command |
|---|---|
| Discard edits to one file | `git restore path/to/file` |
| Unstage but keep the edit | `git restore --staged path/to/file` |
| Fix the last commit message | `git commit --amend` |
| Add a forgotten file to the last commit | `git add f && git commit --amend --no-edit` |
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` |
| Undo last commit and the changes | `git reset --hard HEAD~1` ⚠️ |
| Undo a **pushed** commit | `git revert <sha>` ✅ |
| Put changes aside temporarily | `git stash` … `git stash pop` |
| Find something you think you lost | `git reflog` |
| Abandon a merge in progress | `git merge --abort` |

:::{important} On a shared branch: `revert`, never `reset`
`reset` rewrites history and needs a force-push, which breaks everyone else's next `pull`. `revert`
moves forward. GitOps rollbacks are `revert`.
:::

### Conflicts

```bash
git status                    # which files are conflicted
# edit the files; delete the <<<<<<< ======= >>>>>>> markers
git add <resolved-file>
git commit                    # or: git merge --abort to start over
```

---

## GitLab CI

### Skeleton

```yaml
stages: [lint, test, build, deploy]

variables:
  IMAGE: "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"   # the image, tagged with the commit

.python:                                 # hidden job = reusable template
  image: python:3.12-slim
  before_script: ["pip install -q -r requirements.txt"]

unit-tests:
  extends: .python
  stage: test
  script: ["pytest -v --junitxml=report.xml"]
  artifacts:
    when: always
    reports:
      junit: report.xml
```

### Build and push an image (Kaniko)

```yaml
build-container-image:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.23.0-debug
    entrypoint: [""]                     # mandatory
  script:
    - mkdir -p /kaniko/.docker
    - |
      echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "$CI_REGISTRY_USER" "$CI_REGISTRY_PASSWORD" | base64 | tr -d '\n')\"}}}" \
        > /kaniko/.docker/config.json
    - /kaniko/executor
        --context "$CI_PROJECT_DIR"
        --dockerfile "$CI_PROJECT_DIR/Dockerfile"
        --destination "$IMAGE"
```

### Security scanners — one line each

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/SAST-IaC.gitlab-ci.yml          # KICS: manifests, charts, Dockerfiles
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml  # Trivy
# vulnerable libraries in the source tree: trivy fs --scanners vuln . (section 4.5)
```

### `rules` recipes

```yaml
# Only on the default branch
rules: [{ if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH' }]

# Only on merge requests
rules: [{ if: '$CI_PIPELINE_SOURCE == "merge_request_event"' }]

# Only on tags
rules: [{ if: '$CI_COMMIT_TAG' }]

# Skip docs-only changes, otherwise run (order matters!)
rules:
  - if: '$CI_COMMIT_BRANCH'
    changes: ["**/*.md"]
    when: never
  - if: '$CI_COMMIT_BRANCH'

# A real production gate: pipeline waits for a human
rules:
  - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
    when: manual
    allow_failure: false
```

**First match wins.** Exclusions go first.

### Variables worth knowing

| Variable | Meaning |
|---|---|
| `CI_COMMIT_SHA` / `CI_COMMIT_SHORT_SHA` | Commit hash — use as the image tag |
| `CI_COMMIT_BRANCH` / `CI_COMMIT_REF_NAME` | Branch name |
| `CI_DEFAULT_BRANCH` | Usually `main` |
| `CI_COMMIT_TAG` | Set only on tag pipelines |
| `CI_PIPELINE_SOURCE` | `push`, `merge_request_event`, `schedule`, `web` |
| `CI_PROJECT_DIR` | Checkout path |
| `CI_REGISTRY` / `CI_REGISTRY_IMAGE` | Registry host / this project's image path |
| `CI_REGISTRY_USER` / `CI_REGISTRY_PASSWORD` | Job-scoped registry credentials |
| `CI_ENVIRONMENT_NAME` | Set when the job has `environment:` |

---

## Scanning and signing

Trivy reads three different kinds of thing. Picking the wrong mode is the usual reason a scan
finds nothing:

| Command | Scans | Finds |
|---|---|---|
| `trivy image <ref>` | a built image | vulnerable OS and language packages |
| `trivy fs .` | a source tree, via its lock files | vulnerable dependencies before you build |
| `trivy config .` | Dockerfiles, manifests, charts, Terraform | misconfiguration, such as running as root |

```bash
trivy image python:3.12-slim              # what is already in your base image
trivy fs .                                # what your lock files pull in
trivy config .                            # how your manifests are configured

# A chart is a template, so scan what it renders, not the template.
helm template my-release ./chart > rendered.yaml
trivy config rendered.yaml
```

Turning a report into a gate:

```bash
trivy image \
  --severity CRITICAL,HIGH \
  --ignore-unfixed \
  --exit-code 1 \
  "$IMAGE"
```

`--exit-code 1` fails the job; `--severity` chooses what counts; `--ignore-unfixed` skips findings
with no released fix, without which a base image's unpatchable CVEs block every deployment.

Scanning an image in the project's own registry from a CI job: the registry is private, so give Trivy
the job's credentials, or it fails with `UNAUTHORIZED`:

```yaml
  variables:
    TRIVY_USERNAME: $CI_REGISTRY_USER
    TRIVY_PASSWORD: $CI_REGISTRY_PASSWORD
```

Accepting a finding on purpose — always with a date and a reason, or it becomes permanent by accident:

`.trivyignore`

```text
# Accepted 2026-10-12, review by 2026-12-01.
# Affects a code path this service never calls; upstream fix expected in 2.9.
CVE-2026-12345
```

An SBOM answers "are we affected?" without rebuilding anything:

```bash
trivy image --format cyclonedx --output sbom.json "$IMAGE"
```

Signing, so the image can be traced to the pipeline that built it (key-based: keyless signing does
not trust a self-hosted GitLab):

```bash
cosign generate-key-pair                                   # once; cosign.pub goes in the repo
cosign sign --yes --tlog-upload=false --key env://COSIGN_PRIVATE_KEY "$CI_REGISTRY_IMAGE@<digest>"
cosign verify --key cosign.pub --insecure-ignore-tlog=true "<registry>/<project>@<digest>"
```

## Cookiecutter

```bash
cookiecutter ./my-template                       # interactive
cookiecutter --no-input ./my-template            # all defaults
cookiecutter --no-input ./my-template key=value  # override on the CLI
cookiecutter --no-input --config-file answers.yaml ./my-template
cookiecutter https://gitlab…/template.git --checkout my-branch
cookiecutter --overwrite-if-exists ...           # regenerate in place
```

`answers.yaml`

```yaml
default_context:
  chart_name: "my-app"
  include_ingress: "yes"
```

### Template essentials

```json
{
  "project_name": "my-app",
  "package_name": "{{ cookiecutter.project_name.lower().replace('-', '_') }}",
  "include_ingress": ["yes", "no"],
  "_copy_without_render": ["templates/*"]
}
```

| Gotcha | Remember |
|---|---|
| Helm charts | `_copy_without_render: ["templates/*"]` — **mandatory**, Helm and Jinja share `{{ }}` |
| No booleans | `{% if cookiecutter.x == 'yes' %}` — never `{% if cookiecutter.x %}` |
| Derived defaults | Can only reference keys **above** them |
| Hooks | Jinja-rendered first → `json.loads(r"""{{ cookiecutter \| jsonify }}""")` |
| `post_gen` CWD | Inside the *generated* project |

---

## Helm

```bash
helm lint ./chart                        # sanity check
helm template ./chart                    # render locally — do this before committing
helm template ./chart -f prod.yaml       # render with overrides
helm template ./chart --show-only templates/deployment.yaml

helm list -n <namespace>                 # what is installed
helm get values <release> -n <ns>        # values actually in use
helm get manifest <release> -n <ns>      # what was actually applied
helm history <release> -n <ns>
```

:::{warning} Helm ignores unknown values silently
`replicasCount: 3` (typo) is accepted and does nothing. Always `helm template` and read the output.
:::

---

## Fleet

`fleet.yaml` — raw manifests

```yaml
defaultNamespace: my-app
```

`fleet.yaml` — a Helm chart in the same folder (next to `Chart.yaml`, no `chart:` needed)

```yaml
defaultNamespace: my-app
helm:
  releaseName: my-app
  values:                                # this deployment's settings; chart defaults stay in values.yaml
    image:
      tag: abc123def
```

`fleet.yaml` — a Helm chart elsewhere in the repository ([section 7.3](#d4-7-3))

```yaml
defaultNamespace: my-app
helm:
  chart: /workspace/source/chart         # Fleet clones the repo here
  releaseName: my-app
```

`fleet.yaml` — different values per cluster (first matching entry wins)

```yaml
targetCustomizations:
  - name: production
    clusterSelector:
      matchLabels:
        env: prod
    helm:
      values:
        replicaCount: 3
```

`fleet.yaml` — a Helm chart from a chart repository

```yaml
defaultNamespace: jupyterhub
helm:
  releaseName: jhub
  repo: https://hub.jupyter.org/helm-chart/
  chart: jupyterhub
  version: 4.4.4
  values: { }
```

`fleet.yaml` — ordering, and ignoring a controller-owned field

```yaml
defaultNamespace: my-app
dependsOn:
  - name: my-app-secrets
diff:
  comparePatches:
    - apiVersion: v1
      kind: Service
      name: my-app
      operations:
        - {"op": "remove", "path": "/spec/clusterIP"}
```

`gitrepo.yaml` — registering the repository, once

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app-gitops
  namespace: fleet-default
spec:
  repo: https://gitlab…/my-app-gitops.git
  branch: main
  paths: [prod]                          # omit to auto-discover every fleet.yaml
  clientSecretName: git-credentials      # private repos only
  correctDrift:
    enabled: true                        # revert manual changes in the cluster
  targets:
    - clusterSelector:
        matchLabels:
          environment: production
```

### Plain manifests or a Helm chart?

| Situation | Use |
|---|---|
| Your own service, deployed once | Plain manifests: reviewers read exactly what runs |
| The same thing deployed several times, with differences | A chart, plus `helm.values` / `targetCustomizations` per deployment |
| Third-party software | The upstream chart, `version:` pinned, configured through `helm.values` |

```bash
helm template my-app ./helm | kubectl -n my-app diff --server-side --force-conflicts -f -   # what would change
helm -n my-app list                      # every Fleet bundle is a Helm release
helm -n my-app history my-app            # one revision per deployment
helm show values <repo>/<chart> --version <v>   # what an upstream chart can be configured with
```

### Pulling from a private registry

```bash
kubectl -n my-app create secret docker-registry gitlab-registry \
  --docker-server=<registry host:port> \
  --docker-username=<deploy token user> --docker-password=<deploy token>   # scope: read_registry
```

Reference it from the Pod spec with `imagePullSecrets: [{name: gitlab-registry}]`. On a real
platform, let the External Secrets Operator create it from Vault instead.

### Inspecting Fleet

```bash
# Rancher ☰ → Continuous Delivery → Resources → Git Repos, workspace fleet-traineeNN:
#   your GitRepo, its Bundles, and the Resources box with the objects counted by state.
kubectl -n traineeNN get pods,deploy,svc,ingress   # what Fleet deployed, in your namespace
```

---

## Dagger

```bash
dagger init --sdk=python --source=.dagger --name=my-app   # create a module
#   with engine v0.18.10, then add to .dagger/pyproject.toml and run `dagger develop`:
#   [tool.uv]
#   constraint-dependencies = ["gql<4"]
dagger functions                          # what can this pipeline do?
dagger call test --source=.               # run one step against the working tree
dagger call build --source=. terminal     # open a shell inside that step
dagger develop                            # regenerate the SDK stubs after an edit
dagger install github.com/org/mod@v1.0.0  # add a module dependency
```

Calling it from CI is a thin wrapper — the build logic stays in the module. On this GitLab, the
runner tagged `dagger` provides the engine; the job needs only the CLI:

```yaml
.dagger:
  image:
    name: ghcr.io/purpleclay/dagger-cli:0.18.10   # CLI only, engine's version
    entrypoint: [""]                              # entrypoint is `dagger`
  tags: [dagger]

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
    - dagger call publish --source=. --registry="$CI_REGISTRY"
        --image="$CI_REGISTRY_IMAGE:dagger-$CI_COMMIT_SHORT_SHA"
        --username="$CI_REGISTRY_USER" --password=env:CI_REGISTRY_PASSWORD
```

`env:` passes a *reference* to the variable, so the value never reaches a command line.

## Releases

| Commit type | Release | Version 1.4.2 becomes |
|---|---|---|
| `fix:` | patch | 1.4.3 |
| `feat:` | minor | 1.5.0 |
| `BREAKING CHANGE:` in the body, or `feat!:` | major | 2.0.0 |
| `chore:`, `ci:`, `docs:`, `test:`, `refactor:` | none | 1.4.2 |

```yaml
release:
  stage: release
  needs: [security-audit]                 # no release for an image that failed the gate
  image: node:20
  variables:
    GIT_DEPTH: 0                          # full history, or it finds no previous tag
  before_script:
    - git fetch --tags --force
    - npm install -g semantic-release @semantic-release/gitlab
        @semantic-release/changelog @semantic-release/git
  script:
    - git tag --list > tags-before.txt
    - semantic-release
    - echo "RELEASE_TAG=$(git tag --list | grep -vxFf tags-before.txt)" > release.env  # new tag, or empty
  artifacts:
    reports: { dotenv: release.env }
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

tag-release-image:                        # version tag on the image already scanned
  stage: release
  needs: [release]
  image: { name: "gcr.io/go-containerregistry/crane/debug:v0.22.1", entrypoint: [""] }
  script:
    - if [ -z "$RELEASE_TAG" ]; then exit 0; fi
    - crane auth login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - crane tag "$IMAGE" "$RELEASE_TAG"    # $IMAGE = $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Two common causes of a release job that does the wrong thing:

- **`GIT_DEPTH: 0`.** With a shallow clone the tool sees no tags, concludes nothing has
  been released, and quietly does nothing.
- **`[skip ci]` in the release commit message** (`.releaserc.json`), or the `CHANGELOG.md`
  commit runs the whole pipeline again. It also skips the tag's pipeline: put the version on the
  image from the pipeline on `main` (`tag-release-image` above), not from a `$CI_COMMIT_TAG` job.

## kubectl

```bash
kubectl -n <ns> get pods,deploy,svc,ingress
kubectl -n <ns> get pods -w                       # watch a rollout
kubectl -n <ns> describe pod <pod>                # ← read the Events at the bottom
kubectl -n <ns> logs <pod>
kubectl -n <ns> logs <pod> --previous             # logs from the crashed instance
kubectl -n <ns> logs -l app=my-app --tail=100 -f  # all pods of an app
kubectl -n <ns> get events --sort-by=.lastTimestamp
kubectl -n <ns> exec -it <pod> -- sh
kubectl -n <ns> port-forward svc/my-app 8080:80   # bypass DNS/TLS while debugging
kubectl -n <ns> rollout status deploy/my-app
kubectl -n <ns> auth can-i get pods               # am I allowed?
kubectl -n <ns> get secret <name> -o jsonpath='{.data.KEY}' | base64 -d
```

### Secrets and certificates

```bash
kubectl -n <ns> get externalsecret
kubectl -n <ns> describe externalsecret <name>    # did it sync from Vault?
kubectl -n <ns> get certificate
kubectl -n <ns> describe certificate <name>
kubectl -n <ns> describe certificaterequest       # why the cert is not ready
```

### kustomize

```bash
kubectl kustomize overlays/prod                   # render — do this before committing
kubectl apply --dry-run=server -k overlays/prod   # validate against the API server
```

---

## Troubleshooting table

### Git

| Symptom | Cause | Fix |
|---|---|---|
| `rejected … non-fast-forward` | Remote has commits you do not | `git pull --rebase` then push |
| `Your branch and 'origin/x' have diverged` | Someone force-pushed, or you rebased | `git log --oneline --graph --all`, then rebase or reset onto origin |
| Committed a secret | — | **Rotate the secret.** Then `.gitignore` it |
| `Permission denied (publickey)` | SSH key not registered / not loaded | `ssh -T git@gitlab…`; `ssh-add -l` |
| Lost a commit | — | `git reflog`, then `git switch -c rescue <sha>` |

### GitLab CI

| Symptom | Cause | Fix |
|---|---|---|
| Job absent from the pipeline | `rules` — it was never *created* | Read rules top to bottom; first match wins |
| `yaml invalid` | Indentation, or a literal tab | Use *Pipeline editor → Validate* |
| `needs 'x' is not defined` | Typo, or `x` excluded by its own rules | Check both spellings and `x`'s rules |
| `no such file` in a later stage | Not declared as an `artifact`, or `needs:` narrowed inputs | Add to `artifacts.paths` |
| The same commit sometimes passes, sometimes fails with `No such file` | A job uses a cached folder (e.g. `.venv/`) that it never creates: a cache miss breaks it | Each job installs what it needs; the cache only speeds it up. A file another job *made* is an artifact |
| Every pipeline stays *blocked* | A `when: manual` job with `allow_failure: false` (the default inside `rules:`) nobody meant as a gate | `allow_failure: true` for an optional button |
| Production deploy goes green without approval | `when: manual` with `allow_failure: true` (the default for a job-level `when: manual`) | `allow_failure: false`, on a protected branch whose merge rights only the deployers have |
| `This job is stuck` | No runner matches the job's `tags:` | Check *Settings → CI/CD → Runners* |
| `denied: access forbidden` (registry) | Login failed / registry disabled | Use `$CI_REGISTRY_USER`+`$CI_REGISTRY_PASSWORD`; enable the registry |
| Kaniko ignores `script:` | Missing `entrypoint: [""]` | Add it; use a `-debug` image tag |
| Secret visible in a log | Masking defeated (`base64`, substring) or value unmaskable | Stop printing it; **rotate** |
| Manual job never runs, pipeline hangs | `when: manual` + `allow_failure: false` | That is correct — click it. Or set `allow_failure: true` |

### Fleet / Kubernetes

| Symptom | Cause | Fix |
|---|---|---|
| Bundle `NotReady` / `Modified` | Apply failed, or drift | Open the bundle in Rancher; read per-resource message |
| `cannot be imported into the current release: invalid ownership metadata` | The bundle declares an object that already exists and is not owned by its Helm release — most often a `Namespace` | Fleet renders each bundle as a Helm release and creates `defaultNamespace` itself. Remove the `Namespace` manifest and set `namespaceLabels` in `fleet.yaml` instead |
| Same error, `Service "…" … current value is "<gitrepo>-manifests"`, after changing a GitRepo's **Paths** | The new bundle's install ran while the old bundle's release still owned the objects; Fleet does not retry it | Wait until `kubectl -n <ns> get deploy,svc,ingress` shows the old objects gone, then *⋮ → Force Update* on the GitRepo |
| Bundle flaps Ready ↔ Modified | A controller owns a field you set | `diff.comparePatches` for that exact path |
| New folder not deployed | GitRepo has explicit `paths:`, or wrong branch | Add the path, or remove `paths:` to auto-discover |
| `ImagePullBackOff` | Bad image path/tag, or missing pull secret | `describe pod`; check registry path; `imagePullSecrets` |
| `CreateContainerConfigError` | Referenced Secret/ConfigMap missing | `describe pod`; check the ExternalSecret; add `dependsOn` |
| `CrashLoopBackOff` | App exits on start | `logs --previous` — the reason is there |
| `Pending` forever | No node has the resources, or PVC unbound | `describe pod` Events; `get pvc` |
| `0/1 Running` but not Ready | Readiness probe failing | `describe pod`; check the probe path/port |
| Ingress has no address | Ingress controller issue | `describe ingress`; ask the platform team |
| Certificate never Ready | Wrong issuer, or challenge failing | `describe certificate` → `certificaterequest` |
| ExternalSecret not syncing | Wrong Vault path, or store unhealthy | `describe externalsecret`; check the `ClusterSecretStore` |
| Helm values ignored | Key does not match the chart | `helm template` and read the output |
| Helm values in `fleet.yaml` ignored | `values:` not under `helm:` | Fix the indentation; Fleet does not warn |
| GitRepo error: `forbidden` … `in the namespace "default"` | `fleet.yaml` misnamed (`fleet.yml`, `ffleet.yaml`…), so no `defaultNamespace` | Rename it to exactly `fleet.yaml` |
| GitRepo *Error*: `Authentication required: HTTP Basic: Access denied` | Created after **Edit as YAML**, which drops the token; or the wrong token | Delete the GitRepo and create it from the form; the config project's deploy token, `read_repository` |
| Manual change *not* reverted, bundle `Modified` | Drift correction off | **⋮ → Edit Config → Advanced → Enable self-healing** (`correctDrift`), or *Force Update* |
| Manual `kubectl` change reverted | ✅ Reconciliation working | Change Git instead |

### Scanning, Dagger and releases

| Symptom | Cause | Fix |
|---|---|---|
| Trivy finds nothing on a chart | You scanned the template, not the output | `helm template … > rendered.yaml`, then `trivy config rendered.yaml` |
| Trivy fails the job on day one | Base image has CVEs with no released fix | Add `--ignore-unfixed`; accept the rest in `.trivyignore`, dated |
| `.trivyignore` seems ignored | Wrong working directory, or the ID is a different CVE | Run from the repo root; check the exact ID in the report |
| Scan job runs before the image exists | Scanner job has no `needs:` on the build | Add `needs: [build-image]` and point it at the built tag |
| `cosign verify` fails | Verifying a tag that has since moved | Verify the digest, not the tag |
| `dagger: command not found` | `~/.local/bin` not on `PATH` | `export PATH="$HOME/.local/bin:$PATH"` |
| Dagger hangs at "connecting to engine" | No usable Docker daemon | `docker ps` must succeed |
| Dagger: `Unknown argument "includeDeprecated" on field "__Field.args"` | Module locked gql 4, too new for engine v0.18.10 | `constraint-dependencies = ["gql<4"]` under `[tool.uv]` in `.dagger/pyproject.toml`, then `dagger develop` |
| Dagger `ModuleNotFoundError` | Class name does not match the module | `dagger develop`, then match the generated stub |
| Dagger secret leaked into a log | Passed as a string, not a `Secret` | Type it `dagger.Secret`, pass `--password=env:VAR` |
| "no relevant changes, no new version" | No `feat:`/`fix:` commit since the last tag; less often a shallow clone (then it re-releases 1.0.0) | Check the commit types; `GIT_DEPTH: 0` and `git fetch --tags` |
| The release exists, but no `vX.Y.Z` image | `RELEASE_TAG` empty in `tag-release-image`, or the image was built from a `$CI_COMMIT_TAG` job, which `[skip ci]` skips | dotenv report on `release` and `needs: [release]`; tag with `crane tag` on `main` |
| `403` creating the tag | `GITLAB_TOKEN` missing, unprotected, or lacks `api` | Masked **and** protected, `api` scope, allowed to push to `main` |
| Version bumped when nothing shipped | `chore:`/`ci:`/`docs:` do not release — but `fix(ci):` does | Check the *type*, not the scope |

---

## Seven rules to remember after today

1. **A secret in Git is compromised.** Rotate it; nothing else fixes it.
2. **Base64 is not encryption.** A Kubernetes `Secret` in Git is plaintext in Git.
3. **Never deploy `latest`.** Use the commit SHA or a digest.
4. **`revert`, not `reset`,** on anything shared. That is your rollback.
5. **Render before you commit.** `helm template` / `kubectl kustomize` costs two seconds.
6. **If nothing reverts a manual `kubectl edit`, it is not GitOps.**
7. **A version number is only worth what the commit types are.** `fix:` on a breaking
   change publishes a patch, and everyone who trusts it upgrades and breaks.

---

## Reference documentation

- Git — <https://git-scm.com/docs> · interactive tutorial: <https://learngitbranching.js.org/>
- GitLab CI — <https://docs.gitlab.com/ci/>
- GitLab CI variables — <https://docs.gitlab.com/ci/variables/predefined_variables/>
- GitLab security scanning — <https://docs.gitlab.com/user/application_security/>
- Kaniko — <https://github.com/GoogleContainerTools/kaniko>
- Trivy — <https://trivy.dev/> · all flags: <https://trivy.dev/docs/latest/references/configuration/cli/trivy/>
- `.trivyignore` and accepting findings — <https://trivy.dev/docs/latest/configuration/filtering/>
- cosign signing — <https://docs.sigstore.dev/cosign/signing/overview/>
- CycloneDX SBOM format — <https://cyclonedx.org/>
- Dagger — <https://docs.dagger.io/> · Python SDK: <https://dagger-io.readthedocs.io/en/sdk-python-v0.18.10/>
- Daggerverse, for reusable modules — <https://daggerverse.dev/>
- Conventional Commits — <https://www.conventionalcommits.org/>
- Semantic Versioning — <https://semver.org/>
- semantic-release — <https://semantic-release.gitbook.io/semantic-release/> · plugin list:
  <https://semantic-release.gitbook.io/semantic-release/extending/plugins-list>
- commitlint — <https://commitlint.js.org/>
- Cookiecutter — <https://cookiecutter.readthedocs.io/>
- Helm — <https://helm.sh/docs/>
- kustomize — <https://kubectl.docs.kubernetes.io/references/kustomize/>
- Rancher Fleet — <https://fleet.rancher.io/>
- External Secrets Operator — <https://external-secrets.io/>
- cert-manager — <https://cert-manager.io/docs/>
- OpenGitOps principles — <https://opengitops.dev/>
