# 5. 📖 Exposing an application with Gateway API

Section 4 published `climate-telemetry` with an **Ingress**. This section shows the newer way to do the same thing, the **Gateway API**, and answers one question:

> **What changes when the entry point and the routing rules are separate objects?**

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

:::{important} Reference material
This whole section is 📖: you read it after the lesson. You do not run it on the training cluster. There, Ingress and Gateway API are **mutually exclusive**: only one mode is active at a time, and the cluster administrator switches between them. The day continues with Ingress.
:::

## 5.1 Ingress and Gateway API compared

| | Ingress (section 4) | Gateway API |
|---|---|---|
| Objects | One: the Ingress | Three: GatewayClass, Gateway, HTTPRoute |
| Who owns what | One object for everything | The **platform team** owns the Gateway (entry point, ports, certificates); each **application team** owns its HTTPRoutes |
| Controller-specific features | Annotations (for example `traefik.ingress.kubernetes.io/…`) | Fields in the API (filters, header matching, traffic split) |
| TLS certificate | One annotation; cert-manager creates the Certificate | Usually an explicit **Certificate** object, referenced by the Gateway listener |
| DNS record | external-dns reads the Ingress host | external-dns reads the Gateway (or the HTTPRoute, depending on its configuration) |

When to use which:

- **Ingress**: the simplest first exposure, and most existing platforms. It is still a valid choice.
- **Gateway API**: new platforms, clusters shared by many teams, or when you need traffic rules that Ingress can only express with annotations.

:::{note} Jargon: *GatewayClass*, *Gateway*, *HTTPRoute*
A **GatewayClass** names an implementation (here `traefik`), installed by the platform administrators. A **Gateway** is one entry point of that class: which ports and hostnames it listens on, and with which certificates. An **HTTPRoute** says which requests that arrive on a Gateway go to which Service.
:::

```{mermaid}
flowchart LR
    U[User browser] --> DNS[DNS record]
    DNS --> GW["Gateway<br/>(listeners: ports, hostnames, TLS)"]
    GW --> HR["HTTPRoute<br/>(hostname, path)"]
    HR --> SVC[Service climate-telemetry]
    SVC --> P[Pods]
```

Read it from left to right: the Gateway takes the place of the Ingress controller entry point, the HTTPRoute takes the place of the rules of the Ingress. The Service and the Pods do not change.

## 5.2 Example: `climate-telemetry` behind a Gateway

The example needs a cluster in Gateway mode:

- The Gateway API CRDs and a Gateway controller are installed.
- A `GatewayClass` (here `traefik`) exists.
- cert-manager has the `ClusterIssuer` `ecmwf-meditwin-training-issuer`.
- external-dns creates records under `training.meditwin-project.eu`, and watches Gateway objects.

It uses its own hostname, `climate-gw-traineeNN`, and its own Secret, `climate-gw-tls`, so that it never collides with the objects of section 4.

### Step 1: the Gateway

`gateway-climate.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: climate-gateway
  annotations:
    external-dns.alpha.kubernetes.io/hostname: climate-gw-traineeNN.training.meditwin-project.eu
spec:
  gatewayClassName: traefik
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: climate-gw-traineeNN.training.meditwin-project.eu
  - name: https
    protocol: HTTPS
    port: 443
    hostname: climate-gw-traineeNN.training.meditwin-project.eu
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: climate-gw-tls
```

| Field | What it does | Ingress equivalent (section 4) |
|---|---|---|
| `gatewayClassName: traefik` | Selects the controller that implements this Gateway | `ingressClassName: traefik` |
| `external-dns…/hostname` annotation | Asks external-dns to create the DNS record | Automatic, from the Ingress `host` |
| `listeners` | The entry points: `http` on 80, `https` on 443 | Fixed by the Ingress controller |
| `hostname` (on each listener) | Accepts only requests for this name | `rules[].host` |
| `tls.mode: Terminate` | TLS ends at the Gateway; the Service receives plain HTTP | Same behaviour as `spec.tls` |
| `certificateRefs` | The Secret with the certificate and the key | `spec.tls[].secretName` |

