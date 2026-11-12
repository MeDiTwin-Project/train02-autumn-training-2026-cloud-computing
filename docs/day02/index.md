# Day 2 — Containers & Docker

Welcome to Day 2. Today is about understanding and operating containers with Docker from the ground up: **how you package applications, run them consistently anywhere, persist their data, and compose multi-service systems**.

## The one story this day tells

Everything today hangs off one sentence:

> You write an application, package it with all its dependencies into an immutable container image, run and network containers reliably on any host, persist their data with volumes, and orchestrate them as a multi-tier stack with Docker Compose.

```{mermaid}
flowchart LR
    Code["Source Code + Dockerfile"] -->|docker build| Img[("Immutable Image<br/>(Layered FS)")]
    Img -->|docker run| Ctr["Running Container<br/>(Isolated Process)"]
    Ctr -->|Port Forwarding| User([User Traffic])
    Ctr <-->|Volume Mount| Disk[("Persistent Storage")]
    Ctr <-->|Bridge Network| Ctr2["Database Container"]
    DC["Docker Compose"] -.->|Orchestrates| Ctr
    DC -.->|Orchestrates| Ctr2
```

## What you will be able to do by the end

- Explain the difference between bare metal, Virtual Machines, and Containers.
- Understand how Linux **namespaces** (visibility) and **cgroups** (resource limits) create container isolation.
- Understand how **image layers** and **Copy-on-Write (CoW)** save disk space and make containers instant to start.
- Run, stop, inspect, and debug containers interactively and in detached background mode.
- Expose containerized services to external networks with port forwarding.
- Connect multiple containers on user-defined bridge networks with automatic DNS service discovery.
- Persist stateful data across container lifecycles using named volumes and bind mounts.
- Author clean, production-ready `Dockerfile` specifications with multi-stage builds and non-root security.
- Model, launch, and manage multi-container applications using Docker Compose (`compose.yaml`).

**Focus for the day**: move from running your first simple container command to building, persisting, and orchestrating production-ready containerized applications.

## Prerequisites

You should already have:

- Basic Linux command-line familiarity (`ls`, `cd`, `cat`, `curl`).
- A working Docker installation on your training environment.

Quick environment check:

```bash
docker --version             # Verify Docker CLI (v24.x or higher)
docker compose version       # Verify Docker Compose v2 plugin
docker info                  # Verify Docker daemon connectivity
```

:::{note} Permission Resolution
If `docker info` returns `Got permission denied while trying to connect to the Docker daemon socket`,
ensure your user belongs to the `docker` system group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```
:::

## Core Definitions

Before running commands, keep these two foundational definitions clear:

- **Container Image:** An immutable, read-only snapshot containing the application code, libraries, runtime, and filesystem blueprint. Think of it as a **frozen recipe** or a template.
- **Container:** An active, isolated running process created from an image, with its own private writable filesystem layer. Think of it as the **baked cake** or a running instance.

## Sections

| # | Section | What it covers |
|---|---|---|
| 1 | [Containers and Docker: advantages and disadvantages](01-containers-and-docker.md) | The problem containers solve, first containers, VMs and containers, namespaces and cgroups (Hands-on 1–3) |
| 2 | [How Docker stores images and data](02-storage-and-images.md) | Images, layers, Copy-on-Write (CoW), why containers need volumes; 📖 OverlayFS (Hands-on 4) |
| 3 | [Running containers and exposing services](03-running-containers.md) | Container lifecycle, interactive and detached modes, published ports, networks and DNS, four steps to find a problem (Hands-on 5–7) |
| 4 | [Persisting data and a web interface for Docker](04-persisting-data-and-web-ui.md) | Volumes, bind mounts, tmpfs, file permissions, backup and restore, Portainer; 📖 the Docker socket (Hands-on 8–10) |
| 5 | [Application containerisation: Dockerfiles and registries](05-application-containerisation.md) | Dockerfiles, layer cache, multi-stage builds, non-root users, tags, the GitLab registry (Hands-on 11–12) |
| 6 | [Microservices with Docker Compose](06-docker-compose.md) | `compose.yaml`, service discovery, start order with health checks, start and stop the stack (Hands-on 13) |
| 7 | [Capstone lab](07-lab.md) | The MeDiTwin climate extremes pipeline: multi-stage image, Compose stack, fault injection |
| 8 | [📖 Cheat sheet and troubleshooting](08-cheatsheet.md) | Everyday commands, disk cleanup, common problems, exit codes |

Sections 1 to 6 build the model, the mechanics, and the tools. **Section 7** is where you apply them end-to-end in a realistic multi-service application.

:::{note} Symbols and checkpoints
⌨️ marks the commands that you type. 📖 marks reference material: read it after the lesson, or when you need it; the lesson does not cover it in class.

Sections 1 to 6 end with a **Core takeaway** and a 📖 **Check yourself**: a few questions, with the solutions in a box you open after you have answered.

The last hands-on of each section ends with a **Checkpoint**: a few commands and the output you should see. If your output is different, open *Not there?* under it before you continue: the next sections use the same ports, and Day 3 uses the image and the stack you build here.
:::
