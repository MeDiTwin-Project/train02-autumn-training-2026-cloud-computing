# 3. IaaS, PaaS, SaaS, plus public, private and hybrid

The cloud world sorts its offerings along two independent axes: **how much of the stack you
rent** (the service models: IaaS, PaaS, SaaS, …) and **whose cloud it is** (the deployment
models: public, private, hybrid). 

The main acronyms:
 - IaaS - Infrastructure as a Service
 - PaaS - Platform as a Service
 - SaaS - Software as a Service
 - FaaS - Function as a Service

## 3.1 Who manages what

The service models differ in one thing: which layers of the stack you manage yourself and
which ones you hand over to the provider. This table sums it up:

| Layer | On-premises | IaaS | PaaS | SaaS |
|---|---|---|---|---|
| Application | **you** | **you** | **you** | provider |
| Runtime / middleware | **you** | **you** | provider | provider |
| Operating system | **you** | **you** | provider | provider |
| Virtualisation | **you** | provider | provider | provider |
| Servers, storage, network | **you** | provider | provider | provider |
| Building, power, cooling | **you** | provider | provider | provider |

Read it column by column: as you move right, you hand over more responsibility and also more
control. Neither end is "better"; they are different trade-offs, and real projects use several
columns at once.

The same comparison as a picture, with a slightly finer split of the layers (white: managed
by you, the customer; black: managed by the provider):

:::{figure} img/on-premise-iaas-paas-saas.png
:alt: Four columns (On-Premise, IaaS, PaaS, SaaS), each a stack of layers from Applications down to Data Centres. Customer-managed layers are white and provider-managed layers are black; the black part grows from none (On-Premise) to everything (SaaS).
:width: 100%

Who manages which layer in each service model. Source: P. Rosati and T. Lynn, via
[Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Comparison_of_on-premise,_IaaS,_PaaS,_and_SaaS.png)
(public domain).
:::

A popular way to remember all this is **"Pizza as a Service"**, a metaphor introduced by
Albert Barron, then a software architect at IBM. You want pizza for dinner, and there are
four ways to get it:

- **Made at home** (on-premises): you buy the ingredients, make the dough, own the oven, pay
  for the gas and set the table. Everything is yours.
- **Take & bake** (IaaS): you buy a ready-made pizza, but bake it in your own oven, with your
  own energy, and serve it at your own table.
- **Pizza delivered** (PaaS): the pizzeria makes and bakes it; you only provide the table and
  the drinks.
- **Dined out** (SaaS): you go to a restaurant; everything is theirs, you just eat.

In every case you end up eating pizza. What changes is **who does the work**, and that is
exactly the difference between the service models.

:::{figure} img/pizza-as-a-service.jpeg
:alt: Four columns (Traditional On-Premises, IaaS, PaaS, SaaS), each a stack of layers from Dining Table, Soda, Electric/Gas, Oven and Fire down to Pizza Dough, Tomato Sauce, Toppings and Cheese. Blue layers are managed by you, green layers by the vendor. Below the columns: Made at home, Take & Bake, Pizza Delivered, Dined Out. The green part grows from none (On-Premises) to everything (SaaS).
:width: 90%