:::{note} Listener ports with Traefik
With Traefik, `listeners[].port` must match the port of a Traefik **entryPoint**. The example uses 80 and 443; the defaults of the Traefik Helm chart are 8000 and 8443. If the Gateway stays `Programmed: False` with a port error, the ports do not match.
:::

### Step 2: the HTTPRoute

`httproute-climate.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: climate-route
spec:
  parentRefs:
  - name: climate-gateway
  hostnames:
  - climate-gw-traineeNN.training.meditwin-project.eu
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: climate-telemetry
      port: 80
```

| Field | What it does | Ingress equivalent |
|---|---|---|
| `parentRefs` | Attaches the route to the Gateway `climate-gateway` | — (an Ingress has no separate entry point) |
| `hostnames` | The rule applies only to this name | `rules[].host` |
| `matches.path` (`PathPrefix`, `/`) | Which URLs match: here all of them | `paths[].path` and `pathType: Prefix` |
| `backendRefs` (`climate-telemetry`, `80`) | The Service and port that receive the traffic | `backend.service.name` and `port.number` |

### Step 3: the Certificate

`certificate-climate.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: climate-gw-tls
spec:
  secretName: climate-gw-tls
  issuerRef:
    name: ecmwf-meditwin-training-issuer
    kind: ClusterIssuer
  dnsNames:
  - climate-gw-traineeNN.training.meditwin-project.eu
```

With an Ingress, one annotation made cert-manager create this object for you. A Gateway listener only *references* a Secret, so you write the Certificate yourself. Section 6 explains how cert-manager fills the Secret.

### Step 4: apply and check

Apply the three files:

```bash
kubectl apply -f gateway-climate.yaml -f httproute-climate.yaml -f certificate-climate.yaml
```

Check the three objects:

```bash
kubectl get gateway climate-gateway
kubectl get httproute climate-route
kubectl get certificate climate-gw-tls
```

Call the service:

```bash
curl -s https://climate-gw-traineeNN.training.meditwin-project.eu/health; echo
```

Expected result:

```text
NAME              CLASS     ADDRESS        PROGRAMMED   AGE
climate-gateway   traefik   192.0.2.10     True         5m
NAME            HOSTNAMES                                              AGE
climate-route   ["climate-gw-traineeNN.training.meditwin-project.eu"]   5m
NAME             READY   SECRET           AGE
climate-gw-tls   True    climate-gw-tls   4m
{"status":"healthy","service":"ecmwf-climate-telemetry","version":"1.0.0","node":"climate-telemetry-…"}
```

## 5.3 Problems and solutions

| Symptom | Usual cause | Check |
|---|---|---|
| Gateway stays `Pending`, *Waiting for controller* | The cluster is not in Gateway mode, or the GatewayClass does not exist | `kubectl get gatewayclass` |
| `PROGRAMMED False` | The listener ports do not match the controller entry points | `kubectl describe gateway climate-gateway` |
| The route is not used | `parentRefs` does not name the Gateway, or the hostname differs | `kubectl describe httproute climate-route`: `Parents` → `Accepted` |
| No DNS record | external-dns does not watch Gateway objects, or the name is outside its zone | The sources and the domain filter of external-dns |
| Certificate not ready | As with Ingress: DNS record missing or wrong name | Section 6.6 |

## 5.4 Core takeaway

> **Ingress is still a valid and practical choice; Gateway API is the direction for more flexible and explicit traffic management.**

- Gateway API splits one Ingress into roles: **GatewayClass** (the implementation), **Gateway** (the entry point, owned by the platform), **HTTPRoute** (the routing rules, owned by the application team).
- With a Gateway, the certificate is usually an explicit **Certificate** object, referenced by the listener.
- The Service and the Pods do not change; the DNS record and the TLS flow use the same tools, external-dns and cert-manager.
- On the training cluster, only one exposure mode is active at a time: the day uses Ingress.

Continue with: [6. cert-manager and DNS](06-cert-manager-and-dns.md).
