# Day 3 - Kubernetes and Helm

Welcome to Day 3. Today is about running real applications on Kubernetes from the user perspective: **what you deploy, how users reach it, and how you operate it safely at scale**.

## The one story this day tells

Everything today hangs off one sentence:

> You take the container image you built on Day 2, deploy it as a Kubernetes workload, expose it with Services and Ingress, secure it with cert-manager and DNS, scale it for demand, and automate the full deployment with Helm.

```{mermaid}
flowchart TB
    U([User]) -->|request| DNS[DNS hostname]
    DNS -->|resolves to| ING[Ingress<br/>HTTP / HTTPS routing]
    ING -->|forwards to| SVC[Service<br/>stable endpoint]
    SVC -->|selects ready Pods| PODS[Pods<br/>climate-telemetry]

    IMG[(Day 2 image)] -->|image reference| DEP[Deployment]
    DEP -->|manages| RS[ReplicaSet]
    RS -->|creates and replaces| PODS
    HELM[Helm chart / release] -.->|renders and updates| DEP
    HPA[HPA] -.->|adjusts replica count| DEP

    CM[cert-manager] -->|issues| TLS[TLS Secret]
    TLS -.->|certificate used by| ING
    CFG[ConfigMap / Secret] -.->|provides configuration| PODS
    PVC[PersistentVolumeClaim] -.->|mounts persistent data| PODS
    GPU[GPU request and scheduling] -.->|for GPU workloads| PODS
```

The diagram shows two paths that meet at the Pods: the image and workload controllers establish the application; user requests arrive through DNS, Ingress and a Service. The dashed links are supporting controls or resources, not extra traffic hops.

**The thread with Day 2.** On Day 2 you packaged one application into an image with a `Dockerfile`, and ran it in two ways: `docker run` and a `compose.yaml`. Today you run the *same* image — the `climate-telemetry` service from [Day 2, section 5](../day02/05-application-containerisation.md) — on a cluster. Every concept you meet today has a Docker twin you already know; each section points it out, so that you learn the *difference*, not a new vocabulary from scratch.

## What you will be able to do by the end

- Map a `compose.yaml` onto Kubernetes objects (Deployment, Service, Ingress, ConfigMap, Secret, PVC).
- Explain the role of Pods, Deployments, StatefulSets, Jobs, and CronJobs.
- Deploy an image from a registry, roll it out and roll it back.
- Choose and configure the right Service type for an application.
- Publish applications with Ingress and understand host/path routing.
- Issue and renew TLS certificates with cert-manager.
- Understand how DNS records are connected to cluster entry points.
- Inject configuration and secrets, and persist data across Pod replacement.
- Scale workloads manually and automatically with HPA.
- Run GPU workloads with the right node labels, requests, and limits.
- Use Helm to deploy and operate a non-trivial platform such as JupyterHub, and write a chart for your own application.

**Focus for the day**: move from conceptual understanding to repeatable, production-like operational workflows.

## Prerequisites

You should already have:

