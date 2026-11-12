# 6. Cloud networking

Compute is virtualised (section 1) and so is storage (section 5); networks are the remaining
piece. Cloud networks are built from the same ideas as any other network, so this section
covers those first: **IP addresses**, **subnets** and **routing**. Then it gives you the three
tools you will use most to find out why something does not connect: **`ping`**,
**`traceroute`** and **`curl`**. It ends with three things cloud users meet very early on:
**private and public addresses**, **NAT** (which is what a **floating IP** is) and
**firewalls** (which is what a **security group** is).

## 6.1 What a network does

A network moves **packets**: small chunks of data, each carrying an envelope with two
addresses on it, the **source** (who sent it) and the **destination** (where it must go).
Anything bigger than a packet, such as a web page, a file or an SSH session, is cut into many
packets and reassembled at the other end.

Machines connect to a network through **interfaces**, and every interface has an **address**.
When the destination is close by, the packet is delivered directly. When it is not, it is
handed to a **router**, which passes it on towards the destination. The rest of this section
explains how each of those steps works.

:::{note} Jargon: *interface*
A network **interface** is one attachment point to a network: physical (a real network card)
or virtual (the network card of a VM, which is software). One machine can have several, each
with its own address.
:::

:::{note} Jargon: *port*
An IP address picks the **machine**; a **port** picks the **program** on it. A port is a
number from 1 to 65535. `192.168.1.14:22` reads "port 22 on that machine", where the SSH
server listens. Firewall rules (6.7) are mostly about ports.
:::

:::{note} Jargon: *TCP* and *UDP*
Programs exchange packets using one of two **transport protocols**. **TCP** first sets up a
*connection*, then guarantees that everything arrives, complete and in order: SSH sessions,
web pages, file transfers. **UDP** sends independent packets, with no connection and no
guarantee: lighter and faster, for short question-and-answer exchanges such as DNS, or for
streaming. TCP and UDP each have their own ports 1 to 65535, so a port is always quoted
together with its protocol: "TCP 22", "UDP 53".
:::

Most services listen on a **well-known port**, fixed by convention, so that clients know where
to connect without being told. The ones you will meet this week:

| Service | Protocol and port | What it is for |
|---|---|---|
| SSH | TCP 22 | Logging into your VM (section 0) |
| DNS | UDP 53 (TCP 53 for large answers) | Turning names into addresses (6.4) |
| HTTP | TCP 80 | The web, unencrypted |
| HTTPS | TCP 443 | The web, encrypted; also almost every cloud API (section 7) |
| NTP | UDP 123 | Keeping machine clocks in sync |
| PostgreSQL | TCP 5432 | A popular database (6.7) |
| Kubernetes API | TCP 6443 | Controlling a Kubernetes cluster (Day 3) |
| *(none)* | TCP 8000, 8080 | No standard service: common choices for test and development web servers, like the one in 6.4 |

`ping` (6.4) is the exception: it uses **ICMP**, a third protocol meant for network control
messages, which has no ports at all. Any Linux machine has the full list of well-known ports
in `/etc/services`, e.g. `grep -w 443 /etc/services`.

## 6.2 IP addresses and subnets

### IP addresses

An **IP address** identifies one network interface, the way a postal address identifies one
letterbox. An (IPv4) address is a 32-bit number, written as four numbers from 0 to 255
separated by dots: `192.168.1.14`.

See your training VM's addresses:

```bash
ip -4 -brief address show
```

```text
lo               UNKNOWN        127.0.0.1/8
enp3s0           UP             192.168.1.84/24
docker0          UP             172.17.0.1/16
```

Three interfaces: `lo`, the **loopback**, through which the machine talks to itself
(`127.0.0.1`, also called `localhost`, always means "this machine"); `enp3s0`, the VM's
(virtual) network card; `docker0`, the Docker bridge (we can ignore it for now, we will talk
about Docker in forthcoming lessons).  Your interface name and address may differ. The `/24`
after the address is the netmask, explained next. 

### Netmasks and subnets

An address has two parts: a **network part**, shared by all the machines on the same network,
and a **host part**, different for each of them. The **netmask** says where the split is. It
is usually written as a **prefix length** after a slash, counting the bits of the network
part:

