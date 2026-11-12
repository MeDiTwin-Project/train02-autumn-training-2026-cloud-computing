# 2. Security fundamentals for cloud-native workloads

As soon as a service has a public HTTPS address, anybody on the internet can send it a request.
This section answers three questions about each request:

> **Who sent it? And is that sender allowed to do this?**

The first question is **authentication**, the second is **authorisation**. Under both there is
**HTTPS**: without it, nothing else holds.

:::{important} What this section is not
It is not a full security course, and it is not an application security course: you do not change
the code of the application. Everything here is done by the platform, in front of the application
or around it: TLS certificates and a password check in the Ingress controller. Keeping secrets out
of Git and scanning images in a pipeline are covered in
[Day 4, section 4](../day04/04-pipeline-security.md), and are not repeated here. Limiting what a pod
may do (Kubernetes permissions, security contexts) is not covered: section 2.9 has pointers.
:::

**Symbols in this section:** ⌨️ you type the commands · 📖 reference material: read it after the
lesson, or when you need it. The lesson might not cover it in class.

:::{admonition} What you build in this section
:class: tip

You protect the `climate-stats-api` of section 1, the API in front of PostgreSQL and Redis, in
two hands-ons. The application stays at the version of section 1; only the manifests change.

| Hands-on | What you do | What you see |
|---|---|---|
| 4 ([2.5](#d5-ho4)) | Publish the API on HTTPS only, and follow cert-manager while it gets a certificate from Let's Encrypt | `Certificate` → `CertificateRequest` → `Order` → `Challenge`; the certificate with `openssl` and `curl` |
| 5 ([2.7](#d5-ho5)) | Traefik asks for a password (HTTP Basic Auth), one for each client | `401` without the password, `200` with it; a pod that goes around Traefik |

You need your training VM, `kubectl` access to your namespace `traineeNN`, and the folder
`~/climate-stack-k8s` with the manifests of section 1. Step 0 of Hands-on 4 has them if you lost
them.
:::

## 2.1 Think like an attacker first

Before choosing controls, look at what you are protecting. `climate-stats-api` has three endpoints:

| Endpoint | Does | Who should use it |
|---|---|---|
| `GET /health` | Answers `{"status": "healthy"}` | Anybody: Kubernetes probes, load balancers |
| `GET /stats` | Counts the records in PostgreSQL, and adds one to a counter in Redis | Known clients only |
| `GET /metrics` | Numbers about the service, for the monitoring system (section 1) | The monitoring system, inside the cluster only |

`GET /stats` only reads, but it is not free: each call runs a query on the database and **writes** to
Redis. A request that "only reads" can still change state and use resources.

Here it is, with everything it talks to:

```{mermaid}
flowchart LR
    C([Client]) -->|"HTTPS :443"| T[Traefik<br/>Ingress controller]
    T -->|"HTTP, inside the cluster"| S[Service climate-api]
    S --> P["Pod: /health, /stats,<br/>/metrics"]
    P --> DB[(PostgreSQL<br/>climate-db)]
    P --> R[(Redis<br/>climate-cache)]
    P -.->|service account token| API[(Kubernetes API)]
```

For each arrow, ask: *what if somebody else is at this end?*

| What an attacker wants | How they try it | The control |
|---|---|---|
| Read the statistics, or make the service work for them | Send requests to `/stats` | **Authentication** on `/stats` |
| Overload the database | Call `/stats` in a loop | Authentication; a rate limit at the edge (section 2.9) |
| Your password | Read it on the network, or guess it | **HTTPS**; long random passwords; one password for each client |
| Your users, with a fake copy of your service | Send them to a server they control, with your name | A **trusted certificate**, checked by the client |
| The private key of your certificate | Read the TLS `Secret` | Permissions on Secrets (**RBAC**, Day 3); nobody else in your namespace |
| Your credentials, and the database password | Find them in Git, in the image or in logs | Secrets outside Git; never print them |
| Endpoints you did not mean to publish | Try other paths: `/metrics`, `/admin`, `/` | **Expose only the paths you mean to** |
| The cluster, after a bug in your code | Use the pod's token and permissions | **Least privilege** for the pod: no Kubernetes permissions, no token, non-root (section 2.9) |

This list is a **threat model**. It does not need to be long.

:::{note} Jargon: *attack surface* and *blast radius*
The **attack surface** is everything an attacker can reach: open ports, public paths, accounts. The
**blast radius** is how much an attacker can do *after* one control fails. Good security makes the
first small and the second smaller.
:::

### Defence in depth: the four Cs

Cloud-native security is often shown as four layers, from the outside in. Each layer trusts the layer
outside it, and none is enough alone.

:::{figure} img/four-cs.svg
:alt: Four nested boxes. Cloud (VMs, networks, security groups, Day 1) contains Cluster (Ingress, certificates, a password at the edge, this section; RBAC, Day 3), which contains Container (non-root, minimal image, scanned, Days 2 and 4), which contains Code (input checks, dependencies, scanned on Day 4).
:::

This section works on the layer that a cloud platform gives you in front of every application: the
**cluster**, with its Ingress controller, certificates, and a password check at the edge.

## 2.2 Authentication and authorisation

These two words look alike and are often confused. They are different questions, asked in a fixed
order.

| | Authentication (*AuthN*) | Authorisation (*AuthZ*) |
|---|---|---|
| Question | **Who are you?** | **Are you allowed to do this?** |
| Input | A credential: password, token, certificate | An identity, an action and a resource |
| Example | "This request comes from `dashboard`." | "`dashboard` may read the statistics, but not change the configuration." |
| HTTP failure | **`401 Unauthorized`** | **`403 Forbidden`** |
| Kubernetes | Certificates, tokens, OIDC | **RBAC**: Roles and RoleBindings |

```{mermaid}
flowchart LR
    R[Request] --> A{Authentication:<br/>valid credential?}
    A -->|no| E401["401 Unauthorized"]
    A -->|yes, it is dashboard| Z{Authorisation:<br/>may dashboard do this?}
    Z -->|no| E403["403 Forbidden"]
    Z -->|yes| OK["200 OK"]
    style E401 fill:#ffe6e6,stroke:#c00,color:#5c0a0a
    style E403 fill:#ffe6e6,stroke:#c00,color:#5c0a0a
```

:::{note} The HTTP names are misleading
`401 Unauthorized` actually means *unauthenticated*: "I do not know who you are". `403 Forbidden`
means "I know who you are, and the answer is no". The names are historical; the meaning is the
table above. Remember it, because it tells you which half to debug. You see `401` from Traefik in
Hands-on 5.
:::

### Two kinds of identity

Not every caller is a person.

- **Humans** use passwords, single sign-on (SSO) and second factors. Their sessions are short.
- **Workloads** (a pipeline job, a pod, a script) use tokens, API keys or certificates. Nobody types
  these, so they must be stored somewhere, and *where* is the security question. Examples: the token
  of a CI job, a registry deploy token, an S3 access key. In Kubernetes, a workload's identity is its
  **service account**, and Kubernetes puts a token for it in every pod (section 2.8).

The two clients of `climate-stats-api` in Hands-on 5, `dashboard` and `report`, are workloads.

:::{note} Jargon: *principal*
A **principal** is anything that can be authenticated: a user, a service account, a CI job. Security
policies are written in terms of principals ("this principal may do that"), so the word turns up in
cloud providers' documentation.
:::

(d5-tls)=
## 2.3 HTTPS and TLS: what they protect, and how

HTTPS is HTTP inside **TLS** (*Transport Layer Security*). Here is what TLS gives you, and what it
does not:

| TLS gives you | TLS does **not** give you |
|---|---|
| **Confidentiality**: nobody on the path can read the request, the password or the answer | Any idea of *who the client is* |
| **Integrity**: nobody on the path can change them | Protection after the traffic arrives: in Traefik, in the pod, in logs |
| **Server authentication**: the client knows it talks to the real `stats-traineeNN…`, not an impostor | Protection against a bug in your application |

So HTTPS authenticates the **server** to the client. It says nothing about the **client**. That is
why you still need authentication, and why authentication without HTTPS is bad: the
credential crosses the network in a form anybody on the path can read.

(d5-pki)=
### Public key cryptography in five minutes

Everything in the rest of this section, TLS, certificates, Let's Encrypt and tokens, is built on one
idea. It helps to see it once, without the details.

**Symmetric** cryptography uses **one key** to encrypt and to decrypt (AES is the usual algorithm).
It is fast, and TLS encrypts all your data with it. But both sides need the same key, and they
cannot send it over a network that somebody may be listening to.

**Public key** (*asymmetric*) cryptography uses a **key pair**: two keys that are made together and
match each other.

- The **private key** stays secret, where it was made: in a file, a Secret, a hardware module.
- The **public key** can be given to anybody, and published anywhere.

What one key does, only the other can check or undo. That gives two tools:

| Tool | How it works | What it proves | Where you meet it today |
|---|---|---|---|
| **Signature** | The owner signs some data with the **private** key. Anybody checks it with the **public** key | The data comes from the owner of the key, and nobody changed it | A CA signs a certificate; Traefik signs the handshake; cert-manager signs its requests to Let's Encrypt with its account key; the API server signs a pod's token |
| **Key exchange** | Each side combines **its own** private key with the **other's** public key. Both get the same secret; somebody who sees only the public keys cannot compute it | Nothing about identity: it only gives two sides a shared secret | The session keys of every TLS connection (ECDHE, X25519) |

```{mermaid}
flowchart LR
    D[Data] --> Sign["sign with the<br/>PRIVATE key"]
    Sign --> DS["Data + signature"]
    DS --> Verify{"check with the<br/>PUBLIC key"}
    Verify -->|matches| OK["Made by the owner,<br/>not changed"]
    Verify -->|does not match| KO["Refuse it"]
```

TLS uses both, and symmetric encryption on top: a **key exchange** gives the two sides a fresh
symmetric key for the connection, a **signature** proves that the server is the owner of the
certificate, and a **certificate** is the CA's signature on "this public key belongs to this name".

Signatures do not sign the whole data directly, but a **hash** of it: a short fingerprint (SHA-256
gives 32 bytes) that changes completely if one bit of the data changes. The **digest** of an image,
which Day 4 signed, is such a hash, and so is the fingerprint of a certificate
(`openssl x509 -noout -fingerprint -sha256`).

| Algorithm | Used for | Here |
|---|---|---|
| RSA (2048 bits and more) | Signatures | Your certificate's key (Hands-on 4), Kubernetes service account tokens (`RS256`) |
| ECDSA, Ed25519 | Signatures, with much shorter keys | Many certificates; the SSH key of Day 1 (`ssh-keygen -t ed25519`) |
| X25519 (ECDHE) | Key exchange | Most TLS handshakes today, yours included |

You have used key pairs before: on [Day 1](../day01/04-openstack.md) you put the **public** half of
an SSH key on the VM and kept the private half on your laptop, and on
[Day 4](../day04/04-pipeline-security.md) the pipeline signed images with a private key while the
public key went into the repository. It is the same rule every time: **the private key never
travels, and whoever has it *is* its owner** for everybody who trusts the public key.

:::{note} Quantum computers and "post-quantum" algorithms
A large enough quantum computer could break RSA, ECDSA and X25519. It is not yet feasible today, but traffic
recorded now could be decrypted later ("store now, decrypt later"), so new **hybrid** key exchanges
that add a post-quantum algorithm (ML-KEM) are already turned on in recent browsers and in OpenSSH:
that is what the warning about a "post-quantum key exchange" of a recent `ssh` client means. 
:::

### The handshake

Before the first byte of HTTP, client and server do a **TLS handshake**. With TLS 1.3, the current
version, it takes one round trip:

```{mermaid}
sequenceDiagram
    participant C as Client (curl)
    participant S as Server (Traefik)
    C->>S: ClientHello<br/>TLS versions, ciphers, key share,<br/>SNI: stats-traineeNN.training...
    Note over S: Chooses the certificate for that name
    S-->>C: ServerHello + key share
    Note over C,S: Both compute the same session keys.<br/>From here on, everything is encrypted.
    S-->>C: Certificate chain<br/>+ signature with the private key
    Note over C: 1. Is the chain signed by a CA I trust?<br/>2. Is the name in the certificate?<br/>3. Is it still valid?<br/>4. Does the server have the private key?
    C->>S: Finished
    C->>S: GET /stats<br/>Authorization: Basic ...
    S-->>C: 200 OK
```

1. The client says which versions and ciphers it knows, sends its half of a key exchange, and the
   **name** of the server it wants: the **SNI** (*Server Name Indication*). One Traefik serves
   hundreds of names on one IP address; the SNI tells it which certificate to show.
2. The server answers with its half of the key exchange. Both sides now compute the same
   **session keys**, which never cross the network.
3. The server sends its **certificate chain**, and proves with a signature that it has the
   **private key** of the certificate.
4. The client checks the certificate (the four questions in the diagram). If one answer is no, the
   connection stops: no request is sent, so no password is sent.

:::{note} What an observer on the network still sees
The SNI travels in clear text, and so does the DNS query before it. Somebody on the path sees **that**
you connect to `stats-traineeNN.training.meditwin-project.eu`, when and how much data. They do not
see the path, the headers (with the password), or the body.
:::

:::{note} Jargon: *forward secrecy*
The session keys come from a key exchange (ECDHE) that is new for every connection, not from the
certificate's key. If the private key is stolen next year, traffic recorded today cannot be
decrypted with it. This property is called **forward secrecy**, and TLS 1.3 always has it.
:::

### Certificates and the chain of trust

A certificate is a small signed document. Its main fields:

| Field | Example | Means |
|---|---|---|
| Subject Alternative Name (SAN) | `DNS:stats-traineeNN.training.meditwin-project.eu` | The names the certificate is valid for. Clients check **this** field |
| Subject | `CN=stats-traineeNN.training.meditwin-project.eu` | The old place for the name; clients ignore it today |
| Issuer | `C=US, O=Let's Encrypt, CN=YR1` | Who signed it |
| Validity | `notBefore`, `notAfter` | When it is valid |
| Public key | RSA 2048 bits, or ECDSA | The half of the key pair that everybody may know |
| Signature | | Made by the issuer with **its** private key |

Your certificate is signed by an **intermediate** certificate of Let's Encrypt, and that one by a
Let's Encrypt **root** certificate. The roots come with your operating system and your browser: a
short list of certificate authorities (CAs) that everybody agreed to trust. These are the
certificates of the training cluster:

```{mermaid}
flowchart LR
    X1["ISRG Root X1<br/>(in the OS trust store)"] -->|cross-signs| YR["Root YR<br/>(sent by the server)"]
    YR -->|signs| Int["YR1, intermediate<br/>(sent by the server)"]
    Int -->|signs| Leaf["Your certificate<br/>stats-traineeNN..."]
    Key[["Private key<br/>(only in the Secret climate-api-tls)"]] -.->|matches| Leaf
```

`Root YR` is a new root of Let's Encrypt. Not every system has it yet, so the server also sends a
copy of it signed by the older `ISRG Root X1`, which every system has: a **cross-signature**. The
client follows the links until it reaches a root it already has. If any link is missing or any
signature is wrong, the client refuses the connection.

The **private key** is the one secret in this picture. Whoever has it can pretend to be your
service. In Kubernetes, it is in the TLS `Secret`, next to the certificate.

### Where TLS ends

On the training cluster, TLS **terminates** at Traefik, the Ingress controller.

```{mermaid}
flowchart LR
    C([Client]) -->|"encrypted (TLS)"| T[Traefik]
    T -->|"plain HTTP"| P[Pod]
    subgraph Cluster
      T
      P
    end
```

:::{note} Jargon: *TLS termination*
The component that holds the certificate and decrypts the traffic **terminates** TLS. Behind it, the
traffic is normally plain HTTP on the cluster network. That is acceptable when the cluster network is
trusted. When it is not, you encrypt the inner hop too, often with *mutual TLS* (mTLS) from a service
mesh, where both sides show a certificate.
:::

Two consequences:

1. Anything that runs inside the cluster and can reach your pod's port does **not** go through
   Traefik. That includes the kubelet's probes and every other pod. Controls you put on the Ingress
   do not apply to them.
2. Plain HTTP must not carry credentials. On Day 3
   ([section 4.7.1](../day03/04-exposing-an-application.md)) a middleware redirected `http://` to
   `https://`: right for a website, but a client that forgets the `s` has already sent its first
   request, with its password, in clear text, and the redirect then hides the mistake. For an API, publish **HTTPS only**, with the annotation
   `traefik.ingress.kubernetes.io/router.entrypoints: websecure`: port 80 answers `404`,

(d5-cert-manager)=
## 2.4 How cert-manager gets a certificate from Let's Encrypt

On Day 3 ([section 6](../day03/06-cert-manager-and-dns.md)) one annotation on the Ingress was enough
to get a certificate. This part looks at what happens in between.

### The ACME conversation

Let's Encrypt is a free, automated certificate authority. Clients talk to it with the **ACME**
protocol (*Automatic Certificate Management Environment*, RFC 8555). Before it signs a certificate for
a name, it checks that the client **controls** that name. cert-manager does each step of ACME, and
writes each step as a Kubernetes object:

```{mermaid}
sequenceDiagram
    participant I as Ingress (yours)
    participant CM as cert-manager
    participant LE as Let's Encrypt
    participant T as Traefik (port 80)

    I->>CM: annotation cert-manager.io/cluster-issuer
    Note over CM: Certificate climate-api-tls
    CM->>CM: New private key (stays in the cluster)
    Note over CM: CertificateRequest: a CSR, with the public key only
    CM->>LE: New order for stats-traineeNN...
    LE-->>CM: Prove it: put token X at /.well-known/acme-challenge/X
    Note over CM: Order and Challenge
    CM->>T: A solver pod and a temporary Ingress serve the token
    CM->>T: Self-check: can I fetch the token myself?
    CM->>LE: Ready, check it
    LE->>T: GET http://stats-traineeNN.../.well-known/acme-challenge/X
    T-->>LE: token X + account key fingerprint
    CM->>LE: Finalize: here is the CSR
    LE-->>CM: Signed certificate and intermediate
    Note over CM: Secret climate-api-tls: tls.crt + tls.key.<br/>Solver removed.
```

| Object | Created by | Holds | Lives |
|---|---|---|---|
| `Certificate` | cert-manager, from the Ingress annotation (its name is the `secretName`) | What you want: names, issuer, Secret | As long as the Ingress |
| `CertificateRequest` | cert-manager, one for each issuance or renewal | A **CSR** (*certificate signing request*): the names and the **public** key | Kept, as a record |
| `Order` (`acme.cert-manager.io`) | cert-manager, one for each request to Let's Encrypt | The state of the ACME order: `pending`, `valid`, `invalid` | Kept, as a record |
| `Challenge` (`acme.cert-manager.io`) | cert-manager, one for each name in the order | The token, and the state of the check | **Deleted** when the check is done |
| Solver pod, Service and Ingress `cm-acme-http-solver-…` | cert-manager | A tiny web server that answers with the token | **Deleted** when the check is done |
| `Secret` (`kubernetes.io/tls`) | cert-manager | `tls.crt` (certificate and chain) and `tls.key` (**private key**) | Updated at every renewal |

When something goes wrong, follow the chain from the top: `kubectl describe` the `Certificate`, then
the `CertificateRequest`, the `Order` and the `Challenge`. The first one with an error in its
**Events** or **Status** tells you which step failed. Do **not** delete and re-create the Ingress or
the Secret to "try again": Let's Encrypt issues at most 5 certificates a week for the same names
(the tip at the end of Day 3, [section 4.7.1](../day03/04-exposing-an-application.md)), and everybody
on the training cluster shares one ACME account.

:::{note} No `Challenge` at all? Let's Encrypt remembers
After a successful check, Let's Encrypt remembers for a limited time that this ACME account controls
the name. A new order for the same name in that time is valid at once: no `Challenge`, no solver,
and the certificate is ready in a few seconds. The `Order` shows it in its status, with
`initialState: valid`. On the training cluster everybody uses the same ACME account, the one of the
`ClusterIssuer`, so this happens whenever a name was checked recently: for example, when you apply
an Ingress for a host that you already used this week.
:::

:::{important} The private key never leaves the cluster
cert-manager creates the key pair in the cluster and sends Let's Encrypt only the CSR, which contains
the **public** key. Let's Encrypt never has your private key, so it cannot pretend to be your
service. The only copy is in the `Secret`: who can read Secrets in your namespace can.
:::

### HTTP-01 and DNS-01

There are two common ways to prove that you control a name:

| | HTTP-01 (training cluster) | DNS-01 |
|---|---|---|
| Proof | A file at `http://<name>/.well-known/acme-challenge/<token>` | A `TXT` record `_acme-challenge.<name>` in DNS |
| Needs | The name resolves to your Ingress controller, and **port 80 is open to the internet** | Write access to the DNS zone, through the API of the DNS provider |
| Wildcards (`*.example.org`) | No | Yes |
| Clusters that are not reachable from the internet | No | Yes |

This explains a detail of section 2.3: your Ingress answers only on port 443, but the challenge
uses port 80. That still works, because the challenge goes to cert-manager's **temporary** Ingress,
which has no `websecure`-only annotation. Closing port 80 on the whole Ingress controller, or in a
security group of the cloud, would break HTTP-01.

### Lifetime and renewal

Let's Encrypt certificates are valid for 90 days today. The whole industry is going to shorter
lifetimes: from 2029 no public certificate may be valid for more than 47 days. Short lifetimes limit
the damage of a stolen key, but they make one rule absolute: **renewal must be automatic**.

By default cert-manager renews a certificate when two thirds of its lifetime have passed: for a
90-day certificate, 30 days before it expires. It goes through the same steps again, with a new
`CertificateRequest`, `Order` and `Challenge`, and replaces the content of the Secret. Traefik
notices the new Secret and uses the new certificate, with no restart. You can read both dates in the
status of the `Certificate`: `notAfter` and `renewalTime`.

(d5-ho4)=
## 2.5 ⌨️ Hands-on 4: HTTPS for `climate-stats-api`, step by step

You publish `climate-stats-api` on HTTPS only, follow cert-manager while it gets the certificate, and
check the result from the client side. In every file and command below, replace `traineeNN` with
your account.

### Step 0: start the stack again

Section 1 ended by removing the stack. The files are still in `~/climate-stack-k8s`. Apply them
again, in this order: what the database needs, the database, then Redis and the API.

```bash
cd ~/climate-stack-k8s
kubectl apply -f db-secret.yaml -f db-pvc.yaml -f db-init-configmap.yaml
kubectl apply -f db.yaml
kubectl rollout status deployment/climate-db
kubectl apply -f cache.yaml -f api.yaml
kubectl rollout status deployment/climate-api
kubectl get deploy climate-api climate-db climate-cache
```

```text
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
climate-api     2/2     2            2           40s
climate-db      1/1     1            1           1m
climate-cache   1/1     1            1           40s
```

:::{dropdown} The files are missing?
Create `~/climate-stack-k8s` and these six files (they come from Day 3, sections 9 and 10, and
section 1 of today), then run the commands above.

`db-secret.yaml`: the database credentials. In a real project this file is not in Git; it is here
only to keep the exercise short.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: climate-db
type: Opaque
stringData:
  POSTGRES_DB: climate_db
  POSTGRES_USER: ecmwf_admin
  POSTGRES_PASSWORD: EcmwfPassphrase123!
```

`db-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: climate-db-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

`db-init-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: climate-db-init
data:
  init.sql: |
    CREATE TABLE IF NOT EXISTS climate_telemetry (
      id SERIAL PRIMARY KEY,
      station_id VARCHAR(50) NOT NULL,
      temperature_celsius NUMERIC(6, 2) NOT NULL,
      timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    INSERT INTO climate_telemetry (station_id, temperature_celsius) VALUES
    ('STATION-ATHENS-01', 37.4),
    ('STATION-BOLOGNA-02', 29.8),
    ('STATION-VALENCIA-03', 34.1);
```

`db.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: climate-db
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: climate-db
  template:
    metadata:
      labels:
        app: climate-db
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          envFrom:
            - secretRef:
                name: climate-db
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - name: postgres
              containerPort: 5432
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "ecmwf_admin", "-d", "climate_db"]
            periodSeconds: 5
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: init
              mountPath: /docker-entrypoint-initdb.d
              readOnly: true
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: climate-db-data
        - name: init
          configMap:
            name: climate-db-init
---
apiVersion: v1
kind: Service
metadata:
  name: climate-db
spec:
  selector:
    app: climate-db
  ports:
    - name: postgres
      port: 5432
      targetPort: postgres
```

`cache.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: climate-cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: climate-cache
  template:
    metadata:
      labels:
        app: climate-cache
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - name: redis
              containerPort: 6379
          readinessProbe:
            tcpSocket:
              port: redis
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: climate-cache
spec:
  selector:
    app: climate-cache
  ports:
    - name: redis
      port: 6379
      targetPort: redis
```

`api.yaml`: version 1.1.0 is the one of section 1. If you never pushed it, write `1.0.0`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: climate-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: climate-api
  template:
    metadata:
      labels:
        app: climate-api
    spec:
      imagePullSecrets:
        - name: gitlab-registry
      containers:
        - name: app
          image: gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN/climate-stats-api:1.1.0
          env:
            - name: DB_HOST
              value: climate-db
            - name: REDIS_HOST
              value: climate-cache
          envFrom:
            - secretRef:
                name: climate-db
          ports:
            - name: http
              containerPort: 8000
          readinessProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 5
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: climate-api
  labels:
    app: climate-api
spec:
  selector:
    app: climate-api
  ports:
    - name: http
      port: 80
      targetPort: http
```
:::

:::{dropdown} `climate-api` in `ImagePullBackOff`?
The pods cannot pull the image. Check that the tag in `api.yaml` exists in your registry (GitLab:
your project `climate-telemetry-traineeNN`, **Deploy → Container Registry**), and that the Secret
`gitlab-registry` exists: `kubectl get secret gitlab-registry`. If it is missing, create a deploy
token of `climate-telemetry-traineeNN`: in GitLab, *Settings → Repository → Deploy tokens → Add
token*, name `cluster-pull`, scope **`read_registry`** only. Then:

```bash
read -rp  "Deploy token username: " DT_USER
read -rsp "Deploy token: " DT_TOKEN; echo
kubectl create secret docker-registry gitlab-registry \
  --docker-server=gitlab.data.meditwin-project.eu:6443 \
  --docker-username="$DT_USER" \
  --docker-password="$DT_TOKEN"
unset DT_TOKEN
```
:::

### Step 1: the Ingress

In `~/climate-stack-k8s`, create:

`ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: climate-api
  annotations:
    cert-manager.io/cluster-issuer: ecmwf-meditwin-training-issuer
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - stats-traineeNN.training.meditwin-project.eu
      secretName: climate-api-tls
  rules:
    - host: stats-traineeNN.training.meditwin-project.eu
      http:
        paths:
          - path: /health
            pathType: Exact
            backend:
              service:
                name: climate-api
                port:
                  number: 80
          - path: /stats
            pathType: Exact
            backend:
              service:
                name: climate-api
                port:
                  number: 80
```

| Part | What it does |
|---|---|
| `cert-manager.io/cluster-issuer` | cert-manager creates a `Certificate` for the hosts in `tls`, with the training cluster's Let's Encrypt issuer |
| `tls.secretName` | The Secret where cert-manager writes the certificate and the private key, and where Traefik reads them. The `Certificate` gets the same name |
| `router.entrypoints: websecure` | HTTPS only: port 80 answers `404` (section 2.3) |
| Two `Exact` paths | Only `/health` and `/stats` are published; `/metrics` is not (section 2.1) |

### Step 2: watch cert-manager work

Open a **second terminal** on the VM, and start a view that refreshes every two seconds:

```bash
watch -n 2 kubectl get certificate,certificaterequest,orders.acme.cert-manager.io,challenges.acme.cert-manager.io,ingress,pods
```

In the first terminal, apply the Ingress:

```bash
cd ~/climate-stack-k8s
kubectl apply -f ingress.yaml
```

In the second terminal, objects appear one after the other. After about 20 seconds you see something
like this (the names end with different numbers, and some columns are left out):

```text
NAME                                          READY   SECRET            AGE
certificate.cert-manager.io/climate-api-tls   False   climate-api-tls   20s

NAME                                                   APPROVED   DENIED   READY   ISSUER                           AGE
certificaterequest.cert-manager.io/climate-api-tls-1   True                False   ecmwf-meditwin-training-issuer   19s

NAME                                                      STATE     AGE
order.acme.cert-manager.io/climate-api-tls-1-2145812375   pending   19s

NAME                                                                     STATE     DOMAIN                                         AGE
challenge.acme.cert-manager.io/climate-api-tls-1-2145812375-1462938204   pending   stats-traineeNN.training.meditwin-project.eu   18s

NAME                                                  CLASS     HOSTS                                          ADDRESS   PORTS     AGE
ingress.networking.k8s.io/climate-api                 traefik   stats-traineeNN.training.meditwin-project.eu   ...       80, 443   20s
ingress.networking.k8s.io/cm-acme-http-solver-bx9wd   <none>    stats-traineeNN.training.meditwin-project.eu   ...       80        17s

NAME                              READY   STATUS    RESTARTS   AGE
pod/climate-api-6f9c7d8b5-2xkqp   1/1     Running   0          10m
pod/climate-api-6f9c7d8b5-tq8lm   1/1     Running   0          10m
pod/cm-acme-http-solver-7xk2p     1/1     Running   0          17s
...
```

These are the objects of section 2.4, in the order of the ACME conversation. Two of them are not
yours: the solver pod and its Ingress, which serve the token on port 80. Note that the solver's
Ingress shows port `80` only.

After one to three minutes, the `Challenge`, the solver pod and the solver Ingress disappear, the
`Order` becomes `valid`, and the `Certificate` becomes `READY True`. Stop `watch` with **Ctrl+C**.

`READY True` after a few seconds, with no `Challenge` at all? Then your name was checked recently, and
Let's Encrypt reused that check (section 2.4, *No `Challenge` at all?*). Nothing is wrong: continue
with step 3.

### Step 3: follow the chain backwards

The `Certificate` tells the story in its events:

```bash
kubectl describe certificate climate-api-tls | sed -n '/^Events/,$p'
```

```text
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    3m    cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  3m    cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "climate-api-tls-slm8q"
  Normal  Requested  3m    cert-manager-certificates-request-manager  Created new CertificateRequest resource "climate-api-tls-1"
  Normal  Issuing    1m    cert-manager-certificates-issuing          The certificate has been successfully issued
```

First the private key, then the request. Look at what cert-manager sent to Let's Encrypt, the CSR in
the `CertificateRequest`:

```bash
kubectl get certificaterequest climate-api-tls-1 -o jsonpath='{.spec.request}' | base64 -d \
  | openssl req -noout -text | grep -E 'Public-Key|DNS:'
```

```text
                Public-Key: (2048 bit)
                    DNS:stats-traineeNN.training.meditwin-project.eu
```

A name and a public key: nothing secret. Now the result, in the Secret:

```bash
kubectl get secret climate-api-tls -o yaml | less
kubectl get secret climate-api-tls -o jsonpath='{.data.tls\.crt}' | base64 -d \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

```text
subject=CN=stats-traineeNN.training.meditwin-project.eu
issuer=C=US, O=Let's Encrypt, CN=YR1
notBefore=Nov  5 11:12:08 2026 GMT
notAfter=Feb  3 11:12:07 2027 GMT
X509v3 Subject Alternative Name:
    DNS:stats-traineeNN.training.meditwin-project.eu
```

`tls.crt` has three certificates: yours, the intermediate `YR1`, and `Root YR` cross-signed by
`ISRG Root X1` (section 2.3). `tls.key`, which the first command also shows, is the private key,
only base64-encoded. Never copy it, paste it or show it on a shared screen: anybody who has it can
pretend to be your service until the certificate expires.

Last, when will it be renewed?

```bash
kubectl get certificate climate-api-tls \
  -o jsonpath='notAfter:    {.status.notAfter}{"\n"}renewalTime: {.status.renewalTime}{"\n"}'
```

```text
notAfter:    2027-02-03T11:12:07Z
renewalTime: 2027-01-04T11:12:07Z
```

### Step 4: the client's view

Now look from the outside, as a client does. `curl -v` prints the handshake:

```bash
HOST=stats-traineeNN.training.meditwin-project.eu
curl -sv -o /dev/null https://$HOST/health 2>&1 | grep -E 'SSL connection|subject:|expire date|issuer:|verify ok'
```

```text
* SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
*  subject: CN=stats-traineeNN.training.meditwin-project.eu
*  expire date: Feb  3 11:12:07 2027 GMT
*  issuer: C=US; O=Let's Encrypt; CN=YR1
*  SSL certificate verify ok.
```

TLS 1.3, the certificate of step 3, and **verify ok**: the chain ends at a root that the VM trusts,
and the name matches. Run the same command without the `grep` to see every message of the handshake
of section 2.3: *Client hello*, *Server hello*, *Certificate*, *CERT verify*, *Finished*.

Last, the HTTP side, and the paths:

```bash
curl -s http://$HOST/health   # port 80
curl -s https://$HOST/metrics          # not a published path
curl -s https://$HOST/stats
```

```text
404 page not found
404 page not found
{"cache_access_count":1,"database_record_count":3,"service_state":"synchronized"}
```

`/stats` answered `200` to anybody on the internet, without a password. Hands-on 5 closes it.

**Checkpoint:** `kubectl get certificate climate-api-tls` shows `READY True`; `curl -v` shows
`verify ok`; port 80 and `/metrics` answer `404`.

## 2.6 HTTP Basic Auth

HTTP Basic Auth is the simplest authentication scheme in HTTP, and every client supports it: `curl`,
Python `requests`, browsers. It is a reasonable first step for a small machine-to-machine API.

### How it works

```{mermaid}
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: GET /stats
    S-->>C: 401 Unauthorized<br/>WWW-Authenticate: Basic realm="climate"
    Note over C: base64("alice:s3cret")<br/>= YWxpY2U6czNjcmV0
    C->>S: GET /stats<br/>Authorization: Basic YWxpY2U6czNjcmV0
    S-->>C: 200 OK
```

1. The server answers an unauthenticated request with `401` and a `WWW-Authenticate: Basic` header,
   which says "I accept Basic Auth".
2. The client joins user name and password with a colon, encodes them with base64, and sends them in
   the `Authorization` header.
3. The client sends that header again with **every** request. There is no session and no login step.

:::{danger} The password is in every request, and base64 is not encryption
Base64 is an encoding with no key. Anybody who sees the header has the password:

```bash
echo 'YWxpY2U6czNjcmV0' | base64 -d
```
```text
alice:s3cret
```

Basic Auth is acceptable **only over HTTPS**. Over plain HTTP it is equal to sending the password in
clear text. Over HTTPS, the header is inside the encrypted part (section 2.3).
:::

### Where to put it: at the edge or in the application

You can check the credential in two places.

```{mermaid}
flowchart LR
    C([Client]) --> T["Traefik<br/>(A) basicAuth middleware"]
    T --> P["Pod<br/>(B) check in the code"]
```

| | (A) At the edge: Traefik middleware | (B) In the application |
|---|---|---|
| Code changes | None | A few lines, in every application |
| Who sets it up | Whoever writes the manifests | The developers of each application |
| Where the users are stored | A bcrypt hash of each password, in a `Secret` that Traefik reads | Usually the password itself, in the pod's environment |
| The password reaches the application | No (Traefik can remove the header) | Yes |
| Per-endpoint rules | By path, with one Ingress per rule set. No roles | Any rule you can write in code |
| Callers inside the cluster | **Not** checked; they bypass Traefik | Checked |
| Works without the cluster (Docker Compose, tests) | No | Yes |

Hands-on 5 uses **(A)**. The application gets no authentication code and never sees the password,
and the same middleware can protect any other service, whatever its language. It is a platform
control: the people who run the cluster can add it to a service they did not write.

### Traefik middlewares

A **middleware** is a step that Traefik runs on a request **after** it has chosen the route and
**before** it forwards the request to your Service. Each request goes through the same stages:

```{mermaid}
flowchart LR
    C([Client]) --> E["Entrypoint<br/>web :80 or websecure :443"]
    E --> R["Router<br/>host + path,<br/>from your Ingress"]
    R --> M["Middlewares,<br/>in order"]
    M --> S[Service climate-api]
    M -.->|"answers itself:<br/>301, 401, 403, 429"| C
```

A middleware can change the request (add or remove headers), or answer it itself without calling the
Service (a redirect, `401`, `403`, `429 Too Many Requests`). You used one on Day 3
([section 4.7.1](../day03/04-exposing-an-application.md)): `redirectScheme`, which answers `301`.

| Middleware | Does | Where |
|---|---|---|
| `redirectScheme` | Redirects `http://` to `https://` | Day 3 |
| `basicAuth` | Asks for a user name and password; `401` without them | Hands-on 5 |
| `rateLimit` | Limits the requests of each client; `429` above the limit | Section 2.9 |
| `ipAllowList` | Accepts only some source addresses; `403` for the others | Section 2.9 |
| `headers` | Adds response headers, such as HSTS | Section 2.9 |
| `forwardAuth` | Asks another service (for example `oauth2-proxy`) whether the request may pass | Section 2.8 |

A `Middleware` is an object of your namespace (`apiVersion: traefik.io/v1alpha1`, a CRD added by
Traefik), so you create it with `kubectl apply` like any other. An Ingress uses it through the
annotation `traefik.ingress.kubernetes.io/router.middlewares`, and the middleware then applies to
**every** path of that Ingress. 

(d5-ho5)=
## 2.7 ⌨️ Hands-on 5: Basic Auth in Traefik

You put a password on `/stats`, one for each of two clients, without changing the application. Then
you remove one client, and you look at the limit of a check at the edge.

### Step 1: create the Secret with the users

Traefik's `basicAuth` middleware reads its users from a Secret, as `htpasswd` lines: the user name and
a **bcrypt hash** of the password, never the password itself. Each client gets its own user and
password, so that you can remove one without changing the others.

Generate a long random password for each client, keep them in shell variables, and make the lines
with the `htpasswd` command. It is in the package `httpd-tools`, which the first command installs:

```bash
sudo yum install -y httpd-tools
cd ~/climate-stack-k8s
DASHBOARD_PASSWORD=$(python3 -c 'import secrets; print(secrets.token_urlsafe(32))')
REPORT_PASSWORD=$(python3 -c 'import secrets; print(secrets.token_urlsafe(32))')
htpasswd -nbB dashboard "$DASHBOARD_PASSWORD" | grep . > users
htpasswd -nbB report "$REPORT_PASSWORD" | grep . >> users
cat users
kubectl create secret generic climate-api-users --from-file=users --save-config
```

```text
dashboard:$2y$05$...
report:$2y$05$...
```

`--save-config` lets you change the Secret later with `kubectl apply`, for example to remove a
client, without a warning. Anybody who reads the Secret gets the hashes, which are
useless for logging in; and a bcrypt hash is slow to compute, on purpose, so guessing a password
from it is slow too.

### Step 2: the middleware, and a second Ingress

Create the middleware (section 2.6). It names the Secret of step 1, but contains no credential, so it
is safe to commit:

`edge-auth.yaml`

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: climate-api-auth
spec:
  basicAuth:
    secret: climate-api-users
    realm: climate
    removeHeader: true
```

A middleware attached to an Ingress applies to **every** path of that Ingress, and `/health` must
stay open. So the API gets two Ingress objects, for the same host: one for the open path, one for the
protected path. In `ingress.yaml`, **remove** the `/stats` path, so that only `/health` is left:

`ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: climate-api
  annotations:
    cert-manager.io/cluster-issuer: ecmwf-meditwin-training-issuer
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - stats-traineeNN.training.meditwin-project.eu
      secretName: climate-api-tls
  rules:
    - host: stats-traineeNN.training.meditwin-project.eu
      http:
        paths:
          - path: /health
            pathType: Exact
            backend:
              service:
                name: climate-api
                port:
                  number: 80
```

And create the second one:

`ingress-protected.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: climate-api-protected
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.middlewares: traineeNN-climate-api-auth@kubernetescrd
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - stats-traineeNN.training.meditwin-project.eu
      secretName: climate-api-tls
  rules:
    - host: stats-traineeNN.training.meditwin-project.eu
      http:
        paths:
          - path: /stats
            pathType: Exact
            backend:
              service:
                name: climate-api
                port:
                  number: 80
```

| Part | What it does |
|---|---|
| `basicAuth: secret` | Traefik answers `401` to a request without a valid user name and password |
| `realm: climate` | The name the client sees in the `WWW-Authenticate` header. Without it, the client sees `traefik` |
| `removeHeader: true` | Traefik removes the `Authorization` header after the check. The password never reaches the pod, so it cannot end up in its logs or error reports |
| `router.middlewares` | Attaches the middleware to every path of this Ingress. The value is `<namespace>-<name>@kubernetescrd` (Day 3, [section 4.7.1](../day03/04-exposing-an-application.md)) |
| No `cert-manager.io/cluster-issuer` in `ingress-protected.yaml` | It uses the Secret `climate-api-tls` of Hands-on 4, for the same host. Only `ingress.yaml` asks for it: if both asked, they would both try to manage it |
| `router.entrypoints: websecure` | HTTPS only, on both Ingress objects |

### Step 3: deploy and test

Apply the middleware first, then close the open path, then open the protected one. In this order,
`/stats` is never published without the middleware:

```bash
kubectl apply -f edge-auth.yaml
kubectl apply -f ingress.yaml
kubectl apply -f ingress-protected.yaml
```

No new certificate is needed: `kubectl get certificate` still shows the one of Hands-on 4. Now test:

```bash
STATS=https://stats-traineeNN.training.meditwin-project.eu
curl -s $STATS/health
curl -s $STATS/stats
curl -s -u dashboard:wrong $STATS/stats
curl -s -u "dashboard:$DASHBOARD_PASSWORD" $STATS/stats
curl -s -u "report:$REPORT_PASSWORD" $STATS/stats
curl -si $STATS/stats | grep -i www-authenticate
```

| Request | Answer |
|---|---|
| `GET /health` | `200`, `{"status":"healthy"}`: no password needed |
| `GET /stats` without credentials | `401`, with the body `401 Unauthorized`: Traefik's answer, not the application's |
| `GET /stats` with a wrong password | `401` |
| `GET /stats` as `dashboard` | `200`, with the statistics |
| `GET /stats` as `report` | `200`, with the statistics |
| The `WWW-Authenticate` header | `Basic realm="climate"`: the server asks for Basic Auth |

### Step 4: the door that Traefik does not guard

Everything above went through Traefik. Now call the API from **another pod** of your namespace, the
Redis pod, through the Service `climate-api`. The Redis image has `wget`:

```bash
kubectl exec deployment/climate-cache -- wget -qO- http://climate-api/stats
kubectl exec deployment/climate-cache -- wget -qO- http://climate-api/metrics | head -n 3
```

```text
{"cache_access_count":7,"database_record_count":3,"service_state":"synchronized"}
# HELP python_gc_objects_collected_total Objects collected during gc
# TYPE python_gc_objects_collected_total counter
python_gc_objects_collected_total{generation="0"} 312.0
```

No password, and every path, because the request never passed through Traefik. That is the limit of a
check at the edge. For `/metrics` it is what you want: the monitoring system reads it
from inside the cluster. 

**Checkpoint:** over HTTPS, `/stats` answers `401` without a password and `200` with the password of
each client; `/health` answers without a password; from another pod, `/stats` answers without one.

(d5-oidc)=
## 2.8 From passwords to tokens: OpenID Connect

Basic Auth works, and with one password for each client it is manageable for a few clients. But
look at its limits:

| Limitation of Basic Auth | Consequence |
|---|---|
| The password is in every request | Each request is a chance to leak it: logs, proxies, error reports |
| It does not expire | A leaked password works until somebody changes it |
| Whoever checks it stores the users | Every middleware, or every service, that checks a password needs its own copy of the users |
| No roles | Everybody who has a password may do everything on the paths it protects |
| No single sign-on | A person with ten services has ten passwords |

The usual answer is to let a separate service, an **identity provider**, check the credentials, and
give the client a short-lived, signed **token**. The API never sees a password: it only checks the
token. This part is theory: you do not set up an identity provider. 

:::{note} Jargon: *identity provider*, *OAuth 2.0*, *OpenID Connect*
An **identity provider** (IdP) is a service that authenticates users and workloads and issues tokens:
Keycloak, Dex, Microsoft Entra ID, Google, GitLab. **OAuth 2.0** is the standard for how a client
gets an access token. **OpenID Connect** (OIDC) adds identity on top of it: standard user
information, and a standard discovery address, `/.well-known/openid-configuration`, where an API
finds the provider's settings and public keys. The "Sign in with Google" or "Sign in with GitLab"
buttons on websites are OIDC.
:::

### How a workload gets a token

For a workload, the simplest flow is **client credentials**. The client has an ID and a secret,
registered in the identity provider. It exchanges them for a token, then sends the token to the API:

```{mermaid}
sequenceDiagram
    participant C as Client (dashboard)
    participant I as Identity provider
    participant A as API
    C->>I: POST /token<br/>client_id + client_secret
    I-->>C: access token (JWT, signed, valid 5 min)
    C->>A: GET /stats<br/>Authorization: Bearer eyJhbGciOi…
    A->>I: GET /keys (public keys, once, then cached)
    Note over A: 1. signature, issuer, audience, expiry<br/>2. role "reader"?
    A-->>C: 200 OK
```

The client still has a secret, but it sends it only to the identity provider, and only once every
few minutes. The API receives a token that stops working after a few minutes.

People use a different flow, the **authorization code** flow: the browser is sent to the identity
provider's login page (with single sign-on and a second factor), and comes back with a code that the
application exchanges for tokens.

### What is in a token

Most access tokens are **JSON Web Tokens** (JWT): three base64url parts separated by dots,
`header.payload.signature`. The payload is a set of **claims**:

```json
{
  "iss": "https://idp.example.org",
  "sub": "dashboard",
  "aud": ["climate-api"],
  "exp": 1793803071,
  "groups": ["reader"]
}
```

| Claim | Means | The API checks it, because… |
|---|---|---|
| `iss` (issuer) | Which identity provider made the token | …a token from another provider, even a valid one, must not be accepted |
| `sub` (subject) | Who the token is about: a user or a client | …it is the identity, for authorisation and logs |
| `aud` (audience) | Which APIs the token is for | …a token made for another service must not be replayed against this one |
| `exp` (expiry) | When the token stops being valid (seconds since 1970) | …a stolen token must stop working soon |
| `groups`, `roles`, `scope` | What the client may do. The name depends on the provider | Authorisation: `reader` may read, `editor` may change |

:::{danger} A JWT is signed, not encrypted
Anybody who has the token can read the payload: it is only base64url. Never put a secret in a
claim. What the signature gives is **integrity**: if anybody changes one
character of the payload, the signature no longer matches, and the API refuses the token.
:::

The token is also a **bearer** token: whoever holds it can use it, like a ticket. So HTTPS is still
required, and tokens must not be logged.

The API checks the signature with the identity provider's **public** key. Only the identity
provider has the private key, so only it can make valid tokens. The API needs no secret at all.
This is the same idea as the certificate chain of section 2.3: a private key signs, everybody can
check with the public key.

### Where you meet OIDC as a cloud user

You do not need to write an identity provider to use OIDC every day:

| Where | The identity provider | The token |
|---|---|---|
| Every pod in Kubernetes | The Kubernetes API server | The **service account token**, mounted in every pod in `/var/run/secrets/kubernetes.io/serviceaccount/`, is a JWT. Programs in the pod use it to call the Kubernetes API |
| GitLab CI jobs | GitLab | `id_tokens` in `.gitlab-ci.yml` give a job a JWT that says which project, branch and pipeline it is. A cloud provider can accept it **instead of** a stored access key (*workload identity federation*) |
| `kubectl` on many clusters | The organisation's IdP (Keycloak, Entra ID, …) | You log in with a browser; `kubectl` sends the token to the API server |
| Web applications behind single sign-on | The organisation's IdP | `oauth2-proxy`, or a Traefik `forwardAuth` middleware that calls it, adds the login **in front of** an application, with no code change: the same idea as Hands-on 5, with tokens instead of passwords |

The direction is always the same: **short-lived credentials are better than stored ones**.

### 📖 References

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) and
  [OAuth 2.0, RFC 6749](https://www.rfc-editor.org/rfc/rfc6749) (the client credentials flow is
  section 4.4)
- [JSON Web Token, RFC 7519](https://www.rfc-editor.org/rfc/rfc7519), and the
  [JWT best current practices, RFC 8725](https://www.rfc-editor.org/rfc/rfc8725)
- [OAuth 2.0 for browser-based applications](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps)
- Kubernetes: [service account tokens](https://kubernetes.io/docs/concepts/security/service-accounts/)
  and [service account issuer discovery](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-issuer-discovery)
- GitLab: [OpenID Connect in GitLab CI/CD](https://docs.gitlab.com/ci/secrets/id_token_authentication/)
- [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/), and Traefik's
  [ForwardAuth middleware](https://doc.traefik.io/traefik/middlewares/http/forwardauth/)
- Identity providers you can run yourself: [Dex](https://dexidp.io) (small, configured with one
  file) and [Keycloak](https://www.keycloak.org) (full-featured, with an admin console)
- [jwt.io](https://jwt.io) decodes tokens in the browser. Use it only with test tokens: never paste a
  real token into a website

(d5-beyond)=
## 2.9 📖 Beyond this section

| Method | How it works | Use it when |
|---|---|---|
| **Basic Auth** | User name and password in every request | Simple internal APIs, behind HTTPS |
| **API key** | A long random key in a header, one for each client | Machine clients, when the application checks the key itself |
| **OAuth 2.0 / OpenID Connect** | An identity provider gives short-lived signed tokens; the API checks the signature | Many services, many clients, roles; human users with single sign-on |
| **Mutual TLS (mTLS)** | The client also shows a certificate | Service-to-service traffic, often with a service mesh |

More controls that Traefik and Kubernetes give you without a change in the application:

- **Traffic inside the cluster.** By default, every pod can connect to every other pod. A
  Kubernetes `NetworkPolicy` can restrict which pods may connect to yours, when the cluster's network
  plugin enforces it. Allow only Traefik, and nobody can go around the middleware of Hands-on 5.
- **Rate limits.** The Traefik `rateLimit` middleware limits the number of requests from each
  client, for example 10 a second. It protects the database behind `/stats` from a loop.
- **Network allow lists.** The Traefik `ipAllowList` middleware accepts only some source addresses,
  and answers `403` to the others: authorisation by network, not by identity. It needs Traefik to see
  the real address of the client, which depends on how the load balancer in front of it is set up.
- **Login at the edge.** `oauth2-proxy`, alone or behind a Traefik `forwardAuth` middleware, sends
  people to the organisation's identity provider and lets only logged-in users through (section 2.8).
- **HSTS.** For websites, the `Strict-Transport-Security` header (Traefik `headers` middleware) tells
  the browser to use only HTTPS for this host in the future, so that even the first request after a
  typed `http://` goes to HTTPS.
- **DNS-01 and wildcard certificates.** For clusters that are not reachable from the internet, or for
  `*.example.org`, cert-manager can prove control of a name in DNS instead of HTTP (section 2.4).
- **Your own CA.** For traffic that stays inside an organisation, cert-manager can issue certificates
  from a private CA (`CA` or `Vault` issuer), with no rate limit, for clients that trust that CA.
- **Certificate Transparency.** Every certificate from a public CA is written in public logs, which
  anybody can search, for example on [crt.sh](https://crt.sh). Your host names become public within
  minutes of the certificate, so never put anything confidential in a host name.

Limit what a pod can do, if a bug gives an attacker a shell in it:

- **No Kubernetes credential.** Kubernetes mounts a service account token in every pod. Most
  applications never call the Kubernetes API: `automountServiceAccountToken: false` in the pod
  template removes the token. A program that does need the API gets its own `ServiceAccount` and a
  `Role` with only the verbs it needs; see
  [service accounts](https://kubernetes.io/docs/concepts/security/service-accounts/) and
  [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).
- **A security context.** `runAsNonRoot`, `allowPrivilegeEscalation: false`,
  `readOnlyRootFilesystem: true`, `capabilities.drop: ["ALL"]` and a `RuntimeDefault` seccomp profile
  change nothing in the application. The
  [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
  group them into levels; write your manifests to the **restricted** level. The group project
  (section 3) uses all of them.

## 2.10 Core takeaway

> **Authentication asks who you are; authorisation asks what you may do. You need both, and HTTPS
> under them.**

- `401` means *not authenticated*. `403` means *authenticated, but not allowed*. The code tells you
  which half to debug.
- A key pair: the **private** key signs and never travels; anybody checks with the **public** key.
  Certificates, the TLS handshake, ACME and tokens all rest on this.
- HTTPS proves the identity of the **server** and protects the traffic. It says nothing about the
  client. The host name (SNI) is still visible on the network.
- A certificate is trusted because a chain of signatures leads to a root that the client already
  has. The private key is the one secret: it stays in the TLS Secret.
- cert-manager does each step of ACME as an object: `Certificate`, `CertificateRequest`, `Order`,
  `Challenge`. When a certificate is not ready, follow them from the top, and read the events.
- HTTP-01 needs port 80 open from the internet. Certificates are renewed automatically, and short
  lifetimes are the future.
- Do not delete and re-create certificates to "try again": Let's Encrypt limits how many it issues.
- TLS ends at Traefik. Traffic inside the cluster does not pass Traefik's checks.
- A Traefik middleware is a step between the router and your Service: it can change a request, or
  answer it itself. It is an object of your namespace, attached to an Ingress with an annotation.
- Basic Auth is base64, not encryption: use it **only** over HTTPS. A Traefik middleware checks it
  with no code change, keeps only bcrypt hashes, and protects only the requests that pass through
  Traefik. One password for each client.
- Publish only the paths you mean to publish, and serve APIs on HTTPS only.
- A token from an identity provider is signed, short-lived and carries roles. Check its signature,
  issuer, audience and expiry, then its roles. It is signed, not encrypted. Your pod already has
  one.

## 2.11 📖 Check yourself

1. A teammate says: "Our API uses Basic Auth, so we do not need HTTPS: the password is encoded." What
   do you answer?
2. Over HTTPS, what can somebody on the same Wi-Fi network still learn about your call to
   `https://stats-traineeNN.training.meditwin-project.eu/stats`, and what can they not?
3. You apply a new Ingress. Ten minutes later, `kubectl get certificate` still shows `READY False`.
   What do you look at, in which order? What must you **not** do?
4. Does Let's Encrypt have the private key of your certificate? Who can pretend to be your service?
5. Your API returns `403` to a client. Is the password wrong?
6. As in Hands-on 5, you protect your service with the Traefik `basicAuth` middleware only. Another
   pod in the cluster calls `http://climate-api/stats`. What happens? How could you prevent it?
7. Anybody who has a JWT can read its payload. Is that a problem? Why must an API that accepts JWTs
   check the `aud` claim?

:::{dropdown} Solutions
**1.** Encoding is not encryption. Base64 has no key: `echo … | base64 -d` gives the password back
at once. Without HTTPS, everybody on the network path (a Wi-Fi network, a proxy, a router) reads the
password in every request. Basic Auth without HTTPS is equal to no protection, and also leaks a
password that may work in other places.

**2.** They see the DNS query and the SNI, so the host name `stats-traineeNN…`, and the IP address,
the time and the amount of data. They do not see the path `/stats`, the headers (with the
`Authorization` header), or the answer: all of that is inside the encrypted part.

**3.** Follow the chain of objects from the top, with `kubectl describe`, and read the events and the
status of each: `Certificate`, then `CertificateRequest`, then `Order`, then `Challenge`. The first
one with an error tells you which step failed. The usual causes are a typo in the host, a DNS record
that is missing or wrong, port 80 not reachable from the internet, or a rate limit. Do **not** delete
and re-create the Certificate, the Secret or the Ingress to try again: each attempt counts against
the Let's Encrypt rate limits, and only time fixes those.

**4.** No. cert-manager creates the key pair in the cluster and sends Let's Encrypt only a CSR, with
the public key. The private key is only in the Secret `climate-api-tls`. Whoever can read that Secret
can pretend to be your service: everybody whom RBAC allows to `get` Secrets in your namespace, and
any pod whose service account has that permission.

**5.** Probably not. A wrong or missing password gives `401`. `403` means the service *recognised* the
client and decided that it may not do this. Look at the authorisation rule: the role of the client,
the path, the method. (Some badly written services return `403` for everything. That is a bug in
the service.)

**6.** The request does not go through Traefik, so the middleware does not see it. The pod answers
without a password check: you saw it in step 4 of Hands-on 5. Two ways to prevent it: a
`NetworkPolicy` that lets only Traefik connect to the pods, so that every request must pass the
middleware; or a check in the application too. On a cluster you share with others, you need one of
them.

**7.** Reading it is not a problem, as long as there is no secret in the claims: a JWT is signed, not
encrypted, by design. The danger is **using** it: it is a bearer token, so whoever copies it can call
the API as its subject until it expires. The `aud` check is there because
one identity provider issues tokens for many APIs. Without it, a token issued for *another* service
would be accepted here, and that other service, or anybody who stole a token from it, could replay it
against yours.
:::

---

Continue with: [3. Group project: package and serve a model](03-group-project.md).
