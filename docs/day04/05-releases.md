# 5. Conventional commits and automated releases

[Section 1](01-git.md) introduced Conventional Commits and claimed that the convention
"lets tools generate changelogs automatically". That claim was left unpaid. This
section pays it: the commit message becomes the input to a tool that decides the
version number, writes the changelog, creates the tag and publishes the release.

**Slides:** the slides used in class are in one file, {download}`day04-git-cicd-gitops.pdf`.

## 5.1 The problem with version numbers

Releasing software by hand requires somebody to answer three questions correctly,
every time:

- Which version number comes next?
- What changed since the last one?
- Which artifacts belong to this version?

All three answers already exist in the repository. The commits since the last tag say
what changed. Whether any of them was a breaking change determines the number. The
only reason a person is involved is that the commit messages are not written in a form
a tool can read.

(d4-5-2)=
## 5.2 Semantic versioning

:::{note} Jargon: *semantic versioning*, *SemVer*
**Semantic versioning** is the convention `MAJOR.MINOR.PATCH`, where each component
carries a promise to whoever consumes the software:

- **PATCH** (`1.4.2` → `1.4.3`): a bug is fixed and nothing else changes. Safe to take.
- **MINOR** (`1.4.3` → `1.5.0`): something was added, and existing usage still works.
  Safe to take.
- **MAJOR** (`1.5.0` → `2.0.0`): something existing changed or was removed. Upgrading
  requires work.
:::

The value of the convention is that a consumer can decide whether to upgrade without
reading the changelog. That only holds if the numbers are assigned honestly, which is
the second reason to take the decision away from a person: a breaking change released
as a patch is worse than no versioning at all, because it breaks a promise the
consumer relied on.

## 5.3 From commit message to version

Conventional Commits, from [section 1](01-git.md), give each commit a machine-readable
type. The mapping to a version bump is direct:

| Commit | Meaning | Bump |
|---|---|---|
| `fix: handle an empty input file` | A bug fix | PATCH |
| `feat: add a CSV export endpoint` | A new capability | MINOR |
| `feat!: drop support for the v1 API` | A breaking change | MAJOR |
| `docs: correct a typo in the README` | No release | none |
| `chore: update the linter version` | No release | none |

A breaking change is marked either with `!` after the type, or with a
`BREAKING CHANGE:` line in the commit body:

```text
feat: replace the single config file with a directory

BREAKING CHANGE: config.yaml is no longer read. Move its contents into
conf.d/ and delete it. See MIGRATION.md for the mapping.
```

The body is where the migration instructions belong, because the tool copies them into
the changelog and the release notes. A breaking change with no explanation is a
support ticket waiting to happen.

```{mermaid}
flowchart TB
    A["Commits since the last tag"] --> B["Highest bump wins"]
    B --> C["New version number"]
    C --> D["Changelog entry"]
    C --> E["Git tag"]
    C --> F["Release with notes"]
```

The rule is that the **highest** bump among the commits wins. Twelve fixes and one
breaking change produce a major release.

(d4-5-4)=
## 5.4 Doing it

The most widely used implementation is `semantic-release`. It is configured with a
plugin chain, where each plugin does one step:

`.releaserc.json`

```json
{
  "branches": [
    "main",
    { "name": "dev", "prerelease": true }
  ],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", { "changelogFile": "CHANGELOG.md" }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md"],
      "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
    }],
    "@semantic-release/gitlab"
  ]
}
```

Reading that chain in order: analyze the commits to choose a version, generate notes
from them, write the notes into `CHANGELOG.md`, commit that file back, and create the
GitLab release.

The `dev` branch entry with `prerelease: true` produces versions such as
`2.1.0-dev.3`, so a branch can be released continuously without consuming a real
version number.

The job that runs it:

`.gitlab-ci.yml`