```text
192.168.1.14/24

  192 . 168 .   1   .   14
 └──────────┬─────────┘  └┬┘
   network part           host part
   (first 24 bits =       (last 8 bits =
    first three numbers)   last number)
```

All the addresses sharing the network part form a **subnet**, written with a zero host part:
`192.168.1.0/24` is the 256 addresses from `192.168.1.0` to `192.168.1.255`. Two of them
are reserved and cannot be given to a machine: the first (`.0`, the subnet itself) and the
last (`.255`, the **broadcast** address, meaning "every machine on this subnet"). You will also
meet the old notation for the same netmask, with the network bits set to 1: `/24` =
`255.255.255.0`.

**The prefix length tells you the size.** The larger the number after the slash, the more
bits are fixed, and the *smaller* the subnet:

| Prefix | Netmask | Addresses | Where you will see it |
|---|---|---|---|
| `/32` | `255.255.255.255` | 1 | `136.156.7.42/32`: exactly one machine, e.g. "only my laptop" in a firewall rule |
| `/24` | `255.255.255.0` | 256 | `192.168.1.0/24`: a typical cloud subnet |
| `/16` | `255.255.0.0` | 65,536 | `136.156.0.0/16`: a whole institute |
| `/8` | `255.0.0.0` | ~16.7 million | `10.0.0.0/8`: the whole private 10.x range |
| `/0` | `0.0.0.0` | all of them | `0.0.0.0/0`: "the whole internet" in a firewall rule, "default" in a routing table |

:::{note} Jargon: *CIDR*
**CIDR** (*Classless Inter-Domain Routing*) is the official name of this `address/prefix`
notation: `192.168.1.0/24`, `136.156.7.42/32`, `0.0.0.0/0`. Cloud dashboards and APIs say
"CIDR" wherever they ask for an address range.
:::

A netmask answers the question *is this other address on my subnet?* To find out, compare the
network parts:

| From `192.168.1.14/24` to… | Network part | Same subnet? |
|---|---|---|
| `192.168.1.200` | `192.168.1` | **yes**: deliver directly |
| `192.168.2.14` | `192.168.2` | **no**: needs a router |
| `8.8.8.8` | `8.8.8` | **no**: needs a router |

Prefixes that do not fall on a dot (`/20`, `/26`…) work the same way but are hard to compute
by eye. 

## 6.3 Routing

Machines on the **same subnet** talk directly. For **any other** destination, the packet is
handed to a **router**: a machine with interfaces in several subnets that **forwards**
packets from one to another.

Every machine (your laptop, your VM, every router) decides where to send a packet by looking
it up in its **routing table**: a list of *destination subnet → how to get there*. Look at
your training VM's:

```bash
ip route show
```

```text
default via 192.168.1.254 dev enp3s0 proto dhcp src 192.168.1.84 metric 100
169.254.169.254 via 192.168.1.1 dev enp3s0 proto dhcp src 192.168.1.84 metric 100
192.168.1.0/24 dev enp3s0 proto kernel scope link src 192.168.1.84 metric 100
```

Read it bottom-up:

- `192.168.1.0/24 dev enp3s0`: "this subnet is **directly connected** on `enp3s0`, deliver
  directly." Linux adds this line by itself from the `/24` in `192.168.1.84/24`.
- `169.254.169.254 via 192.168.1.1`: how to reach OpenStack's metadata service (to make
  `curl http://169.254.169.254/openstack/latest/meta_data.json` work).
- `default via 192.168.1.254`: "for **everything else**, hand the packet to the router at
  `192.168.1.254`." `default` is shorthand for `0.0.0.0/0`, the subnet that contains every
  address. The router this line points to is your **default gateway**.

When several lines match, the **most specific** one (the longest prefix) wins. The default
route matches everything but is the least specific of all, so it is only used when nothing
else matches. You can ask which line would be used for any destination:

```bash
ip route get 192.168.1.1        # a neighbour: delivered directly
ip route get 8.8.8.8            # the internet: via the gateway
```

```text
192.168.1.1 dev enp3s0 src 192.168.1.84 uid 1000
    cache

8.8.8.8 via 192.168.1.254 dev enp3s0 src 192.168.1.84 uid 1000
    cache
```

The second answer contains `via 192.168.1.254` and the first does not. That `via` is what
distinguishes a neighbour from a destination somewhere else.

