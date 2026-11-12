# 7. Cloud APIs

Each action in the dashboard demos (launch, attach, associate) was a REST API call made on your
behalf. Using the REST APIs, the instructor drives OpenStack from the
**command line** and from **Python**; then it is your turn, with the API you *do* have
credentials for: the **S3 API** for object storage, from the CLI and from Python. Here the
cloud becomes a programmable resource.

## 7.1 Why bother, when the dashboard works?

The dashboard does not cover all use cases. Consider some real requests:

- "Every night at 02:00, snapshot the data volume."
- "Recreate our project setup (VMs, volumes, networks) on another cloud."
- "Which of our instances have a floating IP but have been shut off for a month?"

These are impractical by clicking, but straightforward with e.g. a `for` loop.
Scripts are also
**documentation that executes**: they can be version-controlled, reviewed and reproduced.

```{mermaid}
flowchart LR
    B["Dashboard<br/>(Horizon)"] --> API["OpenStack REST APIs<br/>Keystone · Nova · Cinder · Neutron · Swift"]
    C["CLI<br/>(openstack ...)"] --> API
    P["Python<br/>(openstacksdk / boto3)"] --> API
    T["Day-4 tools<br/>(Terraform, CI jobs)"] --> API
```

All of these clients are equivalent; each of them calls the same APIs.

## 7.2 Getting in: credentials for machines

Browsers log in interactively; scripts cannot. OpenStack's answer is an **application
credential**: a scoped, revocable ID/secret pair that a script can use without embedding a
personal password.

:::{admonition} 👀 Instructor demo: an application credential
:class: hint
In the dashboard: **Identity → Application Credentials → Create Application Credential**,
named `training01`, roles and expiry at defaults, then **Create**, and **download the
`openrc` file it offers**. It is a small shell script that sets **environment variables**,
the standard way of handing credentials to every OpenStack client:

```bash
# ~/app-cred-training01-openrc.sh
export OS_AUTH_TYPE=v3applicationcredential
export OS_AUTH_URL=https://keystone.example.org:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_REGION_NAME=RegionOne
export OS_INTERFACE=public
export OS_APPLICATION_CREDENTIAL_ID=f0e1d2c3...
export OS_APPLICATION_CREDENTIAL_SECRET=REDACTED
```

The instructor protects the file and loads it into the current shell:

```bash
chmod 600 ~/app-cred-training01-openrc.sh
source ~/app-cred-training01-openrc.sh
env | grep '^OS_' | cut -d= -f1      # which variables are set (names only, no secrets on screen)
```

Every `openstack` command and every Python script started from *this* shell now finds the
credentials in its environment. A new terminal knows nothing until the file is sourced again.
:::

:::{note} Jargon: *environment variable* and *rc file*
An **environment variable** is a named value (`OS_AUTH_URL=...`) that a shell passes on to
every program it starts. An **rc file** ("run commands") is a script of `export` lines that
you `source` to set many of them at once. Credentials in the environment never appear in the
code, so the same script runs unchanged for anyone who sources their own rc file. (OpenStack
also accepts the same information as a YAML file, `clouds.yaml`, with the same content in a
different format.)
:::

:::{danger} Credentials are secrets
The secret in the rc file is a key to the project: anyone holding it can create and delete
resources and run up quota. Never commit it to Git, never paste it in chat, never bake it into
an image. If it leaks, **Identity → Application Credentials → Delete** revokes it immediately.
This revocability is the main reason you make an application credential instead of using your
password; another is that it keeps working when your personal password changes. The same
rules apply to your S3 keys in `~/.s3cfg`. 
:::

## 7.3 Demo: the `openstack` CLI

:::{admonition} 👀 Instructor demo: the CLI
:class: hint
The client is a Python package:

```bash
python3 -m venv ~/venv && source ~/venv/bin/activate
pip install python-openstackclient
source ~/app-cred-training01-openrc.sh      # credentials into the environment (7.2)
```

The CLI is one command, `openstack`, with `<noun> <verb>` subcommands. List the
VMs (a.k.a. *servers* or *instances* in OpenStack jargon):

```bash
openstack server list
```

```text
+--------------------------------------+----------+--------+-----------------------------------------+--------------+----------+
| ID                                   | Name     | Status | Networks                                | Image        | Flavor   |
+--------------------------------------+----------+--------+-----------------------------------------+--------------+----------+
| 8a1c9e2f-3b4d-4c5e-9f6a-7b8c9d0e1f2a | demo-vm1 | ACTIVE | private-net=192.168.1.14, 136.156.x.y   | ubuntu-24.04 | 2cpu-4gb |
+--------------------------------------+----------+--------+-----------------------------------------+--------------+----------+
```

