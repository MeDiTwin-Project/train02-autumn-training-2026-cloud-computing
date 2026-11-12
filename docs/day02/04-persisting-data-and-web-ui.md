# 4. Persisting data and a web interface for Docker

As we discovered in Section 2, a container's writable layer is ephemeral: when you delete a container, everything written inside it disappears.

To build real-world applications, you need persistent storage: **how to keep database data safe across container restarts, how to share code between your host machine and a container during development, and how to manage your containers using a graphical web interface (Portainer)**.

**Symbols:** ⌨️ you type the commands · 📖 reference material: read it after the lesson, or when you need it. The lesson does not cover it in class.

---

(id-4-1-state-volatility-the-problem-in-action)=
## 4.1 ⌨️ Hands-on 8: data lost with the container

To demonstrate state volatility in non-persisted containers, consider an in-memory key-value store:

```bash
# 1. Instantiate a Redis container without external storage mounts
docker run -d --name ephemeral-redis redis:alpine

# 2. Write a key-value record to the running instance
docker exec -i ephemeral-redis redis-cli SET station_bologna "Temp: 21.4C, Pressure: 1013.2hPa"
docker exec -i ephemeral-redis redis-cli SAVE

# 3. Retrieve the record to verify persistence in the container layer
docker exec -i ephemeral-redis redis-cli GET station_bologna
```

```text
"Temp: 21.4C, Pressure: 1013.2hPa"
```

If the container is destroyed and replaced with a new instance from the identical image:

```bash
docker rm -f ephemeral-redis
docker run -d --name ephemeral-redis redis:alpine
docker exec -i ephemeral-redis redis-cli GET station_bologna
```

```text
(nil)
```

The data was written to the container's private `upperdir`. When the container was removed, the runtime
purged that layer. To achieve data durability across container lifecycles, storage must be decoupled
from execution instances.

```bash
docker rm -f ephemeral-redis
```

---

## 4.2 Three ways to give a container storage

Docker provides three distinct mechanisms for mounting storage into a container's filesystem namespace:

```{mermaid}
%%{init: {"flowchart": {"subGraphTitleMargin": {"top": 5, "bottom": 25}}}}%%
flowchart TB
    subgraph Host["Host Operating System Filesystem"]
        subgraph DockerManaged["Managed Daemon Storage: /var/lib/docker/volumes/"]
            Vol[("Named Volume: pg_data<br/>- Bypasses UnionFS (CoW)<br/>- Native I/O performance<br/>- Decoupled from container lifecycle")]
        end
        subgraph UserManaged["User-Defined Host Path: /home/cloud-user/app/"]
            BM[("Bind Mount Directory<br/>- Direct host path mapping<br/>- Real-time file synchronization<br/>- Host-dependent paths")]
        end
        subgraph MemoryStore["Volatile Kernel Memory (RAM)"]
            Tmp[("tmpfs Mount<br/>- Volatile RAM storage<br/>- Zero physical disk write<br/>- Transient sensitive tokens")]
        end
    end

    subgraph Ctr["Container Filesystem Namespace"]
        C1["/var/lib/postgresql/data"]
        C2["/app/src"]
        C3["/run/secrets"]
    end

    Vol ===>|"docker run -v pg_data:/var/lib/..."| C1
    BM ===>|"docker run -v $(pwd):/app/src"| C2
    Tmp ===>|"docker run --tmpfs /run/secrets"| C3
```

### Which one to use

| Mount Type | Host Storage Location | Filesystem Performance | Lifecycle Boundary | Primary Use Case |
|---|---|---|---|---|
| **Named Volume** | `/var/lib/docker/volumes/<name>/_data` | Native filesystem speed (Bypasses CoW) | Managed independently by Docker; persists across container removals | Production databases, message queues, climate telemetry databases |
| **Bind Mount** | Explicit user-specified host path (e.g. `/home/user/src`) | Native filesystem speed (Bypasses CoW) | Bound to host directory; persists on host disk | Local development (hot code reload), host configuration file injection |
| **`tmpfs` Mount** | Host physical RAM only | Maximum speed (in-memory access) | Terminated upon container stop | Cryptographic keys, authentication tokens, transient scratch computations |

