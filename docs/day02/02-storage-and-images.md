# 2. How Docker stores images and data

When you run an Ubuntu container, it boots in 200 milliseconds and takes only ~70 megabytes of disk space. How is that possible when a normal Ubuntu installation requires 4 to 10 gigabytes?

The answer lies in two core storage concepts: **Layered Images** and **Copy-on-Write (CoW)**.

**Symbols:** ⌨️ you type the commands · 📖 reference material: read it after the lesson, or when you need it. The lesson does not cover it in class.

---

## 2.1 Core concepts

- **Container Image:** An ordered, read-only stack of filesystem layers containing your application, runtime, and system packages.
- **Image Layer:** A recorded set of file additions, modifications, or deletions created during an image build.
- **Container Writable Layer:** When a container starts, Docker adds a thin, private, read-write layer on top of the image stack.
- **Copy-on-Write (CoW):** If a container modifies an existing image file, Docker copies that file up into the writable layer before modifying it. The underlying image layer is never changed.
- **Ephemeral Lifecycle:** When a container is deleted (`docker rm`), its writable layer is discarded.

**Practical mental model**:
> An image is like a printed reference book (read-only). Running a container is like laying a sheet of transparent tracing paper on top of the book. You can read the printed text underneath, and you can write notes on your tracing paper. When you throw away the tracing paper, the printed book is completely untouched.

```{mermaid}
%%{init: {"flowchart": {"subGraphTitleMargin": {"top": 5, "bottom": 25}}}}%%
flowchart TB
    subgraph C1["Container A"]
        W1["Writable layer A<br/>/etc/hosts (copy, changed)<br/>/root/notes.txt (new)"]
    end
    subgraph C2["Container B"]
        W2["Writable layer B<br/>(empty)"]
    end
    subgraph IMG["Image python:3.12-slim<br/>(read-only, stored once)"]
        L3["Layer 3: Python 3.12 runtime"]
        L2["Layer 2: C libraries"]
        L1["Layer 1: Debian base<br/>/etc/hosts (original)"]
        L3 --- L2 --- L1
    end
    W1 --> L3
    W2 --> L3
    L1 -. "copy on first write" .-> W1
    RM(["docker rm A"]) -. "deletes only" .-> W1
```

*Two containers, one image: each container writes only to its own layer; the image never changes.*

---

## 2.2 ⌨️ Image commands you use every day

Let's start by learning the fundamental image management commands you will use daily.

### 1. Downloading images (`docker pull`)

To download an image from Docker Hub to your local machine:

```bash
docker pull python:3.12-slim
```

```text
3.12-slim: Pulling from library/python
c96e625a47e7: Pull complete 
75d65427fa83: Pull complete 
8f7e27e8a93e: Pull complete 
Digest: sha256:4f1a238e5d...
Status: Downloaded newer image for python:3.12-slim
```

Notice the lines labeled `Pull complete`? Each line represents an independent **filesystem layer** being downloaded in parallel!

:::{note} Jargon: *digest*
The **digest** (`sha256:…`) is a hash of the image content. A tag such as `3.12-slim` can later point to a different image; a digest always points to the same bytes. Use it when you must be sure that two machines run exactly the same image.
:::

---

### 2. Inspecting local images (`docker images`)

To list all images currently stored on your machine:

```bash
docker images
```

```text
REPOSITORY   TAG         IMAGE ID       CREATED        SIZE
python       3.12-slim   8b7a421f1c98   2 days ago     154MB
ubuntu       24.04       e82ab14d3f5a   3 weeks ago    77.8MB
alpine       latest      91ef0af6111b   4 weeks ago    7.79MB
```

- **`REPOSITORY`**: The name of the image (e.g. `python`, `ubuntu`, `alpine`).
- **`TAG`**: The version or variant (e.g. `3.12-slim`, `24.04`, `latest`). When no tag is specified, Docker defaults to `:latest`.
- **`IMAGE ID`**: A unique cryptographic hash identifying the image.
- **`SIZE`**: The virtual uncompressed disk size of the image.

---

### 3. Removing local images (`docker rmi`)

When you are done with an image and want to free disk space:

```bash
docker rmi hello-world
```

*(Note: You cannot delete an image if an active or stopped container is currently using it).*

---

## 2.3 Layers: the core idea

Why does Docker use layers instead of one big `.zip` or `.iso` archive?

```{mermaid}
%%{init: {"flowchart": {"subGraphTitleMargin": {"top": 5, "bottom": 25}}}}%%
flowchart TB
    subgraph Image["Layered Image: python:3.12-slim"]
        direction TB
        L3["Layer 3: Python 3.12 Binary & Pip (+45 MB)"]
        L2["Layer 2: Shared C Libraries & SSL (+30 MB)"]
        L1["Layer 1: Debian Bookworm Minimal Rootfs (+79 MB)"]
        L3 --> L2 --> L1
    end
```