This is the section-4 demo VM, with its private and floating IP. The inventory continues:

```bash
openstack volume list            # the volume from section 5
openstack network list           # private-net and the external network
openstack security group list    # including demo-ssh from section 6
openstack flavor list            # every machine size this cloud offers
openstack image list             # every boot image
```

This covers what the dashboard showed, and a bit more (`--long`, `-f json` for
machine-readable output). The verbs work too. Here is the section-4 wizard as three commands:

```bash


openstack server create \
    --image ubuntu-jammy --flavor 1cpu-1gbmem-30gbdisk \
    --network private-ecmwf-meditwin-efp --key-name demokey \
    demo-vm2
openstack server show demo-vm2 -c status -c addresses
openstack server delete demo-vm2       # and away it goes
```

Without the rc file sourced, the same commands fail immediately with a message that no
authentication URL or credentials were found: the CLI has no other way of knowing who you are.
:::

:::{tip} Discoverability
`openstack --help` lists hundreds of commands; `openstack server create --help` explains one.
The pattern `openstack <noun> {list,show,create,delete,set}` covers most daily work, and the
nouns map to the section-4 services: `server` is Nova, `volume` is Cinder, `network` is
Neutron.
:::

## 7.4 Demo: OpenStack from Python

The CLI is meant for interactive use; **openstacksdk** gives programs the same access. 

:::{admonition} 👀 Instructor demo: `inventory.py`
:class: hint
```bash
pip install openstacksdk
```

```python
import openstack

conn = openstack.connect()   # reads the OS_* environment variables, like the CLI

print("=== Servers ===")
for server in conn.compute.servers():
    addresses = [
        addr["addr"]
        for net in server.addresses.values()
        for addr in net
    ]
    print(f"{server.name} {server.status} {', '.join(addresses)}")

print("=== Volumes ===")
for vol in conn.block_storage.volumes():
    print(f"{vol.name} {vol.status} {vol.size} GiB")

print("=== Floating IPs ===")
for ip in conn.network.ips():
    print(f"{ip.floating_ip_address} -> {ip.fixed_ip_address}")
```

```bash
source ~/app-cred-training01-openrc.sh      # if this shell has not loaded it yet
python3 inventory.py
```

```text
=== Servers ===
demo-vm1             ACTIVE   192.168.1.14, 136.156.x.y
=== Volumes ===
demo-data            in-use   5 GiB
=== Floating IPs ===
136.156.x.y        -> 192.168.1.14
```
:::

In about twenty lines, the infrastructure has become a set of Python objects. The script
contains no URL and no secret: pointed at a different project or a different cloud, it runs
unchanged, as long as a different rc file is sourced first.

## 7.5 Your turn: S3, the common object-storage API

Object storage (section 5) deliberately speaks HTTP, and most implementations (AWS, Ceph,
MinIO, Swift) support the same dialect, the **S3 API**. Once you have learnt it, you can use it
against most clouds. You already used it in [section 5.7](05-storage.md) through `s3cmd`;
now you look under the hood and drive it from Python.

Everything below runs on your **training VM**.

### Where `s3cmd` gets its credentials

An S3 client needs three things: the **endpoint** (the URL of the object store), an **access
key** (who you are) and a **secret key** (proof that it is you). `s3cmd` reads them from
`~/.s3cfg`, which was prepared for you. Look at the non-secret part of it:

```bash
grep -E '^(host_base|host_bucket|use_https|signature_v2) ' ~/.s3cfg
```

```text
host_base = s3.training.meditwin-project.eu
host_bucket = s3.training.meditwin-project.eu
use_https = True
signature_v2 = False
```

`host_base` is the endpoint. `host_bucket` without a `%(bucket)s` in it means **path-style**
URLs, `https://<s3-endpoint>/<bucket>/<key>`, which every S3-compatible store understands.
`access_key` and `secret_key`, are your credentials: keep them off the
screen, as the `grep` above does, and keep the file private (`-rw-------`).

:::{note} An S3 client, but no Amazon
The S3 API started at Amazon, and so did its vocabulary (`aws` appears in some option names
and environment variables below), but no AWS account or service is involved: every request
goes to the training object store.
:::

