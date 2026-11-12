# 1. Containers and Docker: advantages and disadvantages

Containers are the foundation of modern cloud computing, microservices, and reproducible data science. Before moving to multi-container orchestration with Docker Compose and Kubernetes (Day 3), you need to understand how a single container works: **what problem it solves, how to run one, why it differs fundamentally from a virtual machine, and how the Linux kernel keeps it isolated**.

**Symbols:** ⌨️ you type the commands · 📖 reference material: read it after the lesson, or when you need it. The lesson does not cover it in class.

---

(id-1-1-the-problem-containers-solve-it-works-on-my-machine)=
## 1.1 The problem containers solve: "it works on my machine"

Every developer and student encounters this scenario:

> You write a Python climate analysis script or Earth observation data pipeline on your laptop. It works perfectly. You send it to a classmate, professor, or push it to a compute cluster, and it immediately crashes:
> - Your laptop has Python 3.12; the server has Python 3.10.
> - A geospatial or C-library dependency (`netcdf4`, `gdal`, `libssl`) is missing or has a different version.
> - An environment variable or dataset path exists on your laptop but not on the server.

```
Developer Laptop (macOS / Python 3.12)  ---> "Works on my machine!"
                       |
                       v
Server / Cluster (Ubuntu / Python 3.10) ---> ImportError: libsomething.so not found!
```

This is the **environment mismatch problem**.

### The shipping container metaphor

Before 1956, shipping cargo was slow, expensive, and chaotic. Goods were transported in loose crates, barrels, and sacks of all shapes and sizes. Dockworkers spent days manually loading and unloading ships, and goods frequently broke in transit.

In 1956, the **standardized steel shipping container** was invented:
- It doesn't matter what is inside the container (cars, coffee beans, electronics).
- Every crane, truck, train, and ship in the world is built to handle that exact standardized box.

```
+-------------------------------------------------------------+
| Physical World:                                             |
| Cargo (any shape) + Standard Container  --> Ship / Truck    |
|                                                             |
| Software World:                                             |
| Code + Libraries  + Standard Image      --> Any Cloud / Host|
+-------------------------------------------------------------+
```

**Docker does the exact same thing for software**:
It packages your application code, runtime (e.g. Python 3.12), system libraries, and filesystem configurations into a single standardized unit called a **container image**. If it runs on your laptop, it runs identically on any Linux server, in CI/CD pipelines, or on a Kubernetes cluster.

---

## 1.2 ⌨️ Hands-on 1: your first containers

Let's start directly in the terminal.

### Step 1: run your first container

Type this into your terminal:

```bash
docker run hello-world
```

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client...
```

Notice what Docker did automatically behind the scenes:
1. It checked your local machine for the `hello-world` image.
2. When it didn't find it locally, it downloaded (pulled) it from **Docker Hub** (the public registry).
3. It created an isolated container, executed the program, printed the output, and exited.

---

### Step 2: an interactive Linux terminal inside a container

Now run an entire Ubuntu Linux environment:

```bash
docker run -it --rm ubuntu:24.04 bash
```

Notice your terminal prompt changed:
```text
root@e82ab14d3f5a:/#
```

You are now sitting inside an isolated Ubuntu 24.04 shell! Let's explore:

```bash
# Check who you are
whoami

# Check the operating system version
cat /etc/os-release

# Check the running processes
ps aux

