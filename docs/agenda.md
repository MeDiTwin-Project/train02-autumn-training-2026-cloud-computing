# Agenda

This five-day training is about cloud computing for scientific workloads. You start with virtual
machines and storage. You finish with an application that you deploy, monitor and make secure on
Kubernetes. For each day, this agenda gives:

- The main idea of the day.
- The requirements before you start.
- The sections, in sequence, with their topics.
- The practical work in each section.

| Day | Subject | Sections | Main practical work |
| --- | --- | --- | --- |
| 1 | [Cloud Computing Foundations](day01/index.md) | 11 (0–10) | On your training VM: a block device, network checks, and S3 from the command line and from Python |
| 2 | [Containers & Docker](day02/index.md) | 8 | Capstone lab: put the MeDiTwin climate extremes pipeline into containers |
| 3 | [Kubernetes and Helm](day03/index.md) | 13 (5 and 6 reference, 11 demo) | The Day 2 service on Kubernetes: from a workload to an HTTPS endpoint, then scaled, configured and packaged as a Helm chart |
| 4 | [Git, CI/CD & GitOps](day04/index.md) | 9 (8 **or** 8B) | Your service tested, scanned, released and deployed by merging |
| 5 | [Advanced topics: Monitoring, Security](day05/index.md) | 3 | Monitor and protect `climate-stats-api`; group project: package and serve a model |

**Symbols:**

- 👀 The instructor shows a demonstration. Participants watch. They do not type.
- ⌨️ Participants type all the commands.
- 📖 Reference material. Participants read it after the lesson, or when they need it.

---

## Day 1 — Cloud Computing Foundations

Full material: **[Day 1 — Cloud Computing Foundations](day01/index.md)**

> *A computer is not always a physical box. An API call can create, change and remove compute,
> storage and networks. You pay for these resources only while they exist.*

Participants do not have an OpenStack account. The instructor shows OpenStack in 👀 demonstrations.
Participants do the practical work on a **training VM** (SSH and `sudo`) and on a shared **S3** endpoint.

**Before you start, you must have:**

- A laptop with a terminal and an SSH client.
- Basic knowledge of the shell.
- Your training credentials (sent by e-mail) and access to your training VM ([section 0](day01/00-getting-connected.md)).
- S3 access. It is shown in the credential portal and already configured on your VM.

| # | Section | Topics | Practical work |
| --- | --- | --- | --- |
| 0 | [Getting connected](day01/00-getting-connected.md) | Training credentials; SSH key from the credential portal; SSH and VS Code Remote - SSH; GitLab and Rancher sign-in | ⌨️ Log in to the training VM, GitLab and Rancher |
| 1 | [Virtualization](day01/01-virtualization.md) | The problem that virtualization solves; history; how virtualization works; VMs, containers, unikernels; virtual storage and virtual networks | — |
| 2 | [Cloud computing](day01/02-cloud-computing.md) | The meaning of "cloud"; its history; why to use the cloud; why to use it for scientific workloads | — |
| 3 | [IaaS, PaaS, SaaS, plus public, private and hybrid](day01/03-service-models.md) | IaaS, PaaS, SaaS, FaaS, XaaS; public, private and hybrid cloud | — |
| 4 | [OpenStack: the open-source cloud](day01/04-openstack.md) | From AWS to open source; architecture (Nova, Cinder, Swift, Neutron, Keystone); the European Weather Cloud | 👀 Dashboard; create a VM · ⌨️ Log in to the training VM and look at it from the inside |
| 5 | [Cloud storage](day01/05-storage.md) | Block storage and object storage; the properties of cloud storage; Ceph | 👀 Attach a volume; look at object storage · ⌨️ Format, mount and detach a block device; objects in a bucket with `s3cmd` |
| 6 | [Cloud networking](day01/06-networking.md) | IP addresses, subnets, routes; `ping`, `traceroute`, `curl`; private and public addresses; NAT and floating IPs; firewalls and security groups | ⌨️ Examine the network and find problems · 👀 Security groups in OpenStack |
| 7 | [Cloud APIs](day01/07-apis.md) | The function of APIs; credentials (rc file, environment variables); OpenStack CLI and Python SDK; S3 | 👀 `openstack` CLI, Python SDK · ⌨️ S3 with `s3cmd`, `curl` and Python (`boto3`) |
| 8 | [Lambda functions](day01/08-faas.md) | The problem of a server that does no work; the parts of a lambda function; open-source FaaS | — |
| 9 | [Containers overview](day01/09-containers.md) | From jails to namespaces; containers and VMs compared (introduction to Day 2) | — |
| 10 | [Day 1 recap](day01/10-day01recap.md) | One key idea per section; what you did on your training VM | — |

