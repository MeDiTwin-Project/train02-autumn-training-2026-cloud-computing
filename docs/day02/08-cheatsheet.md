# 8. 📖 Docker cheat sheet and troubleshooting

The Docker commands of the day, how to free disk space, the most common problems and the exit codes, on one page.

**Symbols:** ⌨️ you type the commands · 📖 reference material: read it after the lesson, or when you need it. The lesson does not cover it in class.

---

## 8.1 Commands

### Containers

```bash
# Execution & State Transitions
docker run -d --name <name> -p <host>:<ctr> <image>      # Instantiate detached container with port forwarding
docker run -it --rm <image> bash                         # Instantiate interactive container, purge on exit
docker run -d --restart unless-stopped <image>           # Instantiate with persistent reboot restart policy
docker stop <container>                                  # Graceful termination (SIGTERM -> SIGKILL)
docker kill <container>                                  # Immediate forced termination (SIGKILL)
docker restart <container>                               # Re-execute container process
docker rm <container>                                    # Purge stopped container metadata and upperdir
docker rm -f <container>                                 # Forcibly terminate and purge active container

# Inspection & Runtime Telemetry
docker ps                                                # Enumerate active running containers
docker ps -a                                             # Enumerate all containers (active and stopped)
docker logs <container>                                  # Query standard I/O log streams
docker logs -f --tail 100 <container>                    # Stream live standard I/O logs
docker exec -it <container> sh                           # Attach diagnostic shell to container namespaces
docker top <container>                                   # Enumerate process table within container PID namespace
docker stats                                             # Real-time resource consumption telemetry (CPU, RAM, I/O)
docker inspect <container>                               # Query comprehensive JSON metadata manifest
docker diff <container>                                  # Inspect mutated inodes in container upperdir
```

### Images and registries

```bash
# Image Construction & Local Storage
docker build -t <repository>:<tag> .                     # Compile image from Dockerfile in current context
docker build --no-cache -t <repo>:<tag> .                # Force compilation without layer cache
docker images                                            # Enumerate locally stored image manifests
docker rmi <image>                                       # Purge local image layers from storage engine
docker history <image>                                   # Inspect cryptographic layer sizes and build instructions

# OCI Registry Distribution
docker login <registry-domain>                           # Authenticate against OCI distribution endpoint
docker tag <local-image> <registry>/<ns>/<repo>:<tag>    # Apply canonical remote namespace tag
docker push <registry>/<ns>/<repo>:<tag>                 # Upload serialized layer archives to registry
docker pull <registry>/<ns>/<repo>:<tag>                 # Download image layers from registry
```

### Volumes

```bash
docker volume create <name>                              # Allocate managed persistent volume
docker volume ls                                         # Enumerate local named volumes
docker volume inspect <name>                             # Query volume host mountpoint path
docker volume rm <name>                                  # Purge volume (fails if mounted)
docker volume prune                                      # Reclaim unattached persistent volumes
```

### Networks

```bash
docker network create <name>                             # Allocate custom bridge network (with embedded DNS)
docker network ls                                        # Enumerate allocated network namespaces
docker network inspect <name>                            # Query network subnet CIDR and attached endpoints
docker network connect <network> <container>             # Attach running container to network namespace
docker network disconnect <network> <container>          # Detach container from network namespace
docker network rm <name>                                 # Remove virtual network device
```

### Docker Compose

```bash
docker compose up -d                                     # Converge stack to declared state in background
docker compose up --build -d                             # Force image recompilation before stack convergence
docker compose ps                                        # Enumerate services within stack topology
docker compose logs -f [service]                         # Stream consolidated or service-specific logs
docker compose exec <service> <cmd>                      # Invoke diagnostic command in service container
docker compose restart <service>                         # Restart specific service container
docker compose down                                      # Teardown containers and networks (preserves volumes)
docker compose down -v                                   # Teardown containers, networks, AND persistent volumes
```

---

## 8.2 Disk use and cleanup

Docker retains cached build stages, unreferenced layers, and stopped containers. To inspect and reclaim storage:

