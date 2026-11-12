# 1. Git from zero

Git is the tool underneath everything else today. If your knowledge of Git is shaky,
CI/CD and GitOps will feel unfamiliar rather than straightforward. This section covers
it first.

**Slides:** the slides used in class are in one file, {download}`day04-git-cicd-gitops.pdf`.

## 1.1 The problem Git solves

Imagine you are writing a report with two friends. Without any tooling you end up with:

```
report.docx
report_v2.docx
report_v2_FINAL.docx
report_v2_FINAL_marco_edits.docx
report_v2_FINAL_marco_edits_USE_THIS_ONE.docx
```

You cannot answer basic questions: *what changed between v2 and FINAL? who wrote this paragraph?
can I get back yesterday's version?* And if two people edit at once, one of them loses work.

Git answers all of those. It keeps **every** version, forever, labeled with who did it and why,
and it can merge two people's simultaneous work.

:::{note} Jargon: *version control*
A **version control system** records the history of a set of files so you can inspect it, go back to
any point, and combine changes made by different people. Git is by far the most used one.
:::

## 1.2 The one idea you must get: a commit is a snapshot

Most confusion about Git comes from imagining it stores *changes*. It does not. **Git stores
snapshots.**

A **commit** is a photograph of your entire project at one moment, plus a label saying who took it,
when, and why. Commits are chained: each one knows which commit came before it.

```{mermaid}
flowchart LR
    A["commit a1b2c3<br/>'Add login page'"] --> B["commit d4e5f6<br/>'Fix typo in title'"]
    B --> C["commit 789abc<br/>'Add password check'"]
```

Reading that chain from left to right *is* your project's history. `git log` prints it.

:::{important} Why this matters later
GitOps ([section 6](06-gitops.md)) works because a commit is a complete, exact description of a desired state. If a
commit were only a partial diff, "make the cluster look like commit `789abc`" would be meaningless.
:::

## 1.3 ⌨️ Your first repository, for real

⏱ **Type along with the instructor**, about 5 minutes.

Type along. Every line here works on your training VM.

```bash
mkdir ~/hello-git && cd ~/hello-git
git init
```

```text
Initialized empty Git repository in /home/cloud-user/hello-git/.git/
```

That `.git` folder is the entire database. Delete it and you have a normal folder again; nothing
else on your machine was touched.

Tell Git who you are (once per machine):

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
```

Now make a file and take your first snapshot:

```bash
echo "# My project" > README.md
git status
```

```text
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	README.md

nothing added to commit but untracked files present (use "git add" to track)
```

Git is telling you: *I can see `README.md`, but I am not watching it yet.* Two steps to commit:

```bash
git add README.md          # 1. choose what goes in the snapshot
git commit -m "Add README" # 2. take the snapshot, with a reason
```

```text
[main (root-commit) 4f3a1c9] Add README
 1 file changed, 1 insertion(+)
 create mode 100644 README.md
```

Look at your history:

```bash
git log --oneline
```

```text
4f3a1c9 (HEAD -> main) Add README
```

You have a project with history. That is the whole of the basic loop.

### The two-step feels weird. Why?

Because `git add` lets you commit *some* of your changes. Suppose you fixed a bug **and** changed
some colours, and you want two separate commits so each is easy to review:

```bash
git add src/bugfix.py
git commit -m "Fix crash when the input file is empty"
git add src/theme.css
git commit -m "Make the header dark blue"
```

Two clean commits instead of one messy one. The area you `git add` into is called the **staging
area** (or *index*).

:::{note} Jargon: *staging area*
The **staging area** is a waiting room. `git add` puts changes in the waiting room; `git commit`
takes everything in the waiting room and makes it a snapshot.
:::

```{mermaid}
flowchart LR
    W["Working directory<br/>(files you edit)"] -->|git add| S["Staging area<br/>(waiting room)"]
    S -->|git commit| R["Repository<br/>(history)"]
    R -->|git checkout| W