```yaml
release:
  stage: release
  image: node:20
  variables:
    # semantic-release needs the full history and all tags to work out what the
    # last release was. A shallow clone is the most common reason it silently
    # does nothing.
    GIT_DEPTH: 0
  before_script:
    - git fetch --tags --force
    - npm install -g semantic-release @semantic-release/gitlab
        @semantic-release/changelog @semantic-release/git
  script:
    - semantic-release
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

:::{important} Two settings that cause almost every failure
**`GIT_DEPTH: 0`** plus `git fetch --tags`. By default CI performs a shallow clone with
no tags, so the tool cannot see the previous release and either does nothing or starts
again from `1.0.0`.

**`[skip ci]` in the release commit message.** The tool commits `CHANGELOG.md` back to
`main`. Without the marker, that commit starts a whole second pipeline: it tests, builds
and scans code that has not changed, and its release job finds nothing to release.

One consequence to know: the release tag points at that same commit, so GitLab skips the
tag's pipeline too. A job with `rules: - if: $CI_COMMIT_TAG` never runs. That is why
[section 5.6](#d4-5-6) puts the version on the image from the pipeline on `main`, not from a tag
pipeline.
:::

## 5.5 What it produces

The changelog is generated, so it is accurate by construction:

`CHANGELOG.md`

```markdown
## [1.4.1](https://gitlab.example.com/team/app/compare/v1.4.0...v1.4.1) (2026-10-14)

### Bug Fixes

* **api:** return 404 rather than 500 for an unknown dataset ([a1b2c3d](...))

# [1.4.0](https://gitlab.example.com/team/app/compare/v1.3.2...v1.4.0) (2026-10-11)

### Features

* **export:** add a CSV export endpoint ([e4f5a6b](...))
```

The scope in parentheses comes from the commit message: `fix(api): ...` produces the
`**api:**` prefix. Scopes are optional, and useful once a repository is large enough
that a reader wants to skim for the part they use.

(d4-5-6)=
## 5.6 Connecting the version to the artifact

A version number that is not attached to anything is bookkeeping. The tag has to reach
the image, which is where [section 3](03-gitlab-ci.md) and this section meet.

The pipeline on `main` has already built the image and scanned it. Its name is `$IMAGE`
from [section 3](03-gitlab-ci.md), the image path followed by the commit hash, for example
`…/climate-telemetry-traineeNN:3f2a9c1e`. When the release job creates a version, say `v1.4.1`, one
more job adds `v1.4.1` as a second tag **on that same image**. Nothing is rebuilt.

That takes three steps, one per heading below: find out which version the release job has just created, pass it
to the next job, and tag the image.

### Which version did semantic-release just create?

`semantic-release` prints the version in its log, but a later command cannot read a log.
What it also does is create a Git tag in the job's copy of the repository. So the job
lists the tags before and after it runs, and keeps the one that is new:

```yaml
release:
  # ... as in section 5.4, with this script:
  script:
    - git tag --list > tags-before.txt
    - semantic-release
    - echo "RELEASE_TAG=$(git tag --list | grep -vxFf tags-before.txt)" > release.env
```

Line by line, for a project whose last release was `v1.4.0` and that has just merged a
`fix:` commit:

1. `git tag --list > tags-before.txt` writes the existing tags into a file, one per line:

   ```text
   v1.3.2
   v1.4.0
   ```

2. `semantic-release` creates and pushes the tag `v1.4.1`. Now `git tag --list` prints
   three lines: `v1.3.2`, `v1.4.0`, `v1.4.1`.

3. `grep -vxFf tags-before.txt` filters those three lines. It keeps the lines that are
   **not** in `tags-before.txt`:

   | Option | Meaning |
   |---|---|
   | `-f tags-before.txt` | Take the patterns to search for from this file, one per line |
   | `-F` | The patterns are plain text, not regular expressions (a `.` is just a dot) |
   | `-x` | A pattern must match the whole line, so `v1.4.0` does not match `v1.4.0-rc.1` |
   | `-v` | Invert: print the lines that match **none** of the patterns |

   The only line left is `v1.4.1`. `$( … )` puts that output into the command, so
   `echo` writes this line into the file `release.env`:

   ```text
   RELEASE_TAG=v1.4.1
   ```

   If nothing was released (only `ci:` or `docs:` commits), no tag is new, `grep`
   prints nothing, and the file reads:

   ```text
   RELEASE_TAG=
   ```

### Passing the version to the next job

Each job runs in a new container. A variable set in one job, or a file written in it, is
gone when the job ends. GitLab has one way to hand a variable over: the job declares the
file as a **dotenv report**.

```yaml
release:
  # ...
  artifacts:
    reports:
      dotenv: release.env
