# 11. GPU workloads and scheduling

👀 This section is a demonstration by the instructor: the training cluster has only one GPU, so it cannot be shared among all participants. Watch, and do not apply the examples yourself.

Section 7 was about *how many* Pods run. This section is about *where* a Pod can run: some workloads need hardware that only a few nodes have, such as a GPU, and the scheduler must send them there.

**Symbols:** ⌨️ you type the commands · 👀 the instructor shows it, you watch · 📖 reference: read it after the lesson, or when you need it; it is not covered in class.

**Slides:** the slides used in class are in one file, {download}`day03-kubernetes-slides.pdf`.

## 11.1 GPU workloads in Kubernetes

:::{note} Jargon: *extended resource*
Kubernetes knows CPU and memory by itself. Other hardware, such as a GPU, is an **extended resource**: a plugin on the node (here the NVIDIA device plugin) tells Kubernetes how many units the node has, under a name such as `nvidia.com/gpu`. Pods then request them in `resources`, like CPU and memory, but only in whole units.
:::

GPU resources are exposed as extended resources, typically `nvidia.com/gpu`. With Docker you would use `docker run --gpus 1` (not covered on Day 2); here the GPU is one more line in `resources`, and the scheduler finds a node that has one.

**Prerequisites**:

- GPU nodes and working drivers.
- NVIDIA device plugin (usually via DaemonSet), which advertises `nvidia.com/gpu` to the scheduler.
- A **RuntimeClass** (here `nvidia`) that starts the container with the NVIDIA container runtime, so that the GPU and the driver libraries are visible inside it.
- Node labeling/taints strategy.

Pod example, `gpu-demo.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-demo
spec:
  restartPolicy: Never
  runtimeClassName: nvidia
  containers:
    - name: cuda
      image: nvidia/cuda:12.4.1-runtime-ubuntu22.04
      command: ["nvidia-smi"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

Two lines make this Pod a GPU Pod:

- `resources.limits: nvidia.com/gpu: 1` asks the **scheduler** for a node with a free GPU.
- `runtimeClassName: nvidia` asks the **kubelet** on that node to start the container with the NVIDIA runtime. Without it, the Pod may be scheduled on the GPU node, but `nvidia-smi` does not find the GPU.

A GPU is requested in `limits` only, and in whole units: unlike CPU, a Pod cannot ask for half a GPU (unless the cluster uses GPU sharing, such as MIG or time-slicing).

Run it and read the output:

```bash
kubectl apply -f gpu-demo.yaml
kubectl get pod gpu-demo -w
```

```text
NAME       READY   STATUS              RESTARTS   AGE
gpu-demo   0/1     ContainerCreating   0          2s
gpu-demo   1/1     Running             0          78s
gpu-demo   0/1     Completed           0          79s
```

The Pod stays in `ContainerCreating` for more than a minute the first time: the node pulls the CUDA image, which is large (the CUDA runtime libraries alone are over a gigabyte). `nvidia-smi` itself runs in about one second. Run the demo again and it starts in a few seconds, because the image is now cached on the node.

When the status is `Completed`, stop the watch with `Ctrl+C` and read the log:

```bash
kubectl logs gpu-demo
```

```text
Thu Oct 22 15:02:36 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.159.03             Driver Version: 580.159.03     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  GRID A100D-1-10C               On  |   00000000:05:00.0 Off |                   On |
| N/A   N/A    P0            N/A  /  N/A  |                  N/A   |     N/A      Default |
|                                         |                        |              Enabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| MIG devices:                                                                            |
+------------------+----------------------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |              Shared Memory-Usage |        Vol|        Shared         |
|      ID  ID  Dev |                Shared BAR1-Usage | SM     Unc| CE ENC  DEC  OFA  JPG |
|                  |                                  |        ECC|                       |
|==================+==================================+===========+=======================|
|  0    0   0   0  |               1MiB /  8812MiB    | 14      0 |  1   0    0    0    0 |
|                  |               0MiB /  4096MiB    |           |                       |
+------------------+----------------------------------+-----------+-----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

How to read it:

- **`GRID A100D-1-10C`**: the node does not get a whole physical card. It gets a virtual GPU (NVIDIA GRID/vGPU) that is a slice of an A100: `1` compute slice, `10C` about 10 GB of memory for compute workloads.
- **`MIG M. Enabled`** and the **MIG devices** table: the GPU is partitioned with Multi-Instance GPU. Our Pod sees one MIG instance, with about 8.8 GB of usable memory (`8812MiB`) and 14 streaming multiprocessors (`SM`).
- **`Driver Version` / `CUDA Version: 13.0`**: the driver is installed on the **node**, not in the image. The image brings the CUDA *runtime* 12.4; a newer driver runs older CUDA runtimes, so the two do not have to match exactly.
- **`No running processes found`**: correct, `nvidia-smi` only reads the GPU state and exits. A training or inference job would appear here with its memory use.

Remove the Pod at the end, so that the GPU is free again:

```bash
kubectl delete pod gpu-demo
```

## 11.2 Scheduling controls for GPUs

