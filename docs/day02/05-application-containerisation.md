# 5. Application containerisation: Dockerfiles and registries

Packaging software applications into standardized container images requires writing deterministic,
reproducible build specifications.

This section analyzes **Dockerfile grammar and directive semantics**, layer caching optimization,
**multi-stage compilation patterns** for attack surface minimization, non-root security principles,
and the **OCI image distribution protocol**.

**Symbols:** ⌨️ you type the commands · 📖 reference material: read it after the lesson, or when you need it. The lesson does not cover it in class.

---

(id-5-1-dockerfile-specification-and-directives)=
## 5.1 ⌨️ The application

A **`Dockerfile`** is a declarative script defining the step-by-step instructions for assembling an OCI
image. Each instruction represents a discrete build step that either generates an immutable filesystem layer
or injects runtime metadata into the image configuration manifest.

Consider the following reference architecture for an ECMWF climate telemetry ingestion service:

```bash
mkdir -p ~/climate-service && cd ~/climate-service
```

Create `app.py`, a small web service written with [Flask](https://flask.palletsprojects.com/):

```python
import os
import socket

from flask import Flask

app = Flask(__name__)
app.json.sort_keys = False          # keep the JSON fields in the order written below


@app.get("/health")
def health():
    payload = {
        "status": "healthy",
        "service": "ecmwf-climate-telemetry",
        "version": os.getenv("APP_VERSION", "1.0.0"),
        "node": socket.gethostname()
    }
    return payload                  # Flask sends a dict as JSON, with status 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

`/health` answers with the state of the service, its version (read from the environment variable `APP_VERSION`) and the name of the host that runs it. Any other path returns `404`: Flask does that for you.

Create `requirements.txt`, with the two Python packages that the service needs:

```text
flask==3.1.3
gunicorn==26.2.0
```

`flask` is the web framework. `gunicorn` is the web server that runs it in the container: `app.run()` starts Flask's own server, which is meant for development only (it prints a warning when it starts).

---

(id-5-2-anatomy-of-a-production-grade-dockerfile)=
## 5.2 ⌨️ A Dockerfile for production, line by line

Create `Dockerfile` in `~/climate-service`:

```dockerfile
# 1. Declare immutable parent base image with explicit version pinning
FROM python:3.12-slim-bookworm

# 2. Configure runtime environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    APP_VERSION=1.0.0

# 3. Create an unprivileged user and group with a fixed ID (10001): principle of least privilege
RUN groupadd -g 10001 ecmwf && useradd -u 10001 -g ecmwf -d /app -s /sbin/nologin appuser

# 4. Define execution working directory
WORKDIR /app

# 5. Install the dependencies first: this layer is rebuilt only when requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 6. Transfer application artifact with correct POSIX ownership
COPY --chown=appuser:ecmwf app.py .

# 7. Drop root privileges prior to container execution: a number, so that Kubernetes can check it
USER 10001

# 8. Document exposed network port (Metadata only)
EXPOSE 8000

# 9. Declare executable entrypoint using Exec Form: gunicorn serves the Flask app "app" of app.py
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "1", "--threads", "4", "app:app"]
```

`app:app` means "the object `app` in the file `app.py`". One worker process with four threads answers up to four requests at the same time; to serve more users, you start more containers (on Day 3, more Pods). Day 5 explains why one process per container also matters for metrics.

### What each instruction does

| Instruction | Purpose | Rule |
|---|---|---|
| **`FROM`** | Declares the base image layer. | Explicit version pinning (e.g. `3.12-slim-bookworm`) is required for build reproducibility. Using mutable floating tags such as `latest` is prohibited in production. |
| **`WORKDIR`** | Sets working directory for subsequent instructions. | Guarantees an absolute directory path. **Never** use `RUN cd /path`, as each `RUN` statement spawns an isolated subshell that resets on exit. |
| **`COPY`** | Copies files from host build context into image. | Prefer `COPY` over `ADD`. `ADD` includes complex behaviors (auto-extracting tarballs and fetching remote URLs) that introduce security risks. |
| **`RUN`** | Executes build commands, committing a new layer. | Chain commands via `&&` and purge package manager caches within the **same** instruction (`rm -rf /var/lib/apt/lists/*`) to prevent cache bloat. |
| **`USER`** | Sets UID/GID for subsequent commands and runtime. | **Containers must not run as `root` (UID 0) in production.** Non-root execution restricts the blast radius if an application vulnerability is exploited. |
| **`EXPOSE`** | Documents intended network transport ports. | Documentation metadata only; does not configure host firewall or network routing. |
| **`CMD`** | Declares default command arguments. | Must utilize **Exec Form** (`["executable", "param"]`). |

:::{note} Why a fixed UID such as 10001
With `useradd -r`, the system chooses the first free UID in the image, often 999. That number can change with the base image, and on the host it usually belongs to a system user: in Hands-on 11, `docker top` would then show a name such as `systemd+`, which has nothing to do with your service. A fixed UID above 10000 is the same on every build and every host, never collides with a real user of the host, and, as a number in `USER`, lets Kubernetes check that the container is not `root` (Day 3).
:::

### Signals: exec form and shell form

A critical consideration in container engineering is the distinction between **Exec Form** and **Shell Form**:

- **Shell Form (`CMD gunicorn --bind 0.0.0.0:8000 app:app`):** Docker invokes a subshell process: `/bin/sh -c "gunicorn …"`. The subshell becomes PID 1. Standard Unix termination signals (`SIGTERM`) dispatched by container orchestrators are swallowed by `/bin/sh` and **never forwarded to gunicorn**. Consequently, graceful shutdown routines fail, and after 10 seconds Docker issues an ungraceful `SIGKILL`.
- **Exec Form (`CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]`):** Docker executes the binary directly as PID 1 without an intermediate shell. gunicorn receives `SIGTERM` directly, finishes the requests in progress, and terminates cleanly.

---

## 5.3 Layer cache and build context

Docker optimizes image construction by caching the output of each instruction. When rebuilding an
image, Docker evaluates whether the instruction and its inputs have changed. If unchanged, the cached
layer is reused.

### The order of the instructions matters

Any mutation in a layer invalidates all subsequent layers in the build sequence:

```dockerfile
# ❌ SUB-OPTIMAL CACHE STRUCTURE:
COPY . .                            # Invalidates cache on every single source code edit!
RUN pip install -r requirements.txt # Forces full package re-download on every build!
```

```dockerfile
# ✅ OPTIMIZED CACHE STRUCTURE:
COPY requirements.txt .             # Invalidated only when dependencies change
RUN pip install -r requirements.txt # Reused from cache across 95% of builds!
COPY app.py .                       # Only fast source file transfers execute on edits
```

The `Dockerfile` of section 5.2 follows the second structure: when you change only `app.py`, Flask and gunicorn are not downloaded again.

### Leave files out of the build: `.dockerignore`

When `docker build` executes, the Docker CLI transmits the entire current directory to the daemon as
the **build context**. To prevent large local artifacts, virtual environments, and secrets from entering
the context, define a `.dockerignore` file:

```text
# .dockerignore
.git/
.venv/
__pycache__/
*.pyc
*.pyo
*.pyd
.pytest_cache/
.coverage
htmlcov/
dist/
build/
.env*
```

---

## 5.4 Multi-stage builds

Scientific pipelines and compiled software (C++, Go, Rust, and Python with C-extension libraries)
require build tools (`gcc`, `g++`, `make`, `python3-dev`, `git`) during compilation.

If these utilities are retained in the final runtime image:
1. **Artifact Bloat:** Production images expand from $\approx 100\text{ MB}$ to $\approx 1.2\text{ GB}$.
2. **Elevated Attack Surface:** If an adversary compromises the application process, the container
   provides a full C compiler, package manager, and network download tools directly within the execution environment.

**Multi-stage builds** solve this by employing multiple `FROM` instructions within a single specification.
Artifacts are compiled in an ephemeral build stage, and only the finalized runtime binaries are
transferred into the production image.

```{mermaid}
%%{init: {"flowchart": {"subGraphTitleMargin": {"top": 5, "bottom": 35}}}}%%
flowchart TB
    subgraph BuildStage["Stage 1: builder"]
        direction TB
        B0["FROM python:3.12-bookworm"]
        B1["Install gcc, g++, libpq-dev, make"]
        B2["Compile dependencies: pip wheel --wheel-dir=/wheels"]
        B3["Image Size: ~980 MB"]
        B0 --> B1 --> B2 --> B3
    end

    subgraph RuntimeStage["Stage 2: runtime"]
        direction TB
        R0["FROM python:3.12-slim-bookworm"]
        R1["Install runtime shared libraries only (libpq5)"]
        R2["COPY --from=builder /wheels /wheels"]:::handoff
        R3["RUN install wheels only if present"]
        R4["Add unprivileged user (appuser) & application code"]
        R5["Final Image Size: ~125 MB"]
        R0 --> R1 --> R2 --> R3 --> R4 --> R5
    end

    B2 ==> R2
    linkStyle 8 stroke:#21B795,stroke-width:3px
    classDef handoff fill:#21B795,stroke:#178a70,stroke-width:2px,color:#ffffff,font-weight:bold
```

The highlighted step is the only link between the two stages: `COPY --from=builder` takes the compiled wheels from the first stage. Everything else in the first stage (compilers, headers, caches) stays behind.

:::{note} Jargon: *wheel*
A **wheel** (`.whl`) is a Python package that is already built: `pip` installs it by copying files, without a compiler. The `builder` stage compiles the wheels once; the runtime stage only installs them.
:::

### Example

```dockerfile
# ==========================================
# Stage 1: Compilation Environment
# ==========================================
FROM python:3.12-bookworm AS builder

WORKDIR /build

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir=/build/wheels -r requirements.txt

# ==========================================
# Stage 2: Minimal Production Runtime
# ==========================================
FROM python:3.12-slim-bookworm AS runner

WORKDIR /app

# Install runtime C dependencies only
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Transfer pre-compiled wheels from builder stage
COPY --from=builder /build/wheels /wheels
RUN if ls -1 /wheels/*.whl >/dev/null 2>&1; then pip install --no-cache-dir --no-index --find-links=/wheels /wheels/*.whl; fi && rm -rf /wheels

# Enforce non-root execution
RUN groupadd -g 10001 ecmwf && useradd -u 10001 -g ecmwf -d /app -s /sbin/nologin appuser
USER 10001

COPY --chown=appuser:ecmwf app.py .

EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "1", "--threads", "4", "app:app"]
```

The compilation environment (Stage 1) is completely discarded upon build completion. Only Stage 2
constitutes the final image manifest.

---

(id-5-5-image-construction-docker-build)=
## 5.5 ⌨️ Hands-on 11: build the `climate-telemetry` image

### Step 1: build the image

```bash
cd ~/climate-service
docker build -t climate-telemetry:1.0.0 .
```

```text
[+] Building 14.2s (11/11) FINISHED                                             docker:default
 => [internal] load build definition from Dockerfile                                       0.1s
 => [internal] load metadata for docker.io/library/python:3.12-slim-bookworm               1.4s
 => [internal] load .dockerignore                                                          0.0s
 => [1/6] FROM docker.io/library/python:3.12-slim-bookworm@sha256:e697c818…                4.5s
 => => sha256:3c5439d0… 28.25MB / 28.25MB                                                  2.2s
 => => sha256:5b676d7d… 13.68MB / 13.68MB                                                  1.2s
 => => sha256:a4516a4e… 3.34MB / 3.34MB                                                    0.8s
 => => sha256:560ad164… 250B / 250B                                                        0.2s
 => [internal] load build context                                                          0.1s
 => [2/6] RUN groupadd -g 10001 ecmwf && useradd -u 10001 -g ecmwf -d /app -s /sbin/nol…  1.0s
 => [3/6] WORKDIR /app                                                                     0.4s
 => [4/6] COPY requirements.txt .                                                          0.2s
 => [5/6] RUN pip install --no-cache-dir -r requirements.txt                               3.3s
 => [6/6] COPY --chown=appuser:ecmwf app.py .                                              0.3s
 => exporting to image                                                                     2.7s
 => => naming to docker.io/library/climate-telemetry:1.0.0                                 0.0s
```

The four `sha256:…` lines under `[1/6] FROM` are the layers of the base image `python:3.12-slim-bookworm`, downloaded the first time only (section 2.3). Steps 2 to 6 each add a layer of your own on top of them.

Change one line of `app.py` (for example, the text of a comment) and build again: steps 1 to 5 print `CACHED`, and only step 6 runs.

### Step 2: look at the image

```bash
docker images climate-telemetry:1.0.0
```

```text
REPOSITORY          TAG       IMAGE ID       CREATED          SIZE
climate-telemetry   1.0.0     4b892a0f7c12   20 seconds ago   162MB
```

### Step 3: start a container from the image

Start it in the background (`-d`), with the name `climate-telemetry`, and publish port 8000 of the container on port 8000 of the VM:

```bash
docker run -d --name climate-telemetry -p 8000:8000 climate-telemetry:1.0.0
```

Docker prints the ID of the new container.

### Step 4: call the service

```bash
curl -s http://localhost:8000/health
```

```text
{"status":"healthy","service":"ecmwf-climate-telemetry","version":"1.0.0","node":"7f3c2a9b1d04"}
```

The service answers through the published port. `node` is the host name of the container, that is, its short ID: compare it with `docker ps`.

### Step 5: check that the service does not run as `root`

First ask **inside** the container which user runs the service:

```bash
docker exec climate-telemetry id
```

```text
uid=10001(appuser) gid=10001(ecmwf) groups=10001(ecmwf)
```

Then look at the same processes **from the host**. `docker top` lists the processes of the container; `-o uid,user,pid,comm` shows the user ID, the user name, the process ID and the command:

```bash
docker top climate-telemetry -o uid,user,pid,comm
```

```text
UID     USER       PID       COMMAND
10001   10001      2780466   gunicorn
10001   10001      2780514   gunicorn
```

There are two `gunicorn` processes: the master, which starts and watches the worker, and the worker, which answers the requests. Both run with UID `10001`, not `0`: they are not `root`.

On the host the `USER` column shows the number again: `docker top` runs `ps` on the **host**, and the host looks up UID 10001 in **its own** `/etc/passwd`, where nobody has that number. Inside the container, the same 10001 is `appuser`. The kernel knows only the number; each side gives it the name it finds in its own files.

### Step 6: cleanup

Remove the container, so that port 8000 is free again:

```bash
docker rm -f climate-telemetry
```

---

(id-5-6-image-tagging-taxonomies-and-versioning)=
## 5.6 How to tag images

Image tags specify the version, architecture, or variant of an image manifest:

```text
[registry-domain[:port]/][namespace/]repository:tag
```

### Three kinds of tag

| Kind | Example | Use it? |
|---|---|---|
| **Semantic Versioning** | `climate-telemetry:1.4.2` | **Standard:** Enforces strict immutable release tracking. |
| **Git Commit SHA** | `climate-telemetry:sha-a8f102c` | **Standard:** Direct 1-to-1 traceability between running container and Git commit. |
| **Floating Latest** | `climate-telemetry:latest` | **Antipattern in Production:** Mutable pointer. Induces silent drift across cluster nodes. |

---

(id-5-7-oci-container-distribution-registries)=
## 5.7 ⌨️ Hands-on 12: push the image to your registry

Container images are distributed across networks via **OCI Container Registries** (GitLab Container
Registry, Harbor, Docker Hub, Quay.io).

```{mermaid}
flowchart LR
    Local["Your VM<br/>climate-telemetry:1.0.0"] -->|docker tag| Named["Your VM<br/>REGISTRY/climate-telemetry:1.0.0"]
    Named -->|docker push| Registry[("Container<br/>Registry")]
    Registry -->|docker pull| Node["Any host<br/>(on Day 3: a Kubernetes node)"]
```

`REGISTRY` is the address of the registry, plus the path of the project in it; in the steps below, it is your GitLab project. `docker tag` does not copy the image: it gives the image you built in Hands-on 11 a second name, the full path in the registry. `docker push` uploads it under that name, and any host that can log in to the registry can `docker pull` it.

### The steps

1. **Find your repository in GitLab (Web UI):**
    - Log in to [GitLab](https://gitlab.data.meditwin-project.eu) as `traineeNN`, with your claim code.
    - Open `https://gitlab.data.meditwin-project.eu/training/autumn-2026/climate-telemetry-traineeNN`
      (replace `NN` with your trainee number). It was created for you, private and empty: today it
      only holds your image, and on Day 4 you push the code of `~/climate-service` into this same
      project.
    - You are its Maintainer, and you see only your own projects: the other trainees' are invisible
      to you, as yours are to them.

    The registry belongs to the project, so the project must exist before the first `docker push`:
    pushing to a path with no project behind it is refused with *`insufficient_scope: authorization
    failed`*. If yours is missing, tell the instructor.

2. **Authentication:**
   ```bash
    docker login gitlab.data.meditwin-project.eu:6443
   ```
    Training VM credentials allow login and push to your repository.

3. **Tag the image with the registry path:**
   ```bash
    docker tag climate-telemetry:1.0.0 \
      gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:1.0.0
   ```

4. **Push:**
   ```bash
    docker push gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:1.0.0
   ```

5. **Pull:**
   ```bash
    docker pull gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:1.0.0
   ```

:::{note} On Day 3
On Day 3 a Kubernetes node pulls this image, not your VM: that is why it must be in a registry ([Day 3, section 3](../day03/03-deploying-an-application.md)).
:::

**Checkpoint 12** — `climate-telemetry:1.0.0` is in your GitLab registry, and it runs as `appuser`.

```bash
docker pull gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:1.0.0
docker run --rm gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:1.0.0 whoami
```

```text
1.0.0: Pulling from training/autumn-2026/climate-telemetry-traineeNN
Digest: sha256:…
Status: Image is up to date for gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:1.0.0
…
appuser
```

In GitLab, **Deploy → Container Registry** of your project lists the tag `1.0.0`. Keep `~/climate-service`: you use it again on Day 3 and Day 4.

:::{dropdown} Not there?
- `unauthorized` or `denied` on `docker push`: run `docker login gitlab.data.meditwin-project.eu:6443` again, with your trainee credentials.
- `insufficient_scope: authorization failed`: the path does not match your project. Check `traineeNN` in the tag (for example `climate-telemetry-trainee07`).
- `An image does not exist locally with the tag`: the `docker tag` step is missing. Run it, then push again.
- `whoami` prints `root`: the `USER 10001` line is missing in the `Dockerfile`. Add it, then build, tag and push again.
:::

## 5.8 Core takeaway

> **A good image is small, reproducible, runs as a non-root user, and lives in a registry with a version tag.**

- Pin the base image (`python:3.12-slim-bookworm`), not `latest`.
- Put the instructions that change least first (dependencies), and the code last: the **layer cache** does the rest.
- **Multi-stage builds** keep compilers out of the final image.
- A non-root user with a fixed UID (`USER 10001`), and a production server (gunicorn) started with the **exec form** of `CMD`.
- `docker tag` + `docker push` put the image in the registry, where Day 3 and Day 4 find it.

## 5.9 📖 Check yourself

1. You change only `app.py` and run `docker build` again. Which steps come from the cache, and why?
2. Why `COPY requirements.txt .` and `RUN pip install …` *before* `COPY app.py .`?
3. With `CMD gunicorn --bind 0.0.0.0:8000 app:app`, `docker stop` always takes 10 seconds. Why, and how do you fix it?
4. Why is `climate-telemetry:latest` a bad tag for a cluster?
5. You add a `/metrics` route to `app.py` and build `climate-telemetry:1.1.0`. In `docker history`, which layer changes compared with `1.0.0`?

:::{dropdown} Solutions
**1.** All the steps before `COPY … app.py`: their instruction and their input files did not change. `COPY app.py` and the steps after it run again.

**2.** So that a change to the code does not invalidate the `pip install` layer: the dependencies are downloaded again only when `requirements.txt` changes.

**3.** That is the shell form: it starts `/bin/sh` as PID 1, and the shell does not pass `SIGTERM` to gunicorn; after 10 seconds Docker sends `SIGKILL`. Use the exec form: `CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]`.

**4.** `latest` can point to a different image tomorrow: two nodes can run two different versions with the same name, and you cannot roll back to "the previous latest".

**5.** If you change only `app.py`, only the `COPY … app.py` layer is new; the layers before it keep the same IDs. If you also change `APP_VERSION` in the `ENV` line, every step from `ENV` on runs again, because `ENV` comes early in the Dockerfile.
:::

---

Continue with: [6. Microservices with Docker Compose](06-docker-compose.md).