```

(d4-1-4)=
## 1.4 Writing commit messages that are not useless

You will read your own history far more often than you write it. Compare:

| ❌ Useless | ✅ Useful |
|---|---|
| `fix` | `Fix crash when the input file is empty` |
| `stuff` | `Add retry logic to the S3 upload` |
| `asdf` | `Bump base image to python:3.12 for the CVE fix` (a CVE is a published security flaw; [section 4.3](#d4-4-3)) |
| `changes as requested` | `Rename user_id to account_id for consistency` |

The rule: **finish the sentence "If applied, this commit will…"**. Use the imperative mood
("Add", not "Added"), keep the first line under ~70 characters, and if there is a *why* that is not
obvious, add a blank line and explain it:

```text
Bump base image to python:3.12

3.11 reached end of security support and the scanner flagged
CVE-2025-1234. No application code changes were needed.
```

:::{tip} Conventional Commits
Many teams prefix messages with a type: `feat:`, `fix:`, `docs:`, `refactor:`,
`test:`, `chore:`.
So `fix: handle empty input file`. It reads well and lets tools generate
changelogs automatically.
:::

Here is a real one, as GitLab shows it:

:::{figure} img/gitlab-commit-diff.png
:alt: A GitLab commit page showing a fix(ci) subject line above a detailed explanatory body.
**A commit message doing its job.** The subject says *what* changed in one line, and the body
below it says *why* — the symptom, the cause, and what was done instead. Six months from now
this is the only record of that reasoning. [Section 5](05-releases.md) turns the `fix:` prefix
into a version number automatically.
:::

## 1.5 ⌨️ Branches: working without breaking things

⏱ **Type along with the instructor**, about 4 minutes.

A **branch** is a movable label pointing at a commit. That is the whole definition. Creating one is
instant and costs nothing, because it is literally a 40-character file.

The point of branches is: you can build something half-finished without disturbing the version
everybody else uses.

```bash
git switch -c add-greeting      # create a branch and move onto it
echo "Hello, world!" > hello.txt
git add hello.txt
git commit -m "Add a greeting file"
```

Now compare:

```bash
git log --oneline --all --graph
```

```text
* 8c1d2e5 (HEAD -> add-greeting) Add a greeting file
* 4f3a1c9 (main) Add README
```

`main` is still at the old commit. Your work is on `add-greeting`. Prove it:

```bash
git switch main
ls
```

```text
README.md
```

`hello.txt` is *gone* — because on `main` it never existed. Switch back and it returns:

```bash
git switch add-greeting
ls
```

```text
README.md  hello.txt
```

Nothing was lost. You are just looking at a different snapshot.

```{mermaid}
gitGraph
    commit id: "Add README"
    branch add-greeting
    checkout add-greeting
    commit id: "Add a greeting file"
    checkout main
    commit id: "Update README"
    merge add-greeting
```

### Merging

When the branch is finished, **merge** it back:

```bash
git switch main
git merge add-greeting
```

```text
Updating 4f3a1c9..8c1d2e5
Fast-forward
 hello.txt | 1 +
 1 file changed, 1 insertion(+)
```

"Fast-forward" means nothing had changed on `main` in the meantime, so Git just slid the label
forward. When both branches have moved, Git creates a **merge commit** joining the two histories —
that is normal and fine.

:::{note} Jargon: `HEAD`
`HEAD` means "the commit I am currently looking at", usually via a branch name. When `git status`
says *On branch main*, it means `HEAD` → `main` → some commit.
:::

## 1.6 Remotes: sharing with other people

So far everything lives on your laptop. A **remote** is a copy of the repository somewhere else —
on GitLab, for example. The default remote is conventionally named `origin`.

```bash
git remote add origin https://gitlab.example.com/you/hello-git.git
git push -u origin main
```

`push` sends your commits up; `pull` brings other people's commits down.

```{mermaid}
flowchart TB
    L1["Your laptop"] -->|push| R[("GitLab: origin")]
    R -->|pull| L1
    R -->|clone| L2["A colleague's laptop"]
    L2 -->|push| R