# Leave the container
exit
```

When you typed `exit`, the container stopped. Because we added `--rm`, Docker automatically cleaned up the stopped container.

Notice something remarkable: **starting that Ubuntu environment took less than one second**. Why was it so fast? That brings us to the core distinction of container technology.

---

## 1.3 Image and container

Before going further, keep this distinction clear:

| Concept | What It Is | Real-world Analogy | Programming Analogy |
|---|---|---|---|
| **Image** | A frozen, read-only blueprint containing code, libraries, and files. | A recipe or blueprint | A `class` |
| **Container** | A live, running process created from an image, with its own private writable layer. | The baked cake or house | An instantiated `object` |

You can launch 1, 10, or 100 containers from a single image. They all start from the identical blueprint, but each runs independently.

---

## 1.4 Virtual machines and containers

Why did the Ubuntu container boot in 200 milliseconds, while booting an Ubuntu Virtual Machine in VirtualBox or VMware takes 1 to 2 minutes?

```{mermaid}
flowchart TB
    subgraph VM["Virtual Machine (Hardware Virtualization)"]
        direction TB
        App1["App A"] --> Lib1["Libs / Binaries"] --> GOS1["Guest OS Kernel<br/>(Full Linux / Windows)"]
        App2["App B"] --> Lib2["Libs / Binaries"] --> GOS2["Guest OS Kernel<br/>(Full Linux / Windows)"]
        GOS1 --> Hyp["Hypervisor (KVM / VMware / VirtualBox)"]
        GOS2 --> Hyp
        Hyp --> HostOS["Host OS"]
        HostOS --> HW1["Physical Hardware (CPU, RAM, Disk)"]
    end

    subgraph CTR["Container (OS-Level Virtualization)"]
        direction TB
        cApp1["App A"] --> cLib1["Libs / Binaries"]
        cApp2["App B"] --> cLib2["Libs / Binaries"]
        cLib1 --> Engine["Docker Engine / containerd"]
        cLib2 --> Engine
        Engine --> SharedKernel["Shared Host Linux Kernel<br/>(Namespaces + Cgroups)"]
        SharedKernel --> HW2["Physical Hardware (CPU, RAM, Disk)"]
    end
