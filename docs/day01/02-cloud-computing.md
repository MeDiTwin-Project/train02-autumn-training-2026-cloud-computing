# 2. Cloud computing

Section 1 established that a computer can be simulated in software. This section is about
what that makes possible: *what if computing resources could be requested as a service,
whenever you need them, without anyone having to install a machine for you?*

## 2.1 What "cloud" actually means

Cloud computing is often explained as one particular business: renting servers from a large
company by the hour. That is one form of it, but it is not the definition. The international
standard is more general, and it is the definition used in this training:

:::{note} Jargon: *cloud computing*
**Cloud computing** is a *"paradigm for enabling network access to a scalable and elastic pool
of shareable physical or virtual resources with self-service provisioning and administration
on-demand"* (ISO/IEC 22123-1, the cloud computing vocabulary standard). The resources can be
servers, storage, networks, operating systems or applications.
:::

Notice what the definition does *not* say. It says nothing about who owns the hardware,
whether you pay for it, or whether the provider is a company. A commercial provider such as
AWS fits it; so does a cloud run by research institutions for their community, such as the
European Weather Cloud, and so does a cloud that an organisation runs on its own hardware for
its own staff. What makes something a cloud is *how* the resources are offered, not *who*
offers them or how they are paid for.

The ISO/IEC cloud standards list six key characteristics:

- **On-demand self-service**: you provision resources yourself, when you need them, with no
  human on the provider's side in the loop. There is no purchase order, ticket or waiting
  period; a VM is usually ready seconds or minutes after you ask for it. In practice
  self-service means a web dashboard and, more importantly, an **API**: a *program* can request
  infrastructure, so infrastructure can be automated, version-controlled and reproduced. 
- **Broad network access**: resources are reached over the network through standard
  protocols, from whatever client you use (a laptop, a script, another server).
- **Resource pooling**: the physical resources are gathered into a pool and shared out
  dynamically; you neither know nor need to know which physical machine your VM lands on.
  Virtualization (section 1) is what makes this possible.
- **Multi-tenancy**: many users or groups (*tenants*) share the same pool, isolated from each
  other (the *isolation* of section 1.4).
- **Rapid elasticity and scalability**: resources can be added *and released* quickly, often
  automatically, so capacity follows demand.
- **Measured service**: usage is metered, so it can be monitored, controlled and reported. On
  a commercial cloud the meter drives the bill (*pay per use*); on an institutional cloud it
  may drive quotas, fair-share allocations or reports to funding bodies. Either way, capacity
  stops being a fixed purchase guessed years in advance and becomes something consumed, and
  accounted for, as it is needed.

One practical consequence for you as a user: the hardware is operated by whoever runs the
cloud, be it a company, a research infrastructure or your own IT department. They deal with
power, cooling, failed disks and 3 a.m. alarms, while you concentrate on your own work.

The name itself comes from network diagrams: for decades, engineers drew anything outside
their own system (usually the internet) as a fluffy cloud, meaning *"details unknown and
irrelevant."* "

## 2.2 Where it came from

:::{admonition} 📖 Story: the myth, the memo and the birth of AWS
:class: seealso
You may have heard this story. Amazon is a retailer, and retail is strongly seasonal: every
November and December traffic rises sharply for the holiday rush, then falls away in January.
To cope with Black Friday and Christmas, Amazon supposedly bought enough servers for the
single busiest hour of the year, and for the other eleven months all that hardware sat idle,
costing money in power and rent. So, the tale goes, someone had an idea: *why not rent the
idle machines to other companies?* Amazon packaged up its leftovers, opened them to outside
developers, and arrived, almost by accident and as a thrifty sideline, at a business worth
hundreds of billions of dollars. Seasonal waste led to clever reuse, which led to an
accidental cloud, and Amazon Web Services was born.

It is a good story, but it does not hold up.

The real story starts with an engineering problem. By 2001, Amazon's retail engineers were
struggling with a tangled monolith in which teams kept getting in each other's way. In 2003
**Jeff Bezos** issued a now well-known mandate: teams would communicate only through clean,
hardened interfaces (APIs), with no backdoors. Since everyone was being made to build proper
interfaces anyway, someone realised that the underlying infrastructure could be turned into a
proper service too.