```

To start from an existing project instead, **clone** it:

```bash
git clone https://gitlab.example.com/team/some-project.git
cd some-project
```

`clone` gives you the full history, not just the latest files. You can read the entire past of the
project offline.

:::{danger} Never put a password or token in the remote URL
You will see URLs like `https://user:glpat-AbCdEf123@gitlab.example.com/…`. The `glpat-…` part is a
GitLab **personal access token**: a password-like string GitLab issues to one user, for scripts and
tools, with its own permissions and expiry date. Putting it in the URL works, and it is a
bad idea: the token is stored in plain text in `.git/config`, and it leaks the moment you paste
`git remote -v` output into a chat, a screenshot, or a bug report. Use SSH keys or a credential
helper instead. We come back to this on [section 4](04-pipeline-security.md).
:::

## 1.7 `.gitignore`: what not to track

Some files must never be committed: build output, virtual environments, editor droppings, and above
all **secrets**. List them in a file called `.gitignore`:

```text
# Python
__pycache__/
*.pyc
.venv/

# Build output
dist/
build/

# Secrets — never commit these
.env
*.key
*.pem
credentials.json

# macOS
.DS_Store
```

Commit the `.gitignore` file itself — everyone on the project should share it.

:::{warning} `.gitignore` does not un-commit anything
It only stops Git from *starting* to track a file. If you already committed a secret, it is in the
history forever, on every clone, even if you delete the file in a later commit. The only real fix is
to **rotate the secret** (invalidate it and issue a new one). Rewriting history is possible but does
not help once other people have pulled.
:::

## 1.8 ⌨️ Hands-on 1: put your Day 2 service under Git

⏱ **About 10 minutes.**

Today you keep working on the climate telemetry service you containerised on
[Day 2](../day02/05-application-containerisation.md). By the end of the day it will have a pipeline
([section 3](03-gitlab-ci.md)), a security gate ([section 4](04-pipeline-security.md)), and Fleet will deploy it ([section 7](07-fleet.md)). Everything starts with
the project being in Git.

```bash
cd ~/climate-service
ls
git status
```

It answers `fatal: not a git repository`: the folder is not under Git yet. Create the repository:

```bash
git init -b main
```

Create `.gitignore` first, so nothing unwanted is ever tracked:

`.gitignore`

```text
__pycache__/
*.pyc
.venv/
.pytest_cache/
.env
*.key
*.pem
```

Now make two small commits, one per idea:

```bash
git add .gitignore
git commit -m "chore: ignore Python caches and local secrets"

git add app.py Dockerfile requirements.txt
git status --short        # anything else from Day 2, such as .dockerignore? git add it too
git commit -m "feat: climate telemetry service with a /health endpoint"
```

Prove the `.gitignore` works:

```bash
echo "API_TOKEN=not-a-real-token" > .env
git status --short
```

`.env` does not appear in the output. Git cannot see it, so nobody can commit it by accident.

**Checkpoint 1** — `git log --oneline` shows your two commits, and `git status --short` lists
nothing.

## 1.9 ⌨️ Merge conflicts

⏱ **Type along with the instructor**, about 4 minutes.

A **conflict** happens when two branches changed *the same lines* of the same file. Git will not
guess which one you meant; it asks you.

Create one on purpose.

```bash
cd ~/hello-git
git switch main
echo "Colour: blue" > colour.txt
git add colour.txt && git commit -m "Set colour to blue"

git switch -c make-it-red
echo "Colour: red" > colour.txt
git add colour.txt && git commit -m "Set colour to red"

git switch main
echo "Colour: green" > colour.txt
git add colour.txt && git commit -m "Set colour to green"

git merge make-it-red
```

