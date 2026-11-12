# 3. Running containers and exposing services

Now that you understand images and storage, this section covers the core operational workflow: **how you run containers, expose web services so users can reach them, connect containers on internal networks, and diagnose problems**.

**Symbols:** ⌨️ you type the commands · 📖 reference material: read it after the lesson, or when you need it. The lesson does not cover it in class.

---

(id-3-1-core-concepts-everyday-flags)=
## 3.1 The flags you use every day

When using `docker run`, you combine a few standard flags depending on whether you are running an interactive shell or a continuous background service:

| Flag | Purpose | When to Use |
|---|---|---|
| `-d` (`--detach`) | Runs the container in the background (daemon mode) and returns the container ID. | Web servers, databases, queue workers, long-running services. |
| `-it` (`-i` + `-t`) | Allocates a pseudo-TTY (`-t`) and keeps STDIN open (`-i`). | Interactive debugging sessions, shell exploration (`bash`, `sh`). |
| `--name <name>` | Assigns a memorable, human-readable name to the container. | Always in practice (e.g. `--name climate-api` instead of random names). |
| `--rm` | Automatically deletes the container filesystem when the process exits. | Temporary testing, batch calculations, disposable utilities. |
| `-p <host>:<ctr>` | Forwards traffic from a host port to the container's internal port. | Exposing web APIs or databases to external clients or your browser. |
| `--network <net>` | Attaches the container to a specific Docker virtual network. | Allowing multiple containers to communicate with each other by name. |
| `-v <source>:<ctr-path>` | Mounts a named volume or a host directory into the container. | Keeping data after the container is removed, or sharing files with the host (section 4). |
| `-e <NAME>=<value>` | Sets an environment variable inside the container. Repeat it for each variable. | Configuring an image that reads its settings from the environment (e.g. `-e POSTGRES_PASSWORD=…` for `postgres`, section 4). |

---

(id-3-2-container-lifecycle-states)=
## 3.2 Container lifecycle and states

A container transitions through discrete lifecycle states:

```{mermaid}
stateDiagram-v2
    [*] --> Running: docker run (create + start)
    Running --> Paused: docker pause (freeze)
    Paused --> Running: docker unpause
    Running --> Stopped: Process Exit / docker stop (graceful)
    Running --> Stopped: docker kill (immediate SIGKILL)
    Stopped --> Running: docker start
    Stopped --> [*]: docker rm (delete container)
```

### Lifecycle commands

- **`docker run [flags] <image> [cmd]`**: Download image (if missing), create container, and start it.
- **`docker stop <container>`**: Gracefully stop the container by sending `SIGTERM`, waiting 10 seconds, and sending `SIGKILL` if it hasn't exited.
- **`docker start <container>`**: Re-start an existing stopped container without recreating its filesystem.
- **`docker restart <container>`**: Stop and immediately start the container.
- **`docker rm <container>`**: Delete a stopped container and free its writable layer. Add `-f` to force-stop and delete in one step.

:::{note} Jargon: *SIGTERM* and *SIGKILL*
A **signal** is a short message that the kernel delivers to a process. `SIGTERM` asks the process to stop: it can close its files and connections first. `SIGKILL` stops it at once, and the process cannot catch it. `docker stop` sends `SIGTERM`, then `SIGKILL` after 10 seconds.
:::

---

## 3.3 ⌨️ Interactive mode and detached mode

Let's compare the two primary execution modes:

### Mode 1: interactive (`-it`)

Use this when you need a terminal prompt inside the container to inspect files or run exploratory commands:

```bash
docker run -it --rm python:3.12-slim python3
```

```text
Python 3.12.5 (main, Aug 14 2024, 18:23:45) [GCC 12.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys; print(sys.version)
3.12.5
>>> exit()
```

When Python exits, the container terminates and `--rm` cleans it up.

---

### Mode 2: detached, in the background (`-d`)

Use this for web services, APIs, and databases that should run continuously in the background while keeping your terminal prompt free:

```bash
docker run -d --name web-service nginx:alpine
```

```text
7a3b4c12d3e4...
```

Docker prints the unique container ID and releases your terminal.

Check your running containers:
```bash
docker ps
```

```text
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS     NAMES
7a3b4c12d3e4   nginx:alpine   "/docker-entrypoint.…"   5 seconds ago   Up 4 seconds   80/tcp    web-service
```

Notice the `STATUS` says `Up 4 seconds`, but under `PORTS` it lists `80/tcp` without any host port mapping. That means the server is running, but **no one outside Docker can reach it yet**.

---

(id-3-4-exposing-services-port-forwarding-p)=
## 3.4 Publish a port (`-p`)

By default, every container runs inside its own isolated network namespace. External users (and your host browser) cannot access container ports directly.

To route traffic into a container, you publish ports with `-p`:

```bash
-p <host_port>:<container_port>
```

```{mermaid}
flowchart LR
    Client([External Client / Host Browser]) -->|Request: localhost:8080| Host["Host Port: 8080"]
    Host -->|Port Forwarding / NAT| Bridge["Docker Bridge (docker0)"]
    Bridge -->|Forwarded Packet| Ctr["Container Port: 80 (nginx)"]
```

### The metaphor: an intercom

- Think of the host machine as an apartment building with an intercom on the front door.
- When someone visits port `8080` on the front door, the intercom forwards the call directly to Apartment `80` inside the building.

### ⌨️ Hands-on 5: publish nginx on port 8080

1. Stop and remove the unmapped container:
   ```bash
   docker rm -f web-service
   ```

2. Start a new one with port forwarding:
   ```bash
   docker run -d --name web-service -p 8080:80 nginx:alpine
   ```

3. Test with `curl` or open `http://localhost:8080` in your web browser:
   ```bash
   curl -I http://localhost:8080
   ```

   ```text
   HTTP/1.1 200 OK
   Server: nginx/1.27.0
   Content-Type: text/html
   Connection: keep-alive
   ```

4. Check port mappings with `docker port`:
   ```bash
   docker port web-service
   ```

   ```text
   80/tcp -> 0.0.0.0:8080
   ```

:::{warning} Port Conflict Error (`address already in use`)
If you try to bind a host port that is already used by another service or container, Docker returns:
`bind: address already in use`.
To fix it, choose a different host port (e.g. `-p 8081:80`) or stop the conflicting process.
:::

:::{note} On Day 3
`-p 8080:80` publishes a port on *this* host. In Kubernetes a **Service** gives the Pods one stable address inside the cluster, and `kubectl port-forward` plays the role of `-p` while you test from your machine ([Day 3, section 4](../day03/04-exposing-an-application.md)).
:::

---

## 3.5 Connect containers: networks and built-in DNS

Real applications consist of multiple containers: for example, a web API and a database. **How do containers talk to each other?**

### The default bridge and user-defined bridge networks

- **Default bridge (`bridge`):** The network Docker uses if you don't specify one. It only allows communication via IP addresses. **Automatic DNS resolution by container name is disabled.**
- **User-defined bridge (`docker network create`):** Recommended for all projects. Containers on the same user-defined network can discover and reach each other **by container name** as their hostname!

```{mermaid}
%%{init: {"flowchart": {"subGraphTitleMargin": {"top": 5, "bottom": 25}}}}%%
flowchart TB
    subgraph Net["Custom Bridge Network: climate-net"]
        direction TB
        DNS[("Docker Internal DNS<br/>127.0.0.11")]
        App["API Container<br/>(name: 'api-service')"]
        DB["Database Container<br/>(name: 'db-service')"]
        App -.->|1. Lookup 'db-service'| DNS
        DNS -.->|2. Returns 172.20.0.3| App
        App ==>|3. TCP Connection to 5432| DB
    end
```

### ⌨️ Hands-on 6: reach a container by name

Let's test this directly:

1. Create a custom bridge network:
   ```bash
   docker network create training-net
   ```

2. Run a background Redis database service on this network:
   ```bash
   docker run -d --name db-service --network training-net redis:alpine
   ```

3. Run a temporary client container on the same network and ping `db-service` **by name**:
   ```bash
   docker run --rm --network training-net alpine ping -c 3 db-service
   ```

   ```text
   PING db-service (172.20.0.2): 56 data bytes
   64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.082 ms
   64 bytes from 172.20.0.2: seq=1 ttl=64 time=0.075 ms
   64 bytes from 172.20.0.2: seq=2 ttl=64 time=0.071 ms
   ```

Docker's internal DNS automatically resolved the hostname `db-service` to its internal container IP address! This means your applications never need hardcoded IP addresses.

Clean up the test resources:
```bash
docker rm -f db-service
docker network rm training-net
```

---

(id-3-6-the-recommended-diagnostic-flow)=
## 3.6 ⌨️ Four steps to find a problem

When a container fails or behaves unexpectedly, follow this 4-step diagnostic flow:

```
1. Start Broad  --->  docker ps -a          (Check status & exit code)
2. Focus        --->  docker inspect <name>  (Check config, IP, ports, env)
3. Validate     --->  docker logs -f <name>  (Read application stdout/stderr)
4. Enter        --->  docker exec -it <name> (Debug live inside container)
```

### 1. Start broad (`docker ps` and `docker ps -a`)

- `docker ps`: Lists running containers.
- `docker ps -a`: Lists **all** containers, including stopped and crashed ones. Check the `STATUS` column for exit codes:
  - `Exited (0)`: Exited cleanly (normal completion).
  - `Exited (1)`: Application runtime error (exception, missing file).
  - `Exited (137)`: Killed by OOM Killer (out of memory) or `docker kill`.