Late that year, two network engineers, **Benjamin Black** and his manager **Chris Pinkham**,
wrote a short internal paper. Amazon's infrastructure, they argued, should be fully
standardised and automated, and delivered as web services. Almost as an afterthought, they
noted that the company could even sell virtual servers to outside developers.

The response was larger than Amazon had expected. Developers signed up quickly, and within a
year the platform counted well over a hundred thousand. In 2008 a startup called **Animoto**
became the standard example of the new model: when a Facebook feature went viral, it scaled
from under 100 servers to around **3,400 in days**, which would have been impractical when
hardware had to be bought and racked.

So a bookseller ended up building the modern cloud.  Amazon set out to turn infrastructure
into something you could rent with a credit card.

*Further reading:
[blog.b3k.us/2009/01/25/ec2-origins.html](http://blog.b3k.us/2009/01/25/ec2-origins.html) ·
[aboutamazon.com: the deceptively simple origins of AWS](https://www.aboutamazon.com/news/aws/the-deceptively-simple-origins-of-aws)
[Video🍿: AWS Origin Story & How it Shaped the Cloud](https://youtu.be/QiWP-pEXnhk)*
:::

After 2006 the rest of the industry followed: Google App Engine (2008), Microsoft Azure
(2010) and, most relevant for this training, the **open-source clouds**, which asked: if
renting VMs through an API is so useful, why should only a few companies be able to
offer it? That question is the subject of [section 4](04-openstack.md).

## 2.3 Why cloud computing?

The general arguments come first, since they apply to everyone.

**Speed.** "I need a machine" goes from weeks (procurement) to minutes (API call). This
changes behaviour as well as budgets: experiments that were not worth three weeks of waiting
become worth a few seconds of waiting.

**Elasticity.** You can get *less* on demand as well as *more*: hand resources back when a
burst ends and the bill drops accordingly. A fleet of a thousand machines for one hour may cost
the same as one machine for a thousand hours, which is a new possibility.

**Pay as you go.** You stop paying for peak capacity during off-peak time. The Animoto story
above is the canonical example: capacity followed demand up *and back down*.

:::{note} Jargon: *elasticity* vs *scalability*
**Scalability** is the ability to grow when load grows. **Elasticity** is the ability to grow
*and shrink automatically*, tracking demand in both directions. A system can be scalable
without being elastic; the cloud's economic promise depends on elasticity.
:::

**Offloading operations.** Failed power supplies, disk replacements, firmware updates and
physical security become the provider's problem. This is real work you no longer do. Be clear,
though, about what is *not* offloaded: your OS updates, your application security and your
backups-are-actually-restorable checks remain yours.

**The trade-offs.** Public cloud is not automatically cheaper: a machine that computes at 100% load,
24/7, for years, is usually cheaper to own. Moving petabytes of data in or out
costs time and (on commercial clouds) money. There is also lock-in: the more provider-specific
services you use, the harder leaving becomes. Sensible organisations, including scientific
ones, run *hybrid* setups for these reasons, as we will see in
[section 3](03-service-models.md).

## 2.4 Why cloud computing for scientific workloads?

Scientific computing tends to suit the cloud rather well and owned hardware rather poorly,
because its demand is **bursty by nature**:

- A model run or training job needs 500 cores for two days, then nothing for three weeks.
- A paper deadline or a data release creates a spike; review season creates silence.
- Collaborations form and dissolve; a three-year project should not require buying hardware
  that lives for seven.

Three cloud properties that matter in this context:

**Flexibility.** Different workloads want different shapes of machine: many small VMs for a
data pipeline, one large GPU machine for training, a high memory node for memory-intensive operations.
On owned hardware you get whatever was bought years ago; on a cloud you pick the shape per job.

**Elasticity.** You can size the analysis to the dataset. Reprocessing a full archive with a
corrected algorithm becomes a large fleet of VMs for one weekend instead of a six-month queue.

**Resource optimisation and reproducibility.** Because infrastructure is created by API
calls, the whole environment of an experiment (machines, storage, network, software) can be
written down, version-controlled, and recreated by a colleague or a reviewer with one
command. "It ran on our cluster" becomes "here is the recipe that builds the machine it ran
on." 

There is one more reason, i.e **data locality and sovereignty**.
Some datasets are too large, too sensitive, or too legally constrained to
copy onto a commercial cloud in another jurisdiction. One answer is to build clouds *next to
the data*, run by the institutions that hold it. The **European Weather Cloud** is such a
cloud: it offers cloud-style flexibility, physically adjacent to ECMWF's and EUMETSAT's data
archives, on European infrastructure. You will see the kind of platform it runs on in action
in [section 4](04-openstack.md).

## 2.5 Cloud computing and HPC

Long before the cloud, scientists who needed a lot of computing power had another option: the
supercomputer.

:::{note} Jargon: *HPC*
**High-performance computing (HPC)** means running one large computation on many machines
(*nodes*) at once, as a single *job*. The parts of the job exchange data all the time (usually
with **MPI**, the Message Passing Interface), so the nodes are connected by a very fast,
low-latency network (such as **InfiniBand**) and share a **parallel filesystem** (such as
Lustre). Users do not get machines: they submit jobs to a **batch scheduler** (such as
**Slurm**), which puts them in a queue and runs them when enough nodes are free.
:::

Both the cloud and HPC share a large pool of hardware among many users, but they are built for
different goals:

| | Cloud | HPC |
|---|---|---|
| You ask for | resources (VMs, disks, networks), through an API | a job (*N* nodes for *T* hours), through a batch queue |
| You get them | in seconds or minutes, for as long as you want | when the scheduler finds free nodes, until the job's time limit |
| Built for | flexibility: any OS, root access, services that run for months | performance: many nodes working on one problem |
| Network and storage | general-purpose networks; block and object storage | low-latency interconnect; parallel filesystem |
| Software | whatever you install | modules provided by the centre, or containers (Apptainer) |
| Accounting | pay per use, or quotas | core-hours or GPU-hours, allocated to a project |

As a rule of thumb, **tightly coupled simulations** (a weather or climate model, fluid dynamics,
the physics of a digital twin) belong on HPC. On ordinary cloud networks their nodes would
spend most of their time waiting for each other. **Services** that must stay up (web
applications, databases, portals), **interactive work** (notebooks, dashboards) and **many
independent tasks** (processing 10,000 files one by one) usually fit the cloud better.

The two worlds increasingly overlap. HPC centres run cloud services next to their
supercomputers, and some public clouds rent HPC-style nodes with fast interconnects.
Many real workflows use both: a simulation runs on HPC, and
its results are shared, analysed and served from a cloud.

:::{admonition} 🌍 Example: ECMWF 
:class: seealso
ECMWF produces its weather forecasts on a supercomputer in Bologna: about one million CPU cores
connected by InfiniBand, with a Lustre filesystem and Slurm as the scheduler. Some consumers
of the forecasts work on the **European Weather Cloud**, next to the data
archives. The HPC system computes the forecasts, and the cloud gives users flexible access to
the results.

**Destination Earth** (DestinE), the EU's digital twin of the Earth.
Its digital twins, developed by ECMWF, run on EuroHPC
supercomputers such as LUMI in Finland. Users reach the results through a cloud-based platform
run by ESA and a data lake run by EUMETSAT.

*Further reading:
[ECMWF supercomputer facility](https://www.ecmwf.int/en/computing/our-facilities/supercomputer-facility) ·
[DestinE components](https://destination-earth.eu/destination-earth/destines-components/)*
:::

## 2.6 Cloud computing and AI

Artificial intelligence, and machine learning in particular, is now one of the main reasons
people use clouds, and the main reason clouds buy GPUs.

:::{note} Jargon: *GPU*
A **GPU** (graphics processing unit) is a processor with thousands of small cores that apply
the same arithmetic to many numbers at once, which is exactly what neural networks need. A
cloud VM can get a whole GPU (*passthrough*) or a slice of one (*vGPU*, *MIG*). You will see a
GPU slice in action on [Day 3](../day03/11-gpu-workloads-and-scheduling.md).
:::

A machine learning model goes through two phases, with very different needs:

- **Training** runs over a large dataset many times on GPUs. Large models are trained on
  hundreds or thousands of GPUs that exchange data constantly. That is an HPC workload, and
  the largest AI training systems are supercomputers. Smaller training runs and fine-tuning fit
  on one or a few cloud GPUs, and their demand is bursty (section 2.4): you need GPUs for two
  days, then for nothing for weeks.
- **Inference** (using the trained model to answer requests) is a *service*: it runs
  continuously, must answer quickly and must scale with the number of users. That is a typical
  cloud workload.

The cloud provides what an AI project needs at each step: **GPUs on demand**, rented for a
training run instead of bought and left idle; **object storage** for datasets and trained
models (sections 5 and 7); and **elastic serving** for inference. 

:::{admonition} 🇪🇺 EuroHPC AI Factories
:class: seealso
The EU applies the "build it next to the users" approach to AI as well. **AI Factories** are
AI-optimised supercomputers run by EuroHPC, together with support services for research,
industry and the public sector. About twenty have been selected across Europe. The Greek AI
Factory, **Pharos**, is run by GRNET with partners that include NTUA.
It uses the DAEDALUS supercomputer and focuses on health, language and culture, and
sustainability.

*Further reading:
[EuroHPC AI Factories](https://www.eurohpc-ju.europa.eu/ai-factories_en) ·
[Pharos](https://pharos-aifactory.eu)*
:::

## Recap

- Cloud computing (ISO/IEC 22123-1) = **network access to a shared, elastic pool of
  resources, provisioned self-service and on demand**. Its key characteristics: on-demand
  self-service, broad network access, resource pooling, multi-tenancy, rapid elasticity and
  measured service. Who owns the hardware and whether you pay are not part of the definition:
  commercial, institutional and in-house clouds all qualify.
- The economic core is **elasticity**: capacity tracks demand in both directions, and 1,000
  machines × 1 hour might cost the same as 1 machine × 1,000 hours.
- Cloud is not always cheaper: steady 24/7 loads, data gravity and lock-in argue the other
  way. This is why hybrid setups are common.
- Science is a natural fit (bursty demand, varied machine shapes, reproducibility through
  infrastructure-as-code), and in Europe there are also community clouds built next to the
  data.
- **HPC** runs one large, tightly coupled job on many nodes, through a batch queue; the
  **cloud** provides flexible resources on demand, through an API. Coupled simulations belong
  on HPC; services, interactive work and many independent tasks belong on the cloud. Real
  workflows, such as ECMWF and Destination Earth, combine both.
- In **AI**, training large models is mostly an HPC workload, while inference is a cloud service.
  The cloud provides GPUs on demand, object storage and elastic serving.

## Check yourself

1. Your group runs a simulation cluster at a steady 95% utilisation, year-round. Is the cloud
   likely to save money? What kind of workload *would* it save money on?
2. What is the difference between scalability and elasticity, and which one does the
   pay-per-use model actually depend on?
3. Name two reasons a European medical or meteorological project might prefer a community
   cloud over a large commercial cloud, even at equal price.

:::{dropdown} Solutions
**1.** Probably not. Steady, high-utilisation loads are the case where owning wins, since
you pay the cloud's margin on top of hardware that would rarely idle. The cloud wins on
**bursty** loads: occasional reprocessing campaigns, deadline spikes, short-lived experiments,
or anything needing hardware shapes (GPUs, fat memory) you only need occasionally. Many groups
therefore run the steady base load on owned hardware and *burst* into a cloud (hybrid model).

**2.** Scalability = can grow with load. Elasticity = grows **and shrinks**, automatically,
tracking demand. Pay-per-use depends on elasticity: if you scale up and never release
resources, you have simply rented a fixed cluster at on-demand prices, which is the most
expensive way to own a cluster.

**3.** Any two of: **data locality** (the cloud sits next to the multi-petabyte archives, so
you move the compute to the data instead of copying the data out); **sovereignty and legal
constraints** (data stays under European jurisdiction and institutional governance, which may
be a hard requirement for medical data); **no egress-pricing trap / lock-in** (community
clouds are not designed to make leaving expensive); **community fit** (flavors, images and
support are tailored to scientific workloads rather than web startups).
:::

---

Next: [IaaS, PaaS, SaaS](03-service-models.md).