### The metaphor: transparent sheets

Imagine a stack of transparent plastic sheets placed on an overhead projector:
- **Sheet 1 (Base):** Has the core Debian Linux directory structure (`/bin`, `/etc`, `/usr`).
- **Sheet 2:** Adds Python binaries and libraries to `/usr/local/bin`.
- **Sheet 3:** Adds your application script to `/app`.

When you look down from the top, **you see a single unified filesystem**.

### Why layers save space: sharing and caching

Suppose you run 10 different containers on your server:
- 5 containers run Python APIs built on Debian.
- 5 containers run PostgreSQL databases built on Debian.

On a traditional Virtual Machine setup, you would need 10 separate virtual disks:
$$\text{10 VMs} \times 2\text{ GB} = 20\text{ GB of duplicated disk space}$$

With Docker layers:
- The base Debian layer (~80 MB) is downloaded and stored on your disk **exactly once**.
- All 10 containers share that exact same read-only layer simultaneously!
- **Bandwidth savings:** When you pull a new image that shares an existing layer, Docker prints `Already exists` and skips downloading it.

#### ⌨️ See the layers with `docker history`

Run `docker history` to see the layer construction of your Python image:

```bash
docker history python:3.12-slim
```

```text
IMAGE          CREATED        CREATED BY                                      SIZE
8b7a421f1c98   2 days ago     CMD ["python3"]                                 0B
<missing>      2 days ago     RUN /bin/sh -c set -eux; for src in idle3 pi…   32.4MB
<missing>      2 days ago     ENV PYTHON_GET_PIP_SHA256=...                   0B
<missing>      2 days ago     RUN /bin/sh -c set -eux; ...                    42.8MB
<missing>      2 weeks ago    # debian base layer                             78.9MB
```

Each line corresponds to one instruction of the Dockerfile that built the image (you write your own in Section 5). Only instructions that change files (`RUN`, `COPY`, `ADD`) create a filesystem layer; instructions that only set metadata (like `CMD` or `ENV`) show `0B`.

---

## 2.4 What happens when a container writes: Copy-on-Write (CoW)

If all containers share the same read-only image layers, **what happens when a container writes a log file, modifies a configuration file, or deletes a file?**

```
+-------------------------------------------------------------+
| Container Writable Layer (Upperdir)  <-- [Read / Write]     |
+-------------------------------------------------------------+
| Layer 3: Application Code            <-- [Read-Only]        |
| Layer 2: Python Runtime              <-- [Read-Only]        |
| Layer 1: Debian Base OS              <-- [Read-Only]        |
+-------------------------------------------------------------+
```

When a container is started:
1. Docker places a thin, private, empty **writable layer** on top of the image stack.
2. The running process sees a unified view of the image layers plus its writable layer.

### How file operations work

1. **Reading a file:** Docker looks from top to bottom. The first file found in the stack is read directly from its read-only layer.
2. **Creating a new file:** The file is written directly into the top writable layer. The image below is untouched.
3. **Modifying an existing file (The Copy-on-Write event):**
   - The kernel intercepts the write request.
   - It makes an instant copy of the specific file from the read-only image layer into the container's top writable layer.
   - The modification is applied **to that copy**.
   - The original file in the image layer remains completely unchanged!
4. **Deleting a file:**
   - Docker cannot delete files in the read-only layer.
   - Instead, it creates a special **whiteout marker** in the writable layer that hides the lower file from directory listings.

---

## 2.5 ⌨️ Hands-on 4: see Copy-on-Write with `docker diff`

Let's test this directly. Docker provides a built-in command, `docker diff`, which displays every file added, changed, or deleted in a container's writable layer.

### Step 1: start a test container

```bash
docker run -d --name cow-test alpine sleep 600
```

### Step 2: make three different changes to the files

```bash
# 1. ADD a new file:
docker exec cow-test touch /root/my_experiment.txt

# 2. CHANGE an existing system file:
docker exec cow-test sh -c "echo '127.0.0.1 custom-host' >> /etc/hosts"

# 3. DELETE a base operating system file:
docker exec cow-test rm /etc/alpine-release
```

### Step 3: list the changes

```bash
docker diff cow-test
```

```text
C /etc
C /etc/hosts
D /etc/alpine-release
C /root
A /root/my_experiment.txt
```

Notice the status flags:
- **`A` (Added):** Newly created file in the container's writable layer.
- **`C` (Changed):** Copied from the read-only base layer to the writable layer and modified.
- **`D` (Deleted):** Masked by a whiteout marker so it is invisible inside the container.

### Step 4: check that the image did not change