### 2. Focus (`docker inspect <container>`)

Dumps full JSON metadata about the container:
```bash
# Query the container's IP address
docker inspect --format '{{ .NetworkSettings.IPAddress }}' web-service

# Query container state (running, exited, ...)
docker inspect --format '{{ .State.Status }}' web-service
```

### 3. Read the logs (`docker logs`)

Docker automatically captures everything written to `stdout` and `stderr`:
```bash
# View recent logs
docker logs web-service

# Stream logs continuously as they happen
docker logs -f --tail 50 web-service
```

(id-4-enter-container-for-live-diagnostics-docker-exec)=
### 4. Enter the container (`docker exec`)

`docker exec` runs a new command inside an existing, running container:
```bash
# Run a single quick command
docker exec web-service nginx -v

# Open an interactive shell inside the running container
docker exec -it web-service sh
```

Inside the shell, you can inspect environment variables (`env`), test internal network connectivity, or verify files.

Clean up:
```bash
docker rm -f web-service
```

:::{note} On Day 3
The same four steps exist in Kubernetes: `kubectl get` (start broad), `kubectl describe` (focus), `kubectl logs` (read the logs) and `kubectl exec` (enter). See the `docker` → `kubectl` table in [Day 3, section 2](../day03/02-interacting-with-a-cluster.md).
:::

---

## 3.7 ⌨️ Hands-on 7: investigate a container that fails

Let's practice diagnostic reasoning.

Run a container that fails immediately:

```bash
docker run -d --name faulty-app alpine cat /nonexistent/file.txt
```

Now investigate:

1. Did it stay running?
   ```bash
   docker ps
   ```
   *(It doesn't appear in `docker ps`!)*

2. Why did it stop? Check `docker ps -a`:
   ```bash
   docker ps -a | grep faulty-app
   ```
   *Notice:* Status shows `Exited (1)`.

3. Why did it exit with code 1? Check the logs:
   ```bash
   docker logs faulty-app
   ```
   *Output:* `cat: can't open '/nonexistent/file.txt': No such file or directory`.

You immediately found the root cause! Clean up:
```bash
docker rm faulty-app
```

**Checkpoint 7** — no container from this section is left, and port 8080 is free for the next sections.

```bash
docker ps -a --format '{{.Names}}' | grep -E 'web-service|db-service|faulty-app'
docker network ls --filter name=training-net --format '{{.Name}}'
docker ps --filter publish=8080 --format '{{.Names}}'
```

The three commands print nothing.

:::{dropdown} Not there?
- A container name is printed: remove it with `docker rm -f <name>`.
- `training-net` is printed: `docker network rm training-net`. If Docker says the network has active endpoints, remove `db-service` first.
- A container publishes 8080: remove it, or Day 2 section 4 and the `port-forward` commands of Day 3 fail with `address already in use`.
:::

## 3.8 Core takeaway

> **Run services detached, publish only the ports that users need, connect containers by name, and investigate from the outside in.**

- `-it` for an interactive shell, `-d` for a service in the background; `--rm` deletes the container when it exits.
- `-p <host>:<container>` publishes a container port on the host; without it, nobody outside Docker reaches the service.
- On a **user-defined network**, containers find each other **by name** through Docker's DNS; the default `bridge` has no DNS.
- To find a problem: `docker ps -a` → `docker inspect` → `docker logs` → `docker exec`.

## 3.9 📖 Check yourself

1. `docker ps` shows nginx `Up`, with `80/tcp` under `PORTS`, but `curl http://localhost:80` fails. Why?
2. Two containers are on the default `bridge` network. `ping db-service` fails with `bad address`. What do you change?
3. A container is not in `docker ps`. Where do you look first, and what does `Exited (1)` tell you?
4. What is the difference between `docker stop` and `docker kill`?
5. `docker run -p 8080:80 …` fails with `address already in use`. Give two ways to fix it.

:::{dropdown} Solutions
**1.** The port is not published: `80/tcp` without `0.0.0.0:…->` means it is reachable only inside Docker. Start the container again with `-p 8080:80`.

**2.** Create a user-defined network (`docker network create …`) and attach both containers to it with `--network`: the default bridge has no DNS by name.

**3.** In `docker ps -a`, which also lists stopped containers. `Exited (1)` means the application ended with an error; `docker logs <name>` shows which one.

**4.** `docker stop` sends `SIGTERM` and waits up to 10 seconds before `SIGKILL`; `docker kill` sends `SIGKILL` at once (exit code 137).

**5.** Use another host port (`-p 8081:80`), or stop the container or process that uses 8080 (`docker ps --filter publish=8080`).
:::

---

Continue with: [4. Persisting data & Docker web interface](04-persisting-data-and-web-ui.md).
