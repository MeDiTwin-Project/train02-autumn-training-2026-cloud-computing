# 10. Day 1 recap

The day started from one sentence: **a computer does not have to be a physical box; compute,
storage and networks can be created, resized and destroyed with an API call, and paid for
only while they exist.** Here is how each section filled it in.

| # | Section | The one thing to remember |
|---|---|---|
| 0 | Getting connected | You reach the cloud with **keys, not passwords**: an SSH key for your VM, credentials that stay private. |
| 1 | Virtualization | A **hypervisor** (KVM/QEMU) lets one physical machine pretend to be many isolated ones; this is what makes the cloud *possible*. |
| 2 | Cloud computing | The cloud is self-service, on-demand, elastic, measured access to a shared pool; elasticity means growing **and shrinking**. |
| 3 | Service models | IaaS → PaaS → SaaS (and FaaS): for any service, ask **which layers are mine** to patch, secure and back up. |
| 4 | OpenStack | A team of services with APIs: **Keystone, Nova, Neutron, Cinder, Swift** (+ Glance, Horizon); **cloud-init** configured your VM at first boot. |
| 5 | Storage | **Block** (a volume: a disk for one VM) vs **object** (a bucket: files over HTTP for anyone with keys); Ceph underneath; replication is not backup. |
| 6 | Networking | Subnets and routes; `ping`, `traceroute`, `curl`; **private** addresses + **NAT** out, **floating IPs** in; **security groups** decide what gets through. |
| 7 | Cloud APIs | Dashboard, CLI and Python are equal clients of the same APIs; credentials live in **rc files / config files**, never in code; **S3** is signed HTTP. |
| 8 | Lambda functions | Rent only the **execution of a function**, per event; stateless and short; a FaaS function is a **container plus a convention**. |
| 9 | Containers | A container is a process in **namespaces + cgroups** sharing the host kernel; containers run *inside* VMs. |

What you did with your own hands today, on your training VM:

- logged in with an SSH key and recognised, from the inside, a **KVM guest on OpenStack Nova**
  configured by cloud-init (section 4);
- created, formatted, mounted, detached and re-attached a **block device**, and stored, listed
  and fetched **objects** in a bucket with `s3cmd` (section 5);
- read your VM's addresses and routes, debugged connections with `ping`, `traceroute` and
  `curl`, and found the **security group** that silently dropped your web server's traffic
  (section 6);
- used the **S3 API** from `s3cmd`, from raw `curl` and from Python, and shared a file with a
  **pre-signed URL** instead of your keys (section 7).

---

Next: [Day 2 — Containers & Docker](../day02/index.md), where the containers of section 9 are
built, run and connected with Docker.
