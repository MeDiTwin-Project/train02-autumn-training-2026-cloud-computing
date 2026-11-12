# 4. Exposing an application

This section introduces **Service** as the first step to make a deployed workload reachable inside the cluster, without adding Ingress complexity yet.

On [Day 2](../day02/05-application-containerisation.md#id-5-5-image-construction-docker-build), `docker run -p 8000:8000 climate-telemetry:1.0.0` did two things at once: it gave the container a stable address and it published it on the host. Kubernetes splits that into two objects: a **Service** (stable address inside the cluster) and an **Ingress** (published to the outside world).

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

## 4.1 Why a Service?

```{mermaid}
flowchart TB
    C([Client]) --> SVC[Service climate-telemetry]
    SVC --> P1[Pod: ready]
    SVC --> P2[Pod: ready]
    P3[Pod: not ready]
    SVC -. no traffic .-> P3
```

A Service is a stable network endpoint in front of a changing set of Pods. It sends traffic only to the Pods that are **ready**: a Pod whose readiness probe fails stays out, even if it is running.

## 4.2 Concepts

- **Pod IP**: each Pod gets an IP, but Pods are ephemeral and can be replaced.
- **Service**: stable virtual IP and DNS name that abstracts multiple Pods.
- **ClusterIP**: default Service type, reachable only from inside the cluster.
- **NodePort**: exposes a fixed port on every node for external access patterns.
- **LoadBalancer**: asks the cloud (for example OpenStack) for an external IP address in front of the Service.
- **selector**: label query used by the Service to target matching Pods.
- **service discovery**: built-in DNS naming so workloads can find each other reliably.

:::{note} Jargon: *endpoints*
The **endpoints** of a Service are the IP address and port of each Pod that it sends traffic to *right now*. Kubernetes updates the list all the time: a Pod that is replaced, or that is not ready, leaves the list. `kubectl get endpoints <service>` shows it.
:::

Why this matters:

- Pods change over time.
- A Service gives clients one stable destination.
- Traffic routing follows labels, not static Pod addresses.

## 4.3 ⌨️ Hands-on 3: a Service for `climate-telemetry`

Create `service.yaml` in `~/climate-service-k8s`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: climate-telemetry
spec:
  selector:
    app: climate-telemetry
  ports:
    - name: http
      port: 80
      targetPort: http
```

`port: 80` is what clients call; `targetPort: http` is the named container port (8000, the `EXPOSE 8000` of the Dockerfile) from the Deployment. It is the same idea as the `host:container` pair of `-p` on [Day 2](../day02/03-running-containers.md#id-3-4-exposing-services-port-forwarding-p), except that nothing is published on any host yet: the Service is reachable only inside the cluster.

```bash
kubectl apply -f service.yaml
```

(The imperative equivalent is `kubectl expose deployment climate-telemetry --port=80 --target-port=8000`.)

Then inspect it:

```bash
kubectl get service climate-telemetry
kubectl describe service climate-telemetry
```

Also check whether endpoints were created:

```bash
kubectl get endpoints climate-telemetry
```

You should see one or more `IP:8000` endpoints, one per ready Pod. On recent clusters the command also prints `Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice`: you can ignore it, the list is still correct. If endpoints are empty, the Service selector does not match the Pod labels, or no Pod is **ready** — the readiness probe from section 3.4 decides who is in this list.

## 4.4 Service discovery: `DB_HOST=db`, again

On [Day 2](../day02/06-docker-compose.md#id-6-4-service-discovery-and-embedded-dns-resolution) the API found its database by name (`DB_HOST=db`) thanks to the Docker embedded DNS. Kubernetes does the same with CoreDNS, and the name is the **Service** name. Prove it from a throw-away Pod, the `docker run --rm` of [Day 2](../day02/03-running-containers.md#id-3-1-core-concepts-everyday-flags):

Start a shell in a throw-away Pod:

```bash
kubectl run -it --rm tmp --image=busybox:1.36 --restart=Never -- sh
```

Inside the Pod, resolve the Service name and call it, first with the short name, then with the full one:

```bash
nslookup climate-telemetry
wget -qO- http://climate-telemetry/health
wget -qO- http://climate-telemetry.traineeNN.svc.cluster.local/health
```

Leave the Pod (`--rm` deletes it):

```bash
exit
```

The short name works inside your namespace; the fully qualified name works from anywhere in the cluster.

Now make load-balancing visible with three replicas. Set `replicas: 3` in `deployment.yaml` and apply it — the file is the source of truth:

```bash
kubectl apply -f deployment.yaml
kubectl get pods -l app=climate-telemetry
```

The imperative twin is `kubectl scale deployment climate-telemetry --replicas=3`: quick, but the file would still say 1, and the next `kubectl apply` would scale you back.

Then repeat requests from a temporary Pod:

```bash
kubectl run --rm -i tmp-lb --image=busybox:1.36 --restart=Never -- sh -c 'for i in 1 2 3 4 5 6; do wget -qO- http://climate-telemetry/health; echo; done'
```

You should see different `node` values across responses, because the Service balances traffic across the three Pods.

## 4.5 Local access with port-forward

To reach the Service from your machine without publishing anything, you use two terminals on the VM.

**Terminal 1: open a tunnel to the Service.** Start the port-forward in the foreground and leave it open:

```bash
kubectl port-forward service/climate-telemetry 8080:80
```

```text
Forwarding from 127.0.0.1:8080 -> 8000
```

**Terminal 2: call the service through the tunnel:**

```bash
curl -s http://localhost:8080/health
```

This is the Day 3 counterpart of `-p`: local port 8080 → Service port 80 → container port 8000. One difference: it only exists while the command runs, and only on your machine.

If you get `address already in use`, a [Day 2](../day02/04-persisting-data-and-web-ui.md#id-4-4-bind-mounts-and-posix-permission-boundaries) container (for example `web-doc`) still uses port 8080 on this machine (`docker ps`, then `docker rm -f <name>`), or use another local port: `kubectl port-forward service/climate-telemetry 8081:80`.

**Checkpoint 3** — the Service has three ready endpoints, and you reach it from your machine.

```bash
kubectl get endpoints climate-telemetry
curl -s http://localhost:8080/health; echo      # in terminal 2, with the port-forward of terminal 1 running
```

```text
NAME                ENDPOINTS                                      AGE
climate-telemetry   10.42.1.15:8000,10.42.2.21:8000,10.42.3.9:8000   3m
{"status":"healthy","service":"ecmwf-climate-telemetry","version":"1.0.0","node":"climate-telemetry-…"}
```

(The IP addresses are different on your cluster: what matters is three of them, all on port 8000.)

:::{dropdown} Not there?
- `ENDPOINTS <none>`: the Service selector does not match the Pod labels (`kubectl get pods --show-labels`), or no Pod is ready.
- Only one endpoint: `kubectl scale` was not run, or `deployment.yaml` was re-applied with `replicas: 1`.
- `curl: (7) Failed to connect`: the `port-forward` is not running in terminal 1.
:::

When you have finished, stop the port-forward in terminal 1 with **Ctrl+C**.

A running Pod means the process exists; a Service means the application is reachable in a stable way: **a Pod is not yet a reachable application.** The next step makes it reachable from outside the cluster.

## 4.6 Adding Ingress for external HTTP/HTTPS

Once a Service works, the next step is to expose it externally with **Ingress**.

```{mermaid}
flowchart LR
    U[User browser] --> DNS[DNS record]
    DNS --> ING[Ingress controller]
    ING --> SVC[Service climate-telemetry]
    SVC --> P[Pods]
```

Read the diagram from left to right: DNS sends the user to the Ingress controller, which reads your Ingress rules and forwards the request to the Service, which chooses a Pod.

:::{note} Jargon: *Ingress* and *Ingress controller*
An **Ingress** is only a set of rules (which hostname and path go to which Service). It does nothing by itself. An **Ingress controller** is the program that reads all the Ingress objects of the cluster and actually receives the traffic. On the training cluster it is **Traefik**, installed once by the platform team for everybody.
:::

Key concepts:

- **Ingress** defines host/path routing rules for HTTP and HTTPS traffic.
- **ingressClassName** selects which Ingress controller processes the resource. On the training cluster it is `traefik`.
- **TLS** on Ingress uses a Kubernetes Secret containing certificate and key.
- **cert-manager annotation** can automate certificate issuance for that Secret. The cluster has a ClusterIssuer named `ecmwf-meditwin-training-issuer`.
- **DNS**: the cluster runs external-dns, which creates the record for the Ingress host automatically, as long as the host is under `training.meditwin-project.eu`.

## 4.7 ⌨️ Hands-on 4: Ingress with TLS

Create `ingress.yaml` in `~/climate-service-k8s`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: climate-telemetry
  annotations:
    cert-manager.io/cluster-issuer: ecmwf-meditwin-training-issuer
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - climate-traineeNN.training.meditwin-project.eu
      secretName: climate-telemetry-tls
  rules:
    - host: climate-traineeNN.training.meditwin-project.eu
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: climate-telemetry
                port:
                  number: 80
```

| Field | What it does | Day 2 twin |
|---|---|---|
| `annotations: cert-manager.io/cluster-issuer` | Asks cert-manager for a certificate, from the training ClusterIssuer (section 6) | — (on Day 2 there was no TLS) |
| `ingressClassName: traefik` | Which Ingress controller applies these rules | — |
| `tls[].hosts`, `secretName` | The hostname to protect and the Secret where the certificate is stored | — |
| `rules[].host` | Only requests for this name match; external-dns creates its DNS record | The host part of `http://<host>:8080` |
| `paths[].path: /`, `pathType: Prefix` | Which URLs match: here all of them | — |
| `backend.service` (`climate-telemetry`, `80`) | The Service, and its port, that receive the traffic | The container behind `-p` |

Apply and inspect:

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress climate-telemetry
```

At this stage there is no HTTP->HTTPS redirect yet.

Validate the DNS record and the certificate workflow (both take a minute or two):

```bash
nslookup climate-traineeNN.training.meditwin-project.eu
kubectl get certificate
kubectl describe certificate climate-telemetry-tls
```

Then test both protocols from outside:

```bash
curl http://climate-traineeNN.training.meditwin-project.eu/health
curl https://climate-traineeNN.training.meditwin-project.eu/health
```

You can also open the URL in a browser: with plain HTTP, the connection is not secure (no TLS lock), while HTTPS uses the certificate issued by cert-manager.

### 4.7.1 Enable HTTP -> HTTPS redirect

Force HTTP to HTTPS redirection with a Traefik **Middleware**. Look at its `apiVersion`: `traefik.io/v1alpha1` is not a built-in group. A `Middleware` is your first object of a **CRD** (section 3.3.1), added to the cluster when Traefik was installed:

```bash
kubectl api-resources --api-group=traefik.io        # the kinds that Traefik adds
kubectl explain middleware.spec.redirectScheme
```

Create `middleware-redirect-https.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

Apply it:

```bash
kubectl apply -f middleware-redirect-https.yaml
```

Then tell the Ingress to use it. Add the annotation to `ingress.yaml`:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: ecmwf-meditwin-training-issuer
    traefik.ingress.kubernetes.io/router.middlewares: traineeNN-redirect-https@kubernetescrd
```

The value has three parts: your namespace (`traineeNN`), the name of the Middleware (`redirect-https`), and `@kubernetescrd`, Traefik's name for objects defined by its CRDs.

```bash
kubectl apply -f ingress.yaml
```

Check that plain HTTP is redirected:

```bash
curl -I http://climate-traineeNN.training.meditwin-project.eu/health
```

You should see `301` or `308` and a `Location: https://...` header.
This means HTTP is no longer serving the app directly: the redirect is active, and clients are now sent to the HTTPS URL.

You now have `deployment.yaml`, `service.yaml`, `middleware-redirect-https.yaml` and `ingress.yaml` in `~/climate-service-k8s`. Keep them: on Day 4 the same four objects are deployed automatically from a repository.

:::{tip} Do not delete the TLS Secret
Each time a certificate is requested again for the same hostname, the CA counts it, and Let's Encrypt allows only 5 identical certificates per week. When you delete and recreate the Ingress (later today, and on Day 4), leave the `climate-telemetry-tls` Secret in place: cert-manager reuses it while it is valid.
:::

### ⌨️ Try it: a deliberate failure

Edit `deployment.yaml` and change the readiness probe path to `/healthz` (which the app does not serve).

**Before you apply it, predict:**

1. Will the new Pod be `Running`? Will it be `READY`?
2. Will it appear in the endpoints of the Service?
3. Will `https://climate-traineeNN…/health` still answer?

Apply it, and wait for the rollout (it will time out):

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/climate-telemetry --timeout=45s
```

Look at the Pods and at the endpoints of the Service:

```bash
kubectl get pods -l app=climate-telemetry
kubectl get endpoints climate-telemetry
```

Call the site:

```bash
curl https://climate-traineeNN.training.meditwin-project.eu/health
```

Compare with your predictions, then open the solution.

:::{dropdown} What happened, and why
1. The new Pod is `Running` (the process started) but `0/1`: the readiness probe asks for `/healthz` and gets `404`.
2. It is **not** in the endpoints: only ready Pods receive traffic.
3. Yes. The three old Pods keep serving, and the site stays up. With 3 replicas, the default rolling update starts **one** new Pod at a time and never removes an old one before the new one is ready.

This is the difference between "the process is running" and "the application is ready", and it is what Compose's `healthcheck` could only approximate.
:::

Put `/health` back in `deployment.yaml` and apply again.

**Checkpoint 4** — the service answers over HTTPS with a valid certificate, plain HTTP is redirected, and the readiness probe is back on `/health`.

```bash
kubectl get certificate climate-telemetry-tls
kubectl get deployment climate-telemetry
curl -s https://climate-traineeNN.training.meditwin-project.eu/health; echo
curl -sI http://climate-traineeNN.training.meditwin-project.eu/health | head -1
```

```text
NAME                    READY   SECRET                  AGE
climate-telemetry-tls   True    climate-telemetry-tls   10m
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
climate-telemetry   3/3     3            3           25m
{"status":"healthy","service":"ecmwf-climate-telemetry","version":"1.0.0","node":"climate-telemetry-…"}
HTTP/1.1 308 Permanent Redirect
```

:::{dropdown} Not there?
- Certificate `READY False` for more than a few minutes: `kubectl describe certificate climate-telemetry-tls` and `kubectl describe challenge` (section 6.6). Most often: a typo in the hostname, so the DNS record does not exist.
- `curl: (6) Could not resolve host`: the DNS record is not there yet. `nslookup` the name and wait a minute; check the host in `ingress.yaml`.
- HTTPS works but HTTP answers `200` instead of `301`/`308`: the Middleware or its annotation is missing (section 4.7.1). The annotation value must be `traineeNN-redirect-https@kubernetescrd` with *your* namespace.
- `UP-TO-DATE 1` instead of `3`, and one Pod `0/1` in `kubectl get pods`: the readiness path is still `/healthz`. Fix `deployment.yaml` and apply again.
:::

## 4.8 Core takeaway

> **Service gives stable internal reachability; Ingress + TLS gives stable and secure external reachability.**

- A **Service** is a stable name and virtual IP in front of a changing set of Pods; it selects them by **labels**.
- Only **ready** Pods are in the endpoints of a Service: readiness decides who receives traffic.
- Inside the cluster, the Service name is the hostname (`climate-telemetry`), like `DB_HOST=db` in Compose.
- `kubectl port-forward` is a private tunnel for you; an **Ingress** publishes the Service for everybody, by hostname and path.
- One annotation asks cert-manager for a certificate; external-dns creates the DNS record. A Traefik Middleware adds the HTTP → HTTPS redirect.

## 4.9 📖 Check yourself

1. `kubectl get endpoints climate-telemetry` shows `<none>`, but the Pods are `Running`. Give two causes.
2. In `service.yaml`, what is the difference between `port: 80` and `targetPort: http`?
3. From a Pod in **another** namespace, which hostname reaches your Service?
4. Why does `kubectl port-forward` not replace an Ingress?
5. The certificate stays `READY False`. Which object do you describe next, and what is the most common cause on the training cluster?
6. You copy a `Middleware` manifest from the Traefik documentation. How do you know, from the file alone, that it is not a built-in Kubernetes object? Which part of it would you never write yourself?

:::{dropdown} Solutions
**1.** The Service `selector` does not match the Pod labels (`kubectl get pods --show-labels`), or no Pod is ready (the readiness probe fails).

**2.** `port` is the port clients call on the Service. `targetPort` is the container port that receives the traffic: here the port named `http`, that is 8000.

**3.** The fully qualified name: `climate-telemetry.traineeNN.svc.cluster.local`. The short name works only inside your namespace.

**4.** It exists only while the command runs, only on your machine, and sends all traffic to one Pod. An Ingress is a permanent, shared entry point with a hostname and TLS.

**5.** `kubectl describe certificate climate-telemetry-tls`, then `kubectl describe challenge`. Most often: a typo in the hostname, so the DNS record does not exist and the HTTP-01 challenge fails.

**6.** Its `apiVersion` (`traefik.io/v1alpha1`) names a group that is not part of Kubernetes: the kind comes from a CRD. You never write `status`: Kubernetes, or the controller of the CRD, fills it in.
:::

📖 After the lesson: [5. Exposing an application with Gateway API](05-exposing-an-application-with-gateway-api.md), the newer alternative to Ingress, and [6. cert-manager and DNS](06-cert-manager-and-dns.md), what happened behind the scenes in Hands-on 4.

Continue with: [7. Scaling](07-scaling.md).
