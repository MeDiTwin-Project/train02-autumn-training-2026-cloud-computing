# 5. Cloud storage

Your training VM has a disk. This section is about where that disk actually is, why
clouds offer **two different kinds** of storage, and what sits underneath both of them on
most scientific clouds, which is usually **Ceph**.

## 5.1 Two kinds of storage, two kinds of questions

Clouds separate storage into two models because applications ask two different questions:

**"Give me a disk"** → **block storage.** A raw virtual disk 
that you attach to a VM, format with a filesystem, and mount. It behaves like a
physical disk: fast and random-access, but attached to **one VM at a time**, and only usable
*through* a VM. In OpenStack this is **Cinder**, and one unit of it is a **volume**.

**"Keep this file, give it back later"** → **object storage.** No disk, no filesystem, no
mounting. You hand whole files (**objects**) over HTTP to a service that stores them in named
**buckets**; you get them back by name, from anywhere, with credentials. Capacity is
generally unlimited and objects are accessible from any machine simultaneously, but there
are no random writes: you replace an object rather than editing it. In OpenStack
this is the **Swift** service. The de-facto standard API for it is **S3** (the AWS product,
whose API has become a de facto standard that open-source systems implement too).

| | Block storage (volume) | Object storage (bucket) |
|---|---|---|
| Looks like | a disk (`/dev/sdb`) | a web service (HTTP) |
| Unit | blocks → filesystem → files | whole objects |
| Attach to | one VM at a time | nothing; any client, anywhere |
| Random writes | yes | no (replace the whole object) |
| Typical use | OS disks, databases, scratch space | datasets, results, backups, images, sharing |
| OpenStack service | **Cinder** | **Swift** (S3-compatible) |
| Day-to-day verb | *mount* | *upload / download* |

:::{note} Jargon: *volume*
A **volume** is one unit of block storage: a virtual disk with a size, existing independently
of any VM. It can be attached, detached, re-attached elsewhere and snapshotted, and it
can survive the deletion of the VM it was attached to. 

For sake of completeness, multi-attach volumes, which can be attached to
multiple VMs at the same time, are available in some specific environments, but
we will not consider them in this training.
:::

:::{note} Jargon: *bucket* / *object*
In S3 vocabulary, a **bucket** is a named object container (globally addressable via a URL) and an
**object** is one stored file plus its metadata, identified by a **key**. A key may look like
a path (`results/run-42/output.nc`) but is simply a string; there are no actual directories.
:::

:::{important} The third kind: shared filesystems
There is a middle model, the **shared filesystem** (NFS, and on HPC systems Lustre or GPFS),
mounted by many machines at once. OpenStack offers it too (a service called **Manila**). It
is the most familiar model to HPC users, but it will not be covered in this training.
:::

## 5.2 What makes cloud storage *cloud*

Where is the volume, physically? It is not in the physical server running your VM. That
would be a serious problem, because that server will eventually fail, and because a VM that
migrates to a different host would lose its disk.

Instead, volumes and objects live in a **storage cluster**: a fleet of machines
full of disks, in the datacentre network, where every piece of data is stored
redundantly across several machines. Cinder is actually only a *manager*: it
takes the "create a 10 GB volume" instruction and delegates to a storage
**backend**. Cinder supports dozens of them (vendor specific appliances, LVM on
a plain server, …), but in the open-source cloud world one backend is by far
the most widely used, and its history is worth a short detour.

## 5.3 Ceph: the storage under the storage

:::{admonition} 📖 Story: Sage Weil and the problem nobody wanted
:class: seealso
**Sage Weil** has a habit of taking on problems that most people would rather pay to avoid.
As a teenager in the 1990s he built **WebRing**, a system that stitched thousands of amateur
websites together, and co-founded a web-hosting company, **DreamHost**. Running a hosting
business taught him that reliable, large-scale, affordable storage was hard to build and
even harder to buy. So in **2004** he went to graduate school at **UC Santa Cruz** to work on
the problem.

Working in a research centre, Weil set out to design a storage system with
**no single point of failure**, one that could grow from a handful of disks to thousands
without a central bottleneck. The result became **Ceph**. By **2007** he had a working
prototype, a PhD, and an algorithm called **CRUSH** at its heart.

