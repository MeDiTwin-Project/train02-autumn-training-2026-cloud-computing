# 1. Virtualization

Virtualization is the technology underneath most of what you will see this week. If "virtual
machine" stays a vague phrase, the cloud will be hard to reason about. This section tries to
make it concrete.

## 1.1 The problem virtualization solves

Imagine a research group in the early 2000s. They have five physical servers in a rack:

- one runs the group's website, at 3% CPU load;
- one runs the mail server, at 2%;
- one runs a database, busy for one hour per night;
- two run simulations, but only when somebody has a paper deadline.

Every server was bought for its worst hour and idles the rest of the time. Adding a new service
means a purchase order, a delivery, rack space, cables, and weeks of waiting. Retiring one is
awkward too: nobody dares to switch it off, in case *something* is still using it.

The obvious fix, running everything on one big server, fails for a different reason. The
website, the mail server and the database now share one operating system. A library upgrade
needed by one of them may break another; if one is compromised, all of them are; if one crashes
the kernel, they all go down.

:::{note} Jargon: *kernel*
The **kernel** is the core of the operating system: the first program to start when the
machine boots, and the only one with direct control over the hardware. It decides which program
gets the CPU and when, hands out memory, and talks to disks and network cards. Every other
program (your shell, a web server, Python) has to ask the kernel to do these things on its
behalf. Strictly speaking, "Linux" is the name of a kernel. As you will see in this section,
whether something has *its own* kernel or *shares* one is the main difference between a virtual
machine and a container.
:::

What you would like sounds contradictory: **the isolation of five machines with the hardware of
one.** Virtualization resolves that contradiction.

:::{note} Jargon: *virtualization*
**Virtualization** is making one physical computer pretend to be several independent ones (or,
more generally, making a physical resource, such as a disk or a network, appear as several
independent virtual copies of itself). The pretend computers are **virtual machines (VMs)**;
the physical one is the **host**; each VM is a **guest**.
:::

```{mermaid}
flowchart TB
    subgraph Host["One physical server (host)"]
        HV["Hypervisor"]
        subgraph VM1["VM: website"]
            OS1["Own OS"]
        end
        subgraph VM2["VM: mail"]
            OS2["Own OS"]
        end
        subgraph VM3["VM: database"]
            OS3["Own OS"]
        end
        HV --- VM1
        HV --- VM2
        HV --- VM3
    end
```

Each guest behaves as if it owned a whole computer, with its own CPUs, memory, disk and network
card. In reality all of these are shared or emulated, and from the guest's point of view the
difference is mostly invisible.

:::{note} Jargon: *hypervisor*
The **hypervisor** is the piece of software that runs the guests and keeps the illusion up:
it shares out the real CPU and memory, and emulates the hardware each VM thinks it has.
**Type 1** hypervisors run directly on the hardware (e.g. KVM, Xen, VMware ESXi); **type 2** run as
an application inside a normal OS (e.g. VirtualBox). Clouds use type 1, most often
**KVM**, which is built into the Linux kernel.
:::

## 1.2 An old idea that disappeared and came back

Virtualization was not invented for the cloud. IBM shipped it on mainframes in **1972**
(VM/370): one very expensive machine, divided into virtual machines so that many users and
workloads could share it safely. On a mainframe the hardware costs so much that sharing it
efficiently was the main business case.

Then hardware became cheaper. Through the 1980s and 1990s the industry moved to "one cheap x86
box per task" and virtualization was largely set aside; the x86 processor could not even be
virtualised properly, and few people minded.

It came back for the same reason it was invented: **utilisation**. By the late 1990s many
organisations were struggling with the five-servers-at-3%-load problem from section 1.1. VMware
(1998) found a way to virtualise x86 in software; Intel and AMD then added hardware support
(**Intel VT-x**, **AMD-V**, 2005–2006), which made it fast; and the Linux kernel gained **KVM**
in 2007, which made it free and widely available.

This is worth noticing as a pattern, because it keeps repeating:

:::{important} Centralisation and decentralisation move in cycles
Mainframes (centralised, shared) → personal computers and departmental servers (decentralised,
dedicated) → the cloud (centralised, shared again) → and today "edge computing" pushes some
work back out towards decentralisation. Each swing happens because the economics shift:
hardware cost, network cost, people cost. The cloud is the current position of a pendulum, and
keeping this in mind helps you avoid treating any architecture as permanent.
:::

