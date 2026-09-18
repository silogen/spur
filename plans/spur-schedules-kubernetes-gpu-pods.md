# Plan: Spur schedules Kubernetes GPU pods at GPU granularity

Date 2026-09-18. Based on `spur` main at `9ea65a8`, cluster-forge worktree
`EAI-8560-byok`, and the research in `plans/spur-k8s-gpu-coscheduling.md`.

## 1. Requirement

A customer runs Spur jobs and Kubernetes GPU pods (aim-engine through KServe)
on the same nodes. The customer wants:

- GPU granularity. A node with 8 GPUs can hold 3 GPUs of Spur jobs and 5 GPUs
  of pods at the same time, and the split changes with the load.
- One queue. Spur's priority, QoS, fair-share and reservations apply to the
  pods as well as to the native jobs.
- No change in aim-engine. AIM objects, KServe and the AMD GPU operator stay
  as they are.

## 2. The device conflict

Two allocators hand out the same GPUs and neither sees the other.

- Spur. `spurd` discovers the GPUs from the KFD topology, gives each a
  sequential `device_id`, and reports them once at registration. `spurctld`
  allocates whole GPUs per `device_id` (`crates/spur-sched/src/cons_tres.rs`)
  and writes the allocation to the Raft log. At launch `spurd` installs a
  cgroup-v2 BPF device filter that permits only the allocated
  `/dev/dri/renderD*` and `/dev/kfd`.
- Kubernetes. The AMD device plugin publishes the GPUs to the kubelet with the
  PCI BDF as the ID. The Kubernetes scheduler sees only the count. The kubelet
  picks the IDs and containerd applies a device cgroup for the pod.
- Neither filter bounds the other side. KFD lets any number of processes open
  one GPU, so both launches succeed. The failure appears later as VRAM
  out-of-memory or compute contention in the workload.

