# 7. Scaling

In section 4.4 you scaled `climate-telemetry` by hand with `kubectl scale`, and watched the Service spread requests across the replicas. This section makes that automatic: first the **resources** each Pod asks for, then the **Horizontal Pod Autoscaler**, which changes the number of replicas with the load.

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

## 7.1 Right-size resources first

Autoscaling works well only if **requests and limits** are meaningful.

- **requests** define guaranteed minimum resources, used by the **scheduler** to pick a node.
- **limits** cap maximum resource usage, enforced by the **kernel** on that node.

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

The limits are the cgroups you met on Day 2 (`--memory` in [section 1](../day02/01-containers-and-docker.md#id-2-control-groups-resource-allocation-throttling), and the `-m 30m` OOM test in [Day 2, section 7](../day02/07-lab.md#experiment-6-2-control-group-memory-boundary-enforcement)). Requests are the new part: Docker had one host and no need to choose; the scheduler has many and must know how much to reserve.

### ⌨️ Hands-on 5: exit code 137, again

Repeat the Day 2 OOM test from [section 7](../day02/07-lab.md#experiment-6-2-control-group-memory-boundary-enforcement) as a Pod.

#### Step 1: write the Pod, and predict

Work in the folder of `climate-telemetry`:

```bash
cd ~/climate-service-k8s
```

Create `oom-demo.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  restartPolicy: Never
  containers:
    - name: python
      image: python:3.12-alpine
      command: ["python3", "-c", "x = bytearray(50 * 1024 * 1024); print('allocated 50 MiB')"]
      resources:
        limits:
          memory: 30Mi
```

**Before you apply it, predict:** the program allocates 50 MiB, the limit is 30 MiB. What will `STATUS` show, and which exit code?

#### Step 2: apply it, and observe

```bash
kubectl apply -f oom-demo.yaml
sleep 10
kubectl get pod oom-demo
kubectl describe pod oom-demo | grep -A3 "Last State\|State:"
```

```text
NAME       READY   STATUS      RESTARTS   AGE
oom-demo   0/1     OOMKilled   0          12s
...
      Reason:       OOMKilled
      Exit Code:    137
```

:::{dropdown} What happened, and why
The kernel stopped the process when it went over the cgroup limit: same kernel, same OOM killer, same `128 + 9 = 137` as on Day 2. What Kubernetes adds is the *reason* in the API (`OOMKilled`), and, in a Deployment, the automatic restart with a growing back-off (`CrashLoopBackOff`). This Pod has `restartPolicy: Never`, so it stays stopped.
:::

#### Step 3: raise the limit

Change the limit to `100Mi` in `oom-demo.yaml`. A Pod spec cannot be changed in place, so delete the Pod and create it again:

```bash
kubectl delete pod oom-demo
kubectl apply -f oom-demo.yaml
```

The Pod should now complete.

**Checkpoint 5** — you saw `OOMKilled` / exit code `137` with `30Mi`, and a successful run with `100Mi`.

```bash
kubectl get pod oom-demo
kubectl logs oom-demo
```

```text
NAME       READY   STATUS      RESTARTS   AGE
oom-demo   0/1     Completed   0          20s
allocated 50 MiB
```

:::{dropdown} Not there?
- Still `OOMKilled`: the limit in `oom-demo.yaml` was not changed, or the old Pod was not deleted before `apply` (a Pod spec cannot be changed in place).
- `Completed` already with `30Mi`: rare, but possible if the node reports memory differently. Lower the limit to `20Mi` and try again.
:::

#### Step 4: cleanup

Remove the test Pod and its file, so that `~/climate-service-k8s` keeps only the files of `climate-telemetry`:

```bash
kubectl delete pod oom-demo --ignore-not-found
rm -f oom-demo.yaml
```

The `climate-telemetry` Deployment from Hands-on 2 already has `requests` and `limits`; that is what makes the next step possible.

## 7.2 Horizontal Pod Autoscaler (HPA)

HPA changes replica count based on **metrics** (CPU, memory, custom metrics). There is no Compose equivalent: `docker compose up --scale api=3` was a manual number, and so was `kubectl scale` in section 4.4.

### ⌨️ Hands-on 6: watch the HPA react

#### Step 1: start from one replica

Reset the Deployment to one replica, so that the lab starts from a known baseline:

```bash
kubectl scale deployment climate-telemetry --replicas=1
kubectl get deployment climate-telemetry
```

#### Step 2: create the HPA

In `~/climate-service-k8s`, create `hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: climate-telemetry-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: climate-telemetry
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
    scaleDown:
      stabilizationWindowSeconds: 60
```

| Field | What it does | Compose twin |
|---|---|---|
| `scaleTargetRef` | The Deployment whose `replicas` the HPA changes | — |
| `minReplicas: 2` / `maxReplicas: 6` | The HPA never goes below 2 or above 6 replicas | `docker compose up --scale api=N`, but with a range instead of a fixed number |
| `metrics` (`cpu`, `Utilization`, `70`) | Target: on average, each Pod uses 70 % of its `requests.cpu` | — |
| `behavior.scaleUp.stabilizationWindowSeconds: 0` | When the load rises, add Pods at once (this is the default; a higher value waits, to ignore short peaks) | — |
| `behavior.scaleDown.stabilizationWindowSeconds: 60` | After the load drops, wait 60 s before removing Pods (the default is 300 s) | — |

All other scaling behaviour is the Kubernetes default.

```bash
kubectl apply -f hpa.yaml
kubectl get hpa climate-telemetry-hpa
```

**How it works**:

1. Metrics Server exposes usage metrics (`kubectl top pods` reads the same source).
2. HPA controller compares current usage with the target, *as a percentage of `requests.cpu`*.
3. Controller adjusts Deployment replicas.

Point 2 is why requests matter: with `requests.cpu: 50m`, 70 % means 35 millicores per Pod. No requests, no percentage, no scaling.

#### Step 3: predict

**Before you start the load, predict:**

1. The Deployment has 1 replica and the HPA has `minReplicas: 2`. How many replicas run before any load?
2. Under load, what limits the number of replicas?
3. After the load stops, how quickly do the replicas go down, and to how many?

#### Step 4: start the load

Generate load from inside the cluster, through the Service, from a temporary Pod:

```bash
kubectl run loadgen --image=busybox:1.36 --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://climate-telemetry/health > /dev/null; done'
```

`kubectl run` without `-it` starts the Pod in the background and gives you the prompt back at once: the load keeps running inside the cluster.

#### Step 5: watch the HPA

In the same terminal:

```bash
kubectl get hpa climate-telemetry-hpa -w
```

Within a couple of minutes, `TARGETS` should go above 70 % and `REPLICAS` should grow. Stop the watch with `Ctrl+C`.

Optionally, in a second terminal, watch the new Pods appear at the same time:

```bash
kubectl get pods -l app=climate-telemetry -w
```

#### Step 6: stop the load, and watch the scale-down

```bash
kubectl delete pod loadgen
kubectl get hpa climate-telemetry-hpa -w
```

After the stabilisation window (about one minute in this lab profile), replicas should go down toward `minReplicas`. Compare with your predictions:

:::{dropdown} What happened, and why
1. **2**, as soon as the HPA exists: it never lets the Deployment run fewer than `minReplicas`, even without load.
2. `maxReplicas: 6`, and the free resources of the cluster: a new Pod is scheduled only where its `requests` fit. The **load itself** matters too: the HPA adds Pods only while the average CPU is clearly above the target, and it ignores differences of less than 10 % (74 % on a 70 % target changes nothing). That is why the replicas can stay at 5 for a few minutes, and go to 6 only when the CPU rises to about 78 %.
3. Not at once: the HPA waits for the stabilisation window (60 s here) before it scales down, so that a short pause in the traffic does not remove Pods. Then it goes back to `minReplicas`, **2**, not to the 1 replica of the Deployment.
:::

#### Step 7: cleanup

Remove the HPA, so that it does not keep overriding `replicas:`:

```bash
kubectl delete hpa climate-telemetry-hpa
```

**Checkpoint 6** — under load the HPA raised `REPLICAS` above `2`; without load it went back to `2`; the HPA is now deleted.

What you should have seen in `kubectl get hpa climate-telemetry-hpa -w` (your numbers will differ a little; the replicas can stay at 5 for a few minutes before they reach 6):

```text
NAME                    REFERENCE                      TARGETS        MINPODS   MAXPODS   REPLICAS
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 17%/70%   2         6         2
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 164%/70%  2         6         2
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 191%/70%  2         6         5
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 80%/70%   2         6         5
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 74%/70%   2         6         5
…   (a few minutes at 5 replicas, between 70 % and 77 %)
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 78%/70%   2         6         5
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 77%/70%   2         6         6
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 66%/70%   2         6         6
…   (kubectl delete pod loadgen)
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 15%/70%   2         6         6
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 2%/70%    2         6         6
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 2%/70%    2         6         5
climate-telemetry-hpa   Deployment/climate-telemetry   cpu: 2%/70%    2         6         2
```

And now:

```bash
kubectl get hpa
kubectl get pod loadgen
```

```text
No resources found in traineeNN namespace.
Error from server (NotFound): pods "loadgen" not found
```

:::{dropdown} Not there?
- `TARGETS <unknown>/70%`: metrics are not available yet (wait one minute), or the Deployment has no `requests.cpu`.
- `TARGETS` stays low: the load Pod is not running (`kubectl get pod loadgen`) or calls a wrong URL (`kubectl logs loadgen`).
- `loadgen` still exists: delete it, otherwise it keeps loading the cluster for everybody.
:::

## 7.3 📖 Vertical and cluster scaling

- **Vertical Pod Autoscaler (VPA)**: adjusts requests/limits.
- **Cluster autoscaling**: adds/removes nodes.

**Use case split**:

- HPA for bursty web/API traffic.
- VPA for long-running services with uncertain sizing.
- Cluster autoscaler when Pods are pending due to node exhaustion.

## 7.4 📖 Operational checks

These checks are most useful while the HPA exists. If you already deleted it in the previous step, re-create it first:

```bash
cd ~/climate-service-k8s
kubectl apply -f hpa.yaml
```

```bash
kubectl top pods
kubectl get hpa
kubectl describe hpa climate-telemetry-hpa
```

When you have finished, delete the HPA again: in section 8 it would fight with the `replicas:` of `deployment.yaml`.

```bash
kubectl delete hpa climate-telemetry-hpa
```

**Common issues**:

- HPA not scaling because metrics-server is absent, or because `requests.cpu` is not set.
- Overly low requests causing CPU throttling and unstable autoscaling.

## 7.5 Core takeaway

> **Requests tell the scheduler what a Pod needs; limits tell the kernel what it may use. The HPA scales on usage *as a percentage of requests*: without requests, no autoscaling.**

- **Limits** are the cgroups of Day 2: over the memory limit, the kernel kills the process (`OOMKilled`, exit code 137).
- **Requests** are new: the scheduler places a Pod only on a node with that much free capacity.
- The **HPA** adds and removes replicas between `minReplicas` and `maxReplicas`, following a metric such as CPU.
- Scaling down waits for a stabilisation window; scaling up needs free resources in the cluster.
- **VPA** changes the size of Pods; the **cluster autoscaler** changes the number of nodes.

## 7.6 📖 Check yourself

1. A Pod has `requests.cpu: 50m` and uses 100m. What CPU utilisation does the HPA see?
2. You create an HPA for a Deployment without `requests.cpu`. What does `kubectl get hpa` show, and why?
3. A container is restarted again and again, and `kubectl describe pod` shows `Reason: OOMKilled`. Give two possible fixes.
4. The HPA wants 6 replicas, but 2 of them stay `Pending`. What is the probable cause?
5. Why does the HPA wait before scaling down, but not before scaling up?

:::{dropdown} Solutions
**1.** 200 %: utilisation is measured as a percentage of the request, not of the node.

**2.** `TARGETS <unknown>/70%`: without a request there is no 100 %, so no percentage, and no scaling.

**3.** Raise `limits.memory` if the application really needs more memory; or reduce the memory the application uses (a leak, a cache that is too big, too much data loaded at once).

**4.** The cluster has no node with enough free capacity for their `requests`. `kubectl describe pod` shows `Insufficient cpu` or `Insufficient memory` in the events. A cluster autoscaler would add a node.

**5.** Scaling up late means slow or failed requests for users; scaling down too early means removing Pods during a short pause and adding them back a moment later. The stabilisation window avoids that flapping.
:::

Continue with: [8. Configuration and Secrets](08-configuration-and-secrets.md).
