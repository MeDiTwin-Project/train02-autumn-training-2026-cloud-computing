# 4. OpenStack: the open-source cloud

Sections 1–3 were concepts. From here on you see a real cloud at work. This section
introduces **OpenStack**, the open-source cloud platform under the European Weather Cloud,
watches a virtual machine being provisioned on it, and ends with you logging into your own
**training VM** (a machine provisioned in exactly that way) and examining it from the inside.

## 4.1 From AWS to open source

By 2010, AWS had shown that API-driven infrastructure worked well, which created a new
problem. If your institution wanted that workflow, your only options were the large US
providers, with their terms, prices, jurisdictions and lock-in. For governments, telecoms,
research institutions and anyone with data-sovereignty constraints, "just use AWS" was not a
satisfactory answer.

The natural response was to build the same machinery as **open-source software**, so that
anyone with a rack of servers could run their own cloud.

- **OpenNebula** (2008, born from European research) is a lean, tidy virtualisation manager
  that is still actively maintained; a good choice for smaller deployments that find
  OpenStack heavy.
- **OpenStack** (2010) is the one that became most widely adopted in the data centre. NASA had
  built a compute fabric called *Nova* and Rackspace had built an object store; in July 2010
  they merged the projects and released them as open source. It grew into one of the largest
  open-source projects, with thousands of contributors and six-monthly releases.
  Today it runs public clouds (OVHcloud, Deutsche Telekom, Infomaniak), telecom infrastructure,
  research clouds (CERN runs one of the largest, at hundreds of thousands of cores),
  and community clouds such as the **European Weather Cloud**.

:::{note} Jargon: *cloud platform*
A **cloud platform** is the software that turns a cluster of servers into a cloud: it tracks
the hardware, schedules VMs onto it (deciding which physical host has room), manages disks,
networks and users, and exposes the functionality through APIs and a web dashboard. OpenStack is a
cloud platform which is free, open, and run by whoever installs it.
:::

Why does open source matter here, beyond price? There are several reasons:
**sovereignty** (your cloud, your hardware, your jurisdiction),
**transparency** (auditable code and no undisclosed behaviour, which is
relevant for e.g. medical data), and **community** (institutions with similar needs pool their
engineering, instead of each paying a vendor separately).

## 4.2 OpenStack architecture: a team of services

OpenStack is a *collection of cooperating services*, each with one job and its own REST API,
all talking to each other. Dozens exist, but a few do most of the work, and their names are
worth knowing because OpenStack clouds, dashboards, CLIs and error messages all use them:

| Service | Job | Remember it as |
|---|---|---|
| **Keystone** | Identity: users, projects, permissions, tokens. Every request starts here. | *the gatekeeper* |
| **Nova** | Compute: schedules and runs the virtual machines (via KVM/QEMU, section 1). | *the VM factory* |
| **Neutron** | Networking: virtual networks, subnets, routers, floating IPs, security groups. | *the network plumber* |
| **Cinder** | Block storage: volumes, i.e. virtual disks you attach to VMs. | *the disk shelf* |
| **Swift** | Object storage: buckets/containers of files behind an HTTP API (S3-style). | *the archive* |

Two supporting services you will meet without noticing: **Glance** stores the VM *images*
(the template disks, such as "Ubuntu 24.04" or "Rocky 9", that new VMs boot from), and
**Horizon** is the web dashboard you are about to see in the demo.

```{mermaid}
flowchart TB
    User([You]) -->|dashboard| Horizon
    User -->|CLI / Python| APIs
    Horizon --> APIs["REST APIs"]
    APIs --> Keystone["Keystone<br/>identity"]
    APIs --> Nova["Nova<br/>compute<br/>(QEMU/KVM)"]
    APIs --> Neutron["Neutron<br/>networking"]
    APIs --> Cinder["Cinder<br/>block storage"]
    APIs --> Swift["Swift<br/>object storage"]
    Nova -->|may boot image from| Glance["Glance<br/>images"]
    Keystone -.->|authorises| Nova & Neutron & Cinder & Swift
```

The flow when you ask for a VM: Keystone checks who you are → Nova picks a physical host with
room → Glance supplies the image, from which the VM's root disk is created (Cinder is involved
only if the VM boots from, or is given, a volume: section 5) → Neutron wires up the network.
Every arrow is an API call, which is why, in
[section 7](07-apis.md), you will be able to make the same calls yourself from Python.

:::{note} Jargon: *project* (a.k.a. *tenant*)
In OpenStack, a **project** is the unit of ownership and quota: VMs, volumes and networks
belong to a project, not to a person, and everyone in the project sees and shares them. The
training VMs, for instance, all belong to one project run by the organisers. If you see the
word *tenant*, it means the same thing.
:::

:::{note} Jargon: *flavor* and *image*
A **flavor** is a machine size: a named combination of virtual CPUs, RAM and root disk (e.g.
`2cpu-4gb`). An **image** is the template the machine boots from (e.g. Ubuntu 24.04). 
:::

## 4.3 A real OpenStack cloud: the European Weather Cloud