:::{note} Jargon: *taint* and *toleration*
A **taint** on a node keeps Pods away from it: for example, the GPU node is tainted so that ordinary Pods do not use its CPU and memory. A Pod with a matching **toleration** may still be scheduled there. The taint keeps the others out; the toleration lets the right ones in; a `nodeSelector` makes sure the GPU Pod goes there and nowhere else.
:::

**Common controls**:

- nodeSelector/nodeAffinity to target GPU-capable nodes.
- tolerations for tainted GPU nodes.
- priority classes for high-value workloads.

On the training cluster, `nvidia.com/gpu: 1` is enough: the demo Pod above has no `nodeSelector` and no toleration, and the scheduler still sends it to the GPU node, the only one with a free `nvidia.com/gpu`. On clusters that label and taint their GPU nodes, the Pod spec gets two more blocks:

```yaml
spec:
  nodeSelector:
    nvidia.com/gpu.present: "true"     # run only on GPU nodes
  tolerations:
    - key: nvidia.com/gpu              # accept the taint that keeps other Pods away
      operator: Exists
      effect: NoSchedule
```

Ask the administrators of your cluster which label and which taint they use: `kubectl describe node <gpu-node>` shows them under `Labels` and `Taints`.

### 11.2.1 📖 How to check and influence scheduling

The GPU is only the example of this section: the scheduler places **every** Pod of the cluster, and the same rules apply to any workload. Use them, for example, to keep the replicas of `climate-telemetry` on different nodes, to run a database only on nodes with fast disks, or to reserve some nodes for one team.

For every new Pod, the scheduler first **filters** the nodes where the Pod can run (enough free `requests`, matching labels, tolerated taints), then **scores** the remaining ones and picks the best. `nodeSelector` and tolerations, used above for the GPU, are only two of the tools that change this choice. These are the main ones, all written in the Pod spec except PriorityClass:

| Mechanism | What it does |
|---|---|
| [`nodeSelector`](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) | Simple choice of nodes by label: the Pod runs only on nodes with all the given labels |
| [`nodeAffinity`](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity) | The same, with richer rules (`In`, `NotIn`, `Exists`…), as a requirement or as a preference |
| [`podAffinity`](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity) | Places a Pod close to other Pods (same node or same zone), for example next to its cache |
| [`podAntiAffinity`](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity) | Keeps Pods apart, for example two replicas never on the same node |
| [`topologySpreadConstraints`](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) | Spreads the replicas evenly across nodes, zones or other domains |
| [Taints and tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) | Reserve nodes for some kinds of workload: the taint is on the node, the toleration in the Pod |
| [PriorityClass](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) | Sets the priority of a Pod in the scheduling queue; when the cluster is full, a Pod with higher priority can evict (*preempt*) Pods with lower priority |

When a Pod stays `Pending`, `kubectl describe pod <pod>` explains which filters failed, node by node: for example `0/9 nodes are available: 1 Insufficient nvidia.com/gpu, 8 node(s) didn't match Pod's node affinity/selector`.

The official documentation: [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/) explains how the scheduler filters and scores nodes; [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) has examples of the first four mechanisms; [Scheduler Configuration](https://kubernetes.io/docs/reference/scheduling/config/) is for cluster administrators, who can change how the scheduler itself filters and scores.

## 11.3 📖 Operational checks

```bash
kubectl get nodes -L nvidia.com/gpu.present,meditwin.io/role
# Fallback: discover which labels your cluster actually uses
kubectl get nodes --show-labels | grep -E 'nvidia.com/gpu.present|meditwin.io/role|accelerator'
kubectl describe node <gpu-node-name>
```

**Common issues**:

- Pending Pods due to missing GPU resource or taint mismatch. With one GPU, a second GPU Pod stays `Pending` until the first one is deleted.
- Pod in `StartError` or `ContainerCannotRun`, with `nvidia-smi`: executable file not found in the events: `runtimeClassName: nvidia` is missing. The NVIDIA runtime is what puts `nvidia-smi` and the driver into the container.

## 11.4 Core takeaway

> **A GPU is one more resource in `limits`; labels, taints and tolerations decide which nodes may receive the Pod.**

- `nvidia.com/gpu: 1` asks the **scheduler** for a node with a free GPU; `runtimeClassName: nvidia` asks the **kubelet** to make it visible in the container.
- GPUs are requested in whole units (unless the cluster shares them, for example with MIG).
- The driver is on the node; the image brings the CUDA runtime.
- A GPU is scarce: delete the Pod when it has finished, so that the next one can run.

## 11.5 📖 Check yourself

1. The GPU Pod is `Pending`. Give two possible causes.
2. The Pod runs, but `nvidia-smi` finds no GPU. What is missing?
3. The node driver reports CUDA 13.0 and the image contains CUDA 12.4. Is that a problem?

:::{dropdown} Solutions
**1.** No node has a free `nvidia.com/gpu` (here: another Pod already uses the only GPU), or the GPU node has a taint that the Pod does not tolerate.

**2.** `runtimeClassName: nvidia`: without the NVIDIA runtime, the container does not see the GPU device and the driver libraries.

**3.** No: a newer driver runs older CUDA runtimes. The opposite (an image that needs a newer CUDA than the driver supports) would fail.
:::

Continue with: [12. Helm: installing charts and writing your own](12-helm-jupyterhub.md).