"Pizza as a Service": blue layers are managed by you, green layers by the vendor. Original
idea and graphic by Albert Barron,
[LinkedIn, 2014](https://www.linkedin.com/pulse/20140730172610-9679881-pizza-as-a-service).
:::

It is a fine mnemonic, but do not push the analogy too far: the individual ingredients do not
map one-to-one onto the technical layers of the table above.

:::{important} The boundaries are blurred in practice
The canonical distinction is IaaS / PaaS / SaaS, but real services often straddle the
boundaries. Is a managed database IaaS or PaaS? Is a managed Kubernetes cluster infrastructure
or platform? There is little point arguing about the labels. Use them to ask the question that
matters most: **exactly which layers am I responsible for here?** 
:::

## 3.2 IaaS: Infrastructure as a Service

**Infrastructure as a Service** is the bottom column: the provider gives you virtualised
computing resources (servers, storage, networks) through an API, and everything from the
operating system upwards is yours. 

- **You get:** virtual machines, virtual disks (volumes), virtual networks, public IP
  addresses, firewalls; in other words, raw building blocks.
- **You do:** install and patch the OS, configure services, manage backups, secure what you
  build.
- **Examples:** AWS EC2, Google Compute Engine, and **OpenStack**, the open-source
  implementation running the European Weather Cloud, which you will see in action in
  [section 4](04-openstack.md).

## 3.3 PaaS: Platform as a Service

**Platform as a Service** hands over the OS and runtime too: you bring *code* (or a
container), and the platform runs it, scales it and restarts it when it crashes. The
underlying VMs are hidden from you.

- **You get:** "run this application" as a primitive.
- **You do:** write the application; declare what it needs.
- **Open-source examples:** **Kubernetes**,
  **CloudFoundry**, **Dokku** (a tiny self-hosted Heroku), **Coolify**. GitLab's CI runners,
  which you will meet in the next days, are also platform-shaped: you bring a job, they bring the
  execution environment.
- **Commercial examples:** Heroku, Google App Engine, AWS Elastic Beanstalk.

The trade-off is considerable convenience in exchange for less control. If your app needs a
kernel module or an unusual network setup, PaaS will get in the way.

## 3.4 SaaS: Software as a Service

**Software as a Service** is a complete, finished application typically running in a browser. You
administer nothing but your account.

- **Examples you use daily:** Google Docs, Overleaf, GitHub/GitLab.com.
- **Open-source examples** (SaaS you or your institution can *host yourselves*):
  **OnlyOffice** and **Collabora Online** (documents), **Nextcloud** (files and
  collaboration), **GitLab CE** (the training GitLab is exactly this), **Jupyter Hub**
  (notebooks for a whole group).

## 3.5 FaaS: Function as a Service

**Function as a Service** shrinks the unit you deploy one step further, down to a single
**function**, run on demand when an event arrives and billed per invocation. Between calls
nothing runs, so nothing is billed.

- **Examples:** AWS **Lambda** (the name-giver), Google Cloud Functions.
- **Open-source examples:** **OpenFaaS**, **Knative** (both run on Kubernetes),
  **Nuclio** (aimed at data science workloads).

FaaS gets its own section, [section 8](08-faas.md), because it takes the cloud idea furthest:
the server disappears from your view altogether (hence its other name, *serverless*).

## 3.6 XaaS: anything as a Service

The pattern has caught on, and the industry now attaches "as a Service" to a great many
things:

- **DaaS** (Desktop as a Service): a full graphical desktop, streamed from a cloud VM.
- **KaaS** (Kubernetes as a Service): a ready-made Kubernetes cluster on demand.
- **DBaaS** (Database as a Service): a managed PostgreSQL/MySQL, backups and failover
  included.
- **MLaaS**, **STaaS** (storage), **BaaS** (backend)… and the list keeps growing.

The umbrella term is **XaaS**, anything as a Service. For each of them, the question from
section 3.1 still applies: which layers are mine?

```{mermaid}
flowchart LR
    IaaS["IaaS<br/>rent machines"] --> PaaS["PaaS<br/>rent a platform"]
    PaaS --> FaaS["FaaS<br/>rent per function call"]
    PaaS --> SaaS["SaaS<br/>rent the finished app"]
```

## 3.7 Public, private, hybrid

The second, independent axis is **whose cloud it is**.

:::{note} Jargon: *tenant*
A **tenant** is one customer/project sharing a cloud with others. Clouds are **multi-tenant**:
isolation between tenants (section 1) is what makes the sharing safe. On the EWC, a team's
*project* is its tenancy: the pool of resources the team shares.
:::

**Public cloud.** Owned by a provider, rentable by anyone with a credit card: AWS, Azure,
Google Cloud, plus European providers such as OVHcloud, Hetzner, Scaleway. It
offers practically unlimited capacity and global reach, and your data lives under the
provider's terms and jurisdiction.

**Private cloud.** The same self-service, API-driven machinery (very often OpenStack), but
owned and operated by one organisation for itself. You get the workflow benefits of the cloud
and full control over hardware, data and jurisdiction, but *you* run it, and capacity is
whatever you bought.

**Hybrid cloud.** Both, connected: a private base plus a public overflow. The classic pattern
is **cloud bursting**: steady load runs on owned hardware (cheap at high utilisation,
section 2.3), and demand spikes burst into a public cloud, then shrink back.

**Community cloud.** This is the fourth model, and the one this course keeps returning to: a
cloud built and governed by a *community of institutions* with shared needs. The **European
Weather Cloud** is a clear example. It is operated by ECMWF and EUMETSAT for the European
meteorological community, next to the data archives. From the user's perspective
it feels like a public cloud (you log in and provision), but governance, data locality and the
user community are those of a private one.

## Recap

- Two independent axes: **how much of the stack you rent** (IaaS → PaaS → SaaS, plus FaaS and
  the XaaS zoo) and **whose cloud it is** (public / private / hybrid / community).
- The responsibility table is the main tool: for any service, know **exactly which layers are
  yours** (e.g. to patch, secure and back up). The acronym boundaries blur, but your responsibilities
  should stay clear.
- IaaS = raw machines (OpenStack, EC2). PaaS = bring code (Kubernetes, Dokku). SaaS = finished
  app (Google Docs; self-hostable: OnlyOffice, Nextcloud, GitLab). FaaS = single functions,
  billed per call (Lambda; open source: OpenFaaS, Knative).
- The European Weather Cloud is a **community cloud** running **OpenStack**: IaaS operated
  by and for the European meteorological community.

## Check yourself

1. Your team deploys a web app on a rented VM: you installed Ubuntu, nginx and PostgreSQL on
   it. A critical OpenSSL vulnerability is announced. In the IaaS model, who patches what?
2. Classify these, and note where the label gets blurry: (a) the training GitLab, (b) a
   Kubernetes cluster your team installed on EWC VMs, (c) EWC itself, (d) OpenFaaS running on
   that Kubernetes cluster.
3. Why might a hospital consortium choose hybrid, rather than all-private or all-public, for a
   medical digital-twin workload?

:::{dropdown} Solutions
**1.** **You do.** IaaS providers are responsible for the hypervisor and below; the guest OS
and everything installed on it (including OpenSSL, nginx, PostgreSQL) is yours to patch. The
provider patches *their* control plane and hosts. 

**2.** (a) SaaS from your point of view: a finished application you just use, in this case
*open-source, self-hosted* SaaS. (b) PaaS built on IaaS: the VMs are IaaS from EWC, and the
Kubernetes layer is a platform your team now *operates*, so you are simultaneously a PaaS
provider (to your colleagues) and an IaaS consumer. (c) IaaS, on a community cloud. (d) FaaS
layered on your PaaS. This stacking of FaaS on PaaS on IaaS is the key observation: the models
nest.

**3.** Patient-derived data may legally have to stay on infrastructure under the consortium's
control and jurisdiction → private/community for the data and the steady workload. But
occasional heavy compute (training runs, reprocessing) would force them to buy hardware that
idles most of the year → burst those jobs (on properly de-identified or synthetic data, or
with the data staying put and only compute-heavy, non-sensitive stages moving) to a public
cloud. Hybrid gives the compliance of private with the elasticity of public.
:::

---

Next: [OpenStack](04-openstack.md), the open-source cloud platform.