**A packet crosses the internet hop by hop.** No router knows the full path. Each one only
knows the **next hop** towards the destination, like a chain of postal sorting offices:

Each router a packet crosses decrements a counter in it, the **TTL** (*time to live*). A
packet whose TTL reaches 0 is discarded, so a packet caught in a routing loop cannot circle
forever. `traceroute`, in the next section, uses this counter to discover the path.

## 6.4 Debugging tools: `ping`, `traceroute`, `curl`

"It doesn't connect" is one of the most common cloud problems. Three tools answer most of
the questions you need to ask, each one at a different level:

| Tool | The question it answers |
|---|---|
| `ping` | Can a packet reach that **machine** and come back? |
| `traceroute` | Which **routers** does the packet cross on the way, and where does it stop? |
| `curl` | Does the **service** on that port actually answer? |

All the commands below run on your **training VM**.

### `ping`: is the machine there?

`ping` sends small "are you there?" packets (ICMP *echo requests*) and waits for the
answers. Start with your default gateway (the address after `default via` in `ip route`),
then something on the internet:

```bash
ping -c3 192.168.1.254      # your default gateway: use the address from your ip route
ping -c3 8.8.8.8            # a public DNS server, on the internet
```

```text
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=112 time=12.8 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=112 time=11.6 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=112 time=10.3 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 10.276/11.562/12.780/1.023 ms
```

- `time=` is the **round trip**: there and back. A few ms is the same city or country; 100+
  ms is another continent.
- `ttl=` is what was left of the reply's TTL when it arrived: it went down by one at every
  router on the way back.
- The last two lines summarise: here, every packet came back.

`-c3` stops after three packets; without it, `ping` runs until you press Ctrl-C.

:::{warning} No answer to `ping` does not mean "down"
Many machines and firewalls deliberately drop ping. A server can ignore `ping` completely and
still serve web pages normally. `ping` working proves the machine is reachable; `ping`
failing proves little. Test the actual service with `curl` before concluding anything.
:::

:::{note} Jargon: *DNS*
Packets carry addresses, not names. **DNS** (*Domain Name System*) is the internet's
phone book: it turns a name such as `example.com` into an address. Every tool here accepts
names and looks them up first. The `getent` command, e.g. `getent hosts meditwin-project.eu`
can be used to debug DNS look up.
:::

### `traceroute`: which way does it go?

`traceroute` reveals the path by making use of the TTL: it sends packets with TTL 1, then 2, then
3… Each one dies one router further along, and the router where it dies reports back. 

```bash
traceroute 1.1.1.1
```

```text
traceroute to 1.1.1.1 (1.1.1.1), 30 hops max, 60 byte packets
 1  _gateway (192.168.1.254)  1.194 ms  1.318 ms  1.394 ms
 2  136.156.136.1 (136.156.136.1)  8.961 ms  8.937 ms  8.914 ms
 3  10.107.10.12 (10.107.10.12)  5.816 ms  5.778 ms  5.749 ms
 4  136.156.159.84 (136.156.159.84)  3.717 ms 136.156.159.80 (136.156.159.80)  8.929 ms
136.156.159.84 (136.156.159.84)  3.671 ms
 5  10.127.2.163 (10.127.2.163)  5.186 ms  5.164 ms  5.131 ms
 6  136.156.237.39 (136.156.237.39)  3.174 ms  1.589 ms  2.149 ms
 7  136.156.237.0 (136.156.237.0)  8.449 ms  8.404 ms  8.378 ms
 8  136.156.239.1 (136.156.239.1)  2.547 ms  10.601 ms  1.820 ms
 9  ru-ecmwf-rl1-bo01.bo01.garr.net (193.206.128.245)  2.412 ms  2.032 ms  2.176 ms
10  rl1-bo06-rs1-bo01.bo01.garr.net (212.189.220.25)  2.265 ms rl1-bo06-rs1-mi01.mi01.garr.net
(212.189.220.23)  6.793 ms rl1-bo06-rs1-bo01.bo01.garr.net (212.189.220.25)  1.881 ms
11  rs1-mi01-re1-mi02.mi02.garr.net (185.191.180.158)  8.774 ms  8.557 ms  8.790 ms
12  * * *
13  * * 172.68.196.21 (172.68.196.21)  10.493 ms
14  one.one.one.one (1.1.1.1)  10.520 ms  10.376 ms *
```