```

:::{note} Jargon: *dotenv*
A **dotenv file** (named after the usual file name `.env`) is a text file of
`NAME=value` lines, nothing else. `release.env` above is one.

When a job declares it under `artifacts: reports: dotenv:`, GitLab stores the file at the
end of the job and reads each line. Any later job that lists this job under `needs`
starts with those variables already set: `tag-release-image` below can use
`$RELEASE_TAG` as if it were a CI/CD variable.
:::

### Tagging the image

```yaml
tag-release-image:
  stage: release
  needs: [release]                     # runs after release, and receives RELEASE_TAG
  image:
    name: gcr.io/go-containerregistry/crane/debug:v0.22.1
    entrypoint: [""]
  script:
    - if [ -z "$RELEASE_TAG" ]; then echo "No release in this pipeline"; exit 0; fi
    - crane auth login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - crane tag "$IMAGE" "$RELEASE_TAG"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

:::{note} Jargon: *crane*
**crane** is a small command-line tool that works on images directly in the registry.
`crane tag` adds a tag to an image that is already there, without downloading or
uploading any layer. The `debug` variant of its image includes a shell, which GitLab
needs to run the `script:` lines; `entrypoint: [""]` stops the image from starting
`crane` by itself, as with Kaniko in [section 3](03-gitlab-ci.md).
:::

The three lines of the script:

1. `[ -z "$RELEASE_TAG" ]` is true when the variable is empty. No release: the job says
   so and ends successfully.
2. `crane auth login` logs in to the project's registry with the job's own credentials,
   the same ones Kaniko uses.
3. With the values of the example, `crane tag "$IMAGE" "$RELEASE_TAG"` becomes
   `crane tag …/climate-telemetry-traineeNN:3f2a9c1e v1.4.1`. The job log shows the result:

   ```text
   …/climate-telemetry-traineeNN:v1.4.1: digest: sha256:19a54404…2025f size: 2303
   ```