The **European Weather Cloud (EWC)** is the community cloud operated by **ECMWF** and
**EUMETSAT** for the European meteorological community. It runs on OpenStack and sits
physically next to the meteorological archives, so data-heavy work happens beside the data
instead of moving petabytes across the internet. 

## 4.4 How the rest of this lesson works

You do **not** get an OpenStack account in this training. Instead, the day uses two kinds of
activity, and the material marks each one clearly:

| Box | Who acts | Where |
|---|---|---|
| 👀 **Instructor demo** | The instructor drives a real OpenStack cloud on the projector. **Watch, don't type**: you have no credentials for it. | An OpenStack cloud |
| ⌨️ **Your turn** | You type every command yourself. | **Your training VM** |

Your **training VM** is a Linux virtual machine the organisers provisioned for you, on a
cloud, in exactly the way the demo below shows. In [section 0](00-getting-connected.md) you
received:

- its **name** and **IP address**, and your **trainee username** (the login user on the VM
  itself is `cloud-user`);
- an **SSH key**, downloaded from the credential portal;
- **`sudo`** rights inside it: it is your sandbox for the whole week.

The demo therefore shows you the cloud from the *provider's API* side (how a VM, a disk and
a network get created), while your VM lets you examine the same things from the *inside*
and, in sections 5 and 6, work with them hands-on.

:::{admonition} 👀 Instructor demo: watch, don't type
:class: hint
Boxes like this one describe what happens on the projector. Commands inside them need
OpenStack credentials, which only the instructor has. Read along; the commands are there so
that you can recognise them later, and are not meant to be run now.
:::

## 4.5 Demo: a tour of the dashboard

:::{admonition} 👀 Instructor demo: the Horizon dashboard
:class: hint
The instructor logs into the Horizon dashboard of an OpenStack cloud and lands on the
**Overview** page, a set of gauges showing the project's quota (instances used, CPUs, RAM,
volumes, floating IPs). This is followed by a walk through the left-hand menu:

1. **Compute → Instances**: the project's VMs. Resources belong to the *project*, so
   everyone in it sees the same list.
2. **Compute → Images**: the boot templates available (Ubuntu, Rocky, …).
3. **Compute → Key Pairs**: SSH public keys registered for VM login.
4. **Volumes → Volumes**: block storage (covered in section 5).
5. **Network → Network Topology**: a live diagram of the project's networks and routers
   (covered in section 6).
:::

While you watch, answer these for yourself, as they come back later:

- How many vCPUs and how much RAM does the project's quota allow? How much is used?
- What is the smallest flavor on offer?
- In the network topology, how many networks are there, and which one is *external*?

:::{tip} The dashboard is just an API client
Everything Horizon shows, it learned by calling the same REST APIs that the CLI and Python
call in [section 7](07-apis.md). The dashboard has no special access; it is one client among
many.
:::

## 4.6 Demo: provisioning a virtual machine

:::{admonition} 👀 Instructor demo: launch an instance
:class: hint
**Step 1: keypair.** VMs have no passwords; you log in with an SSH key that OpenStack
injects into the machine at first boot. The instructor generates one on their laptop…

```bash
ssh-keygen -t ed25519 -f ~/.ssh/demo-key -N ""
cat ~/.ssh/demo-key.pub
```

…and registers the public half under **Compute → Key Pairs → Import Public Key**.

**Step 2: launch.** **Compute → Instances → Launch Instance**. The wizard has tabs on the
left:

| Tab | Setting |
|---|---|
| Details | Instance name: `demo-vm1` |
| Source | Boot source: **Image** → an Ubuntu image |
| Flavor | A small flavor (2 vCPU / 4 GB class) |
| Networks | The project's internal network, e.g. `private-net` |
| Key Pair | The key just imported |

After **Launch Instance**, the status column goes `Build` → `Spawning` → **`Active`**,
typically in under a minute.

**Step 3: reach it.** The new VM has an address like `192.168.1.14` on the internal
network. This is a **private** address, reachable only inside the project. To reach it from the
laptop it needs a **floating IP** (a public address): row menu → **Associate Floating IP** →
allocate one from the external pool → **Associate**. A **security group** rule must also
allow SSH in (section 6). Then:

```bash
ssh -i ~/.ssh/demo-key ubuntu@<floating-ip>
```

```text
Welcome to Ubuntu 24.04 LTS (GNU/Linux ... x86_64)
ubuntu@demo-vm1:~$
```
:::

It is worth thinking about what just happened, because it summarises most of the day in
about a minute. Keystone verified the instructor, Nova chose a physical server in a hall
nobody in the room is likely to see, Glance handed over an Ubuntu disk image, QEMU/KVM
(section 1) started a guest from it, Neutron attached it to a virtual network.

## 4.7 Your turn: log into your training VM

Your training VM went through that sequence before you arrived. Now you can log in.

### Step 1: connect

On **your laptop**, use the SSH shortcut from [section 0](00-getting-connected.md):

```bash
ssh -i <path-to-your-key> <user>@<vm-address>
```

```text
[cloud-user@<vm-name> ~]$
```

### Step 2: is this really a virtual machine?

