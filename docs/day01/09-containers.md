# 9. Containers overview

This section provides a short introduction to containers: what a container *is*, where the idea
came from, and how it differs from virtual machines. 

## 9.1 From jails to namespaces: the long prehistory

The idea of isolating processes by giving them a restricted view of the system, rather than by
emulating hardware, goes back about forty years:

- **1979, `chroot`** (Unix V7): change a process's idea of where the root of the filesystem
  is. It sees `/` but is actually confined to a subdirectory. This covers the filesystem only.
- **2000, FreeBSD `jails`**: an extended `chroot`, with filesystem, process list, network and
  users isolated, designed so hosting providers could give customers safely separated slices
  of one machine. The first thing recognisable as a container.
- **2001–2005, Linux-VServer, Solaris Zones, OpenVZ**: the same idea, re-implemented per OS,
  each with patched kernels and its own vocabulary.
- **2002–2013, Linux namespaces**, added one by one to the mainline kernel: mount, PID,
  network, user, and more. Plus **cgroups**
  (Google, 2007): limit how much CPU, memory and I/O a group of processes may consume.
- **2008, LXC**: combines namespaces and cgroups into "Linux containers", which look and feel
  very similar to "lightweight VMs".

By 2008, then, the Linux kernel contained the mechanisms a container needs and a container
implementation (LXC). What was missing was a convenient way for an ordinary developer to *use*
them.

:::{note} Jargon: *namespaces* and *cgroups*
The two kernel mechanisms every Linux container is made of. **Namespaces** control what a
process can *see* (its own filesystem, process list, network stack, hostname…); **cgroups**
control what it can *use* (CPU shares, memory ceiling, I/O bandwidth). A container is a
process wrapped in both; there is no "container" object in the kernel at all.
:::

## 9.2 The five-minute talk

:::{admonition} 📖 Story: Solomon Hykes and Docker
:class: seealso
In March 2013, **Solomon Hykes** went to **PyCon** in Santa Clara to give a five-minute
lightning talk. He expected a back room and perhaps thirty people. PyCon, however, runs its
lightning talks on the *main* stage, in front of several hundred. The talk was titled "The
Future of Linux Containers." It turned out to have a considerable effect on how software is
shipped.

Hykes had not set out to pitch containers. His startup, **dotCloud**, founded in France in
2008 and later part of Y Combinator, was a platform-as-a-service business that was not doing
particularly well. **Docker** was an internal tool, the plumbing they had built to package and
run customers' apps. The technology underneath was not new: Linux already had
**namespaces**, **cgroups**, and **LXC** to isolate processes. What Docker added was a
well-designed interface.

It made containers **pleasant to use**. A **Dockerfile** let you describe an app's
environment in a few readable lines. An **image** bundled your code and what it needed into
one portable, versioned artefact that ran the same way on a laptop, a test server, or a
cloud. A **registry** let you share it with a single command. `docker run` simply worked. For
developers used to the "well, it works on my machine" problem, this was a welcome change.

The reaction was quick. Someone posted a link to the not-yet-released Docker website on
Hacker News, where a sceptic dismissed it as "vaporware." Within two weeks, the team
**open-sourced Docker**. The response was strong enough that Hykes moved the company away
from its paying product to concentrate on the free tool. dotCloud became **Docker, Inc.**

The decision paid off. Docker was adopted by developers unusually quickly, engineers began
writing Dockerfiles, and the cloud providers added support for containers. Google
contributed the core runtime code that became the **Open Container Initiative** standard, so
that no single company owned the format, and Docker's design became the foundation that
**Kubernetes** would orchestrate.

Hykes stepped down as Docker's CTO in 2018 and started a new company, **Dagger**, in 2019.  He
had not invented containers: he made them simple enough that a great many people wanted to use
them, starting from a five-minute talk he had expected to give to a nearly empty room.

*Further reading:
[Video🍿 pyvideo.org: The future of Linux Containers](https://pyvideo.org/pycon-us-2013/the-future-of-linux-containers.html) ·
[en.wikipedia.org/wiki/Docker_(software)](https://en.wikipedia.org/wiki/Docker_(software))*
:::

## 9.3 Containers vs VMs, precisely

You now know both sides well enough for a proper comparison:

```{mermaid}
flowchart TB
    subgraph VMside["Virtual machines"]
        direction TB
        HW1["Physical host"] --> HV["Hypervisor (KVM/QEMU)"]
        HV --> G1["Guest kernel + OS<br/>App A"]
        HV --> G2["Guest kernel + OS<br/>App B"]
    end
    subgraph Cside["Containers"]
        direction TB
        HW2["Physical host"] --> K["One shared Linux kernel"]
        K --> C1["namespaces+cgroups<br/>App A + its libraries"]
        K --> C2["namespaces+cgroups<br/>App B + its libraries"]
    end
```

| | Virtual machine | Container |
|---|---|---|
| Isolates by | emulating hardware; guest runs its **own kernel** | kernel features; all share the **host kernel** |
| Starts in | tens of seconds | milliseconds |
| Weighs | gigabytes | megabytes |
| Can run | any OS (Windows on Linux, other kernels) | Linux userlands only, matching the host kernel |
| Isolation strength | strong (hypervisor boundary) | good, but thinner (kernel boundary) |

In practice, containers run *inside* VMs. The cloud rents your project VMs with strong
isolation, and you fill them with fast, portable containers. 

A container image that bundles *code plus its environment* is the artefact that Continuous
Integration (CI) builds and Kubernetes runs.

## Recap

- Containers are processes wrapped in **namespaces** (what you see) and **cgroups** (what you
  may use), with a lineage running `chroot` (1979) → FreeBSD jails (2000) → Linux namespaces.
- **Docker (2013)** added no new kernel mechanism; it added the **Dockerfile, image and
  registry**, a packaging layer that was widely adopted. Usability improvements can be
  significant advances.
- VMs bring their own kernel: heavier, stronger isolation, any OS. Containers share the host
  kernel: near-instant start, small, Linux-only, thinner isolation.
- They are complementary: **containers inside VMs** is the standard cloud arrangement, with
  the VM for isolation and the container for packaging. 

## Check yourself

1. Linux had namespaces and cgroups years before Docker. In one sentence: what did Docker
   add, given that the isolation mechanism is the same?
2. A vendor claims their container platform runs "Windows containers on our Linux hosts,
   fully native." Why should you raise an eyebrow?
3. Why do clouds like the EWC typically run Kubernetes on OpenStack VMs rather than
   directly on the physical servers?

:::{dropdown} Solutions
**1.** Docker wraps the same kernel isolation in a *distributable artefact* (an image built
from a Dockerfile, carrying the filesystem and environment, shareable via a registry and
runnable with one command), whereas a bare namespace is empty isolation with no content, no
packaging and no way to hand it to a colleague.

**2.** A container **shares the host's kernel**, and a Windows application needs a Windows
kernel, which a Linux host does not have. Either the product runs a Windows VM under each
"container" (fine, but then it has VM isolation and VM weight, so it is not "fully native"),
or the claim is wrong. Kernel sharing is the defining constraint; claims that violate it
usually hide a VM somewhere.

**3.** Multi-tenancy and blast radius. The physical servers are shared by many projects, and
container/kernel isolation alone is too thin a wall between tenants that may not trust each
other. VMs give each project a hypervisor-level boundary and its own kernel to manage; inside
that boundary, containers and Kubernetes provide fast packaging and orchestration. 
:::

---

Next: [Day 1 recap](10-day01recap.md)