- Basic Linux and terminal usage.
- The container knowledge from Day 2, and the `~/climate-service` project from [Day 2, section 5](../day02/05-application-containerisation.md).
- The `climate-telemetry:1.0.0` image pushed to the GitLab Container Registry on [Day 2, section 5.7](../day02/05-application-containerisation.md#id-5-7-oci-container-distribution-registries).
- Access to the training Kubernetes cluster, with a working kubeconfig and a namespace named after you (`traineeNN`).
- `kubectl` and `helm`.

Quick checks:

```bash
kubectl version
kubectl get nodes
kubectl get namespace traineeNN
helm version
```

:::{note} Replace `traineeNN` everywhere
All examples use `traineeNN` as the namespace, the GitLab project suffix and the hostname prefix. Replace it with your own trainee number (`trainee07`, for example) every time you see it.
:::

## Sections

| # | Section | What it covers |
|---|---|---|
| 1 | [Kubernetes from the user perspective](01-kubernetes-user-perspective.md) | Core architecture and object model |
| 2 | [Interacting with a cluster](02-interacting-with-a-cluster.md) | kubeconfig, contexts, diagnostics, labels/selectors, `docker` → `kubectl` translation |
| 3 | [Deploying an application](03-deploying-an-application.md) | Imperative vs declarative workflow, registry images, probes, rollout, rollback |
| 4 | [Exposing an application](04-exposing-an-application.md) | Service, service discovery, Ingress, TLS |
| 5 | [📖 Exposing an application with Gateway API *(reference)*](05-exposing-an-application-with-gateway-api.md) | GatewayClass, Gateway, HTTPRoute, external-dns, cert-manager |
| 6 | [📖 cert-manager and DNS](06-cert-manager-and-dns.md) | Certificate automation, issuers, HTTP-01 validation, DNS records |
| 7 | [Scaling](07-scaling.md) | Requests/limits (cgroups again), OOMKilled, HPA under load, VPA and cluster autoscaling |
| 8 | [Configuration and Secrets](08-configuration-and-secrets.md) | ConfigMap for `climate-telemetry`, Secret (encoded, not encrypted), `env` vs `envFrom` |
| 9 | [Persistent storage](09-persistent-storage.md) | PVC and PV, PostgreSQL on a PVC, the [Day 2 persistence experiment](../day02/04-persisting-data-and-web-ui.md#id-4-3-named-volume-management-and-database-durability) |
| 10 | [From Compose to Kubernetes](10-from-compose-to-kubernetes.md) | The [Day 2 `climate-stack`](../day02/06-docker-compose.md) by hand, then with `kompose` (optional); homework: the [Day 2 capstone](../day02/07-lab.md) |
| 11 | [GPU workloads and scheduling 👀](11-gpu-workloads-and-scheduling.md) | Extended resources (`nvidia.com/gpu`), node selectors, taints and tolerations |
| 12 | [Helm: installing charts and writing your own](12-helm-jupyterhub.md) | Installing an upstream chart, writing your own chart for `climate-telemetry` |
| 13 | [Cheat sheet](13-cheatsheet.md) | `compose.yaml` → Kubernetes recap, `docker` → `kubectl` Rosetta stone, high-value commands, troubleshooting |

Most sections end with a hands-on on the same application, so that by section 12 the Day 2 service runs on the cluster, exposed over HTTPS, autoscaled, and packaged as a chart.

## Which application, which image

All images go to your GitLab project `training/autumn-2026/climate-telemetry-traineeNN`. Its registry can hold several images, with the usual GitLab convention:

- The **main application** of the project, `climate-telemetry`, uses the project path itself. This is the image that GitLab CI calls `$CI_REGISTRY_IMAGE` on Day 4.
- **Other applications** add a name after a `/` that says what they do: `climate-stats-api` (the `/stats` API with Redis and PostgreSQL) and `climate-ingestion-api` (the FastAPI ingestion API of the capstone lab).

| Day 2 source | Used in | Image in the registry |
|---|---|---|
| `~/climate-service` ([section 5](../day02/05-application-containerisation.md)) | Hands-on 2–7, 11 (chart), sections 7, 8 and 12 | `…/climate-telemetry-traineeNN:1.0.0` |
| `~/climate-stack` ([section 6](../day02/06-docker-compose.md)) | Sections 9–10 (Hands-on 8–9) | `…/climate-telemetry-traineeNN/climate-stats-api:1.0.0` |
| `~/meditwin-climate-capstone` ([section 7](../day02/07-lab.md)) | Section 10.6 (homework) | `…/climate-telemetry-traineeNN/climate-ingestion-api:1.0.0` |

`…` is `gitlab.data.meditwin-project.eu:6443/training/autumn-2026`. Each tag is the version *of that application*, so every application starts at `1.0.0`.

:::{note} Symbols and checkpoints
⌨️ marks the commands that you type. 👀 marks a demonstration by the instructor: watch, do not run it yourself. 📖 marks reference material: read it after the lesson, or when you need it; it is not covered in class.

The slides used in class, with the theory of each section, are in one file: {download}`day03-kubernetes-slides.pdf`.

Most sections end with a **Core takeaway** and a 📖 **Check yourself**: a few questions, with the solutions in a box you open after you have answered.

Each hands-on ends with a **Checkpoint**: a few commands and the output you should see. If your output is different, open *Not there?* under it before you continue: the next hands-on builds on this one.
:::