```text
Auto-merging colour.txt
CONFLICT (content): Merge conflict in colour.txt
Automatic merge failed; fix conflicts and then commit the result.
```

Open `colour.txt`:

```text
<<<<<<< HEAD
Colour: green
=======
Colour: red
>>>>>>> make-it-red
```

Read it as: *between `<<<<<<<` and `=======` is what **your current branch** says; between
`=======` and `>>>>>>>` is what **the branch you are merging in** says.*

Fix it by editing the file into whatever is actually correct — including the possibility of
something new — and deleting all three marker lines:

```text
Colour: teal
```

Then tell Git you have resolved it:

```bash
git add colour.txt
git commit -m "Merge make-it-red, settling on teal"
```

Done. Three things worth knowing:

- `git merge --abort` backs out completely if you want to start over. Nothing is lost.
- `git status` during a conflict tells you exactly which files still need attention.
- Conflicts are a *sign of collaboration*, not a mistake. Small, frequent merges produce fewer of
  them than one giant merge at the end of the month.

## 1.10 Merge requests: how teams actually work

In practice you do not push straight to `main`. You push a branch and open a **merge request**.

:::{note} Jargon: *merge request* (MR) / *pull request* (PR)
A **merge request** is a proposal: "please merge my branch into `main`". It gives you a place to
discuss the change, an automatic list of every line you touched, and — importantly for today — a
hook where the CI pipeline runs *before* anything reaches `main`. GitLab calls it a merge request;
GitHub calls the same thing a pull request.
:::

The everyday loop, which you will use in Hands-on 2 and for the rest of the day:

```bash
git switch main
git pull                                  # 1. get up to date
git switch -c fix/empty-input-crash       # 2. branch, named after the work
# ... edit files ...
git add -p                                # 3. stage (interactively, if you like)
git commit -m "fix: handle empty input file"
git push -u origin fix/empty-input-crash  # 4. push the branch
```

GitLab replies with a link that opens the merge request for you:

```text
remote: To create a merge request for fix/empty-input-crash, visit:
remote:   https://gitlab.example.com/you/project/-/merge_requests/new?...
```

Then: pipeline runs → colleague reviews → merge → branch deleted. That merge is the event that will
trigger a deployment later today.

:::{figure} img/gitlab-mr-overview.png
:alt: A merged GitLab merge request showing a passing branch pipeline, the merge, and a second pipeline on the main branch afterwards.

**A merge request after it has been merged.** Read it top to bottom: the branch was proposed into
`main`; **Pipeline #429 passed** on the branch, *before* anyone merged; the merge happened and
deleted the source branch; then **Pipeline #430 passed** on `main`. Two pipelines for one change
is the point — the first one decides whether merging is safe, the second one is what actually
ships. Names are blurred because this is somebody else's project.
:::

```{mermaid}
%%{init: {"sequence": {"actorMargin": 12, "width": 130, "diagramMarginX": 12}}}%%
sequenceDiagram
    participant You
    participant GitLab
    participant CI as CI pipeline
    participant Rev as Reviewer
    You->>GitLab: push branch
    You->>GitLab: open merge request
    GitLab->>CI: run pipeline on the branch
    CI-->>GitLab: ✅ tests pass
    Rev->>GitLab: review, approve
    You->>GitLab: merge into main
    GitLab->>CI: run pipeline on main
```

## 1.11 ⌨️ Hands-on 2: your first merge request

⏱ **About 15 minutes.**

Log in to `https://gitlab.data.meditwin-project.eu` as `traineeNN`, with your claim code as the
password: the same code that opened the key portal. To list your projects, use **Search or go to**
in the centre of the top bar → **Your work**, or open `https://gitlab.data.meditwin-project.eu/dashboard/projects`.
You see exactly three projects, all yours, all created for you in the group **training/autumn-2026**: `climate-telemetry-traineeNN`,
`climate-telemetry-traineeNN-deploy` and `hello-ci-traineeNN`. The other trainees' projects are
invisible to you, and you cannot create projects in that group.

