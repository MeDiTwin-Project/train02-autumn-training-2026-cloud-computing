# 2. Interacting with a cluster

You know the architecture; now you use it. This section answers one question:

> **How do I find out what is running in my namespace, and why it does or does not work?**

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

## 2.1 Concepts

`kubectl` needs to know **which cluster**, **who you are**, and **which namespace**:

| Term | What it is | On the training cluster |
|---|---|---|
| **kubeconfig** | The file with the cluster address, your credentials and your contexts (`~/.kube/config`, or `$KUBECONFIG`) | Already prepared on your VM |
| **context** | A name for one combination of cluster, user and default namespace | One context, already pointing at your namespace |
| **namespace** | A separate space for objects inside the cluster | `traineeNN`, yours only |

Four commands answer most questions, from the broad view to the detail:

| Step | Command | Question it answers |
|---|---|---|
| 1. Start broad | `kubectl get` | What exists, and in which state? |
| 2. Focus | `kubectl describe` | Why is this object in this state? Look at the **events** at the end |
| 3. Look at the application | `kubectl logs` | What did the program write? |
| 4. Go inside | `kubectl exec` | What does the container see (files, environment, network)? Only when 1–3 are not enough |

:::{note} Jargon: *labels* and *selectors*
A **label** is a `key: value` pair on an object, for example `app: climate-telemetry`. A **selector** finds all objects with given labels. A Deployment finds its Pods, and a Service finds the Pods it sends traffic to, by selector: if the labels do not match, nothing works, and no error tells you why.
:::

Your speed in Kubernetes depends on how quickly you go from step 1 to the right detail. Practise it on every problem of the day.

## 2.2 From `docker` to `kubectl`