### From the CLI

```bash
BUCKET=s3://meditwin-training-2026-trainee00
echo "measured at $(date)" > measurement.txt

s3cmd put measurement.txt $BUCKET/
s3cmd get --force $BUCKET/measurement.txt retrieved.txt
diff measurement.txt retrieved.txt && echo "round trip OK"
```

```text

upload: 'measurement.txt' -> 's3://meditwin-training-2026-trainee00/measurement.txt'  [1 of 1]
 41 of 41   100% in    0s   401.49 B/s  done
download: 's3://meditwin-training-2026-trainee00/measurement.txt' -> 'retrieved.txt'  [1 of 1]
 41 of 41   100% in    0s   730.92 B/s  done
round trip OK
```

To copy a whole directory, `s3cmd sync` uploads only what has changed since the last run, a
cheap way to back up results:

```bash
mkdir -p results && date > results/run-001.txt && date > results/run-002.txt
s3cmd sync results/ $BUCKET/results/
s3cmd sync results/ $BUCKET/results/      # second run: nothing to upload
s3cmd ls $BUCKET/results/
```

### Under the hood: it is just HTTP

`s3cmd` is one client among many. To see that there is nothing behind it but signed HTTP
requests, fetch the same object with plain `curl`, which knows how to sign S3 requests. The
small `s3cfg` shell function reads one value from `~/.s3cfg`, so that the keys go straight
from the file into the request without appearing on screen:

```bash
s3cfg() { awk -F' *= *' -v k="$1" '$1 == k { print $2; exit }' ~/.s3cfg; }

curl --aws-sigv4 "aws:amz:us-east-1:s3" \
     --user "$(s3cfg access_key):$(s3cfg secret_key)" \
     -H "x-amz-content-sha256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855" \
     "https://$(s3cfg host_base)/${BUCKET#s3://}/measurement.txt"
```

```text
measured at Thu Oct 22 14:11:58 UTC 2026
```

That is a `GET` on a URL (`${BUCKET#s3://}` is your bucket name, with the `s3://` prefix
removed), with a signature header proving that you hold the secret key. The long hexadecimal
header is the SHA-256 hash of the request body, which for a `GET` is empty; S3 wants it as
part of the signature. `us-east-1` is the region name the signature is computed for; it is
also what `s3cmd` uses with `bucket_location = US`.

The S3 API as a whole consists of verbs (`GET`, `PUT`, `DELETE`, `LIST`) on URLs. Drop the
`--aws-sigv4`, `--user` and `-H` options and the request is refused with `AccessDenied`,
because the bucket is private. (To watch `s3cmd` itself make these requests, add `--debug`
to any `s3cmd` command.)

### From Python

The most widely used Python library for the S3 API is **boto3**. Install it in a virtual
environment:

```bash
python3 -m venv ~/venv && source ~/venv/bin/activate
pip install boto3
```

boto3 does not read `~/.s3cfg` by itself, so the script below reads it with the standard
library's `configparser` and hands the values to boto3. Save it as `s3demo.py`, and replace
`<your-bucket>` with your bucket name (without `s3://`):

```python
import configparser
from pathlib import Path

import boto3
from botocore.config import Config

# Endpoint and keys come from s3cmd's ~/.s3cfg, never from the code
cfg = configparser.ConfigParser(interpolation=None)
cfg.read(Path.home() / ".s3cfg")
s3cfg = cfg["default"]
scheme = "https" if s3cfg.getboolean("use_https", fallback=True) else "http"

s3 = boto3.client(
    "s3",
    endpoint_url=f"{scheme}://{s3cfg['host_base']}",
    aws_access_key_id=s3cfg["access_key"],
    aws_secret_access_key=s3cfg["secret_key"],
    region_name="US",
    config=Config(
        signature_version="s3v4",
        s3={"addressing_style": "path"},
        # only send checksums the S3 standard requires; some non-AWS stores reject the rest
        request_checksum_calculation="when_required",
        response_checksum_validation="when_required",
    ),
)

bucket = "meditwin-training-2026-trainee00" # CHANGE THIS
key = "runs/run-001/measurement.txt"

# upload: an object's key can look like a path
s3.upload_file("measurement.txt", bucket, key)

# list
for obj in s3.list_objects_v2(Bucket=bucket).get("Contents", []):
    print(f"{obj['Size']:>8}  {obj['Key']}")

# retrieve straight into memory, no file needed
body = s3.get_object(Bucket=bucket, Key=key)["Body"].read()
print("content:", body.decode().strip())

# share: a pre-signed URL works without any keys, until it expires
url = s3.generate_presigned_url(
    "get_object", Params={"Bucket": bucket, "Key": key}, ExpiresIn=3600
)
print("share this link for one hour:", url)
```