### How to write it

| Mount type | Flag | Example |
|---|---|---|
| **Named volume** | `-v <volume-name>:<container-path>` | `-v climate_db_data:/var/lib/postgresql/data` |
| **Bind mount** | `-v <host-path>:<container-path>` | `-v $(pwd):/usr/share/nginx/html` |
| **`tmpfs`** | `--tmpfs <container-path>` | `--tmpfs /run/secrets` |

The left part of `-v` decides the type: a **name** (`climate_db_data`) is a named volume, and Docker creates it if it does not exist; a **path** that starts with `/` (or `$(pwd)`, which expands to one) is a bind mount. Add `:ro` at the end to mount it read-only, for example `-v $(pwd):/usr/share/nginx/html:ro`.

---

(id-4-3-named-volume-management-and-database-durability)=
## 4.3 ⌨️ Hands-on 9: a database that survives its container

Named volumes are managed entities in Docker, isolated from host directory structures and accessible
via dedicated CLI commands.

(volume-management-api)=
### Volume commands

```bash
docker volume create <name>      # Allocate a new named volume
docker volume ls                 # Enumerate all existing volumes
docker volume inspect <name>     # Query volume metadata and host mount path
docker volume rm <name>          # Delete a volume (fails if currently mounted)
docker volume prune              # Reclaim all unattached volumes
```

(postgresql-state-durability-verification)=
### PostgreSQL on a named volume

To verify state durability, start PostgreSQL 16 with its data on a named volume:

```bash
# 1. Allocate a dedicated persistent volume
docker volume create climate_db_data

# 2. Deploy PostgreSQL mounting the volume to its internal database cluster path
docker run -d \
  --name climate-db \
  -e POSTGRES_DB=telemetry \
  -e POSTGRES_USER=ecmwf_user \
  -e POSTGRES_PASSWORD=SecurePassphrase2026! \
  -v climate_db_data:/var/lib/postgresql/data \
  postgres:16-alpine

# 3. Wait for engine initialization, then create a table and insert records
sleep 5
docker exec -i climate-db psql -U ecmwf_user -d telemetry -c "
CREATE TABLE weather_stations (id SERIAL PRIMARY KEY, station_name VARCHAR(100), location VARCHAR(100));
INSERT INTO weather_stations (station_name, location) VALUES ('Station-Athens-01', 'Attica, Greece (NTUA)'), ('Station-Bologna-02', 'Emilia-Romagna, Italy (ECMWF)'), ('Station-Valencia-03', 'Valencia, Spain (UV)');
"
```

Verify that the records are stored:
```bash
docker exec -i climate-db psql -U ecmwf_user -d telemetry -c "SELECT * FROM weather_stations;"
```

```text
 id |    station_name    |             location             
----+--------------------+----------------------------------
  1 | Station-Athens-01  | Attica, Greece (NTUA)
  2 | Station-Bologna-02 | Emilia-Romagna, Italy (ECMWF)
  3 | Station-Valencia-03| Valencia, Spain (UV)
(3 rows)
```

Now, simulate complete container destruction:
```bash
docker rm -f climate-db
```

Deploy a brand-new container instance mounting the **identical** volume:

```bash
docker run -d \
  --name climate-db-replica \
  -e POSTGRES_DB=telemetry \
  -e POSTGRES_USER=ecmwf_user \
  -e POSTGRES_PASSWORD=SecurePassphrase2026! \
  -v climate_db_data:/var/lib/postgresql/data \
  postgres:16-alpine

sleep 3
docker exec -i climate-db-replica psql -U ecmwf_user -d telemetry -c "SELECT * FROM weather_stations;"
```

```text
 id |    station_name    |             location             
----+--------------------+----------------------------------
  1 | Station-Athens-01  | Attica, Greece (NTUA)
  2 | Station-Bologna-02 | Emilia-Romagna, Italy (ECMWF)
  3 | Station-Valencia-03| Valencia, Spain (UV)
(3 rows)
```

