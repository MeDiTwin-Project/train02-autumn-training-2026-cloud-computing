# 13. 📖 Day 3 cheat sheet

## Your `compose.yaml` seen through Kubernetes eyes

**Official documentation:** [kubernetes.io/docs](https://kubernetes.io/docs/home/) (concepts, step-by-step tasks, example manifests) and the [kubectl reference](https://kubernetes.io/docs/reference/kubectl/). For the fields of an object on *this* cluster: `kubectl explain <kind>.<field>`.

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

The `api` and `db` services of the Day 2 `climate-stack` ([Day 2, section 6](../day02/06-docker-compose.md)), line by line, as you ported them in sections 9 and 10:

| `compose.yaml` (Day 2) | Kubernetes object | What changes |
|---|---|---|
| `services.api` | **Deployment** (also for a single-replica `db` or `cache`, as in Hands-on 8–9); **StatefulSet** when each replica needs its own volume | One service becomes one workload; the workload owns N identical Pods |
| `image: climate-stats-api:1.0.0` | `spec.template.spec.containers[].image` | Must be pullable by every node, so it comes from a **registry**, never from the local `docker build` cache |
| `build:` | — | Kubernetes never builds. Building is the job of the CI pipeline ([Day 4, section 3](../day04/03-gitlab-ci.md)) |
| `ports: "8000:8000"` | **Service** (inside the cluster) + **Ingress** (from outside) | Port publishing is split in two objects with two different scopes |
| `environment:` | **ConfigMap** (plain values) and **Secret** (passwords) | Same variables, with the sensitive ones kept out of the manifests you share |
| `healthcheck:` and `HEALTHCHECK` in the Dockerfile | `readinessProbe` / `livenessProbe` | Kubernetes **ignores** the Dockerfile `HEALTHCHECK`: you declare it again, in the Pod template |
| `depends_on: condition: service_healthy` | readiness probes + retries in the application | There is no start ordering; Pods are expected to retry until dependencies are ready |
| `./init.sql:/docker-entrypoint-initdb.d/init.sql:ro` | **ConfigMap** mounted as a file | The file travels through the API instead of the host filesystem |
| `volumes: postgres_data:` | **PersistentVolumeClaim** | A named volume becomes a *claim* that the cluster satisfies with a PersistentVolume |
| `networks:` and `DB_HOST=db` | cluster DNS: `climate-db`, `climate-db.traineeNN.svc.cluster.local` | Same idea (name = service), provided by CoreDNS instead of the Docker embedded DNS |
| `restart: unless-stopped` | the controller loop (section 1.4) | Restart is the default; replacement across nodes is the new part |
| `mem_limit`, `cpus` (and `docker run -m 30m` from the Day 2 lab) | `resources.requests` / `resources.limits` | Same kernel cgroups, declared per container in YAML |
| `container_name:` | — | Pod names are generated; you address Pods via **labels**, never by name |

## `docker` → `kubectl` Rosetta stone

| Day 2 | Day 3 |
|---|---|
| `docker ps` | `kubectl get pods` (`-A` for all namespaces, `-o wide` for nodes and IPs) |
| `docker inspect <ctr>` | `kubectl describe pod <pod>` · `kubectl get pod <pod> -o yaml` |
| `docker logs -f <ctr>` | `kubectl logs -f <pod>` (`-c <container>`, `--previous`) |
| `docker exec -it <ctr> sh` | `kubectl exec -it <pod> -- sh` |
| `docker run -d --name x img` | `kubectl create deployment x --image=img` |
| `docker run --rm -it alpine sh` | `kubectl run -it --rm tmp --image=alpine --restart=Never -- sh` |
| `docker run -p 8080:80` | Service + `kubectl port-forward svc/<name> 8080:80` (local) · Ingress (public) |
| `docker run -e KEY=val` | `env:` · ConfigMap · Secret |
| `docker run -v vol:/path` | PVC + `volumes:` / `volumeMounts:` |
| `docker run -m 128m --cpus 1` | `resources.limits` (`requests` for scheduling) |
| `docker run --gpus 1` | `resources.limits: {nvidia.com/gpu: 1}` + `runtimeClassName: nvidia` |
| `HEALTHCHECK` / `healthcheck:` | `readinessProbe` / `livenessProbe` (the Dockerfile one is ignored) |
| `docker login` (private images) | `kubectl create secret docker-registry` + `imagePullSecrets` |
| `docker compose up -d` | `kubectl apply -f dir/` · `helm upgrade --install` |
| `docker compose ps` | `kubectl get deploy,po,svc,ing -l app=<name>` |
| `docker compose down` (`-v`) | `kubectl delete -f dir/` (+ `kubectl delete pvc`) |
| `docker compose up --scale api=3` | `kubectl scale deploy/<name> --replicas=3` · HPA |
| `docker stats` | `kubectl top pods` |
| `DB_HOST=db` (embedded DNS) | `<service>` · `<service>.<namespace>.svc.cluster.local` (CoreDNS) |

## Kubernetes core

```bash
kubectl config view --minify -o jsonpath='{..namespace}'   # default namespace of the context
kubectl get deploy,po,svc,ing
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get events --sort-by=.lastTimestamp
```

## Workloads

```bash
kubectl apply -f deployment.yaml
kubectl scale deployment climate-telemetry --replicas=3
kubectl set image deployment/climate-telemetry app=<registry>/<repo>:<tag>
kubectl rollout status deployment/climate-telemetry
kubectl rollout history deployment/climate-telemetry
kubectl rollout undo deployment/climate-telemetry
kubectl rollout restart deployment/climate-telemetry   # after a ConfigMap/Secret change
```

## Registry access

```bash
docker tag climate-telemetry:1.0.0 gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:1.0.0
docker push gitlab.data.meditwin-project.eu:6443/training/autumn-2026/climate-telemetry-traineeNN:1.0.0
# registry credentials for the cluster, from your docker login (Hands-on 2)
kubectl create secret generic gitlab-registry \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
# production: a dedicated read_registry token instead
# kubectl create secret docker-registry gitlab-registry \
#   --docker-server=gitlab.data.meditwin-project.eu:6443 --docker-username=<user> --docker-password=<token>
```

## Services and Ingress

```bash
kubectl get svc
kubectl get endpoints climate-telemetry
kubectl get ingress
kubectl describe ingress climate-telemetry
kubectl port-forward svc/climate-telemetry 8080:80
```

## cert-manager and TLS

```bash
kubectl get clusterissuers
kubectl get certificates
kubectl describe certificate climate-telemetry-tls
kubectl describe challenge
```

## Configuration and storage

```bash
kubectl get configmap climate-config -o yaml
kubectl get secret climate-db -o yaml
kubectl get pvc
kubectl describe pvc climate-db-data
```

## Scaling

```bash
kubectl top pods
kubectl autoscale deployment climate-telemetry --cpu-percent=70 --min=2 --max=6
kubectl get hpa
kubectl describe hpa climate-telemetry       # kubectl autoscale names the HPA after the Deployment (section 7 used climate-telemetry-hpa in explicit YAML)
```

## GPU checks

```bash
# instructor: nodes are cluster-scoped
kubectl get nodes -L nvidia.com/gpu.present,meditwin.io/role
kubectl describe node <gpu-node-name>
```

## Helm

```bash
helm repo list
helm repo update
helm show values <repo>/<chart> --version <v>     # the defaults of a chart
helm upgrade --install <release> <repo>/<chart> --version <v> -f values.yaml
helm list -A
helm status <release> -n <namespace>
helm get values <release> -n <namespace>
helm lint <chart-dir>                              # check a chart you are writing
helm template <release> <chart> -f values.yaml     # render without installing
helm install <release> <chart-dir>                 # install a local chart
helm upgrade <release> <chart-dir> --set key=value # quick override
helm get manifest <release>                        # the YAML of the release
helm get manifest <release> | kubectl get -f -     # the objects it created
helm uninstall <release>
helm create <name>                                 # a starter chart
helm package <chart-dir>                           # build the .tgz
helm history <release> -n <namespace>
helm rollback <release> <revision> -n <namespace>
```

## JupyterHub Helm quick start

```bash
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update
helm show values jupyterhub/jupyterhub --version 4.4.4 > jhub-default-values.yaml   # the defaults
helm upgrade --install jhub jupyterhub/jupyterhub --version 4.4.4 -f values-jhub.yaml   # in your namespace
helm get values jhub --all                                                            # what the release uses
helm get manifest jhub | kubectl get -f -                                             # the objects of the release
```

## From Compose to Kubernetes

```bash
kompose convert -f compose.yaml -o k8s-kompose/   # a draft: check every file before applying it
```

## Problems and solutions

| Symptom | Usual cause | Check |
|---|---|---|
| `ErrImagePull` / `ImagePullBackOff` | Wrong image path or tag (`not found`); for a private project, missing or wrong `imagePullSecrets` (`unauthorized`) | `kubectl describe pod <pod>`, events (section 3.5) |
| `InvalidImageName` | `traineeNN` not replaced in `image:` | The `image:` line of the manifest |
| Pod `Running` but `0/1` | Readiness probe failing: wrong path or port, app not listening on `0.0.0.0` | `kubectl describe pod`, then `kubectl logs` |
| `CreateContainerConfigError` | A ConfigMap or Secret referenced by the Pod does not exist, or has another name or key | `kubectl describe pod` names the missing object (section 8) |
| `OOMKilled`, exit code 137 | `limits.memory` too low; same OOM killer as [Day 2](../day02/01-containers-and-docker.md#the-oom-killer-exit-code-137) | `kubectl describe pod`, *Last State* (section 7.1) |
| `CrashLoopBackOff` | The application exits at start | `kubectl logs <pod> --previous` |
| Pod `Pending` | Not enough CPU or memory for its `requests`; node selector or taint mismatch; PVC not bound | `kubectl describe pod`, events (`Insufficient cpu`, `didn't match`, `unbound`) |
| PVC `Pending` | No default StorageClass, or the class waits for the first Pod (`WaitForFirstConsumer`) | `kubectl describe pvc` (section 9) |
| Service has no endpoints | Selector does not match the Pod labels, or no Pod is ready | `kubectl get endpoints`, `kubectl get pods --show-labels` (section 4.3) |
| Service name does not resolve from another Pod | The Service does not exist (for example, not created by `kompose`), or another namespace | `kubectl get svc`; use `<svc>.<namespace>.svc.cluster.local` (section 10) |
| Ingress not reachable | DNS record missing (hostname outside `training.meditwin-project.eu`, or a typo), or wrong `ingressClassName` | `nslookup <host>`, `kubectl describe ingress` (section 6.4) |
| HTTP not redirected to HTTPS | The Middleware or its annotation is missing, or the annotation names another namespace | `kubectl get middleware.traefik.io`, the Ingress annotations (section 4.7.1) |
| Certificate not ready | ACME challenge failed: DNS not pointing to the Ingress; or a rate limit after many failed attempts | `kubectl describe certificate`, `kubectl describe challenge` (section 6.6) |
| HPA shows `<unknown>` and does not scale | metrics-server missing, or `requests.cpu` not set | `kubectl describe hpa`, `kubectl top pods` (section 7.2) |
| `kubectl apply` changed nothing after a new build | Same tag (for example `latest`) | Use a new tag for each build (section 3.3) |
| `helm install`: *`clusterroles … is forbidden`* | A component of the chart needs cluster-wide permissions | Switch it off in the values (section 12.4) |
| `helm install`: *`cannot be imported into the current release`* | An object with the same name exists and was not created by this release | Delete it (for example `kubectl delete -f`), then install again (section 12.7) |
| Helm upgrade failed, or values ignored | Invalid or misspelled values key, incompatible chart version | `helm template … -f values.yaml`; compare with `helm show values` |
| GPU Pod `Pending` | The only GPU is in use, or a taint without toleration | `kubectl describe pod`; delete the previous GPU Pod (section 11) |
| `nvidia-smi` finds no GPU | `runtimeClassName: nvidia` missing | The Pod spec (section 11.1) |