One line per router, three timings per line (three probes). Hop 1 is always your default
gateway, here the cloud's router. Your path will differ.

- `* * *` means that router did not report back. Many routers are configured not to; if
  later hops still answer, ignore it.
- If the stars start at some hop and **never stop**, the packets are being lost there: you
  have found where to look.

### `curl`: does the service answer?

`curl` talks to a program listening on a port (usually a web server), so it tests the service
itself. Fetch only the response headers from a public website:

```bash
curl -s --head https://meditwin-project.eu| head -n 1
```

```text
HTTP/1.1 200 OK
```

`200` is HTTP for "OK". To see the network steps as well, add `-v` (verbose). The lines
starting with `*` show the name lookup and the connection:

```bash
curl -sv https://meditwin-project.eu 
```

```text
*   Trying 38.19.201.179:443...
* Connected to meditwin-project.eu (38.19.201.179) port 443 (#0)
```

`Trying` is the address DNS returned; `Connected` means the server accepted the connection on
port 443.

### Reading the error

When `curl` (or `ssh`, or any other client) cannot connect, *how* it fails tells you where to
look:

| Error | What happened | Where to look |
|---|---|---|
| *Could not resolve host* | The **name lookup** failed: no packet was even sent to the server | The name (typo?) and DNS: `getent hosts <name>` |
| *Connection refused*, immediately | The packet **reached the machine**, but no program listens on that port, so the machine answered "no" | On the machine: is the service running, and on that port? |
| *Network is unreachable*, immediately | There is **no route** to the destination: the packet never left | The routing table (`ip route`, 6.3) |
| Hangs, then *timed out* | The packet was **dropped silently** somewhere on the way, very often by a firewall (6.7) | Firewalls and security groups |

To see which ports have a program listening on them, run on the machine itself:

```bash
ss -tln          # TCP sockets in LISTEN state, with numeric ports
```

```text
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
LISTEN  0       128           0.0.0.0:22         0.0.0.0:*
```

Here only SSH (port 22) is listening.

### Your turn: debug a connection

You will now cause these errors on purpose, using a small test web server on your
**training VM**, and diagnose each one with the tools above. First create a page for the
server to serve:

```bash
mkdir -p ~/www
echo "hello from my training VM" > ~/www/index.html
```

**1. Nothing listening.** Before starting any server, request the page:

```bash
curl http://localhost:8000/
```

```text
curl: (7) Failed to connect to localhost port 8000: Connection refused
```

*Connection refused*, immediately: the packet reached the machine (here, the VM itself), but
no program listens on TCP 8000. `ss -tln` confirms it: there is no line with `:8000`.

**2. Listening, but only on the loopback.** Start Python's built-in web server in the
background, listening on `127.0.0.1` only:

```bash
python3 -m http.server 8000 --bind 127.0.0.1 --directory ~/www > ~/www.log 2>&1 &
curl http://localhost:8000/
```

```text
hello from my training VM
```

It works. Now request the same page through the VM's network address (the `enp3s0` address
from `ip -4 -brief address show`; use yours):

```bash
curl http://192.168.1.84:8000/
```

```text
curl: (7) Failed to connect to 192.168.1.84 port 8000: Connection refused
```

Same machine, same port, and yet refused. `ss -tln` shows why:

```text
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
LISTEN  0       5           127.0.0.1:8000       0.0.0.0:*
LISTEN  0       128           0.0.0.0:22         0.0.0.0:*
```

The web server listens on `127.0.0.1`, so it only accepts connections that come through the
loopback, from the machine itself. SSH listens on `0.0.0.0`, which here means **every**
interface. 

**3. Listening everywhere.** Stop the server and start it again on all interfaces:

```bash
kill %1
python3 -m http.server 8000 --bind 0.0.0.0 --directory ~/www > ~/www.log 2>&1 &
curl http://192.168.1.84:8000/
```

```text
hello from my training VM
```

`ss -tln` now shows `0.0.0.0:8000`.

**4. From your laptop.** On your **laptop**, request the page through the floating IP you
use for `ssh` (on Windows, type `curl.exe`). Before you press Enter, predict what will
happen:

```bash
curl -m 10 http://<your-vm-floating-ip>:8000/
```

`-m 10` gives up after 10 seconds. Use the table above to read the result:

- **It hangs, then times out**: the packets are dropped somewhere on the way. `ssh` to the
  same address works, so the floating IP, the routers and the VM are all fine: something lets
  TCP 22 through but not TCP 8000. That is a firewall: most likely the VM's **security group**
  (6.7), or possibly the network your laptop is on.
- **The page arrives**: every firewall on the way allows TCP 8000 from where you are.
- ***No route to host***, **immediately**: a firewall *rejected* the packet instead of
  dropping it silently. Host firewalls such as `firewalld` do this by default.

**5. A name that does not exist.** Back on the VM:

```bash
curl http://no-such-host.invalid/
```

```text
curl: (6) Could not resolve host: no-such-host.invalid
```

This error comes from the DNS lookup, before any packet is sent to a server. Compare
`getent hosts no-such-host.invalid` (prints nothing) with `getent hosts meditwin-project.eu`
(prints an address). Names ending in `.invalid` are reserved and never exist.

Tidy up:

```bash
kill %1
rm -r ~/www ~/www.log
```

You moved outwards one step at a time: is a program listening (`ss -tln`)? Does it answer
on the machine itself (`curl localhost`)? On the machine's own network address? From
outside? The first step that fails shows you where to look.

## 6.5 Private and public addresses

Some address ranges are **private**: reserved for internal networks, reused by every
organisation and every home, and **never routed on the public internet**. No router on the
internet has a route for them. Everything else (apart from a few special ranges) is
**public**: globally unique, and reachable from anywhere on the internet.

You recognise a private address by how it starts:

| Private range | How to recognise it | Typical use |
|---|---|---|
| `10.0.0.0/8` | starts with `10.` | Large organisations, cloud internal networks |
| `172.16.0.0/12` | starts with `172.16.` to `172.31.` | Docker's default networks (Day 2) |
| `192.168.0.0/16` | starts with `192.168.` | Home routers, small cloud networks |

Note the middle row: `172.20.1.5` is private, but `172.32.1.5` is public. Two more special
ranges you have already met:

| Special range | Meaning |
|---|---|
| `127.0.0.0/8` | Loopback: "this machine". Never leaves it. |
| `169.254.0.0/16` | Link-local: only valid on the local wire. The cloud metadata service you queried in section 4.7 lives at `169.254.169.254`. |

:::{tip} Check an address with `ipcalc`
:class: dropdown
The `ipcalc` command tells you which kind of range an address belongs to:

```text
$ ipcalc 10.4.7.20
Address:        10.4.7.20
Address space:  Private Use

$ ipcalc 172.20.1.5
Address:        172.20.1.5
Address space:  Private Use

$ ipcalc 172.32.1.5
Address:        172.32.1.5
Address space:  Internet
```

It also does the subnet arithmetic of 6.2 for you, which helps for prefixes that do not fall
on a dot: try `ipcalc 192.168.1.14/26`.
:::

Now look at your training VM's address from `ip -4 -brief address show`. It is private, yet you
`ssh` into it from your laptop, across the internet. The next section explains how.

## 6.6 NAT and floating IPs

Private addresses cannot appear on the internet. So the router that joins a private network
to the internet performs **Network Address Translation (NAT)**: it **rewrites the addresses**
in the packets passing through it, and remembers each rewrite, so that the replies can be
rewritten back and find their way home.

Clouds use NAT in two ways.

**Outbound NAT: reaching out.** Many private machines share the router's single public
address. When a VM sends a packet to the internet, the router replaces the private source
address with its own public one. The reply comes back to the router, which knows which VM
asked and passes it on. Your home router does the same for the devices in your house. In a
cloud, the project's router does it, which is why your VM can `dnf update` without any extra
configuration.

Outbound NAT only works one way: the internet can **reply** to the VM, but cannot **start** a
connection to it, because nothing outside knows the VM's private address, and the router
would not know which VM a new incoming packet is for.

**Floating IPs: being reached.** To be reachable from outside, a VM needs a public address of
its own: a **floating IP**. The cloud owns a pool of public addresses; you *allocate* one to
your project and *associate* it with a VM. The router then translates, in both directions,
between that public address and the VM's private one. This is one-to-one NAT:

```{mermaid}
sequenceDiagram
    participant L as Your laptop<br/>203.0.113.5
    participant R as Cloud router<br/>owns 136.156.x.y
    participant V as Your VM<br/>192.168.1.14
    L->>R: 203.0.113.5 → 136.156.x.y:22
    Note over R: rewrite the destination<br/>136.156.x.y becomes 192.168.1.14
    R->>V: 203.0.113.5 → 192.168.1.14:22
    V->>R: reply 192.168.1.14:22 → 203.0.113.5
    Note over R: rewrite the source back<br/>192.168.1.14 becomes 136.156.x.y
    R->>L: reply 136.156.x.y:22 → 203.0.113.5
```

Your laptop only ever sees the public address; the VM only ever sees its own private one.
The VM does not know its floating IP at all: `ip -4 address show` inside it shows only `192.168.1.x`.

The address is called "floating" because it is not tied to a particular VM. If you
disassociate it and associate it with another VM, the public identity of your service moves
within seconds. This is a simple way to swap a broken server for its replacement without
anyone updating DNS. On paid clouds, allocated-but-unused floating IPs usually cost money,
since public IPv4 addresses are scarce.

### Your turn: find your own floating IP

On your training VM, compare the address the VM *knows*
with the address the internet *sees*:

```bash
ip -4 -brief address show dev enp3s0  # the address configured inside the VM
curl -s ifconfig.me                   # the address the outside world sees you coming from
```

```text
enp3s0           UP             192.168.1.84/24
136.156.x.y
```

`ifconfig.me` is a website that answers with the source address of your request, as it
arrived. If the two addresses differ, there is NAT between your VM and the internet, and if
the second one is the address you `ssh` to, you are looking at your VM's floating IP, which
the VM itself cannot see. 

Try the same on your **laptop**: compare its own address (`ip -4 -brief address show` on Linux,
`ipconfig getifaddr en0` on macOS, `ipconfig` on Windows) with `curl -s ifconfig.me`. At home
or in an office they usually differ, because your laptop is behind NAT too.

## 6.7 Firewalls and security groups

A VM with a public address is reachable from the internet, and connection attempts usually
start within minutes, because automated scanners probe the public IPv4 address space
continuously. Count the failed SSH login attempts on your training VM:

```bash
sudo journalctl -u sshd --since "4 hours ago" | grep "Invalid user"
```

To protect against threats from the network,a **firewall** decides which packets get through.
It holds a list of **rules**. Each rule describes some traffic by its **direction** (incoming
or outgoing), its **protocol** and **port** (e.g. TCP 22) and its **remote address range** in
CIDR notation (6.2), and says whether to let it through. A well-configured firewall is an
**allow-list**: anything not explicitly allowed is dropped.

Firewalls sit in several places, and a packet must get past all of them:

- **on the network**: your institute's firewall, at the edge of its network;
- **in the cloud, outside the VM**: the **security group**;
- **on the machine itself**: a host firewall such as `firewalld` on Rocky Linux (or `ufw` on Ubuntu).

:::{note} Jargon: *security group*
A **security group** is the cloud's firewall, attached to VMs and enforced *outside* the VM
by the cloud: the VM's own operating system cannot see or disable it. It is a list of
**allow rules**; anything not allowed is dropped. 
:::

Each security group rule has a direction (**ingress** = incoming, **egress** = outgoing), a
protocol and port, and a **source**: either an address range, or *another security group*,
meaning "allow from any VM that carries that group". The prefix length matters here:
`/32` is one machine, `0.0.0.0/0` is the whole internet.

A sensible pattern for a small web service:

| Group | Rule | Why |
|---|---|---|
| `ssh-access` | ingress TCP 22 from `<your institute's range>`, e.g. `136.156.0.0/16` | Admin access, not from the whole world |
| `web` | ingress TCP 443 from `0.0.0.0/0` | The actual service, public |
| `db` | ingress TCP 5432 from group `web` | Only web servers reach the database; no IP range is involved |