```bash
# Query disk utilization by subsystem
docker system df
```

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          14        3         3.82GB    2.91GB (76%)
Containers      5         2         12.4MB    8.2MB (66%)
Local Volumes   8         3         450MB     310MB (68%)
Build Cache     42        0         1.25GB    1.25GB (100%)
```

### Free disk space

```bash
# Purge stopped containers
docker container prune -f

# Purge dangling (untagged) image layers
docker image prune -f

# Purge BuildKit compilation cache
docker builder prune -f

# Reclaim all unused images, networks, and stopped containers
docker system prune -f

# Include persistent volume purging
docker system prune --volumes -f
```

---

## 8.3 Common problems

| Symptom | Cause | What to do |
|---|---|---|
| **Container exits immediately with status 0** | PID 1 completed execution. Containers terminate automatically upon exit of the primary process. | Verify entrypoint configuration. For debugging base distributions (`alpine`, `ubuntu`), attach an interactive TTY (`-it sh`) or supply a non-terminating foreground process. |
| **Container terminates with status 137** | **Out-Of-Memory (OOM) Termination:** The process RSS exceeded declared cgroup boundaries (`--memory`), triggering kernel `SIGKILL` ($128 + 9$). | Inspect termination status: `docker inspect <ctr> --format '{{.State.OOMKilled}}'`. Increase memory allocation quota or profile memory consumption within the application. |
| **`bind: address already in use`** | The target host port is already bound by an existing container or host daemon (`EADDRINUSE`). | Identify conflicting process: `sudo ss -tulpn \| grep :<PORT>`. Remap host port allocation (e.g. `-p 8081:80`). |
| **`permission denied: /var/run/docker.sock`** | The host user account lacks membership in the `docker` system group. | Grant socket access: `sudo usermod -aG docker $USER && newgrp docker`. |
| **`Permission denied` in volume/bind mount** | POSIX ownership mismatch: Container executes as unprivileged UID (e.g. 1000) while host directory is owned by UID 0 (`root`). | Harmonize ownership: `sudo chown -R 1000:1000 /host/path`, or specify execution identity: `docker run --user $(id -u):$(id -g)`. |
| **Inter-container DNS resolution failure** | Containers are attached to the default `bridge` rather than a user-defined bridge network. | The default bridge does not support internal DNS resolution. Allocate a custom network (`docker network create my-net`) or utilize Docker Compose. |
| **Runtime mutations lost across restarts** | Mutations were written to the volatile container writable layer (`upperdir`) rather than an external volume mount. | Bind target path to a named persistent volume: `-v managed_vol:/data/path`. |
| **Termination takes exactly 10 seconds** | Shell form entrypoint (`CMD python app.py`) spawned `/bin/sh` as PID 1, swallowing `SIGTERM` signals. | Convert entrypoint to Exec Form JSON array: `CMD ["python3", "app.py"]`. |
| **`no space left on device` during build** | Host root partition or Docker storage driver directory exhausted available disk inodes/blocks. | Reclaim storage via `docker system prune -a --volumes` and verify host capacity via `df -h`. |
| **`exec format error` upon launch** | CPU instruction set architecture mismatch (e.g. image compiled for ARM64 executed on x86_64). | Compile target image for appropriate architecture using Buildx: `docker buildx build --platform linux/amd64 ...`. |

---

## 8.4 Exit codes

The container process exit code indicates the mechanism of process termination:

| Exit code | Meaning | What it tells you |
|---|---|---|
| **0** | Success / Normal Completion | Primary process executed to completion without unhandled exceptions. |
| **1** | Application Error | Runtime failure, uncaught exception, or fatal initialization error. |
| **125** | Runtime Execution Error | Docker daemon failure to initialize container (e.g., conflicting flags). |
| **126** | Command found but not executable | Entrypoint binary lacks POSIX execute permissions (`+x`). |
| **127** | Command Not Found | Target executable path does not exist within container rootfs. |
| **137** | Fatal Error: `SIGKILL` | Terminated by host Linux kernel OOM Killer ($128 + 9$) or forced via `docker kill`. |
| **143** | Graceful Termination: `SIGTERM` | Process terminated via standard termination signal ($128 + 15$) dispatched by `docker stop`. |

---

Next: **[Day 3 — Kubernetes and Helm](../day03/index.md)**