From inside, a VM looks like an ordinary computer. A few commands tell you more:

```bash
cat /etc/os-release | head -2          # which image the VM was created from
nproc                                  # vCPUs: set by the flavor
free -h                                # RAM: set by the flavor
df -h /                                # root disk: created from the image
systemd-detect-virt                    # which hypervisor is underneath?
cat /sys/class/dmi/id/product_name     # what the "hardware" claims to be
```

```text
NAME="Rocky Linux"
VERSION="9.7 (Blue Onyx)"
4
               total        used        free      shared  buff/cache   available
Mem:            15Gi       792Mi        11Gi        16Mi       3.3Gi        14Gi
Swap:             0B          0B          0B

Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        38G  4.7G   32G  13% /

kvm

OpenStack Nova
```

`kvm` is the hypervisor from section 1. The product name is
whatever the emulated motherboard claims: **`OpenStack Nova`** if your VM runs on an
OpenStack cloud, something like `Standard PC (Q35 + ICH9, 2009)` on plain QEMU, other names on
other clouds. 

### Step 3: how did my key get in here?

```bash
sudo cloud-init status --long
```

**cloud-init** is the first-boot agent: it asked the cloud for the instance's
configuration (hostname, SSH keys, any setup script) and applied it; the SSH public key, for
instance, is written into `~/.ssh/authorized_keys`, the list of keys allowed to log in. Most
clouds serve that
configuration from a **metadata service** at a special link-local address, which you can
query directly:

```bash
curl -s http://169.254.169.254/openstack/latest/meta_data.json | jq '.'
```

```text
{
    "uuid": "8a1c9e2f-3b4d-4c5e-9f6a-7b8c9d0e1f2a",
    "name": "meditwin-training-0",
    "availability_zone": "nova",
    "public_keys": {
        "trainee00": "ssh-rsa AAAAC3NzaC1lZDI1NTE5AAAA..."
    },
    ...
}
```

That is Nova describing your VM to the VM itself.  (If the `curl` returns nothing, usually
means that your VM is not on an OpenStack cloud. Other clouds use the same address with a
different path; that is fine, just skip this step.)

### Step 4: note your addresses

```bash
ip -4 -brief address
```

```text
lo               UNKNOWN        127.0.0.1/8
enp3s0           UP             192.168.1.84/24
docker0          UP             172.17.0.1/16
```

Note the address on the main interface (here `192.168.1.84`), and compare it with the
address you typed into `ssh`. Section 6 explains why these addresses differ.

## Recap

- Open-source cloud platforms exist so that *anyone* can run a cloud, for sovereignty,
  transparency and pooled engineering. **OpenNebula** is the lean option; **OpenStack** (NASA +
  Rackspace, 2010) is the de-facto standard and runs the EWC.
- OpenStack is a **team of services**: **Keystone** (identity), **Nova** (compute),
  **Neutron** (network), **Cinder** (volumes), **Swift** (objects), plus Glance (images) and
  Horizon (dashboard). Learn the five names.
- From the inside, a VM can be recognised: `systemd-detect-virt` names the hypervisor, the
  "hardware" calls itself `OpenStack Nova`, and **cloud-init** configured it at first boot from
  the cloud's **metadata service**.

## Check yourself

1. Trace a "Launch Instance" click: name, in order, the OpenStack services involved and what
   each contributes.
2. Your colleague launches a VM and cannot SSH to `192.168.1.14`, the only address shown for
   it, from their laptop. Two distinct things are missing. What are they?
3. Why did institutions like CERN or ECMWF build OpenStack clouds instead of using AWS,
   which existed and worked?
4. Nobody typed your SSH public key into your training VM, yet it was there at first boot.
   Which component put it there, and where did that component get it from?

:::{dropdown} Solutions
**1.** **Keystone** authenticates you and checks that you may act in the project → **Nova**
schedules the VM onto a physical compute host and drives KVM/QEMU → **Glance** supplies the
boot image, from which the root disk is created → (**Cinder**, only if the VM boots from a
volume or is given one) → **Neutron** creates the virtual network port, wires it to the
network, and applies security groups. Horizon was merely the client making the API calls.

**2.** First, `192.168.1.14` is a **private** address on the project's internal network. The
laptop, on the internet, has no route to it; the VM needs a **floating IP** associated.
Second, even with a floating IP, the **security group** must allow inbound TCP 22, or the
connection will hang/refuse. The two failures look identical from the laptop, which is why
the section 6 material teaches you to check both.

**3.** Any of: data gravity (the compute must sit next to petabyte archives, not across the
Atlantic), sovereignty, privacy, cost at steady scale (their base
load is constant and large, so the own-vs-rent maths favours owning), and
control/customisation of the platform.

**4.** **cloud-init**, running inside the VM at first boot. It fetched the instance's
configuration (including the public key registered with the keypair when the VM was
launched) from the cloud's **metadata service** at `169.254.169.254`, which Nova serves to
each VM, then wrote the key into `~/.ssh/authorized_keys`. 
:::

---

Next: [Cloud storage](05-storage.md): where your VM's disk actually lives, and why there are
two kinds of storage.