After a release, the registry holds one image under two tags, for example `3f2a9c1e`
and `v1.4.1`. Both are deliberate. The version is the human-meaningful reference: it
carries the compatibility promise of [section 5.2](#d4-5-2). The commit hash is the reference
that GitOps requires in [section 6](06-gitops.md), because it is never moved to
another image. A person reads the version; the deployment manifest uses the hash or the
digest.

Tagging the scanned image, instead of building again for the tag, has one more
advantage: `v1.4.1` is *exactly* the image that passed the security gate, with the same
digest. A second build could pick up a different base image or different packages.

## 5.7 Why this usually fails

The tooling is the easy part. The convention is the hard part, and the failure is
consistent enough to be predictable.

A repository adopts `semantic-release`, and the commit history continues to read
`update`, `fix stuff`, `wip`, `changes as requested`. The commit analyzer finds nothing
it recognizes, so it releases nothing. The tool appears broken; the input is the
problem.

Two mitigations, in increasing strength:

**Check the message.** A linter on the commit message, run in CI or as a local
`commit-msg` hook, rejects a message that does not match the convention:

```yaml
lint-commits:
  stage: lint
  image: node:20
  script:
    - npm install -g @commitlint/cli @commitlint/config-conventional
    - echo "module.exports = {extends:['@commitlint/config-conventional']}" > commitlint.config.js
    - npx commitlint --from "$CI_MERGE_REQUEST_DIFF_BASE_SHA" --to HEAD
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

**Control the merge commit.** Where a project squashes merge requests, the squash
message becomes the only commit on the default branch. Requiring the merge request
*title* to follow the convention then makes the whole thing enforceable through the
review process, which is where somebody is already reading it.

:::{warning} A convention followed inconsistently is worse than none
If most commits follow the convention and some do not, the tool still releases, but the
version numbers stop being trustworthy: a breaking change committed as `fix:` ships as
a patch, and every consumer who trusted that number is broken. Either enforce the
convention mechanically or do not derive versions from it.
:::

## 5.8 ⌨️ Hands-on 6: let your commit messages cut the release

⏱ **About 20 minutes.** Most of it is waiting for pipelines: read the next steps while they run.

Look at what you have written today:

```bash
cd ~/climate-service
git switch main && git pull
git log --oneline --no-merges
```

```text
6f7a8b9 ci: fail the pipeline on fixable HIGH and CRITICAL findings
5e6f7a8 ci: scan every push for committed secrets
4d5e6f7 ci: name the image once, cache pip, declare job dependencies
3c4d5e6 test: cover the /health endpoint and run it in CI
2b3c4d5 docs: explain how to run the service locally
1a2b3c4 feat: climate telemetry service with a /health endpoint
0f1e2d3 chore: ignore Python caches and local secrets
```

Every message already carries a type, because [section 1.4](#d4-1-4) asked you to write them that way. Nothing
has used it yet. Turn it on.

### Step 1: a token that may create releases

The release job pushes `CHANGELOG.md` and a tag, and creates a GitLab release, so it needs more than
the job token. In the app project, create a **project access token**: *Settings → Access tokens*,
role **Maintainer**, scopes **`api`** and **`write_repository`**, with an expiry date after the course.

Store it under *Settings → CI/CD → Variables* as `GITLAB_TOKEN`, with **Masked** and **Protected**
both ticked.

:::{danger} Why *Protected* matters here
This token can write to your repository. *Protected* means GitLab only hands it to jobs running on
protected branches such as `main`. Without it, anybody who can push any branch gets a job with a
repository-writing token in its environment. This is [section 4.2](#d4-4-2) in practice.
:::

### Step 2: configure and add the job

```bash
git switch -c ci/semantic-release
```

`.releaserc.json`

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", { "changelogFile": "CHANGELOG.md" }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md"],
      "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
    }],
    "@semantic-release/gitlab"
  ]
}
```

In `.gitlab-ci.yml`, add a fourth stage and the two jobs from sections [5.4](#d4-5-4) and [5.6](#d4-5-6). `release` gets one
addition: `needs`, so a release is only cut after the security gate has passed.

```yaml
stages:
  - test
  - build
  - scan
  - release

release:
  stage: release
  needs: [security-audit]            # no release for an image that failed the gate
  image: node:20
  variables:
    GIT_DEPTH: 0
  before_script:
    - git fetch --tags --force
    - npm install -g semantic-release @semantic-release/gitlab
        @semantic-release/changelog @semantic-release/git
  script:
    - git tag --list > tags-before.txt
    - semantic-release
    - echo "RELEASE_TAG=$(git tag --list | grep -vxFf tags-before.txt)" > release.env
  artifacts:
    reports:
      dotenv: release.env
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

tag-release-image:
  stage: release
  needs: [release]
  image:
    name: gcr.io/go-containerregistry/crane/debug:v0.22.1
    entrypoint: [""]
  script:
    - if [ -z "$RELEASE_TAG" ]; then echo "No release in this pipeline"; exit 0; fi
    - crane auth login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - crane tag "$IMAGE" "$RELEASE_TAG"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

```bash
git add .releaserc.json .gitlab-ci.yml
git commit -m "ci: release automatically from commit messages"
git push -u origin ci/semantic-release
```

Open the merge request and merge it. On `main`, `release` and `tag-release-image` run last.

**Checkpoint 6a** — *Deploy → Releases* shows **v1.0.0**, its notes list your `feat:` commits, and
`CHANGELOG.md` now exists in the repository. `ci:`, `test:` and `docs:` commits are not listed: they
do not change what a user of the service gets. *Deploy → Container registry* lists the tag `v1.0.0`
next to the commit hash of your merge, and both have the same digest: one image, two names. The
`chore(release): 1.0.0 [skip ci]` commit has a pipeline marked **skipped**, and so has the tag
`v1.0.0`.

### Step 3: watch the number move by itself

```bash
git switch main && git pull
git switch -c fix/health-content-type
```

`/health` answers with the content type `application/json`, without saying which character set the
text is in. In `app.py`, add this line under `app.json.sort_keys = False`:

```python
app.json.mimetype = "application/json; charset=utf-8"
```

```bash
git commit -am "fix: declare UTF-8 in the /health content type"
git push -u origin fix/health-content-type
```

Merge it.

**Checkpoint 6b** — the next release is **v1.0.1**: a patch, because you wrote `fix:`. Had you
written `feat:`, it would have been v1.1.0. Nobody chose the number; the commit message did.

:::{important} From now on, `HEAD` of `main` is often a release commit
After every `feat:` or `fix:` merge, the release job pushes a `chore(release): … [skip ci]` commit
that only updates `CHANGELOG.md`. It builds no image. When a later step asks for "the tag of the image
on `main`", use the version (`v1.0.1`), or the commit before the release commit:

```bash
git switch main && git pull
git log -1 --invert-grep --grep='^chore(release)' --format=%h --abbrev=8
```

That prints the 8 characters of the newest commit that is not a release commit: the one the image
was built from.
:::

:::{dropdown} "There are no relevant changes, so no new version is released"
Either the job cannot see the history, or there is nothing releasable in it. Check that
`GIT_DEPTH: 0` is set and `git fetch --tags` ran. Then check the commit types since the last release:
`chore:`, `ci:` and `docs:` do not trigger a release, by design.
:::

:::{dropdown} `403 Forbidden` when the job creates the tag
`GITLAB_TOKEN` is missing, not protected while `main` is, or lacks the `api` or `write_repository`
scope. Also check the branch rule for `main` (*Settings → Repository → Branch rules*, [section 4.2](#d4-4-2)):
Maintainers must be allowed to push to `main`, or the `CHANGELOG.md` commit is refused.
:::

:::{dropdown} The release commit started another pipeline
`[skip ci]` is missing from the release message in `.releaserc.json`. The second pipeline does no
harm, as its `release` job finds nothing to release, but it rebuilds and rescans unchanged code.
:::

:::{dropdown} The release exists, but the registry has no `vX.Y.Z` image
Open the `tag-release-image` job of the pipeline on `main` that made the release:

- `No release in this pipeline`: `RELEASE_TAG` arrived empty. Check that the `release` job has the
  `echo "RELEASE_TAG=…"` line and the `dotenv` report, and that `tag-release-image` has
  `needs: [release]`.
- `MANIFEST_UNKNOWN`: `$IMAGE` names a tag that is not in the registry. It must be the same name
  `build-container-image` pushed.

Do not wait for a pipeline on the tag: the release commit carries `[skip ci]`, so it is skipped.
:::

## 5.9 Core takeaway

> **Commit types decide the version — `fix` a patch, `feat` a minor, a breaking change a major — so
> nobody picks numbers by hand.**

- Semantic versioning is a **promise to the consumer**: patch is safe, minor is safe,
  major requires work.
- Conventional Commits make that promise **derivable**, because `fix:`, `feat:` and
  `feat!:` map onto patch, minor and major.
- The **highest** bump among the commits since the last tag wins.
- `semantic-release` runs a plugin chain: analyze, generate notes, write the changelog,
  commit, publish.
- **`GIT_DEPTH: 0`** and **`[skip ci]`** cause almost every reported failure: without the
  first the job cannot see the previous tag; the second stops the release commit from running
  the whole pipeline again, and also skips the tag's pipeline.
- Tag both the version and the commit SHA on the image. People read versions,
  deployments use immutable references. Add the version to the image that was scanned,
  with `crane tag`, rather than building it again.
- The convention is the hard part. Enforce it with a commit linter, or with the merge
  request title where merges are squashed.

## 5.10 Check yourself

1. A repository has `semantic-release` configured and a `release` job that succeeds on
   every merge to `main`, but no release has ever been created. What is the most likely
   cause?
2. A commit reads `fix: rename the --output flag to --out`. What is wrong with it?
3. Why tag an image with both the version and the commit SHA, rather than choosing one?

:::{dropdown} Solutions
**1.** Almost certainly that the commit messages do not follow the convention, so the
commit analyzer finds no releasable change and exits successfully having done nothing.
A successful job is what makes this confusing: nothing failed, so nothing draws
attention to it. Check the job log for a line reporting that there are no relevant
changes.

The second candidate is a shallow clone. Without `GIT_DEPTH: 0` and a tag fetch, the
tool cannot see the previous release. That failure usually presents differently, as the
tool repeatedly trying to release `1.0.0`.

**2.** The type is wrong. Renaming a flag removes the old one, so any script using
`--output` breaks. That is a breaking change and must be `fix!:` or carry a
`BREAKING CHANGE:` footer, which makes it a major release. Released as a plain `fix:`
it ships as a patch, which tells every consumer the upgrade is safe when it is not.

The commit should also explain the migration in its body, because that text becomes the
release note the affected users will read.

**3.** They answer different questions for different audiences. A person deciding
whether to upgrade needs `1.4.1`, which carries the compatibility promise. A deployment
needs a reference that cannot change meaning, and a version tag can be moved or
rebuilt, whereas a commit SHA or a digest cannot. Using only the version breaks the
GitOps requirement from [section 6](06-gitops.md) that a Git commit describes exactly
one running state; using only the SHA leaves nobody able to tell which release is
deployed. Publishing both costs one extra line.
:::

---

Continue with: [6. What GitOps is](06-gitops.md).