This is the first time you use GitLab today, so here is where things are:

:::{figure} img/ui-gl-overview.png
:alt: A GitLab project page with the top bar and the left sidebar marked.
**A project page.** ① **Search or go to**: type the name of a project to open it. ② **+**
(*Create new*). ③ Your **avatar**. ④ The **left sidebar**, with one group per area of the project:
Manage, Plan, **Code**, **Build**, Secure, **Deploy**, Operate, Monitor, Analyze, and **Settings**
last. ⑤ The **Code** button: the URLs to clone the project.
:::

A path such as *Deploy → Container Registry* means: in the left sidebar, open the group **Deploy**,
then click **Container Registry** in the list that opens under it. The **Pinned** list at the top of
the sidebar repeats a few common pages, such as Merge requests and Pipelines.

Open the project you pushed your image to on [Day 2, section 5.7](../day02/05-application-containerisation.md),
`training/autumn-2026/climate-telemetry-traineeNN`. It already holds your Day 2 image (left sidebar:
**Deploy → Container Registry**), but no code: the repository is empty, waiting for the history on
your VM.

Connect the folder to it and push:

```bash
cd ~/climate-service
git remote add origin git@gitlab.data.meditwin-project.eu:training/autumn-2026/climate-telemetry-traineeNN.git
git push -u origin main
```

GitLab shows the exact clone URL under the blue **Code** button on the project page. Copy it from
there rather than typing it.

:::{dropdown} There is no `climate-telemetry-traineeNN` project
Tell the instructor: your projects are created with your account, and you cannot create projects in
`training/autumn-2026` yourself.
:::

:::{dropdown} The push is rejected: *`! [rejected] main -> main (fetch first)`*
GitLab's `main` already has a commit that your VM does not have, for example a README created in
the web interface.
Join the two histories once, then push:
```bash
git pull origin main --allow-unrelated-histories --no-edit
git push -u origin main
```
:::

:::{figure} img/ui-gl-avatar.png
:alt: The GitLab avatar menu open, with Edit profile and Preferences marked.
**The avatar menu** (top right). ① **Edit profile**, ② **Preferences**.
:::

Your VM is already set up for this: your key is registered on your GitLab account (your
**avatar**, top right → **Edit profile** → left menu **Access → SSH keys** lists it) and `~/.ssh/config` tells `ssh` to use it. Check with:

```bash
ssh -T git@gitlab.data.meditwin-project.eu
```

```text
Welcome to GitLab, @traineeNN!
```

:::{dropdown} `Permission denied (publickey)`
GitLab does not know the key `ssh` is offering. Check that **avatar → Edit profile → Access → SSH keys** lists a key, and that
`~/.ssh/config` has a `Host gitlab.data.meditwin-project.eu` entry. To use a key of your own instead,
create one with `ssh-keygen -t ed25519` and add `~/.ssh/id_ed25519.pub` there. If you would rather use
HTTPS, use the HTTPS URL and a credential helper — never a token inside the URL:
```bash
git config --global credential.helper store
```
:::

Now make a change the way teams do, on a branch:

```bash
git switch -c docs/readme
```

`README.md`

```markdown
# climate-telemetry

A small HTTP service that reports its health on `/health`, port 8000.

Run it locally:

    docker build -t climate-telemetry:dev .
    docker run --rm -p 8000:8000 climate-telemetry:dev
    curl -s localhost:8000/health
```

```bash
git add README.md
git commit -m "docs: explain how to run the service locally"
git push -u origin docs/readme
```

Open the link GitLab prints after the push (or, on the project page, the **Create merge request**
button GitLab shows for a branch you just pushed), and press **Create merge request**. Before merging,
open the merge request's **Changes** tab: that list of lines is what a reviewer reads. Review it
yourself (your neighbour cannot open your project, but can look at your screen), then press
**Merge**.