**Security groups are stateful.** You write rules about who may *start* a connection; the
replies to an allowed connection are let through automatically. With only the three ingress
rules above, the VM can still `dnf update`: the request goes out (egress is allowed by
default), and the downloads coming back in are replies to a connection the VM started, so no
ingress rule is needed for them:

```{mermaid}
flowchart TD
    P(["packet arriving at the VM"]) --> E{"reply to a connection<br/>already allowed?"}
    E -->|yes| A1["let through"]
    E -->|no| I{"matches an<br/>ingress rule?"}
    I -->|yes| A2["let through"]
    I -->|no| D["dropped<br/>(silently)"]
```

In the last box, a blocked packet is **dropped silently**, with no error sent back. 
A firewall that drops silently looks, from outside, just like a machine that is not there. 

### Demo: security groups in OpenStack

:::{admonition} 👀 Instructor demo: a real security group
:class: hint
In the dashboard: **Network → Security Groups → Create Security Group** named `demo-ssh`,
**Manage Rules → Add Rule**: rule *SSH*, remote *CIDR*, CIDR `<the instructor's public IP>/32`.
Then **Instances → row menu → Edit Security Groups** attaches it to the demo VM. SSH still
works, and stops working (it hangs silently) when the rule's CIDR is changed to something
wrong. The CLI equivalent:

```bash
openstack security group create demo-ssh
openstack security group rule create demo-ssh \
    --protocol tcp --dst-port 22 --remote-ip $(curl -s ifconfig.me)/32
openstack server add security group demo-vm1 demo-ssh
```
:::

Two habits to leave today with:

1. **Never solve a connectivity problem by allowing everything.** `0.0.0.0/0` on port 22 "to
   debug it later" is a common way for research VMs to end up in botnets. Open the narrowest
   thing that works.
2. **Security groups don't replace patching.** They limit *who can connect*; a vulnerable
   service on an allowed port is still vulnerable (see the responsibility table in section
   3.1: the OS and services are yours).

## 6.8 The whole picture in OpenStack

Each concept in this section has a name in OpenStack's networking service, **Neutron**:

| Concept from this section | Neutron calls it | Notes |
|---|---|---|
| A subnet and the "wire" it lives on | **network** + **subnet** | The subnet is the address range (e.g. `192.168.1.0/24`), plus its gateway and DHCP settings |
| The default gateway | **router** | Joins the project's subnets to each other and to the outside, and does the outbound NAT |
| One-to-one NAT to a public address | **floating IP** | Allocated from the cloud's pool, associated with one VM, movable |
| The firewall outside the VM | **security group** | Stateful allow-list, attached to many VMs |

None of these is hardware: Neutron builds them all in software, on the cloud's own servers,
when someone clicks a button or calls the API. Put together, the path of an SSH connection
from your laptop to your training VM looks like this:

```{mermaid}
flowchart LR
    Laptop(["Your laptop<br/>(behind its own NAT)"]) -->|"to 136.156.x.y:22"| Inet["Internet<br/>routers"]
    Inet --> RT["Cloud router<br/>floating IP NAT:<br/>136.156.x.y ⇄ 192.168.1.14"]
    RT --> SG{{"Security group<br/>ingress TCP 22 allowed?"}}
    SG -->|yes| VM["Your VM<br/>192.168.1.14"]
```

## Recap

- Networks move **packets**, each carrying a source and a destination address. An **IP
  address** identifies an interface; a **port** identifies a program on it.
- Ports are quoted with their transport protocol, **TCP** or **UDP**. Well-known ones: SSH =
  TCP 22, DNS = UDP 53, HTTP = TCP 80, HTTPS = TCP 443. `ping` uses ICMP, which has no ports.
- The **netmask** (`/24`) splits an address into a network part and a host part. Addresses
  sharing the network part form a **subnet**. Larger prefix = smaller subnet: `/32` is one
  machine, `/0` is all addresses.
- Same subnet: deliver directly. Anything else: hand it to a **router**. The **routing table**
  decides, most specific line first; `default via …` points at the **default gateway**.