CRUSH is the central idea. Most storage systems keep a large central directory: to find your
data, you ask the directory where it lives. A central directory is, however, also a central
bottleneck and a central thing to break. Ceph does without it. Instead, every client runs the
same calculation (a hash, plus a map of the cluster) and *computes* where any piece of data
must be. This removes the lookup, and with it the bottleneck and the single point of failure.
When disks are added or removed, only a minimal amount of data has to move.

*Further reading:
[docs.ceph.com/en/reef/architecture](https://docs.ceph.com/en/reef/architecture/) ·
[en.wikipedia.org/wiki/Sage_Weil](https://en.wikipedia.org/wiki/Sage_Weil)
[Video🍿: Linux Event TV: Sage Weil, CEO and Chief Architect, Inktank](https://www.youtube.com/watch?v=9ibYoZbSBSI)*
:::

This matters to you as a *user* because Ceph is one cluster that speaks **all three storage
languages**: block (RBD, which backs Cinder volumes), object (RGW, which serves the S3/Swift
API), and file system (CephFS). On a typical OpenStack cloud, your volume and your bucket are stored
on the same pool of disks, exposed through different interfaces. Ceph's self-healing is also
why cloud storage tolerates hardware failure without users noticing: when a disk fails, Ceph
detects it within seconds and re-replicates the affected data across the remaining disks,
without anyone having to intervene.

```{mermaid}
flowchart TB
    VM["Your VM"] -->|"/dev/sdb"| Cinder["Cinder (block)"]
    Client["Any client, anywhere"] -->|"HTTPS (S3 API)"| RGW["Object gateway"]
    Cinder --> Ceph[("Ceph cluster<br/>many servers, many disks,<br/>every object stored 3×")]
    RGW --> Ceph
```

:::{note} Jargon: *replication* vs *backup*
Ceph usually keeps ~3 copies of everything, but all three follow your commands. Delete the
volume, and all replicas are lost. **Replication protects against hardware failure; only
backups protect against you.** Snapshots and copies to a *separate* bucket are backups;
replication by Ceph is not.
:::

## 5.4 Demo: provision and attach a volume

:::{admonition} 👀 Instructor demo: a Cinder volume
:class: hint
In the dashboard, go to **Volumes → Volumes → Create Volume**, set the name to `demo-data`
and the size to **5 GiB**, and click **Create**. It appears with status `Available`, a disk
belonging to no machine.

Row menu → **Manage Attachments** → select the demo VM → **Attach Volume**. Status becomes
`In-use`, and the row shows it as `/dev/sdb` on the instance. Inside the VM, `lsblk` now
lists a new, empty disk:

```text
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda    252:0    0   20G  0 disk
├─sda1 252:1    0   19G  0 part /
...
sdb    252:16   0    5G  0 disk            <-- the volume: no partitions, no filesystem
```

The instructor formats and mounts it, writes a file, then **detaches** the volume and
**attaches it to a second VM**, where the file is waiting. The same steps with the CLI, for
reference:

```bash
openstack volume create --size 5 demo-data
openstack server add volume demo-vm1 demo-data
openstack server remove volume demo-vm1 demo-data
openstack server add volume demo-vm2 demo-data
```
:::

## 5.5 Your turn: a volume of your own

Before the session, the organisers did for every training VM what the instructor just did for
the demo VM: they created a **1 GiB Cinder volume** and attached it to your VM. It is a
**raw block device**, a numbered array of blocks with nothing on it. You will find it,
format it, mount it, use it, and then check that your data survives a detach and a
re-attach.

### Step 1: find the disk

SSH into your training VM and list the block devices:

```bash
lsblk
```

```text
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   40G  0 disk
├─sda1   8:1    0   39G  0 part /
...
sdb      8:16   0    1G  0 disk            <-- your volume: no partitions, no filesystem
```

`sda` is the root disk, created from the image when the VM was launched. `sdb` is the volume.
Store its name in a variable and check that it carries no filesystem yet (the `FSTYPE`
column is empty):

```bash
DEV=/dev/sdb
lsblk -f $DEV
```

:::{tip} No `sdb`, or not 1G?
If `lsblk` shows no second disk, ask the instructor. If the volume cannot be attached, do
the same exercise with the loop-device fallback at the end of this section.
:::

### Step 2: format once, mount, use

:::{warning} Check the device before you format
`mkfs` erases whatever is on the device it is given. Make sure `$DEV` is the empty 1G disk
with no mountpoint. Running it on `/dev/sda` would destroy your VM.
:::

```bash
sudo mkfs.ext4 -q $DEV                  # format (ONCE: this erases the disk's contents)
sudo mkdir -p /mnt/vol
sudo mount $DEV /mnt/vol                # mount it
sudo chown $USER: /mnt/vol
echo "hello from a block device" > /mnt/vol/proof.txt
df -h /mnt/vol
```

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb        974M   28K  907M   1% /mnt/vol
```

### Step 3: detach

Next, check that the data outlives the attachment, which is a property that makes volumes
worth having. Detaching is a *cloud* operation (Nova and Cinder), not something the VM can do
by itself, so the instructor detaches the volumes. Your part comes first: **unmount** the
filesystem, so that everything is written to the volume and it is cleanly closed.

```bash
sudo umount /mnt/vol
```

:::{warning} Unmount before the detach
Detaching a volume that is still mounted is like pulling a USB stick out without ejecting it:
data that is still buffered may be lost, and the filesystem may be left damaged. Let the
instructor know once you have unmounted.
:::

:::{admonition} 👀 Instructor demo: detach every volume
:class: hint
When everyone has unmounted, the instructor detaches the participants' volumes. In the
dashboard, each one goes from `In-use` to **`Available`**: a disk belonging to no machine,
with your `proof.txt` on it. For each VM, with the CLI:

```bash
openstack server remove volume <trainee-vm> <trainee-volume>
```
:::

On your VM, the disk has gone:

```bash
lsblk                      # sdb is no longer listed
```

The data was never inside your VM: it lives in the storage cluster (section 5.2), and the
volume now sits there attached to nothing.

### Step 4: re-attach

:::{admonition} 👀 Instructor demo: re-attach every volume
:class: hint
The instructor attaches each volume to its VM again. Status goes back to `In-use`.

```bash
openstack server add volume <trainee-vm> <trainee-volume>
```
:::

Look for the disk again, mount it, and check your file:

```bash
lsblk                      # the 1G disk is back
sudo mount /dev/sdb /mnt/vol && cat /mnt/vol/proof.txt
```

```text
hello from a block device
```

:::{tip} Device names may change
A re-attached volume usually comes back as `sdb`, but the kernel may give it another name
(for instance `sdc`). Use the name `lsblk` shows. For permanent mounts (in `/etc/fstab`), use
the filesystem's **UUID** (shown by `lsblk -f`) instead of the device name, because the UUID
does not change.
:::

The disk exists independently of any VM, and can be attached and detached as needed. The
instructor could just as well have attached your volume to *another* VM, as in the demo of
section 5.4, and `proof.txt` would have been there.

Tidy up by unmounting the volume. Leave it attached; the organisers remove the volumes after
the training.

```bash
sudo umount /mnt/vol
```

::::{dropdown} Fallback: no volume attached? Emulate one with a loop device
Linux can present an ordinary file as a block device, called a **loop device**. The steps
you perform on it (format once, mount, use, detach, re-attach) are the same steps as on a
Cinder volume, and since you do the attaching yourself, they need no cloud at all.

:::{note} Jargon: *loop device*
A **loop device** (`/dev/loop0`, `/dev/loop1`, …) is a block device whose blocks are stored in
a regular file. To the filesystem on top it is indistinguishable from a disk. Some
distributions already use several (Ubuntu, for its snap packages); on your Rocky Linux VM
`lsblk` probably shows none yet.
:::

First, create the "storage backend", a 1 GiB file:

```bash
sudo mkdir -p /srv/volumes
sudo truncate -s 1G /srv/volumes/data.img      # a 1 GiB empty "volume"
ls -lsh /srv/volumes/data.img                   # sparse: 1G in size, ~0 actually used
```

**Attach** it by turning the file into a block device. `losetup` prints which device it
picked:

```bash
DEV=$(sudo losetup --find --show /srv/volumes/data.img)
echo $DEV
lsblk $DEV
```

```text
/dev/loop0
NAME  MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
loop0   7:0    0   1G  0 loop               <-- your volume: no filesystem yet
```

Now follow **Step 2** above (format once, mount, use) unchanged: the commands use `$DEV`,
which is now `/dev/loop0` rather than `/dev/sdb`.

**Detach** the volume yourself, by unmounting it and then releasing the device:

```bash
sudo umount /mnt/vol
sudo losetup -d $DEV
sudo losetup -j /srv/volumes/data.img    # prints nothing: attached to no device
```

The data now exists only in `/srv/volumes/data.img`, attached to nothing. This is the
equivalent of a volume sitting `Available` in the dashboard. **Re-attach** it and look:

```bash
DEV=$(sudo losetup --find --show /srv/volumes/data.img)
sudo mount $DEV /mnt/vol && cat /mnt/vol/proof.txt
```

```text
hello from a block device
```

:::{important} Where the analogy ends
This "volume" is a file on your VM's own root disk, so it would die with the VM. A Cinder
volume lives in a storage cluster *outside* any VM (section 5.2), which is why it survives
the VM and can be re-attached to a *different* one over the network. If you wanted to move
your `data.img` to a teammate's VM, you would have to copy the whole file, a rather clumsy
manual version of what Cinder does with one API call.
:::

Tidy up:

```bash
sudo umount /mnt/vol
sudo losetup -d $DEV
sudo rm /srv/volumes/data.img
```
::::

## 5.6 Demo: browse the object storage

:::{admonition} 👀 Instructor demo: the object store in the dashboard
:class: hint
**Object Store → Containers** (Horizon uses Swift's word "container" for what S3 calls a
bucket; this is an unfortunate clash with Day 2's containers, and the two are unrelated). The
instructor creates one, clicks into it, and **uploads** a small file from the laptop. Clicking
the object shows its metadata: size, hash and a URL. The file is now addressable over HTTPS,
with no VM involved.
:::

Note that none of the steps from 5.5 were needed: no disk was created, sized, formatted,
attached or mounted. Object storage skips that sequence, which is much of its appeal for
data.

## 5.7 Your turn: objects in a bucket

You do have object-storage access of your own: the training object store speaks the **S3**
API (section 7 explains what that means), and your VM comes with **`s3cmd`**, a command-line
S3 client, already configured with your keys. Everything below runs on your **training VM**.

:::{note} Jargon: *S3 URL*
`s3cmd` names buckets and objects with URLs of the form `s3://<bucket>/<key>`. They are not
real web addresses, just a compact notation that S3 clients understand.
:::

First, check that `s3cmd` is there and configured. The configuration lives in `~/.s3cfg`
(the endpoint of the object store and your access key / secret key); `ls` lists the buckets
your keys can see:

```bash
s3cmd --version
ls -l ~/.s3cfg            # your configuration: private to you (-rw-------)
s3cmd ls                  # the buckets you can see
```

```text
s3cmd version 2.4.0
-rw-------. 1 cloud-user cloud-user 1257 Oct 20 09:12 /home/cloud-user/.s3cfg
2026-10-23 12:26  s3://meditwin-training-2026-trainee00
```

:::{tip} `s3cmd: command not found`, or no `~/.s3cfg`?
Install the client with `sudo dnf install -y s3cmd` (it comes from the EPEL repository;
if `dnf` cannot find it, run `sudo dnf install -y epel-release` first). Then run
`s3cmd --configure` and enter the endpoint, access key and secret key shown in the
[credential portal](00-getting-connected.md). Accept the defaults for the other questions.
:::

The listed bucket was created for you.

Store your bucket's URL in a shell variable, to save typing:

```bash
BUCKET=s3://<your-bucket>
```

Upload a file, list the bucket, and download the file again under a new name:

```bash
echo "hello from object storage, $(date)" > hello.txt
s3cmd put hello.txt $BUCKET/
s3cmd ls $BUCKET/
s3cmd get $BUCKET/hello.txt hello-back.txt
cat hello-back.txt
s3cmd info $BUCKET/hello.txt
```

```text
upload: 'hello.txt' -> 's3://meditwin-training-2026-trainee00/hello.txt'  [1 of 1]
 56 of 56   100% in    0s   644.49 B/s  done

2026-10-23 12:29           56  s3://meditwin-training-2026-trainee00/hello.txt

download: 's3://meditwin-training-2026-trainee00/hello.txt' -> 'hello-back.txt'  [1 of 1]
 56 of 56   100% in    0s   467.82 B/s  done

hello from object storage, Fri Oct 23 12:29:48 UTC 2026

s3://meditwin-training-2026-trainee00/hello.txt (object):
   File size: 56
   Last mod:  Fri, 23 Oct 2026 12:29:54 GMT
   MIME type: text/plain
   Storage:   STANDARD
   MD5 sum:   115127ec96ad32f7fddf521ee602fe69
   SSE:       none
   Policy:    none
   CORS:      none
   ACL:       ecmwf-meditwin-efp: FULL_CONTROL
   x-amz-meta-s3cmd-attrs: atime:1792758588/ctime:1792758588/gid:1000/gname:cloud-user/md5:115127ec96ad32f7fddf521ee602fe69/mode:33188/mtime:1792758588/uid:1000/uname:cloud-user
```

`info` shows the object's size, modification time, **MD5** hash and content type.

Finally, remove the copy (an object cannot be edited in place, only replaced or deleted):

```bash
s3cmd del $BUCKET/hello.txt
```

Compare this with 5.5: no `mkfs`, no `mount`, no device, and nothing on your VM's disks
changed except the files you uploaded from and downloaded to. Any machine with the endpoint
and keys sees the same bucket. 

## Recap

- Two models: **block** (Cinder volumes: a disk for one VM, used for the machine) and
  **object** (Swift/S3 buckets: files over HTTP for anyone with credentials, used for the
  data). Shared filesystems (Manila/NFS) exist as a familiar middle, but big shared data
  belongs in buckets.
- Volumes live in a **storage cluster** rather than in your VM's host, which is why they
  survive VM deletion, detach/re-attach, and hardware failure. Cinder delegates to a
  **backend**; the most widely used open-source one is **Ceph**.
- **Ceph** (Sage Weil's PhD project) replaced the central directory with the **CRUSH**
  computation, removing the single point of failure, and serves block, object *and* file
  from one self-healing pool.
- Attaching a volume hands a VM a **raw block device**; format once, mount, use. You did
  this with a real **Cinder volume** and saw the data survive detach and re-attach.
- Object storage needs none of that: with **`s3cmd`** you created/used a bucket and
  `put`, `ls`, `get`, `info` and `del` whole objects, from a machine that mounted nothing.
- **Replication is not backup.** A deletion is applied to every replica.

## Check yourself

1. Your pipeline produces a 200 GB dataset on a volume, and three collaborators (each with
   their own VMs) need to read it. What goes wrong with the volume approach, and which would
   be a better approach?
2. You deleted a VM. Is the data gone? (Careful: there are two answers, depending on where
   the data was.)
3. Why can Ceph clients find data without asking a central directory, and why does that matter
   at petabyte scale?

:::{dropdown} Solutions
**1.** A volume attaches to one VM at a time, so the collaborators would take turns
detaching and re-attaching, or you would build an NFS server VM in front of it (a new single
point of failure that you now administer). A solution would be to upload the dataset to an
**object storage bucket**; all three read it simultaneously over the S3 API from their own
VMs, with access controlled by credentials and no attaching or detaching at all.

**2.** The **root disk** (created from the image as part of the VM) is destroyed with the
instance, unless the VM was explicitly booted from a volume. An **attached Cinder volume**
survives: it detaches and sits `Available`, data intact, just as your volume did with
`proof.txt` on it in section 5.5. (The loop-device fallback would *not* have survived: its
"volume" is a file on the VM's own root disk.)
This asymmetry is a design hint: put anything you care about on a volume or in a bucket, and
treat root disks as disposable.

**3.** With CRUSH, every client knows the current **cluster map** and runs the same
deterministic hash, so the *location of data is computed, not looked up*. At small scale a
directory server works; at petabyte scale it becomes the bottleneck every read must pass
through, the limit on growth, and the single point whose failure takes down the whole
cluster. Removing it lets the cluster grow (and heal) with no central choke point, which is
the property large scientific archives need.
:::

---

Next: [Cloud networking](06-networking.md), the third virtualised resource, and how to find
out why something does not connect.