---

## Day 2 — Containers & Docker

Full material: **[Day 2 — Containers & Docker](day02/index.md)**

> *Put an application and its dependencies into an image that does not change. Run containers
> on any host and connect them. Keep their data in volumes. Run many containers together as
> one stack with Docker Compose.*

**Before you start, you must have:**

- Basic knowledge of the Linux command line.
- Docker and Docker Compose v2 on the training environment.

| # | Section | Topics | Practical work |
| --- | --- | --- | --- |
| 1 | [Containers and Docker: advantages and disadvantages](day02/01-containers-and-docker.md) | "It works on my machine"; image and container; VMs and containers compared; namespaces and cgroups; 📖 the architecture stack; advantages and disadvantages | ⌨️ Hands-on 1–3 (first containers; a container is a process on the host; every container uses the host kernel) |
| 2 | [How Docker stores images and data](day02/02-storage-and-images.md) | Image commands; layers; Copy-on-Write (CoW); 📖 OverlayFS; why containers need volumes | ⌨️ Hands-on 4 (Copy-on-Write with `docker diff`) |
| 3 | [Running containers and exposing services](day02/03-running-containers.md) | The everyday flags (`-d`, `-it`, `-p`, `-v`, `-e`); container states; interactive mode and detached mode; published ports; networks and DNS; four steps to find a problem | ⌨️ Hands-on 5–7 (publish nginx on a port; reach a container by name; investigate a container that fails) |
| 4 | [Persisting data and a web interface for Docker](day02/04-persisting-data-and-web-ui.md) | Data loss in containers; volumes, bind mounts, tmpfs and how to write them; file permissions (UID and GID); backup and restore; Portainer CE; 📖 security risk of the Docker socket | ⌨️ Hands-on 8–10 (data lost with the container; PostgreSQL on a named volume, backup and restore; Portainer) |
| 5 | [Application containerisation: Dockerfiles and registries](day02/05-application-containerisation.md) | A Flask service served by gunicorn; a Dockerfile for production (dependencies before code, non-root user with a fixed UID, exec form); layer cache and `.dockerignore`; multi-stage builds; tags; the GitLab registry | ⌨️ Hands-on 11 (build `climate-telemetry:1.0.0`); Hands-on 12 (push it to your GitLab registry) |
| 6 | [Microservices with Docker Compose](day02/06-docker-compose.md) | The whole stack in one file; the climate stack (API, PostgreSQL, Redis); `compose.yaml` and what each key does; service discovery; start order with health checks; start and stop the stack | ⌨️ Hands-on 13 (start and check the `climate-stack`) |
| 7 | [Capstone lab: the MeDiTwin climate extremes pipeline in containers](day02/07-lab.md) | MeDiTwin climate extremes pipeline: multi-stage image, database schema, Compose stack, fault injection, removal | ⌨️ Full lab |
| 8 | [📖 Docker cheat sheet and troubleshooting](day02/08-cheatsheet.md) | Commands; disk use and cleanup; table of problems and solutions; exit codes | 📖 Reference |