- `ping` tests whether a machine answers (but many don't, on purpose); `traceroute` shows the
  routers on the path; `curl` tests the actual service.
- *Connection refused* = reached the machine, nothing on that port. *Network unreachable* =
  no route, the packet never left. *Timeout* = dropped on the way, often by a firewall.
  *Could not resolve host* = the DNS lookup failed.
- A service listening on `127.0.0.1` can only be reached from the machine itself; to be
  reachable from outside it must listen on the network interface (or `0.0.0.0`, all of them).
- **Private** addresses start with `10.`, `172.16.`–`172.31.` or `192.168.`, and are never
  routed on the internet. 
- **NAT** rewrites addresses at the router. Outbound NAT lets private VMs reach out; a
  **floating IP** is one-to-one NAT that lets a VM be reached, movable between VMs. The VM
  never sees it.
- A **firewall** is an allow-list of rules by direction, protocol, port and source. A
  **security group** is the cloud's stateful firewall, enforced outside the VM. Blocked
  packets are dropped silently. Open narrowly; patching remains your job.

## Check yourself

1. Your VM is `10.4.7.20/24`. Which of these can it reach **without** a router:
   `10.4.7.1`, `10.4.8.20`, `10.4.7.254`? How many addresses does its subnet contain?
2. A security group rule allows SSH from `136.156.7.42/32`; another from `136.156.0.0/16`.
   How many addresses does each allow, and which would you use for "only my laptop"?
3. A VM's routing table has only the three lines shown in 6.3. Which line does it use for a
   packet to `192.168.1.50`, and which for `9.9.9.9`? Why does the default route not "steal"
   the first packet, although `0.0.0.0/0` matches every address?
4. Private or public: `172.31.200.1`, `172.33.0.1`, `192.169.1.1`, `10.250.3.3`?
5. `ping server.example.org` gets no answer at all, but `curl https://server.example.org`
   returns the web page. Is the server down? What is going on?
6. `curl http://<vm>:5000/` from your laptop answers *Connection refused*. A colleague's
   attempt at `curl http://<vm>:5001/` just hangs. Where should each of you look?
7. A teammate's VM can `dnf update` fine but nobody can SSH to it from outside. Using this
   section's vocabulary, list the two most likely causes and how to check each.

:::{dropdown} Solutions
**1.** `10.4.7.1` and `10.4.7.254`: they share the network part `10.4.7` with the VM.
`10.4.8.20` has network part `10.4.8`, a different subnet, so it needs a router. A `/24`
contains 256 addresses (`10.4.7.0`–`10.4.7.255`), of which `.0` (the subnet itself) and
`.255` (broadcast) are reserved.

**2.** `/32` fixes all 32 bits: exactly **one** address. `/16` fixes the first two numbers:
65,536 addresses (`136.156.0.0`–`136.156.255.255`), e.g. a whole institute. "Only my laptop"
is the `/32`: use your laptop's public address, as `curl ifconfig.me` shows it (6.6), not the
private one your laptop reports about itself.

**3.** `192.168.1.50` matches two lines, and the **most specific** wins: the `/24` line, so
it is delivered directly. `9.9.9.9` matches only `default`, so it goes via `192.168.1.254`. The
default route is `/0`, the least specific prefix possible, so it only ever wins when nothing
else matches.

**4.** `172.31.200.1`: private (`172.16.`–`172.31.`). `172.33.0.1`: public (just past the
range). `192.169.1.1`: public (only `192.168.` is private). `10.250.3.3`: private (anything
starting with `10.`).

**5.** Not down: the web server answers. The server, or a firewall in front of it, simply
drops ping. No answer to `ping` proves little; test the service itself.

**6.** *Connection refused* came back immediately, so your packets reached the VM and nothing
was blocking them. The problem is on the VM: is the service running, and on port 5000?
(`ss -tln` on the VM.) The colleague's hang means the packets were dropped on the way: most
likely the security group does not allow port 5001 (or a host firewall on the VM drops it).

**7.** Outgoing traffic works and incoming does not, which is a common pattern. Either (a)
**no floating IP**: the VM has only its private address, so "from outside" has nothing to aim
at (check the instance's addresses in the dashboard); or (b) floating IP present but the **security group** doesn't
allow ingress TCP 22 from where you are (check the groups attached to the instance and their
rules, including whether the source CIDR covers your current network). Outbound NAT works in
both cases, which is why `dnf update` proves nothing about reachability.

:::

---

Next: [Cloud APIs](07-apis.md), where the dashboard operations of the last three sections are
done from the command line and from Python.