```

### The metaphor: a house and an apartment

- **A Virtual Machine is like building an entire standalone house**:
  - It has its own private foundation, its own roof, its own heating furnace, its own water supply, and its own electricity generator (**its own full Guest OS Kernel**).
  - Building one requires significant resources (1–2 GB RAM overhead per VM) and takes minutes to initialize.
- **A Container is like renting an apartment in a building**:
  - All apartments share the building's foundation, water pipes, and power lines (**the shared host Linux kernel**).
  - However, each apartment has its own front door, its own key, its own rooms, and private space (**namespaces and isolation boundaries**).
  - Moving into an apartment takes seconds, and the overhead is negligible.

### Side by side

| Feature | Virtual Machine (VM) | Container |
|---|---|---|
| **Virtualization Layer** | Hardware level (Hypervisor) | Operating system level (Linux Kernel) |
| **Operating System** | Each VM runs its own full guest OS kernel | All containers share the single host Linux kernel |
| **Startup Time** | Tens of seconds to minutes | Milliseconds (instant) |
| **Memory Footprint** | Large (~1–2 GB minimum per VM) | Minimal (only the process memory itself) |
| **Image Size** | Gigabytes (typically 4–20 GB ISO) | Megabytes (often 5–200 MB) |
| **Isolation** | Strong hardware isolation | Logical process isolation |
| **Best Used For** | Running different OS kernels (e.g. Windows on Linux), untrusted multi-tenancy | Microservices, web APIs, reproducible pipelines, CI/CD |

---

## 1.5 Under the hood: namespaces and cgroups

If a container is just a process running on the host Linux kernel, **how does it not interfere with other processes?**

The Linux kernel achieves container isolation using two complementary primitives:
1. **Namespaces** $\rightarrow$ Control what the container can **SEE**.
2. **Control Groups (cgroups)** $\rightarrow$ Control what the container can **USE**.

```
+-------------------------------------------------------------------+
|  Linux Kernel Isolation:                                          |
|                                                                   |
|  Namespaces  ==> "What can I see?"   (Processes, Network, Files) |
|  Cgroups     ==> "How much can I use?" (CPU, RAM, Disk I/O)       |
+-------------------------------------------------------------------+
```

### 1. Namespaces: what a container can see

When you are inside a container, Linux restricts your visibility so you only see what belongs to you:

| Namespace | Syscall Flag | What it Isolates | What the Container Sees |
|---|---|---|---|
| **PID** | `CLONE_NEWPID` | Process IDs | The container's main process is **PID 1**. Host processes are invisible. |
| **NET** | `CLONE_NEWNET` | Network stack | Its own private network card (`eth0`), private IP, and routing table. |
| **MNT** | `CLONE_NEWNS` | Mount points | Its own isolated root filesystem (`/`). It cannot see host folders unless mounted. |
| **UTS** | `CLONE_NEWUTS` | Hostname | Its own independent hostname (e.g., container ID `e82ab14d3f5a`). |
| **IPC** | `CLONE_NEWIPC` | Inter-process comms | Private shared memory and message queues. |
| **USER** | `CLONE_NEWUSER` | User IDs | Root inside the container (`UID 0`) can map to a regular unprivileged user outside. |

#### ⌨️ Hands-on 2: a container is a process on the host

Let's prove this with an experiment.

1. Start a background container running a sleep process:
   ```bash
   docker run -d --name test-sleep alpine sleep 3600
   ```

2. Check the process table **inside** the container:
   ```bash
   docker exec test-sleep ps aux
   ```
   *Notice:* Inside the container, `sleep 3600` is **PID 1**.

3. Now search for that same process on your **host machine**:
   ```bash
   ps aux | grep "sleep 3600" | grep -v grep
   ```
   *Notice:* The exact same process appears in your host process list, but with a completely different host PID (e.g., PID 28415)!

This confirms the core principle: **a container is just a normal Linux process running on the host kernel**, wrapped in a PID namespace that renumbers it to PID 1 inside.

Clean up the test container:
```bash
docker rm -f test-sleep
```

---

(id-2-control-groups-resource-allocation-throttling)=
### 2. Control groups: what a container can use

Namespaces hide processes from each other, but what stops one container from consuming 100% of your CPU or running your machine out of RAM?

**Control Groups (cgroups)** enforce strict resource budgets:
- **CPU Quotas:** Limit how many CPU cores or what percentage of CPU time a container can consume (e.g. `--cpus="1.5"`).
- **Memory Ceilings:** Limit how much RAM a container can allocate (e.g. `--memory="512m"`).
- **Process Density:** Limit the maximum number of processes to prevent fork-bomb attacks (`--pids-limit=100`).

(the-oom-killer-exit-code-137)=
#### The OOM killer (exit code 137)

If a container process tries to allocate more memory than its cgroup limit allows, the Linux kernel invokes the **Out-Of-Memory (OOM) Killer** to immediately terminate the offending process.

In Docker, an OOM termination is marked by **exit code 137**:

$$\text{Exit Code } 137 = 128 + \text{SIGKILL }(9)$$

Exit code 137 means the process was killed with `SIGKILL`. The most common cause is the OOM killer, but `docker kill` also gives 137: check `docker inspect --format '{{.State.OOMKilled}}' <container>` to be sure.

:::{note} On Day 3
In Kubernetes you do not pass `--memory` and `--cpus` to `docker run`: you write **requests and limits** in the Pod spec, and the kubelet turns them into the same cgroup settings. A container that goes over its memory limit is killed with the same exit code 137, and Kubernetes shows it as `OOMKilled` ([Day 3, section 7](../day03/07-scaling.md)).
:::

---

## 1.6 📖 The container architecture stack

When you run `docker run`, who is doing the work?

```{mermaid}
flowchart TB
    CLI["1. Docker CLI ('docker run')"] -->|REST API over /var/run/docker.sock| Daemon["2. Docker Daemon (dockerd)"]
    Daemon -->|gRPC| Containerd["3. containerd (Image management & lifecycle)"]
    Containerd --> Runc["4. runc (OCI low-level runner)"]
    Runc -->|clone, unshare syscalls| Kernel["5. Linux Kernel (Namespaces & Cgroups)"]
    Runc -.->|Starts process & exits| App["Running Containerized Application"]