The database engine recognized the pre-existing database cluster in `/var/lib/postgresql/data` and
resumed operation with zero data loss.

Keep `climate-db-replica` and the `climate_db_data` volume: Section 4.5 backs them up.

:::{note} On Day 3
A named volume becomes a **PersistentVolumeClaim** (PVC). On Day 3 you repeat this same experiment: delete the PostgreSQL Pod, and the data is still there in the new one ([Day 3, section 9](../day03/09-persistent-storage.md)).
:::

---

(id-4-4-bind-mounts-and-posix-permission-boundaries)=
## 4.4 ⌨️ Bind mounts and file permissions

A **bind mount** bridges a directory on the host filesystem directly into the container's mount namespace.

```bash
# Mount current host directory into /usr/share/nginx/html with read-only restriction (:ro)
docker run -d --name web-doc -p 8080:80 -v $(pwd):/usr/share/nginx/html:ro nginx:alpine
```

Remove it when done, to free port 8080 (it is used again later and on Day 3):

```bash
docker rm -f web-doc
```

### Who owns the files: UID and GID

:::{note} Jargon: *UID* and *GID*
Linux identifies users and groups by number: the **UID** (user ID) and the **GID** (group ID). `root` is UID 0. The kernel checks permissions with these numbers, not with the names, so UID 1000 in a container and UID 1000 on the host are the same user for the files of a bind mount.
:::

When a containerized process runs as `root` (UID 0) and creates a file within a bind-mounted directory,
the resulting file on the host filesystem is owned by host UID 0 (`root:root`). This frequently leads to
host permission denial errors when non-root developers attempt to modify the generated files.

To enforce POSIX user identity parity, the container process should be executed matching the host user's UID and GID:

```bash
docker run --user $(id -u):$(id -g) -v $(pwd):/workspace my-build-image
```

---

## 4.5 ⌨️ Back up and restore a volume

Because named volumes reside within root-protected daemon paths (`/var/lib/docker/volumes/`), standard
archival workflows utilize disposable utility containers:

### Back up (volume to archive)

Stop the database first: copying the files of a running database can produce an inconsistent backup.

```bash
docker stop climate-db-replica

docker run --rm \
  -v climate_db_data:/source:ro \
  -v $(pwd):/backup \
  alpine tar -czf /backup/climate_db_backup.tar.gz -C /source .
```

### Restore (archive to volume)
```bash
docker run --rm \
  -v climate_db_restored:/target \
  -v $(pwd):/backup \
  alpine tar -xzf /backup/climate_db_backup.tar.gz -C /target
```

Clean up the database resources of Sections 4.3–4.5:

```bash
docker rm -f climate-db-replica
docker volume rm climate_db_data climate_db_restored
```

---

## 4.6 ⌨️ Hands-on 10: a web interface with Portainer CE

In production and laboratory settings, graphical web interfaces provide consolidated monitoring of
container lifecycles, resource consumption curves, and storage allocations.

**Portainer Community Edition (CE)** is an open-source management interface deployed as a container.

```{mermaid}
flowchart LR
    Browser["Client Web Browser<br/>(http://host-ip:9000)"] -->|HTTP / WebSocket API| Portainer["Portainer CE Container"]
    Portainer -->|Mounts Unix Domain Socket<br/>/var/run/docker.sock| Daemon["Host Docker Daemon<br/>(dockerd)"]
    Daemon -.->|Manages| Resources["Containers, Networks,<br/>Volumes, and Images"]
```

### Step 1: start Portainer

```bash
# 1. Allocate a persistent volume for Portainer's internal database
docker volume create portainer_data

# 2. Deploy Portainer CE binding the host Docker socket
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:2.21.0
```

### Step 2: open the interface

Access the interface at `http://<HOST-IP>:9000`. Upon initial access, establish an administrative
passphrase (minimum 12 characters) and select the **local** Docker environment.

### Step 3: find the volume on the host