Back on your machine, catch up and tidy:

```bash
git switch main
git pull
git branch -d docs/readme
git log --oneline --graph
```

**Checkpoint 2** — the project page in GitLab shows your `README.md`, the merge request is marked
*Merged*, and your local `main` contains the merge.

(undoing-things)=
## 1.12 Undoing things

Choose the command that matches the situation:

| I want to… | Command |
|---|---|
| Throw away uncommitted edits to one file | `git restore path/to/file` |
| Unstage something I `git add`ed (keep the edit) | `git restore --staged path/to/file` |
| Change the message of the commit I just made | `git commit --amend` |
| Undo a commit but keep the changes as edits | `git reset --soft HEAD~1` |
| Undo a commit *and* the changes (careful) | `git reset --hard HEAD~1` |
| Undo a commit that is already pushed | `git revert <commit>` |
| See what I did, even if it looks lost | `git reflog` |

The important distinction:

- **`git reset`** rewrites history. Acceptable on your own unpushed branch, disruptive on a shared one.
- **`git revert`** makes a *new* commit that undoes an old one. History stays intact, so it is safe
  on shared branches. **This is the command GitOps rollbacks are built on** — remember it for
  [section 6](06-gitops.md).

:::{tip} `git reflog` is the undo button for your undo button
Git keeps a log of everywhere `HEAD` has been for ~90 days, even across a `--hard` reset. If you
think you destroyed work, run `git reflog`, find the commit, and `git switch -c rescue <sha>`. It is
very hard to lose a commit that was once made.
:::

## 1.13 Core takeaway

> **A commit is a snapshot, a branch is a label, and a merge request is where a change is checked
> before it reaches `main`.**

- A **commit is a snapshot** of the whole project, not a diff. History is a chain of snapshots.
- **`add` then `commit`**: the staging area exists so you can make small, meaningful commits.
- A **branch is just a label** on a commit. Creating one is free; use them for everything.
- A **remote** (`origin`) is a shared copy. `push` up, `pull` down, `clone` to start.
- **Never commit secrets.** `.gitignore` prevents; only rotation fixes.
- **`revert`** is the safe undo for anything already shared. It is how you roll back a deployment.

## 1.14 Check yourself

Try these before looking. Use a throwaway repository.

1. You committed a file containing your API token. You then deleted the file and committed again.
   Is the token safe? What must you actually do?
2. You are on `main` with uncommitted work, and you need to switch to a colleague's branch right
   now. What are your options?
3. `git reset --hard` versus `git revert` — for a commit already pushed to `main`, which one, and
   why?

:::{dropdown} Solutions
**1.** No. The token is in the commit that added it, and that commit is in the history of every
clone and every fetch. Deleting the file later changes nothing about the past. You must **rotate the
token**: revoke it in the provider's UI and issue a new one. Then add the file to `.gitignore` so it
cannot happen again. Rewriting history (`git filter-repo`) may be worth doing afterwards for
hygiene, but it is *not* the fix — anyone who already cloned still has it.

**2.** Either commit the work (even a scrappy `wip:` commit on your own branch — you can always
`--amend` or reset it later), or stash it:
```bash
git stash              # tuck the changes away
git switch colleagues-branch
# ... look around ...
git switch main
git stash pop          # bring them back
```
Git also refuses to switch branches if it would silently destroy your edits, so you will get a clear
error rather than data loss.

**3.** **`git revert`.** `reset --hard` rewrites history: your `main` and everybody else's `main`
would disagree about what commits exist, and you would need a force-push to publish it — which
breaks every colleague's next `pull` and can destroy their work. `revert` adds a new commit that
undoes the old one, so history only ever moves forward and everyone's `pull` just works.
:::

---

Continue with: [2. What CI/CD actually is](02-ci-cd.md).