## 1.3 How the illusion works

You do not need hypervisor internals this week, but one level of understanding is useful.
There are three classic ways to virtualise a computer:

| Approach | How it works | Cost |
|---|---|---|
| **Full virtualization (emulation)** | Software reads every guest instruction and imitates its effect. The guest can even be a different CPU architecture (run ARM code on x86). | Slow, since every instruction is interpreted or translated |
| **Paravirtualization** | The guest OS *knows* it is virtualised and asks the hypervisor instead of touching hardware. | Fast, but requires a modified guest OS (popularised by Xen) |
| **Hardware-assisted virtualization** | The CPU itself (VT-x / AMD-V) runs guest code directly at full speed, trapping to the hypervisor only for the few operations that need supervision. | Near-native speed; this is what KVM uses |

In practice, open-source virtualisation software such as libvirt combine the last two: KVM lets guest code run
directly on the CPU, and **QEMU** provides the emulated devices (disk controllers, network cards) around it,
usually paravirtualised ones (**virtio**) for speed.

:::{admonition} 📖 Story: Fabrice Bellard
:class: seealso
In 1989, a 17-year-old at a lycée in Montpellier wrote a program called **LZEXE** that
compressed MS-DOS executables so they ran from smaller files. It spread across Europe. The
teenager, **Fabrice Bellard**, moved on to other things, as he would keep doing throughout a
rather unusual career in computing that relatively few people have heard of.

Bellard has mostly stayed away from founding companies or seeking attention. He tends to enter
a contest, or simply get curious, and produce something that much of the industry ends up
relying on. In **2000** he launched **FFmpeg**, the multimedia library now used in VLC, Chrome,
YouTube, Blender and a large share of the software that handles video. A year later he won the
International Obfuscated C Code Contest, and the winning entry grew into the **Tiny C
Compiler**, small and fast enough to compile and boot a Linux kernel in about fifteen seconds.

Then, in **2003**, working essentially alone, he wrote **QEMU**, and he remained almost its
only author until 2005. QEMU can pretend to be an entire computer: hand it a program built
for a different processor, such as ARM or PowerPC, and it translates the instructions on the
fly and runs them on your ordinary x86 laptop. This technique, dynamic binary translation, lets
one machine stand in for many others. Paired a few years later with Linux's **KVM**, it became
the standard open-source virtualization stack. OpenStack clouds run their VMs on KVM and QEMU,
and so does Oracle Cloud. The training VM you will log into in [section 4](04-openstack.md) is
running on QEMU.

In 1997 he had already published a new formula for computing digits of pi, and 
in 2009 he announced he had calculated pi to nearly **2.7 trillion digits**, using a single
desktop PC left running for about 90 days. This beat a record set by machines that cost a
great deal more. He also built **BPG** (an image format), **QuickJS** (a compact JavaScript
engine), and **JSLinux**, which boots Linux inside a web browser. The programmer John Carmack,
well known in his own right, once said Bellard is "almost certainly a better overall programmer
than I am."