```

- **Docker CLI (`docker`):** The user command-line tool you type in.
- **Docker Daemon (`dockerd`):** The background service that manages networks, images, volumes, and API requests.
- **`containerd`:** A lightweight industry-standard runtime that pulls images and manages container execution.
- **`runc`:** The low-level component that makes the actual Linux kernel system calls (`clone`, `unshare`) to set up namespaces and cgroups, starts the process, and hands control over.

:::{note} Jargon: *daemon* and *OCI*
A **daemon** is a program that runs in the background and waits for requests: `dockerd` waits for the commands of the `docker` CLI. **OCI** (Open Container Initiative) is the standard for the image format and for the way a runtime starts a container. Because Docker, containerd, Podman and Kubernetes all follow it, an image built with `docker build` runs on all of them.
:::

---

## 1.7 Advantages and disadvantages

Containers are powerful, but understanding their trade-offs helps you design robust systems:

### Advantages

1. **Environmental Parity:** Eliminates "works on my machine". The exact image tested in CI runs in production.
2. **Speed and Efficiency:** Starts in milliseconds; allows running dozens or hundreds of containers on a single host.
3. **Reproducibility:** Scientific workflows, data pipelines, and web APIs run with identical dependencies across diverse computers.
4. **CI/CD Friendly:** Ephemeral containers can be spun up, tested, and destroyed in seconds.

### Disadvantages

1. **Shared Kernel Security Boundary:** Because containers share the host kernel, a kernel vulnerability could potentially compromise the host or co-located containers. For untrusted code, VMs provide stronger hardware isolation boundaries.
2. **Ephemeral Filesystem by Default:** When a container is deleted, any data written inside it disappears. Stateful data (like databases) requires **volumes** (covered in Section 4).
3. **Architecture Coupling:** A container image compiled for x86_64 (Intel/AMD) will not run natively on ARM64 (Apple Silicon, Raspberry Pi) without emulation (QEMU).

---

## 1.8 ⌨️ Hands-on 3: every container uses the host kernel

Let's verify that containers share the host's Linux kernel, regardless of what Linux distribution is running inside:

1. Check your host kernel version:
   ```bash
   uname -r
   ```

2. Run an **Alpine Linux** container (which uses `musl` libc):
   ```bash
   docker run --rm alpine uname -r
   ```

3. Run an **Ubuntu Linux** container (which uses `glibc`):
   ```bash
   docker run --rm ubuntu:24.04 uname -r
   ```

**Observation:** All three commands print the exact same kernel release! Despite having completely different user-space tools and libraries inside their filesystems, both containers share the exact same underlying host Linux kernel.

**Checkpoint 3** — Docker runs containers, and no test container from this section is left.

```bash
docker run --rm alpine echo "Docker works"
docker ps -a --filter name=test-sleep --format '{{.Names}}'
```

```text
Docker works
```

The second command prints nothing.

:::{dropdown} Not there?
- `permission denied while trying to connect to the Docker daemon socket`: your user is not in the `docker` group. See the note *Permission Resolution* in the [Day 2 introduction](index.md).
- `Unable to find image … pull access denied` or a timeout: the VM cannot reach Docker Hub. Tell the instructor.
- `test-sleep` is listed: remove it with `docker rm -f test-sleep`.
:::

## 1.9 Core takeaway

> **A container is a normal Linux process that the kernel isolates; it is not a small virtual machine.**

- An **image** is the read-only template; a **container** is a running process created from it.
- Containers share the **host kernel**: they start in milliseconds and use little memory, but the isolation is weaker than a VM.
- **Namespaces** decide what a container can see; **cgroups** decide how much CPU and memory it can use.
- Exit code **137** means `SIGKILL`: often the OOM killer, after the container went over its memory limit.

## 1.10 📖 Check yourself

1. You start 10 containers from the same image. How many copies of the image are on the disk?
2. Inside a container, `ps aux` shows your process as PID 1. On the host it has PID 28415. How can it have two PIDs?
3. An Alpine container and an Ubuntu container print the same `uname -r`. Why?
4. A container exits with code 137. Give two possible causes, and the command that tells you which one.
5. You must run code that you do not trust. Would you choose a container or a VM? Why?

:::{dropdown} Solutions
**1.** One. All 10 containers use the same read-only image; each one only adds its own small writable layer (section 2 shows how).

**2.** The PID namespace gives the container its own numbering: inside, the process is PID 1; the host kernel sees the same process with its real PID.

**3.** A container has no kernel of its own. Both use the host kernel; only the files in the image (libraries, tools) are different.

**4.** The OOM killer (the container used more memory than its limit), or `docker kill`. `docker inspect --format '{{.State.OOMKilled}}' <container>` prints `true` in the first case.

**5.** A VM. Containers share the host kernel, so a kernel vulnerability can affect the host and the other containers; a VM has its own kernel behind the hypervisor.
:::

---

Continue with: [2. How Docker stores images and manages data](02-storage-and-images.md).