Inspect the `portainer_data` volume, and find the `Mountpoint` field: it is the directory on the host that holds the volume.

```bash
docker volume inspect portainer_data
```

```text
[
    {
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/portainer_data/_data",
        "Name": "portainer_data",
        …
    }
]
```

### Step 4: cleanup

Remove Portainer when you are done: the capstone lab (section 7) starts its own Portainer on port `9000`.

```bash
docker rm -f portainer
docker volume rm portainer_data
```

**Checkpoint 10** — no container from this section is left, and ports 8080 and 9000 are free.

```bash
docker ps -a --format '{{.Names}}' | grep -E 'ephemeral-redis|climate-db|web-doc|portainer'
docker volume ls --format '{{.Name}}' | grep -E 'climate_db|portainer_data'
docker ps --filter publish=8080 --filter publish=9000 --format '{{.Names}}'
ls climate_db_backup.tar.gz
```

The first three commands print nothing; `ls` finds the backup archive of section 4.5.

:::{dropdown} Not there?
- A container is listed: `docker rm -f <name>`. Portainer has `--restart=always`, so `docker stop` alone is not enough.
- A volume is listed: `docker volume rm <name>`. If Docker says it is in use, remove the container that uses it first.
- `ls: cannot access 'climate_db_backup.tar.gz'`: you are in another directory, or the backup ran after the volume was removed. Go to the directory where you ran section 4.5.
- Portainer does not open in the browser: check `docker ps` (is it `Up`?) and use the IP address of your VM, not `localhost`, if the browser runs on your laptop.
:::

---

## 4.7 📖 Security: the Docker socket

Notice that deploying Portainer requires mounting the host Docker socket:
```bash
-v /var/run/docker.sock:/var/run/docker.sock
```

:::{warning} Privilege Escalation Vector
The Unix domain socket `/var/run/docker.sock` is the control plane interface for the Docker daemon.
Because the Docker daemon runs with root privileges on the host kernel, any process with read/write
access to `/var/run/docker.sock` possesses **unrestricted root equivalence over the host system**.

An adversary within a compromised container with socket access can execute:
```bash
docker run -v /:/host-root alpine chroot /host-root
```
This mounts the host's entire physical filesystem into a new container, completely bypassing all
namespace and cgroup boundaries. Never mount `/var/run/docker.sock` into unverified or public-facing
workloads!
:::

## 4.8 Core takeaway

> **Keep data outside the container: the container can be replaced, the volume stays.**

- A **named volume** is managed by Docker and survives `docker rm`: use it for databases.
- A **bind mount** maps a host directory into the container: use it for development and configuration files; watch the UID of the files.
- A **`tmpfs`** mount is in RAM only and disappears when the container stops.
- Back up a volume with a throw-away container and `tar`, after you stop the database.
- Mounting `/var/run/docker.sock` gives the container root access to the host.

## 4.9 📖 Check yourself

1. You remove the PostgreSQL container and start a new one with the same `-v climate_db_data:…`. Why are the rows still there?
2. Which mount type would you choose for: (a) PostgreSQL data, (b) source code you edit on the host, (c) a temporary token?
3. A container that runs as `root` writes files into a bind mount. What problem do you see on the host, and how do you avoid it?
4. Why do you stop the database before you back up its volume?
5. Why is `-v /var/run/docker.sock:/var/run/docker.sock` dangerous?

:::{dropdown} Solutions
**1.** The data is in the volume, not in the writable layer of the container. The new container mounts the same volume and PostgreSQL finds its existing data directory.

**2.** (a) A named volume. (b) A bind mount. (c) A `tmpfs` mount.

**3.** The files belong to `root` on the host, and your user cannot change or delete them. Run the container with `--user $(id -u):$(id -g)`.

**4.** A running database changes its files while `tar` copies them, so the archive can be inconsistent.

**5.** Whoever can use the socket controls the Docker daemon, which runs as root: they can start a container that mounts the whole host file system.
:::

---

Continue with: [5. Application containerisation](05-application-containerisation.md).