*Further reading: [bellard.org](https://bellard.org) ·
[en.wikipedia.org/wiki/Fabrice_Bellard](https://en.wikipedia.org/wiki/Fabrice_Bellard) ·
[qemu.org](https://www.qemu.org)*
:::

## 1.4 The forms of virtualization

"Virtual machine" is not the only way to slice a computer. There is a spectrum, trading
isolation against weight:

```{mermaid}
flowchart LR
    A["Virtual machine<br/>own kernel, emulated hardware<br/><b>heaviest, strongest isolation</b>"]
    B["Kata container<br/>container packaging,<br/>minimal VM around it"]
    C["Container<br/>shared kernel,<br/>isolated view of the OS"]
    D["Unikernel<br/>app compiled together<br/>with a mini-OS<br/><b>lightest</b>"]
    A --- B --- C --- D
```

**Virtual machines** run a complete operating system of their own, on emulated hardware. They
take tens of seconds to boot and gigabytes of disk, and they can run practically any OS: Linux
VMs on a Linux host, Windows on Linux, and so on. They offer the strongest isolation, since to
escape, an attacker must break the hypervisor itself.

**Containers** do not emulate hardware and do not run their own kernel. They are ordinary
processes on the host, given an *isolated view* of the operating system (their own file
system, network, process list) using Linux kernel features called namespaces and cgroups.
They start in milliseconds and weigh megabytes. The downside is that every container shares the
host's kernel, so the isolation is weaker. More about this topic to come.

**Kata containers** are a hybrid: each container is wrapped in its own minimal VM, so you keep
the container workflow and get VM-grade isolation. Clouds use this style of sandboxing to run
other people's containers safely on shared hardware (AWS built **Firecracker**, a minimal VM
monitor, for this purpose).

**Unikernels** go the other way: instead of putting a whole OS in the VM, you compile your
application *together with* just the OS pieces it needs into a single small bootable image,
with no shell, no other processes and very little to attack. Milliseconds to boot and secure.
However, debugging might turn out to be a bit harsh.

:::{note} Jargon: *isolation*
**Isolation** is the guarantee that tenants sharing hardware cannot read each other's data,
exhaust each other's resources, or crash each other. How much isolation you need, and how much
weight you accept to get it, is the axis along which the spectrum above is arranged.
:::

## 1.5 Not only compute: virtual storage and virtual networks

The same approach applies well beyond CPUs.

**Virtual storage.** The "disk" inside a VM is typically just a file (or a slice of a storage
cluster) on the host. This means that the disk can be *resized*, *snapshotted*,
*copied* and *moved to another physical machine* while the VM keeps running.
[Section 5](05-storage.md) covers this in detail.

**Virtual networks.** The network card inside a VM is software too, and so are the switches
and routers it plugs into. A cloud builds complete private networks, with subnets, routers
and firewalls, that exist only as kernel data structures and packets-inside-packets on the
real wires. [Section 6](06-networking.md) covers this in detail, and you will learn to find
your way around one.

Keep this generalisation in mind: **anything the cloud gives you (machine, disk, network) is a
software object pretending to be hardware.** That is why it can be created and destroyed by an
API call, which is the subject of the next section.

## Recap

- Virtualization makes one physical machine act as several isolated virtual ones: **isolation
  of many, hardware of one.** It was invented for 1970s mainframes and revived for the same
  reason, namely utilisation.
- The **hypervisor** maintains the illusion; open-source clouds such as OpenStack use **KVM**
  (in the Linux kernel) with **QEMU** providing devices, at near-native speed thanks to CPU support (VT-x/AMD-V).
- The spectrum of forms trades isolation against weight: **VMs** (own kernel, heaviest) →
  **Kata containers** → **containers** (shared kernel, light) → **unikernels** (app + mini-OS,
  lightest).
- Storage and networks are virtualised too; what the cloud hands you is software pretending to
  be hardware.
- Centralisation ↔ decentralisation is a **cycle** driven by economics.

## Check yourself

1. Your colleague says "containers are just lightweight VMs." What is the important technical
   difference this glosses over, and when does it matter?
2. A cloud provider wants to run containers from *different customers* on the same physical
   host. Why might plain containers make them nervous, and what are two ways out?
3. Why did virtualization die out in the PC era and come back in the 2000s, when the underlying
   idea never changed?

:::{dropdown} Solutions
**1.** A VM runs its **own kernel** on emulated hardware; a container is a normal process
sharing the **host's kernel**, with an isolated view of it. It matters for security (a kernel
vulnerability can let a container escape, while a VM escape needs a hypervisor bug, which is
much rarer), and for capability (a container cannot run a different OS or kernel version than
the host; a VM can run anything).

**2.** Because all containers share one kernel, a single kernel exploit could let customer A
read customer B's data, which is an unacceptable risk between mutually distrusting tenants.
Ways out: give each customer their own VMs and let them run containers *inside* them (the usual
answer, and what you will do this week), or use sandboxed containers such as **Kata
containers**, which wrap each container in a minimal VM.

**3.** The economics changed twice. Mainframe hardware was so expensive that sharing one
machine safely was worth heavy engineering. Cheap x86 boxes made "one machine per task" easier
than virtualising (and x86 couldn't be cleanly virtualised anyway). By the late 1990s that
habit had produced rooms full of servers idling at a few percent load, each one bought,
racked and maintained separately: the **utilisation** problem of section 1.1. Consolidating
them onto fewer machines became valuable again, VMware solved x86 virtualization in software,
and CPU vendors then built it into silicon.
:::

---

Next: [Cloud computing](02-cloud-computing.md), or what happens when someone offers
virtualization as a service.
