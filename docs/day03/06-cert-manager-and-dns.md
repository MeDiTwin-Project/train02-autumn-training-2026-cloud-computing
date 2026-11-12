# 6. 📖 cert-manager and DNS

After exposing your app with Service and Ingress, this section deepens **certificate automation** and **DNS correctness**. In Hands-on 4 both "just happened" after one annotation and one hostname; here is what did the work.

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

:::{important} A deeper look
This section is 📖: it explains what happened behind the scenes in Hands-on 4, and you can read it after the lesson. The commands only **read** objects that already exist: run them if you want to look inside, nothing in the rest of the day depends on them.
:::

## 6.1 Why cert-manager

Without cert-manager you manually create and rotate TLS certificates. That does not scale and is error-prone.

**cert-manager provides**:

- Automated issuance and renewal.
- Integration with ACME providers (for example Let's Encrypt).
- Native Kubernetes resources: **Issuer, ClusterIssuer, Certificate**.

:::{note} Jargon: *CA*, *ACME*, *HTTP-01 challenge*
A **certificate authority (CA)** signs certificates that browsers trust; Let's Encrypt is a free one. **ACME** is the protocol to ask a CA for a certificate automatically. Before it signs, the CA checks that you control the name: in the **HTTP-01 challenge**, cert-manager publishes a token at `http://<your-host>/.well-known/acme-challenge/…`, and the CA downloads it. If the CA can reach the token at that name, the name is yours.
:::

## 6.2 Core resources

- **Issuer**: namespaced certificate authority configuration.
- **ClusterIssuer**: cluster-wide issuer. The training cluster provides one, `ecmwf-meditwin-training-issuer`.
- **Certificate**: desired certificate for a DNS name. Its output is a plain TLS **Secret**, the one your Ingress references in `spec.tls[].secretName`.

In many setups you request certificates indirectly by annotating an Ingress resource, as you did in Hands-on 4. Look at what the annotation created on your behalf:

```bash
kubectl get clusterissuer
kubectl get certificate climate-telemetry-tls -o yaml
kubectl get secret climate-telemetry-tls
```

## 6.3 Typical HTTPS flow

The flow has two phases. The certificate is issued **once**, when you apply the Ingress (and renewed automatically before it expires); user requests come later and only use it.

```{mermaid}
sequenceDiagram
    participant K as You (kubectl apply)
    participant X as external-dns
    participant CM as cert-manager
    participant CA as ACME CA (Let's Encrypt)
    participant IC as Ingress controller
    participant D as DNS
    participant U as User

    Note over K,D: Phase 1 — issue the certificate (once)
    K->>X: Ingress with a host
    X->>D: Create the DNS record for the host
    K->>CM: Ingress with the cert-manager annotation
    CM->>CA: Order a certificate for the hostname
    CA->>D: Resolve the hostname
    CA->>IC: HTTP-01 challenge: GET http://host/.well-known/acme-challenge/...
    IC-->>CA: Token served by cert-manager's solver
    CA-->>CM: Signed certificate
    Note over CM,IC: cert-manager writes the Secret climate-telemetry-tls, and the Ingress controller reads it

    Note over D,U: Phase 2 — every user request
    U->>D: Resolve climate-traineeNN.training.meditwin-project.eu
    D-->>U: IP address of the Ingress controller
    U->>IC: HTTPS request
    IC-->>U: Response, encrypted with the certificate
```

The DNS record comes **first**: the CA must resolve the hostname to reach the challenge token on your Ingress controller. If the record is missing or wrong, the certificate cannot be issued.

## 6.4 DNS integration

At minimum you need:

- A DNS record (A or CNAME) pointing to the ingress endpoint.
- Correct hostname in the Ingress and Certificate resources.

On the training cluster the record is created by **external-dns**: it watches Ingress objects and writes one record per `spec.rules[].host` under `training.meditwin-project.eu`. That is why the hostname has to be under that zone, and why a typo in the host silently gives you a record for the wrong name — and then a failed ACME challenge, because the CA cannot reach your Ingress at the name you asked a certificate for.

**Validate quickly**:

```bash
kubectl get ingress
nslookup climate-traineeNN.training.meditwin-project.eu
kubectl get ingress climate-telemetry \
  -o jsonpath='{range .status.loadBalancer.ingress[*]}{.ip}{.hostname}{"\n"}{end}'
```

The address from `nslookup` must be the one in the Ingress status (the `ADDRESS` column of `kubectl get ingress` shows the same value).

:::{note} `status.loadBalancer` without a cloud load balancer
The field has this name for historical reasons, but it is the **Ingress controller** that fills it, with the address where it receives traffic: the IP of a cloud load balancer, the IPs of the nodes that run the controller, or an address set in its configuration. Some providers publish a DNS name instead of an IP (AWS, for example), so the command prints `.ip` or `.hostname`, whichever is there. If the output is empty, the controller does not publish an address: compare `nslookup` with the address that the platform team gives you.
:::

Elsewhere, without external-dns, you create that record by hand in your DNS provider; the rest of the flow is identical.

## 6.5 📖 Explicit issuer and certificate example

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ecmwf-meditwin-training-issuer
spec:
  acme:
    email: your-email@example.org
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: ecmwf-meditwin-training-issuer-account-key
    solvers:
    - http01:
        ingress:
          ingressClassName: traefik
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: climate-telemetry-tls
spec:
  secretName: climate-telemetry-tls
  issuerRef:
    name: ecmwf-meditwin-training-issuer
    kind: ClusterIssuer
  dnsNames:
  - climate-traineeNN.training.meditwin-project.eu
```

The ClusterIssuer is cluster-wide and already exists: you do not apply it (and you could not, in your namespace). It is shown so that you can read the one on the cluster with `kubectl get clusterissuer ecmwf-meditwin-training-issuer -o yaml`.

Use the explicit `Certificate` style when you want direct lifecycle control over certificate resources, or when the consumer is not an Ingress (section 5's Gateway, for example). If you prefer a simpler path, the Ingress annotation approach from Section 4 is usually enough.

## 6.6 Practical checks

```bash
kubectl describe certificate climate-telemetry-tls
kubectl describe challenge
kubectl get events --sort-by=.lastTimestamp
```

A Challenge exists only while a certificate is being issued: once the certificate is `READY True`, `kubectl describe challenge` prints `No resources found`, and that is normal.

**Common issues**:

- DNS not pointing to ingress endpoint (hostname outside the external-dns zone, or a typo).
- Wrong `ingressClassName`.
- Firewall/security group blocking HTTP-01 or HTTPS.
- Rate limits from repeated failed ACME attempts: fix the cause before re-applying in a loop.

## 6.7 Core takeaway

> **An annotation on the Ingress is enough: cert-manager obtains and renews the certificate, external-dns creates the DNS record.**

- A **ClusterIssuer** says *how* to get certificates (here: Let's Encrypt with the HTTP-01 challenge); a **Certificate** says *for which name*. Its result is a TLS **Secret**.
- The certificate is issued **once** when you apply the Ingress and renewed automatically; user requests only use it.
- HTTP-01 works only if the hostname already points to the Ingress controller: the DNS record comes first.
- When something fails: `describe certificate`, then `describe challenge`, then the events.

## 6.8 📖 Check yourself

1. You deleted and recreated your Ingress five times today, with the same hostname. What can go wrong, and how do you avoid it?
2. Why does a typo in the Ingress host break the certificate, and not only the DNS?
3. Where does the private key of your certificate live in the cluster?
4. When do you write a `Certificate` object yourself instead of the Ingress annotation?

:::{dropdown} Solutions
**1.** Let's Encrypt limits identical certificates for the same set of names (5 per week). Keep the TLS Secret when you delete the Ingress: cert-manager reuses it while it is valid.

**2.** external-dns creates the record for the wrong name, so the CA cannot reach your Ingress at the name in the certificate request, and the HTTP-01 challenge fails.

**3.** In the TLS Secret (here `climate-telemetry-tls`), next to the certificate. Who can read Secrets in your namespace can read the key.

**4.** When the consumer is not an Ingress (for example a Gateway listener, section 5), or when you want to manage the certificate lifecycle directly.
:::

Continue with: [7. Scaling](07-scaling.md).