Remove the container:
```bash
docker rm -f cow-test
```

Now start a brand-new container from the same `alpine` image:
```bash
docker run --rm alpine cat /etc/alpine-release
```

Notice the output: the file `/etc/alpine-release` is right there, completely intact!
Because changes were isolated to `cow-test`'s writable layer, destroying the container destroyed the changes, leaving the base image 100% pristine.

**Checkpoint 4** — the `alpine` image is unchanged, and `cow-test` is gone.

```bash
docker run --rm alpine cat /etc/alpine-release
docker ps -a --filter name=cow-test --format '{{.Names}}'
```

The first command prints the Alpine version (for example `3.20.3`); the second prints nothing.

:::{dropdown} Not there?
- `docker diff` shows more lines than the example (for example `/root/.ash_history`): that is normal. Look for the `A`, `C` and `D` lines of your three changes.
- `Conflict. The container name "/cow-test" is already in use`: a `cow-test` from an earlier try still exists. `docker rm -f cow-test`, then start again from Step 1.
- `cow-test` is still listed by the second command: `docker rm -f cow-test`.
:::

---

## 2.6 📖 Under the hood: the OverlayFS storage driver

In Linux, this layering is implemented by the **OverlayFS** kernel module (called `overlay2` in Docker):

```{mermaid}
flowchart TB
    Merged["Merged View: What the Container Process Sees (/)"]
    Upper["upperdir: Container Writable Layer (Read/Write)"]
    Lower2["lowerdir 2: Application Layer (Read-Only)"]
    Lower1["lowerdir 1: Base OS Layer (Read-Only)"]

    Merged -.-> Upper
    Merged -.-> Lower2
    Merged -.-> Lower1

    Upper --> Lower2 --> Lower1
```

- **`lowerdir`**: The directory paths on the host holding the read-only image layers.
- **`upperdir`**: The container's private writable directory on the host.
- **`workdir`**: An internal staging directory used by the kernel to ensure atomic file operations.
- **`merged`**: The virtual mount point presented to the container as its root directory (`/`).

### How much disk Docker uses (`docker system df`)

To see how much host disk space is consumed by images, containers, and volumes:

```bash
docker system df
```

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          3         1         239MB     161MB (67%)
Containers      1         0         12B       12B (100%)
Local Volumes   0         0         0B        0B
Build Cache     0         0         0B        0B
```

Notice that containers themselves consume virtually zero extra disk space (`12B` in this example), because almost all data is shared in the base images!

---

## 2.7 Why containers need volumes

Now you understand the trade-off of the container writable layer:

1. **State Volatility:** Containers are disposable. When a container is removed (`docker rm`), its writable layer is permanently erased.
2. **Performance:** Modifying large files (like database tables) through Copy-on-Write adds I/O overhead.

> **Rule of Thumb:**
> Never store persistent database data or user uploads directly inside a container's writable layer.
>
> For stateful workloads, Docker provides **Volumes** and **Bind Mounts**, which bypass the layered filesystem and write directly to host storage at full native speed. We will cover this in [Section 4: Persisting Data](04-persisting-data-and-web-ui.md).

## 2.8 Core takeaway

> **An image is a stack of read-only layers; a container adds one thin writable layer on top, and loses it when it is removed.**

- Each `RUN`, `COPY` or `ADD` instruction creates a **layer**; layers are shared between images and containers, and downloaded once.
- When a container changes a file of the image, **Copy-on-Write** copies it to the writable layer first: the image never changes.
- `docker diff` shows what a container added (`A`), changed (`C`) and deleted (`D`).
- Data that must survive the container goes in a **volume** (section 4), not in the writable layer.

## 2.9 📖 Check yourself

1. You pull `python:3.12-slim` and then an image built on it. Why do some lines print `Already exists`?
2. A container deletes `/etc/alpine-release`. Is the file deleted from the image? What does Docker write instead?
3. `docker history` shows `0B` for `CMD` and `ENV`. Why?
4. You run 10 containers from the same 150 MB image. Roughly how much extra disk do they use before they write anything?
5. Why is the writable layer a bad place for a database?

:::{dropdown} Solutions
**1.** The two images share those layers. Docker already has them on the disk, so it does not download them again.

**2.** No: the image layers are read-only. Docker writes a **whiteout** marker in the writable layer, which hides the file inside that container only.

**3.** They only set metadata in the image configuration; they do not add or change files, so they create no file system layer.

**4.** Almost nothing: the 10 containers share the image layers, and each writable layer is empty until the container writes.

**5.** The data is lost when the container is removed, and Copy-on-Write makes writes to large files slower. A volume avoids both problems.
:::

---

Continue with: [3. Running containers & exposing services](03-running-containers.md).