---

## Day 3 — Kubernetes and Helm

Full material: **[Day 3 — Kubernetes and Helm](day03/index.md)**

> *Package an application. Deploy it as a Kubernetes workload. Make it available with Services
> and Ingress. Make it secure with cert-manager and DNS. Scale it. Use Helm to automate the
> deployment.*

**Before you start, you must have:**

- Basic knowledge of Linux and of the terminal.
- The container knowledge from Day 2.
- The `~/climate-service` project from Day 2, section 5, and the `climate-telemetry:1.0.0` image pushed to the GitLab Container Registry in section 5.7.
- A GitLab account (`traineeNN`) and your namespace `traineeNN` on the training cluster.
- Access to a Kubernetes cluster with a kubeconfig that works.
- `kubectl` and `helm`.

| # | Section | Topics | Practical work |
| --- | --- | --- | --- |
| 1 | [Kubernetes from the user perspective](day03/01-kubernetes-user-perspective.md) | Orchestration; why Kubernetes after Compose; control plane and workers; the controller loop; workloads; Services; Ingress; namespaces and RBAC | — |
| 2 | [Interacting with a cluster](day03/02-interacting-with-a-cluster.md) | kubeconfig and context (your namespace is already set); `docker` → `kubectl`; `get` → `describe` → `logs` → `exec`; labels and selectors; `spec` and `status` | ⌨️ Hands-on 1 (hello, Kubernetes: `kubectl run`, then follow the Pod with `get`, `describe`, `logs` and `-o yaml`) |
| 3 | [Deploying an application](day03/03-deploying-an-application.md) | Why a Deployment instead of a bare Pod; imperative and declarative (YAML) methods; registry images and pull secrets; probes instead of `HEALTHCHECK`; rollout and rollback | ⌨️ Hands-on 2 (deploy the Day 2 image) |
| 4 | [Exposing an application](day03/04-exposing-an-application.md) | Why a Service; Service types (ClusterIP, NodePort, LoadBalancer); service discovery; port-forward; Ingress; TLS; automatic DNS; a deliberate readiness failure | ⌨️ Hands-on 3 (a Service for `climate-telemetry`); Hands-on 4 (Ingress and TLS) |
| 5 | [📖 Exposing an application with Gateway API](day03/05-exposing-an-application-with-gateway-api.md) | Ingress and Gateway API compared; GatewayClass, Gateway, HTTPRoute; external-dns; cert-manager | 📖 Read after the lesson |
| 6 | [📖 cert-manager and DNS](day03/06-cert-manager-and-dns.md) | Issuers and Certificates; the HTTPS sequence, from the DNS record to the TLS Secret; external-dns; checks | 📖 Read after the lesson |
| 7 | [Scaling](day03/07-scaling.md) | Requests and limits (cgroups again, exit 137); HPA, with scale-up and scale-down windows; HPA under load; vertical scaling and cluster scaling | ⌨️ Hands-on 5 (OOMKilled, exit 137); Hands-on 6 (HPA under load) |
| 8 | [Configuration and Secrets](day03/08-configuration-and-secrets.md) | ConfigMap; Secret (base64 is not encryption); `env` and `envFrom` | ⌨️ Hands-on 7 (ConfigMap for `climate-telemetry`); a throw-away Secret; cleanup of `climate-telemetry` |
| 9 | [Persistent storage](day03/09-persistent-storage.md) | PV and PVC; StorageClass; a ConfigMap mounted as a file; the Day 2 PostgreSQL persistence experiment on Kubernetes | ⌨️ Hands-on 8 (PostgreSQL on a PVC) |
| 10 | [From Compose to Kubernetes](day03/10-from-compose-to-kubernetes.md) | The Day 2 `climate-stack` ported service by service; service discovery without `depends_on`; the same stack with `kompose`; homework: the Day 2 capstone | ⌨️ Hands-on 9 (API + Redis + PostgreSQL); `kompose` comparison (optional) |
| 11 | [GPU workloads and scheduling](day03/11-gpu-workloads-and-scheduling.md) | Extended resources (`nvidia.com/gpu`); node selectors, taints and tolerations | 👀 GPU Pod on the single GPU of the training cluster |
| 12 | [Helm: installing charts and writing your own](day03/12-helm-jupyterhub.md) | Why Helm (chart ≈ Dockerfile for a deployment); charts, releases, values; deploy, monitor, upgrade and roll back JupyterHub; write a chart for `climate-telemetry` (`Chart.yaml`, `values.yaml`, templates, `helm template`/`lint`/`install`/`package`) | ⌨️ Hands-on 10 (deploy JupyterHub); Hands-on 11 (your own chart) |
| 13 | [📖 Day 3 cheat sheet](day03/13-cheatsheet.md) | `compose.yaml` → Kubernetes recap, `docker` → `kubectl` Rosetta stone, core commands, workloads, network, TLS, scaling, GPU, Helm, table of problems | 📖 Reference |