Today Spur avoids the conflict with a node-level rule. A node that `spur k8s
up` enrolled has a `k0s_role`, and `NodePlacement::matches()` excludes it from
Spur placement (`crates/spur-sched/src/node_match.rs:160-164`). The job pends
with "ReqNodeNotAvail, Reserved for Kubernetes cluster". The PR that added the
rule (ROCm/spur#558) says "dual-use would need arbitration that doesn't exist
yet. Dual-use workers can be a future opt-in." No issue or roadmap item tracks
that follow-up. On a cluster that Spur does not manage there is no rule at all.

## 3. What the roadmap covers

`plans/implementation-roadmap.md`, dated 2026-03-17, Phase 11 "Kubernetes
Integration" and Phase 13 "Inference Workloads".

| Item | What it says | Status | Does it address the conflict |
|---|---|---|---|
| 11.1 SpurJob CRD + operator | A SpurJob resource becomes a Spur job and Spur creates the pod with `amd.com/gpu: N`. | Done (`crates/spur-k8s`). | No. The kubelet still picks the device IDs. Ordinary pods, such as KServe pods, do not pass through it. |
| 11.2 Virtual kubelet | Kubernetes hands pod specs to `spurd` as `LaunchJobRequest`. Spur executes the pod. | Not started. | Yes, but only by making Spur the only executor. The real kubelet, the AMD device plugin and KServe would no longer run the pod. |
| 11.3 Node pool unification | One `spur nodes` view for native and Kubernetes nodes. | Partial. `node_watcher.rs` registers Kubernetes nodes. The `[kubernetes]` config block exists but nothing reads it. | No. |
| 11.4 GPU topology | XGMI-aware placement. | Not started. Data is collected, the allocator ignores it. | No. |
| 11.5 Gang scheduling | All-or-nothing multi-node placement. | Partial, for heterogeneous job groups. | No. |
| 13.1 to 13.5 Inference | Service jobs, partitions, autoscaling, router, templates, all as native Spur jobs. | Not started. | No. It replaces the pod stack instead of sharing with it. |

Conclusion. The roadmap gives GPU granularity only when Spur executes every
GPU workload itself (11.2). It has no item for a real kubelet and `spurd`
sharing the GPUs of one node. This plan adds that item and builds the
customer's queue requirement on top of it.

## 4. Design

Two layers. The first one removes the conflict. The second one gives Spur the
queue. The second one depends on the first one.

### 4.1 Layer 1: spurd as the device driver of the kubelet

On a dual-use node `spurd` publishes the GPUs to the kubelet. The AMD device
plugin is off on that node. The AMD GPU operator keeps its node labeller, its
metrics exporter and its driver management.

- `spurd` publishes only the GPUs that Spur has not allocated.
- Before `spurd` launches a Spur job, it withdraws the job's GPUs from the
  published list, waits for the kubelet to acknowledge, then installs the
  cgroup filter and launches. The kubelet lowers the allocatable count. Pods
  that already run keep their GPUs.
- When the kubelet gives a GPU to a pod, `spurd` records the GPU as held by a
  foreign owner and reports it to `spurctld`. The scheduler treats it as
  allocated. When the pod ends, `spurd` releases the hold.
- `spurd` rebuilds its hold table on restart from the kubelet's device
  checkpoint or from the DRA prepare calls, and from the controller's
  allocation for its own jobs.

Two protocols can do this. The device plugin API is count-based; the
Kubernetes scheduler binds by count and the kubelet picks the ID, so there is
a window in which both sides claim the last GPU. The loser fails at launch and
retries: Spur through the existing `ResourcesUnavailable` requeue path,
Kubernetes through the Deployment that recreates the pod. The DRA driver API
names devices: `spurd` publishes a `ResourceSlice` with one device per free
GPU, the scheduler allocates a named device, and `spurd` learns it at
`NodePrepareResources`. A `NoSchedule` device taint hides a GPU, a `NoExecute`
taint evicts the pod that holds it. DRA extended-resource mapping lets a
`DeviceClass` answer `amd.com/gpu` requests, so AIM pods need no change. DRA
is the target. The device plugin API is the fallback if the k0s version cannot
give DRA.

### 4.2 Layer 2: Spur places the pods

The pod keeps everything KServe gives it. It gets one more field,
`spec.schedulerName: spur`. The default scheduler ignores such a pod. A Spur
scheduler component picks it up.

1. The component sees an unbound pod with `schedulerName: spur`. It submits a
   placeholder job to `spurctld` with the GPU count, CPU and memory of the
   pod, the node constraints from the pod's affinity and selectors, and the
   account that the pod's namespace maps to. The quota controller already
   defines the account-to-namespace mapping.
2. `spurctld` queues the placeholder as an ordinary job. Priority, QoS,
   fair-share, reservations and backfill apply. It allocates a node and, with
   layer 1, the GPUs.
3. The component binds the pod to the node. The kubelet starts it. `spurd`,
   as the device driver, gives the pod the GPUs of the placeholder.
4. When the pod ends, the component cancels the placeholder. When Spur
   preempts the placeholder, the component evicts the pod. KServe recreates
   the pod and it re-enters Spur's queue.

Two ways to build the component. A kube-scheduler framework plugin is a Go
binary with `Filter`, `Reserve` and `Bind` hooks. A scheduler extender is
three HTTP endpoints, filter, prioritize and bind, that the stock
kube-scheduler calls; `spurctld` can serve them in Rust. The extender is the
smaller change and it keeps the logic in `spurctld`. Spur already renders the
k0s configuration, so it can register the extender itself.

The layer is optional per namespace. A namespace without the field keeps the
default scheduler and gets only layer 1.

## 5. Work packages

In dependency order. Each package is one or more PRs.

### WP1 Stable device identity

- Append `pci_bdf` and `render_minor` to `GpuResource` in `proto/slurm.proto`
  with new tags. `spurd` fills them from `discovery.rs`, which already computes
  the BDF. Compatible: a new field, no renumbering.
- Carry the BDF through `ResourceSet` into the controller's `NodeAllocation`.
  Show it in `spur show node`.
- Test: unit test of the BDF mapping; a proto round-trip test with an old
  message that lacks the field.

### WP2 Live resource updates from the node

Upstream issue ROCm/spur#800. The heartbeat carries no resources, so a hold
change cannot reach the controller.

- Add an optional resource delta to the heartbeat: held device IDs, released
  device IDs, and an inventory version.
- `spurctld` applies the delta with a new WAL operation, `NodeDeviceHold`,
  with `#[serde(default)]` on every new field.
- Test: replay of an old Raft log without the operation; a hold that makes a
  pending job wait; a release that lets it run.

### WP3 Dual-use opt-in

- New config field `[cluster] dual_use = false`. When true, a node with a
  `k0s_role` stays eligible in `NodePlacement::matches()`. The pending-reason
  classifier reports `Resources`, not `K8sReserved`, for such a node.
- Per-node override with a node label, for clusters where only some nodes are
  dual-use.
- Docs: `docs/deployment/managed-kubernetes.rst`, the configuration reference,
  and a new page on GPU sharing.
- Test: the e2e `test_k8s_scheduling.py` gets a dual-use case in which a job
  runs on an enrolled node.

### WP4 spurd as device driver

- New module in `spurd`, active only when `dual_use` is on and the node has a
  k0s worker or single role.
- DRA kubelet plugin: registration on the kubelet plugin socket, a
  `ResourceSlice` per node with one device per free GPU and the attributes
  BDF, product, VRAM, partition, XGMI peers. `NodePrepareResources` records
  the hold and answers with the CDI device name from the spec `spurd` already
  writes to `/etc/cdi/amd.json`. `NodeUnprepareResources` releases the hold.
- Withdraw-before-launch in the executor: taint the job's devices
  `NoSchedule`, confirm, then install the cgroup filter.
- Restart: rebuild the hold table from the kubelet's checkpoint and the
  controller's allocation.
- Fallback: a device plugin API variant with the same hold table, if DRA is
  not available.
- Test: a fake kubelet gRPC peer in the unit tests; an e2e on a Kaytoo VM
  cluster that runs one AIM pod and one Spur job on the same node and reads
  the GPU each one got.

### WP5 Placeholder scheduling for pods

- `spurctld` serves the scheduler-extender endpoints, gated by a config flag.
  Filter maps the pod to a placeholder job spec and answers with the nodes
  Spur allows. Bind waits for the allocation and binds the pod. A watch on
  pod deletion cancels the placeholder. A preempted placeholder evicts its
  pod.
- k0s configuration: Spur renders the `KubeSchedulerConfiguration` with the
  extender and the profile name `spur`.
- Accounting: the placeholder runs under the account of the namespace and
  counts in fair-share and TRES usage.
- Test: unit tests of the pod-to-job mapping; an e2e in which a pod with
  `schedulerName: spur` pends while a higher-priority Spur job holds the GPUs
  and runs after it ends.

### WP6 cluster-forge and byok

- `amd-gpu-operator-config`: on dual-use nodes set
  `devicePlugin.enableDevicePlugin: false`, keep `enableNodeLabeller: true`
  and the metrics exporter. If the operator's own DRA driver is installed,
  turn it off on those nodes; the operator enforces one driver per node.
- A Kyverno mutate policy that sets `schedulerName: spur` on pods in the AIM
  namespaces, for the case in which the AIM CRDs do not pass the field
  through. Kyverno is already in the `default` profile.
- A byok capability `gpu.spur-arbitration` with a probe that reads the
  `ResourceSlice` of a node.
- Docs in `byok/docs`.

### WP7 Upstream housekeeping

- File the follow-up issue that PR ROCm/spur#558 promised, with this plan as
  the design.
- Propose a roadmap entry between 11.3 and 11.4, "Dual-use nodes: spurd as
  the kubelet device driver", and a note on 11.2 that it covers clusters where
  Spur executes the pods itself.
- Ask upstream to reserve the plugin name `inference`, or add a CI check in
  cluster-forge that `spur inference` is not a built-in command.

## 6. Risks and open points

- k0s version. DRA core is GA in Kubernetes 1.34. Device taints and
  extended-resource mapping are stable in 1.37 and beta before. Spur pins k0s
  v1.36.2 and controls the pin. Confirm which feature gates are on by default
  in the pinned version before WP4 starts.
- Compute partitions. In CPX mode one GPU is 8 devices that share the BDF. The
  device key must include the partition index. Spur reads the mode, the DRA
  driver publishes `amdgpu-partition` devices.
- Preemption of pods. `NoExecute` evicts a pod at once. Check KServe's restart
  behaviour and the cache volume of an AIM before the default policy is set.
- Backfill. A GPU that a pod holds has no end time. Backfill must treat it as
  unavailable for the whole planning window.
- Reservations. An advance reservation must withdraw its GPUs from the kubelet
  before its start time, or a pod can take them.
- Scheduling latency. The placeholder path adds Spur's scheduler interval, 2
  seconds by default, to each pod start. Acceptable for an inference replica,
  not for a pod that starts every second.
- Node identity. On a dual-use node the native `spurd` is the only Spur agent.
  The `spur-k8s` operator mode must not also register the node.

## 7. Out of scope

- Time-sharing one GPU between a pod and a Spur job. Nothing on AMD enforces
  it; KFD lets both processes open the device with no VRAM or compute share.
- Roadmap items 11.2, 11.4, 11.5 and 13.x.
- A change in aim-engine or KServe.