```bash
python3 s3demo.py
```

```text
      41  measurement.txt
      29  results/run-001.txt
      29  results/run-002.txt
      41  runs/run-001/measurement.txt
content: measured at Thu Oct 22 14:11:58 UTC 2026
share this link for one hour: https://<s3-endpoint>/meditwin-training-2026-trainee00/runs/run-001/measurement.txt?X-Amz-Algorithm=...
```

Open the shared link in the browser on **your laptop**: the file appears, although your laptop
has no keys at all. The signature in the URL grants access to that one object, for one hour.
This is how you hand a result to a collaborator without handing over your credentials.

Note that the script contains no secrets, so you can commit it to Git, share it or paste it in
a chat. Whoever runs it brings their own credentials.

This pattern, where one program writes results to a bucket and another reads them from
anywhere, is at the core of many cloud-native scientific pipelines.

## Recap

- The dashboard, the CLI, Python, are **equal clients of the same REST APIs**.
- Machines authenticate against OpenStack through **application credentials**, 
  which can be loaded from an **rc file** into
  **environment variables** (`source ...-openrc.sh`): scoped, revocable, and separate from
  your personal password. Credentials are secrets, and a leaked one is dealt with by revoking
  it.
- The **`openstack` CLI** is `<noun> <verb>` over the section-4 services;
  **openstacksdk** gives Python the same access (`openstack.connect()` → iterate resources).
- **S3** is the widely supported object-storage API: endpoint + access key + secret key,
  usable from `s3cmd`, **boto3** (or `minio`, `s3fs`, …) or plain `curl` (since it is signed
  HTTP) against any compliant store. Keep the keys in a private configuration file
  (`~/.s3cfg`), not in your code, and share objects with **pre-signed URLs**, not with keys.

## Check yourself

1. A colleague's deployment script contains their personal OpenStack password. Give two
   concrete reasons an application credential is better, beyond "it feels safer".
2. Using only this section's tools, sketch (in words or pseudo-code) the nightly job "delete
   every VM whose name starts with `scratch-`".
3. Your Python S3 script works on your training VM but fails on your laptop, same code. What
   are the two most likely causes?
4. A collaborator needs one of your result files today. Why send a pre-signed URL rather than
   your `~/.s3cfg`?

:::{dropdown} Solutions
**1.** (a) **Blast radius**: the password unlocks everything the person can do (every
project, the dashboard, possibly linked services), whereas the application credential is
scoped to one project and can carry restricted roles and an expiry. (b) **Easy revocation**: a
leaked credential is deleted in one click and the script gets a new one; a leaked password
forces a password change and a search for every place it was used. (Bonus: credentials also
survive password-rotation policies, so the script does not break when the human's password
changes.)

**2.**
```python
conn = openstack.connect()        # credentials from the sourced rc file
for server in conn.compute.servers():
    if server.name.startswith("scratch-"):
        conn.compute.delete_server(server)
```
…run nightly by any scheduler (cron today; a CI schedule on Day 4, where the rc file's
variables become CI variables). The point of the question is that once infrastructure is
represented as objects in a programming language, a policy like this becomes a `for` loop.

**3.** (a) **Missing configuration on the laptop**: the keys and endpoint live in `~/.s3cfg`
on the VM, and the laptop does not have that file; the same code does not imply the same
environment. (b) **Missing library**: `boto3` was installed in a virtual environment on the
VM (`~/venv`), not on the laptop, so the script stops at `import boto3` with
`ModuleNotFoundError`. The error message tells you which of the two it is: a missing
module, or a missing file or key (`KeyError: 'default'` when `~/.s3cfg` is absent). The network
is an unlikely culprit here: you already opened a pre-signed URL from the laptop's browser,
which proves the laptop can reach the endpoint.

**4.** The pre-signed URL grants **read access to one object, for a limited time**, and
needs no software beyond a browser. `~/.s3cfg` grants **everything your keys can do, on every
bucket, until the keys are revoked**, including deleting your data, and you would then have to
rotate your keys. Share data, not credentials.
:::

---

Next: [Lambda functions](08-faas.md), where the VM disappears as well.