---

## Day 4 — Git, CI/CD & GitOps

Full material: **[Day 4 — Git, CI/CD & GitOps](day04/index.md)**

> *You commit a change to Git. An automated system finds the change, checks it, builds it and
> deploys it. If the change is bad, you revert the commit. This restores the previous state.*

All work on this day uses one application: the **climate telemetry service** from Day 2.

**Before you start, you must have:**

- The `~/climate-service` project from Day 2, section 5.
- The Ingress, TLS and Helm knowledge from Day 3.
- A GitLab account and a Rancher account (`traineeNN`).
- Your namespace on the training cluster.
- An SSH key on your GitLab account.
- These tools: `git`, `docker`, `kubectl`, `helm`, `jq`, `trivy`, `dagger`, `cookiecutter`.

| # | Section | Topics | Practical work |
| --- | --- | --- | --- |
| 1 | [Git from zero](day04/01-git.md) | A commit is a snapshot; commit messages; branches; remotes; `.gitignore`; merge conflicts; merge requests; how to undo changes | ⌨️ Type along: first repository, branches, a merge conflict; Hands-on 1 (put the service in Git); Hands-on 2 (first merge request) |
| 2 | [What CI/CD actually is](day04/02-ci-cd.md) | CI and CD compared; runners, jobs, stages; the usual sequence of a pipeline; push-based CD and its problem | Investigation exercise (discussion, no typing) |
| 3 | [GitLab CI hands-on](day04/03-gitlab-ci.md) | Images; stages; artifacts; cache; variables; `rules`; `needs`; reuse; build and push with Kaniko; find pipeline problems | ⌨️ Hands-on 3 (your service's first pipeline); Hands-on 4 (variables, caching, `rules` and `needs`) |
| 4 | [Security in pipelines](day04/04-pipeline-security.md) | Secrets; automatic scans; supply chain (tags and digests compared); Trivy as a gate; SBOMs; image signatures; manual approval | ⌨️ Hands-on 5 (find a leaked secret; make Trivy a gate) |
| 5 | [Conventional commits and automated releases](day04/05-releases.md) | Semantic versioning; from commit message to version; changelogs; connect the version to the artifact; usual failures | ⌨️ Hands-on 6 (let your commit messages cut the release) |
| 6 | [What GitOps is](day04/06-gitops.md) | Pull and push compared; the four properties; rollback with `git revert`; what GitOps is not; update of image tags | Investigation exercise (discussion, no typing) |
| 7 | [Rancher Fleet](day04/07-fleet.md) | GitRepo, Bundle, BundleDeployment; repository layout; `fleet.yaml`; `dependsOn`; cluster targets; External Secrets Operator; `diff.comparePatches`; problems and solutions | 👀 The training cluster in Fleet; a secret from Vault; GitRepos and Bundles on the management cluster · ⌨️ Hands-on 7 (deploy with Fleet from plain manifests); Hands-on 8 (deploy with a merge); Hands-on 9 (reconciliation and rollback) |
| 8 | [Fleet with plain manifests or Helm charts](day04/08-fleet-manifests-vs-helm.md) | How to select manifests or Helm; change manifests into a chart; values for each deployment; upstream charts with Fleet | ⌨️ Hands-on 10, 11, 12 |
| 8B | [Templates and pipelines as code: Cookiecutter and Dagger (advanced path)](day04/08b-cookiecutter-and-dagger.md) | Templates for projects and Helm charts; hooks; templates for many projects; the Dagger model; run pipeline steps on your computer and on GitLab's Dagger runner | ⌨️ Hands-on 10B, 11B, 12B |
| 9 | [📖 Day 4 cheat sheet](day04/09-cheatsheet.md) | Git, GitLab CI, scans and signatures, Cookiecutter, Helm, Fleet, Dagger, releases, kubectl, table of problems, reference documents | 📖 Reference |

For section 8, each participant selects **one** path:

- **8:** More Helm with Fleet.
- **8B:** Advanced. Templates and pipelines on your computer.

Both paths end the day. The day has no separate final lab: Hands-on 7 to 9 take one change from a
commit to the cluster and back.

---

## Day 5 — Advanced topics: Monitoring, Security

Full material: **[Day 5 — Advanced topics: Monitoring, Security](day05/index.md)**

> *Find out how your application behaves while it runs, and protect it. Then use the knowledge
> from all five days in a group project.*

**Before you start, you must have:**

- The Day 3 knowledge of Kubernetes, and the `climate-stack` of Day 3, section 10.
- The Day 4 knowledge of GitLab CI, Helm and Fleet (for the group project).
- Your training VM and `kubectl` access to your namespace `traineeNN`.

| # | Section | Topics | Practical work |
| --- | --- | --- | --- |
| 1 | [Monitoring with Prometheus, Grafana, and the Python SDK](day05/01-monitoring-prometheus-grafana-python-sdk.md) | Why monitoring is necessary (probes and monitoring compared, RED); how Prometheus works (pull, labels, `up`); the four metric types; PromQL in five patterns; labels and cardinality; ServiceMonitor; Grafana; what to monitor first; problems and solutions | ⌨️ Hands-on 1 (metrics on the VM); Hands-on 2 (`climate-stats-api` 1.1.0, ServiceMonitor, shared Grafana dashboard); Hands-on 3 (stop the database and see the outage on the dashboard) |
| 2 | [Security fundamentals for cloud-native workloads](day05/02-security-fundamentals.md) | A threat model; authentication and authorisation (`401` and `403`); publish only the paths you mean to; what HTTPS protects; public key cryptography in brief (key pairs, signatures, key exchange, hashes); the TLS handshake, SNI and the chain of trust; TLS termination and HTTPS only; cert-manager and ACME step by step (Certificate, CertificateRequest, Order, Challenge), HTTP-01 and DNS-01, renewal; Traefik middlewares; HTTP Basic Auth at the Ingress controller, with no code change; OpenID Connect and signed tokens (JWT: issuer, subject, audience, expiry), theory and references; pointers to service accounts, security contexts and Pod Security Standards | ⌨️ Hands-on 4 (publish `climate-stats-api` on HTTPS only and follow cert-manager; the CSR and the certificate with `openssl`, the client's view with `curl`); Hands-on 5 (Basic Auth in Traefik, one user for each client; a pod that goes around Traefik) |
| 3 | [Group project: package and serve a model](day05/03-group-project.md) | Use the knowledge from all five days: make a machine learning model and put it in a bucket; an API; a container; GitLab projects and a pipeline; a Helm chart; deployment with Fleet; security; a Grafana dashboard; load and autoscaling | ⌨️ Group work and demonstration |
