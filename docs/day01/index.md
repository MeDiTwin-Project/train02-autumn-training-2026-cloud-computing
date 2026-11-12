# Day 1: Cloud Computing Foundations

Welcome to Day 1, and welcome to the training. Today is about **foundations**: what a cloud
actually is, why it exists, and how a virtual machine, a disk, a network and a bucket come out
of one. 

## The one story this day tells

Most of today follows from a single sentence:

> **A computer does not have to be a physical box. Compute, storage and networks can be
> created, resized and destroyed *with an API call*, and paid for only while they exist.**

Virtualization makes this *possible*, and cloud computing offers it as a *service*. The rest of
the day covers the names of the moving parts and how they fit together.

```{mermaid}
flowchart TB
    You([You]) -->|API call / CLI / dashboard| Cloud[Cloud platform<br/>OpenStack]
    Cloud -->|creates| VM[Virtual machine]
    Cloud -->|creates| Vol[(Volume<br/>block storage)]
    Cloud -->|creates| Net[Virtual network<br/>+ floating IP]
    Cloud -->|creates| S3[(Object storage<br/>S3 bucket)]
    Vol --- VM
    Net --- VM
```

## What you will be able to do by the end

- Explain what virtualization is, name its main forms (virtual machines, containers,
  unikernels) and say why each exists.
- Explain what cloud computing is, where it came from, and why it matters for scientific
  workloads.
- Tell IaaS, PaaS, SaaS and FaaS apart, and tell public, private and hybrid clouds apart.
- Describe what OpenStack is and name its core services (Nova, Cinder, Swift, Neutron, Keystone),
  and trace what happens when a virtual machine is provisioned.
- Log in to your **training VM** and tell, from the inside, that it is a virtual machine.
- Explain block storage versus object storage, and format, mount, detach and re-attach a
  **block device** yourself.
- Read a VM's network setup (addresses, subnets, routes) and debug connectivity with
  `ping`, `traceroute` and `curl`.
- Tell **private** from **public** addresses, and explain **NAT**, **floating IPs** and
  **security groups**.
- Read cloud automation written for the **OpenStack CLI and Python SDK**, and use the **S3**
  API yourself, from the command line and from Python, to upload, retrieve and share files.
- Explain what a lambda function is and what a container is.

## Prerequisites

| You need | Notes |
|---|---|
| A laptop with a terminal and an SSH client | Linux, macOS, or Windows (PowerShell or WSL) |
| Basic shell comfort | `cd`, `ls`, editing a file, running a command |
| Your **training credentials** | Trainee username, claim code, VM name and VM IP address, sent to you by e-mail |
| Your **training VM** | A Rocky Linux 9 VM provisioned for you, with `sudo`; you log in as `cloud-user` with the SSH key from the credential portal ([section 0](00-getting-connected.md)) |
| **S3 access** | Shown in the credential portal, and already configured on your VM in `~/.s3cfg` (sections 5 and 7) |

Check you are ready, from your laptop:

```bash
ssh -V                                         # any version is fine
ssh -i <path-to-your-key> <user>@<vm-address>  # you should land on your VM
```

and, once on the VM:

```bash
python3 --version          # 3.9 or later is fine
sudo -v                    # must not ask for a password you do not have
s3cmd ls                   # your S3 access works: lists buckets, or prints nothing
```

## How today works

Nobody gets an OpenStack account in this training, but the day is still hands-on. The
material marks the two kinds of activity clearly:

| Box | Who acts | Where |
|---|---|---|
| 👀 **Instructor demo** | The instructor drives a real OpenStack cloud on the projector. **Watch, don't type**, as you have no credentials for it. | An OpenStack cloud |
| ⌨️ **Your turn** | You type every command yourself. | Your training VM (and your laptop, for SSH) |

The demos show the cloud from the API side. The hands-on parts let you see the same mechanisms
from the inside, and test them yourself: a block device, a web service behind a
security group, a floating IP, an S3 bucket.

## Sections

| # | Section | What it covers |
|---|---|---|
| 0 | [Getting connected](00-getting-connected.md) | Your credentials, SSH key and VM login; VS Code Remote - SSH; GitLab and Rancher sign-in |
| 1 | [Virtualization](01-virtualization.md) | Pretending to be a computer, and why that turned out to be useful |
| 2 | [Cloud computing](02-cloud-computing.md) | What the cloud is, where it came from, why scientists use it |
| 3 | [IaaS, PaaS, SaaS](03-service-models.md) | The service models, and public vs private vs hybrid |
| 4 | [OpenStack](04-openstack.md) | The open-source cloud, a VM provisioned live, and your training VM from the inside |
| 5 | [Cloud storage](05-storage.md) | Block storage, object storage, Ceph underneath, and a block device of your own |
| 6 | [Cloud networking](06-networking.md) | IP addresses, ports, subnets and routing; debugging with `ping`, `traceroute` and `curl`; private vs public addresses, NAT and floating IPs, firewalls and security groups |
| 7 | [Cloud APIs](07-apis.md) | Driving OpenStack from the CLI and Python; S3 yourself, from the CLI and Python |
| 8 | [Lambda functions](08-faas.md) | Running code without owning a server |
| 9 | [Containers overview](09-containers.md) | A first look at containers, ready for Day 2 |
| 10 | [Day 1 recap](10-day01recap.md) | The whole day on one page, and what you did with your own hands |

Section 0 gets you connected; sections 1–3 are the concepts; sections 4–7 combine live demos on
a real cloud with hands-on on your training VM; sections 8–9 look ahead; section 10 sums up the
day.

## A word on jargon

Cloud computing uses a large amount of vocabulary, most of which is simpler than it sounds.
Whenever a new word appears today it is defined in a box like this one:

:::{note} Jargon: *provision*
To **provision** a resource is to ask the cloud to create it (a VM, a disk, a network). The
opposite is to *deprovision* or *release* it.
:::

If a word gets used without a box, that is an oversight, so please ask.