You already know this flow from [Day 2, section 3.6](../day02/03-running-containers.md#id-3-6-the-recommended-diagnostic-flow) (`docker ps` → `docker inspect` → `docker logs` → `docker exec`). The verbs are the same; what changes is that you talk to an API server instead of a local daemon, and you address Pods via labels instead of container names:

| Day 2: `docker` | Day 3: `kubectl` | Difference |
|---|---|---|
| `docker ps` | `kubectl get pods` | Shows only *your namespace* by default; `-A` for all |
| `docker inspect <ctr>` | `kubectl describe pod <pod>` / `kubectl get pod <pod> -o yaml` | `describe` also shows **events**, the first place to look when a Pod is not running |
| `docker logs -f <ctr>` | `kubectl logs -f <pod>` | Add `-c <container>` when a Pod has more than one container; `--previous` after a crash |
| `docker exec -it <ctr> sh` | `kubectl exec -it <pod> -- sh` | Note the `--` |
| `docker run --rm -it alpine sh` | `kubectl run -it --rm tmp --image=alpine -- sh` | Throw-away debugging Pod |

:::{dropdown} 📖 More equivalents
| Day 2: `docker` | Day 3: `kubectl` | Difference |
|---|---|---|
| `docker ps -a` | `kubectl get pods` + `kubectl get events` | Finished Pods are kept only until their controller cleans them up |
| `docker run -d --name x -p 8080:80 img` | `kubectl create deployment x --image=img` + `kubectl port-forward` | There is no `-p`: publishing a port is a separate object (section 4) |
| `docker stop` / `docker rm` | `kubectl delete pod <pod>` | The controller recreates it immediately: delete the **Deployment** to really stop it |
| `docker compose up -d` | `kubectl apply -f dir/` or `helm upgrade --install` | `apply` is idempotent, like `compose up` |
| `docker compose ps` | `kubectl get deploy,po,svc -l app=<name>` | Labels replace the Compose project name |
| `docker compose down` | `kubectl delete -f dir/` | Deletes everything defined in `dir/`, PVCs included (unlike `compose down` without `-v`): keep the PVC in its own file if the data must survive |
| `docker stats` | `kubectl top pods` | Requires metrics-server on the cluster |
| `docker network ls` / `docker volume ls` | `kubectl get svc` / `kubectl get pvc` | Networking and storage are *objects* in the API, not daemon-local resources |

The [cheat sheet](13-cheatsheet.md) has the full list.
:::

(hands-on-1)=
## 2.3 ⌨️ Hands-on 1: hello, Kubernetes

### Step 1: check where you are

The kubeconfig on your VM has one context, and its default namespace is already `traineeNN`: you never have to type `-n traineeNN`. Check it:

```bash
kubectl config current-context
kubectl config view --minify -o jsonpath='{..namespace}'; echo
kubectl get pods
```

The second command prints `traineeNN`; the third prints `No resources found in traineeNN namespace.`, or the objects that are already there.

Now look at the cluster itself: the machines, or **nodes**, of the [architecture in section 1.3](01-kubernetes-user-perspective.md#id-1-3-architecture-at-a-glance-control-plane-and-workers):

```bash
kubectl get nodes -o wide
```

```text
NAME                                   STATUS   ROLES                       VERSION          OS-IMAGE             CONTAINER-RUNTIME
ecmwf-meditwin-training-b6b68-4pkjn    Ready    control-plane,etcd,master   v1.33.3+rke2r1   Ubuntu 24.04.4 LTS   containerd://2.0.5-k3s2
ecmwf-meditwin-training-b6b68-r4xxz    Ready    control-plane,etcd,master   v1.33.3+rke2r1   Ubuntu 24.04.4 LTS   containerd://2.0.5-k3s2
ecmwf-meditwin-training-b6b68-tnxkk    Ready    control-plane,etcd,master   v1.33.3+rke2r1   Ubuntu 24.04.4 LTS   containerd://2.0.5-k3s2
ecmwf-meditwin-training-gpu-md-0-…     Ready    gpu,worker                  v1.33.3+rke2r1   Rocky Linux 9.8      containerd://2.0.5-k3s2
ecmwf-meditwin-training-ingress-md-0-… Ready    ingress,worker              v1.33.3+rke2r1   Ubuntu 24.04.4 LTS   containerd://2.0.5-k3s2
ecmwf-meditwin-training-ingress-md-1-… Ready    ingress,worker              v1.33.3+rke2r1   Ubuntu 24.04.4 LTS   containerd://2.0.5-k3s2
ecmwf-meditwin-training-worker-md-0-…  Ready    worker                      v1.33.3+rke2r1   Ubuntu 24.04.4 LTS   containerd://2.0.5-k3s2
…   (four more workers)
```

(Names and columns shortened: the real output also shows `AGE`, the IP addresses and the kernel version.)

Read it with the architecture of section 1.3 in mind:

- **Three control-plane nodes** (`control-plane,etcd,master`) run the API server, the scheduler, the controllers and etcd, the database of the cluster. Three, so that the cluster keeps working if one fails.
- **Workers** run your Pods. Some have an extra role: `ingress` nodes run Traefik, the Ingress controller of section 4; the `gpu` node has the GPU of section 11.
- `STATUS` `Ready`: the kubelet of the node answers. `VERSION`: Kubernetes 1.33, in the RKE2 distribution.
- `CONTAINER-RUNTIME`: `containerd`, the same runtime that runs your containers under Docker on Day 2.
- The GPU node runs Rocky Linux, the others Ubuntu: each node has its own operating system and kernel, and the containers on it share that kernel ([Day 2, section 1](../day02/01-containers-and-docker.md)).

:::{note} Not every cluster shows you its nodes
Nodes are cluster-wide objects, not objects of a namespace. On this training cluster you may list them. On many production clusters (for example EKS, GKE or AKS run by a platform team), users who only deploy applications get permissions in their own namespace only, and `kubectl get nodes` returns `forbidden`. Then the platform team tells you what the nodes offer.
:::

:::{warning} One cluster, many trainees
Everybody shares the same cluster. Trainee users are RBAC-scoped to their own namespace: operations outside `traineeNN` are intentionally blocked and usually return `forbidden` (for example `kubectl get namespaces`, `kubectl get pods -A`, `kubectl get pods -n kube-system`, or any `kubectl delete` outside your namespace). Work only in your own namespace and never run `kubectl delete` with `-A`.
:::

### Step 2: run your first Pod

On Day 2 you ran a container that prints one line and exits:

```bash
docker run --rm busybox echo "hello kubernetes"
```

The Kubernetes twin is `kubectl run`:

```bash
kubectl run hello --image=busybox --restart=Never -- echo "hello kubernetes"
```

```text
pod/hello created
```

| Part | Meaning | Day 2 twin |
|---|---|---|
| `hello` | The name of the Pod | `--name hello` |
| `--image=busybox` | The image of its only container | `busybox` |
| `--restart=Never` | Run the command once. Without it the Pod has `restartPolicy: Always`: `echo` ends, the container restarts, ends again… and the Pod goes into `CrashLoopBackOff` | no `--restart` flag |
| `-- echo "hello kubernetes"` | Everything after `--` is the command of the container, not an option of `kubectl` | `echo …` after the image |

:::{note} Why `busybox`?
`busybox` is a very small Linux image that includes a basic shell and core utilities. In this course we use it as a disposable troubleshooting container for quick checks.
:::

Two things are different from `docker run`. The text `hello kubernetes` does **not** appear in your terminal, and the command returns at once. `kubectl` only sent a request to the API server, and the cluster does the rest:

1. The **API server** stores the new Pod, with status `Pending`.
2. The **scheduler** chooses a node for it.
3. The **kubelet** on that node pulls `busybox` and starts the container, with the same containerd and runc as `docker run` on Day 2.
4. `echo` prints its line and exits with code `0`: the Pod is `Completed`.

The next steps follow this story with the four commands of section 2.1.

### Step 3: what exists, and in which state? (`get`)

```bash
kubectl get pods -o wide
```

```text
NAME    READY   STATUS      RESTARTS   AGE   IP           NODE                                          …
hello   0/1     Completed   0          15s   10.42.7.85   ecmwf-meditwin-training-gpu-md-0-…-5t6dz.novalocal   …
```

`-o wide` adds the IP address of the Pod and the **node** that runs it, one of the workers of Step 1: the scheduler chose it, not you. If you are fast, you see `ContainerCreating` first; add `-w` to watch the changes live (Ctrl+C to stop).

### Step 4: why is it in this state? (`describe`)

```bash
kubectl describe pod hello
```

Scroll to the **Events** at the end:

```text
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  40s   default-scheduler  Successfully assigned traineeNN/hello to ecmwf-meditwin-training-…-5t6dz.novalocal
  Normal  Pulling    40s   kubelet            Pulling image "busybox"
  Normal  Pulled     37s   kubelet            Successfully pulled image "busybox" in 2.512s (…). Image size: 2236767 bytes.
  Normal  Created    37s   kubelet            Created container: hello
  Normal  Started    37s   kubelet            Started container hello
```

They are the steps of the story above, written by the component that did each one: the scheduler, then the kubelet. When a Pod does not start, the last event tells you where it stopped (for example `Failed … ErrImagePull`).

### Step 5: what did the program write? (`logs`)

```bash
kubectl logs hello
```

```text
hello kubernetes
```

The output did not go to your terminal: it went to the log of the container, on the node. `kubectl logs` fetches it, like `docker logs`.

The fourth command, `kubectl exec`, needs a container that is still running; you use it in Hands-on 2.

(hands-on-1-step-6)=
### Step 6: the whole object (`-o yaml`)

```bash
kubectl get pod hello -o yaml | less
```

The output is long (press `q` to quit `less`), but it has only five top-level fields. Here they are, with the lines that matter for `hello`:

```yaml
apiVersion: v1                  # which API defines this kind of object
kind: Pod                       # which kind of object
metadata:                       # who it is
  name: hello
  namespace: traineeNN
  labels:
    run: hello                  # added by kubectl run
spec:                           # what you asked for: the desired state
  containers:
  - name: hello
    image: busybox
    args: ["echo", "hello kubernetes"]
    imagePullPolicy: Always     # default for an image without a version tag
  restartPolicy: Never
  nodeName: ecmwf-meditwin-training-…-5t6dz.novalocal   # written by the scheduler
status:                         # what is true now: written by Kubernetes
  phase: Succeeded
  podIP: 10.42.7.85
  containerStatuses:
  - name: hello
    state:
      terminated:
        exitCode: 0
        reason: Completed
```

Three things to notice:

- You gave `kubectl run` a name, an image, a restart policy and a command. Everything else in `spec` (`imagePullPolicy`, `dnsPolicy`, the service account volume, the tolerations…) is a **default** that the API server filled in.
- The command is in `args`, not in `command`: `args` replaces the `CMD` of the image ([Day 2, section 5](../day02/05-application-containerisation.md)). For `busybox`, the `CMD` is `sh`; here it becomes `echo hello kubernetes`.
- `kubectl run` added the label `run: hello`. Labels are how Kubernetes finds objects (section 2.1).

Compare it with `docker inspect` from [Day 2](../day02/03-running-containers.md#id-2-focus-docker-inspect-container): both describe one container, but every Kubernetes object, of every kind, has these same five fields. The `spec` is what you asked for, the `status` is what is true now, and the gap between the two is what the controller loop closes. In section 3.3.1 you write these fields yourself, in a manifest.

### Step 7: cleanup

```bash
kubectl delete pod hello
```

**Checkpoint 1** — your context points at your namespace, you followed a Pod from creation to `Completed`, and you removed it.

```bash
kubectl config view --minify -o jsonpath='{..namespace}'; echo
kubectl get pods
```

```text
traineeNN
No resources found in traineeNN namespace.
```

:::{dropdown} Not there?
- Empty first line: the context has no default namespace, so your kubeconfig is not the one prepared for the course. Ask the instructor.
- `forbidden` on `kubectl get pods`: you are in another namespace, or your kubeconfig is not the one for `traineeNN`. Check `kubectl config current-context`.
- `connection refused` / `Unable to connect`: the kubeconfig is missing or wrong. Check `echo $KUBECONFIG` and ask the instructor.
- `hello` is in `CrashLoopBackOff`: `--restart=Never` is missing. `kubectl delete pod hello` and run Step 2 again.
- `hello` stays in `Pending` or `ErrImagePull`: the last event of `kubectl describe pod hello` says why. Tell the instructor if the node cannot pull the image.
- `pods "hello" already exists`: a `hello` from an earlier try is still there. `kubectl delete pod hello`, then run Step 2 again.
:::

## 2.4 Core takeaway

> **Move from a broad view to a focused one: `get` → `describe` → `logs` → `exec`.**

- The context sets the default namespace: on your VM it is `traineeNN`, so every command works there without `-n`.
- `kubectl` has the same verbs as `docker`, but it talks to an API server and finds Pods by **labels**, not by name.
- `describe` shows the **events**: the first place to look when something does not start.
- Every object has a `spec` (what you asked for) and a `status` (what is true now).

## 2.5 📖 Check yourself

1. You run `kubectl get pods` and see nothing, but your colleague sees Pods. What do you check first?
2. A Pod is not `Running`. Which command shows *why*, and in which part of its output?
3. You delete a Pod created by a Deployment, to stop the application. What happens? How do you really stop it?
4. What is the difference between `kubectl logs <pod>` and `kubectl logs <pod> --previous`?

:::{dropdown} Solutions
**1.** The namespace: `kubectl config view --minify -o jsonpath='{..namespace}'`. Without `-n`, `kubectl` shows only the namespace of the current context.

**2.** `kubectl describe pod <pod>`, the **Events** section at the end: image pull errors, scheduling problems, failed probes.

**3.** The Deployment creates a new Pod at once (controller loop). To stop the application, delete or scale the **Deployment**: `kubectl delete deployment <name>` or `kubectl scale deployment <name> --replicas=0`.

**4.** `logs` shows the current container. After a crash and restart, `--previous` shows the logs of the container that crashed: usually the one you need.
:::

Continue with: [3. Deploying an application](03-deploying-an-application.md).
