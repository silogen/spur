# Spur and Kubernetes GPU workloads on the same node

Research report, 2026-09-18. Sources: `spur` main at `9ea65a8`, cluster-forge
worktree `EAI-8560-byok`, and the upstream ROCm projects on GitHub.

## 1. The question

Spur has its own GPU allocator. Kubernetes GPU pods (aim-engine through KServe,
GPUs from the AMD GPU operator) have their own. How can a Kubernetes pod and a
Spur job share the GPUs of one node without a conflict?

## 2. How each side allocates a GPU today

### 2.1 Spur, native-host path

| Step | Where | What happens |
|---|---|---|
| Discovery | `crates/spur-devices/src/cdi/discovery.rs` | `spurd` reads `/sys/class/kfd/kfd/topology/nodes/*/properties`. It knows the render minor, the PCI BDF (`location_id` + `domain`), the KFD `unique_id`, the VRAM size, the XGMI links and the compute/memory partition mode. |
| Identity | `crates/spur-devices/src/registry/device_registry.rs:177-187` | The registry gives each GPU a sequential `device_id` per GRES name. This integer, not the BDF or render minor, is the identity in the scheduler, in the WAL and in dispatch. |
| Report | `crates/spurd/src/reporter.rs:276-326` | The inventory goes to `spurctld` once, at registration. A heartbeat carries no resources (upstream issue ROCm/spur#800). |
| Allocation | `crates/spur-sched/src/cons_tres.rs:153-216` | `NodeAllocation.gpu_allocated: Vec<bool>`. Whole GPUs only, one job per `device_id`. The allocation is persisted in `WalOperation::JobStart.per_node_alloc`. |
| Isolation | `crates/spurd/src/device_cgroup.rs`, `executor.rs:1184` | A cgroup-v2 BPF device filter, default deny, allows only the allocated `/dev/dri/renderD*` and `/dev/kfd`. `ROCR_VISIBLE_DEVICES` and `HIP_VISIBLE_DEVICES` are set as a hint. |

Spur has no per-GPU "held by someone else" state. The only external control is
a whole-node drain. The `[[devices.gres]]` config with `file =
"/dev/dri/renderD[128-131]"` and `auto_detect = false` can give Spur a static
subset of the GPUs (`crates/spur-core/src/config.rs:1285-1350`).

### 2.2 Kubernetes, AMD device plugin path

1. The AMD device plugin (`rocm/k8s-device-plugin`, in the operator's
   `DeviceConfig`) enumerates the GPUs from `/sys/module/amdgpu/drivers/pci:amdgpu`
   and publishes them to the kubelet with the PCI BDF as the device ID, for
   example `0000:19:00:0`. The extended resource is `amd.com/gpu: 8`.
2. The Kubernetes scheduler sees only the count. It binds a pod to a node that
   has enough free count.
3. The kubelet device manager picks the concrete device IDs on the node
   (`healthyDevices` minus allocated, then `GetPreferredAllocation` of the
   plugin) and writes them to
   `/var/lib/kubelet/device-plugins/kubelet_internal_checkpoint`.
4. `Allocate` of the plugin returns `/dev/kfd`, `/dev/dri/card<N>` and
   `/dev/dri/renderD<N>` for the chosen IDs. No env var, no CDI name. containerd
   applies a device cgroup that permits only these nodes.
5. The kubelet never takes a device back from a running pod. A plugin can
   mark a device `Unhealthy` or drop it from `ListAndWatch`; the allocatable
   count goes down, running pods stay.

The stock plugin has exactly two flags, `-pulse` and `-resource_naming_strategy`
(verified in `cmd/k8s-device-plugin/main.go` on master). There is no
allowlist, no exclusion, no device count limit and no sharing mode. The
`AMD_GPU_DEVICE_COUNT` setting in the user guide is not in the code. The
AMD GPU operator `DeviceConfig` has no "exclude GPUs" field either.

The host process that wants to know which GPU a pod holds reads the kubelet
Pod Resources API: `/var/lib/kubelet/pod-resources/kubelet.sock`, `List` and
`GetAllocatableResources`. On a Spur-managed k0s the kubelet root is
`/var/lib/k0s/kubelet`, which byok already sets as `podResourceAPISocketPath`
for the metrics exporter (`byok/profiles/default.yaml:22-28`).

### 2.3 Why the conflict is silent at the OS level

KFD lets any number of processes open one GPU. Neither side checks the other:
Spur's cgroup filter bounds only Spur's own job processes, containerd's device
cgroup bounds only the pod. Two independent allocators that pick the same BDF
both succeed at launch. The failure shows later, as VRAM out-of-memory or
compute contention in the workload.

### 2.4 The two Spur to Kubernetes modes, and the gate that exists today

**Managed k0s** (`docs/deployment/managed-kubernetes.rst`, `crates/spurd/src/cluster.rs`).
Spur owns the nodes and starts k0s on them. Upstream PR ROCm/spur#558
(SPUR-114) added a gate: a node with a `k0s_role` is excluded from Spur batch
placement (`crates/spur-sched/src/node_match.rs:160-164`,
`Node::is_k0s_reserved`), with the pending reason `ReqNodeNotAvail, Reserved for
Kubernetes cluster`. The PR text says: "Dual-use workers can be a future
opt-in." Known hole: a Spur job that already runs when `spur k8s up` claims the
node keeps its GPUs. In this mode the reported conflict cannot happen for a
new Spur job, because Spur places nothing on a k0s node. The node is the unit
of partition, so a single-node cluster such as `itg1` is all Kubernetes.

`spurd` also writes a CDI spec `/etc/cdi/amd.json` with kind `amd.com/gpu` for
k0s's containerd (`cluster.rs:508-538`). The comment there says: "`amd.com/gpu`
is also claimed by the ROCm k8s-device-plugin; acceptable for Phase-2
containerd injection, a native spur-device-plugin is a later milestone." That
later milestone is the natural home of the solution.

**Spur inside Kubernetes** (`docs/deployment/kubernetes.rst`, `crates/spur-k8s`).
The `spur-k8s` operator registers Kubernetes nodes as Spur nodes from
`allocatable["amd.com/gpu"]` (`node_watcher.rs:234`) and runs each Spur job as
one pod with `limits: amd.com/gpu: N` (`agent.rs:212-300`, `989-1000`). The
kubelet is the only allocator. No conflict is possible by construction, but a
Spur job is then a container, not a native process, and the native features
(`spurstepd`, cgroup limits, PMIx steps, srun) do not apply.

**If the conflict was seen on a node where both spurd and a kubelet run and
the cluster is not Spur-managed** (for example an RKE2 node of cluster-bloom
with a native `spurd`), then no gate exists at all, and 2.3 is the full
explanation.

### 2.5 The common key between the two identities

Spur knows the BDF of every GPU (`bdf_from_location_id`, `discovery.rs:247`).
The device plugin's ID is the BDF with a colon before the function
(`0000:19:00:0`). The AMD DRA driver publishes `resource.kubernetes.io/pciBusID`
(`0000:19:00.0`). So a stable mapping `device_id <-> BDF <-> renderD<minor>`
exists on both sides today. It is not in the proto yet; `GpuResource` carries
`device_id`, type, memory, peers and link only.

## 3. Prior art

Every production integration makes Kubernetes own the GPUs:

- **SUNK (CoreWeave)**: each Slurm node is a pod that requests the GPUs.
  Ordinary pods can use the SUNK scheduler, which creates a placeholder Slurm
  job for the pod. Slurm decides placement, the kubelet still hands out the
  device IDs.
- **Slinky slurm-bridge (SchedMD)**: kubelet and slurmd on the same host, a pod
  becomes an "external job" in Slurm, whole-node allocations by default. The
  documentation says a node must not run native Slurm jobs and bridge pods at
  the same time.
- **Soperator (Nebius)**, **kube-slurm**: Slurm inside pods, or Slurm jobs as
  pods.
- **NVIDIA**: no way to hide a subset of GPUs from the device plugin (issue
  435 closed as not planned). The operator can only turn the plugin off per
  node.

None of them splits the GPUs of one node dynamically between a host scheduler
and the kubelet. The Kubernetes primitives that permit such a split are:
(a) the Pod Resources API to learn what the kubelet handed out, (b) a device
plugin that withdraws host-held IDs, (c) under DRA, a smaller `ResourceSlice`
or a device taint (`NoSchedule` hides a device, `NoExecute` also evicts).

## 4. Options

### Option A. Node-level partition (what exists)

`spur k8s up --nodes/--partition/--selector` scopes k0s to some nodes. The rest
stay native. No code change.

- For: works now, no race, matches what every prior-art system does.
- Against: a node is all Spur or all Kubernetes. Useless on a single node and
  wasteful when both sides are half idle.

### Option B. Static GPU split inside a node

Spur takes GPUs 0-3, Kubernetes takes 4-7, by configuration.

Spur side, small work:

1. `[devices] auto_detect = false` and one `[[devices.gres]]` entry with the
   render nodes. Needs a test that the GRES path gives the same env and cgroup
   result as the CDI path.
2. An opt-in that relaxes the node gate: for example `[cluster] dual_use =
   true`, or a per-node label. `is_k0s_reserved()` then no longer excludes the
   node; the scheduler trusts the reduced inventory. The pending-reason
   classifier must not emit `K8sReserved` for such a node.

Kubernetes side, the hard part, because the stock plugin cannot hide a GPU:

- B1. Add a device allowlist or exclude flag to `ROCm/k8s-device-plugin` and
  expose it as a `devicePluginArguments` key in the operator. Both projects are
  in the ROCm organization, so this is an internal upstream change. Until it
  lands, a fork image in the `DeviceConfig`.
- B2. DRA with a `DeviceTaintRule`: an admin object that selects devices by
  `resource.kubernetes.io/pciBusID` and taints them `NoSchedule`. No driver
  change. Needs the AMD DRA driver (GPU operator 1.5.0 or later, cluster-forge
  pins 1.4.1, 1.5.1-beta.0 is on disk), Kubernetes 1.32 or later for DRA and
  the device-taint feature (stable in 1.37, beta before). Spur pins k0s
  v1.36.2, and Spur controls that pin.
- B3. Spur ships its own device plugin that publishes only the GPUs in the
  Kubernetes set. This is B and C sharing one component; see C.

For: simple to reason about, no race, keeps Spur's topology placement inside
its own set. Against: static, both halves idle at different times, two config
files must agree on the split, and the k8s side needs new code somewhere.

### Option C. Spur as the single source of truth, dynamic per GPU

Spur's roadmap (`plans/implementation-roadmap.md`, Phase 11) says: "Spur
becomes the scheduling brain for GPU workloads across both native-host and
Kubernetes." The `cluster.rs` comment plans a native spur-device-plugin. This
option builds that.

On a dual-use node `spurd` is the kubelet's device driver:

1. `spurd` publishes to the kubelet the GPUs that Spur has not allocated. When
   `spurctld` allocates GPU 0 to a job, `spurd` withdraws GPU 0 before it
   launches the job. The allocatable count drops; running pods are untouched.
2. When the kubelet gives GPU 5 to a pod, `spurd` sees it (it is the plugin
   that answered `Allocate`, or, under DRA, `NodePrepareResources` names the
   device). `spurd` reports GPU 5 as externally held. `spurctld` records it,
   and the scheduler treats it as allocated by a foreign owner.
3. When the pod ends, the hold is released and the GPU returns to both pools.

Two ways to be the driver:

- **Device plugin API**: count-based. The k8s scheduler only sees a count, the
  kubelet picks the ID. There is a window between a scheduler bind and a Spur
  withdrawal in which both pick the same count; the loser fails at `Allocate`
  and the pod is recreated by its Deployment or KServe. Spur's own loser path
  exists already: dispatch confirmation fails with `ResourcesUnavailable` and
  the job is requeued.
- **DRA driver** (preferred): `spurd` publishes a `ResourceSlice` with one
  device per free GPU, attributes `pciBusID`, product, VRAM, partition. The
  k8s scheduler allocates named devices, so `spurd` learns the exact GPU at
  prepare time with no polling. To hide a GPU Spur puts a `NoSchedule` taint
  on the device in its slice; to preempt a pod for a higher-priority Spur job
  it can use `NoExecute`. DRA extended-resource mapping (GA in 1.37, beta in
  1.36) lets a `DeviceClass` answer `amd.com/gpu` requests, so aim-engine and
  KServe need no change. The AMD DRA driver must then be off on dual-use
  nodes; the operator already enforces "one of device plugin or DRA driver".

Work in Spur, in dependency order:

1. Put the BDF (and render minor) in `GpuResource` as appended proto fields,
   so the controller and the WAL name a GPU by a stable key. Additive, compatible.
2. Heartbeat resource updates (upstream #800), so an inventory or hold change
   reaches the controller without a re-registration.
3. A per-device external hold in `NodeAllocation` and a WAL op with
   `#[serde(default)]`, plus a `spur show node` column and a `sinfo` GRES view
   that shows held GPUs.
4. The dual-use opt-in that replaces the node gate with the per-device state.
5. The driver in `spurd`: a DRA kubelet plugin (gRPC on
   `/var/lib/k0s/kubelet/plugins/`), `ResourceSlice` publish, taint on
   allocate, hold on prepare, release on unprepare. The device-plugin variant
   is smaller but count-based.
6. Order the local launch: `spurd` withdraws first, confirms the withdrawal,
   then installs the cgroup filter and launches. On restart `spurd` rebuilds
   its hold table from the kubelet checkpoint or from `NodePrepareResources`
   replay, and from the controller's allocation for its own jobs.
7. Docs: `managed-kubernetes.rst` dual-use section, the configuration
   reference, and a new page on GPU sharing.

For: one allocator, both sides see the truth, topology-aware placement in
Spur, fair-share and accounting can count Kubernetes usage (ROCm/spur#439 is
open on GPU fair-share), no change to aim-engine. Against: the largest
option, DRA maturity on k0s must be checked, and the k8s scheduler still has
no view of Spur's queue, so a Spur job with higher priority waits unless the
`NoExecute` path is used.

### Option D. Kubernetes as the source of truth on dual-use nodes

D1. Run the `spur-k8s` operator mode on the Spur-managed k0s: Spur jobs on
dual-use nodes become pods with `amd.com/gpu: N`. Exists today. The cost is
that native execution is lost for those jobs.

D2. Placeholder pod for native jobs, the inverse of SUNK: before `spurd`
launches a native job it creates a placeholder pod that requests N
`amd.com/gpu`, reads the assigned BDFs from the Pod Resources API, maps them
to `device_id`s and launches the native job on those GPUs with the cgroup
filter. Spur's controller must then accept "any N GPUs on this node" and take
the concrete IDs back at dispatch confirmation. No device plugin change, stock
operator, Kueue and ResourceQuota see the usage.

For: no Kubernetes-side component. Against: two-phase allocation with latency,
Spur loses topology placement (only `GetPreferredAllocation` of the AMD plugin
decides), a pod per job on every node, and the Spur scheduler's plan can be
wrong at dispatch time.

## 5. Comparison

| | A node split | B static GPU split | C Spur owns, dynamic | D2 placeholder pod |
|---|---|---|---|---|
| Code in Spur | none | small | large | medium |
| Code in k8s stack | none | plugin flag or DRA taint | none (spurd is the driver) | none |
| aim-engine change | none | none | none | none |
| Granularity | node | GPU, fixed | GPU, dynamic | GPU, dynamic |
| Race | none | none | small window, safe loser path | none |
| Topology placement in Spur | yes | inside its set | yes | no |
| Single-node demo | no | yes | yes | yes |
| Fits the roadmap | interim | stepping stone | yes | no |

## 6. Recommendation

1. Now: state the current behaviour in the docs. On Spur-managed k0s a node
   is all Kubernetes; on a foreign cluster nothing arbitrates. Confirm which
   of the two the reported conflict was.
2. Next: Option B with the Spur-side opt-in and, on the Kubernetes side, the
   allowlist flag in `ROCm/k8s-device-plugin` (B1). It is the smallest change
   that gives a mixed single node, and the Spur-side pieces (BDF in proto,
   dual-use opt-in, reduced inventory) are the first steps of C.
3. Target: Option C with `spurd` as a DRA driver on dual-use nodes. It is the
   design the code comment and the roadmap already point at, and it is the
   only option where accounting, fair-share and topology stay in one place.

## 7. Open points to verify before a design is fixed

- Which k0s version brings DRA device taints and extended-resource mapping on
  by default (1.36 beta, 1.37 stable per upstream). Spur controls the pin.
- Whether `[[devices.gres]] file = "/dev/dri/renderD[...]"` yields the same
  cgroup and env result as CDI auto-detection. A native-host e2e exists for
  the CDI path (`tests/native_host/e2e/test_device_isolation.py`).
- Compute partition modes (CPX, 64 devices) on both sides: Spur reads the
  mode, the AMD plugin groups by it in `mixed` mode, the DRA driver publishes
  `amdgpu-partition` devices. The BDF key is shared by a parent and its
  partitions, so the key must include the partition index.
- The Kubernetes scheduler binds by count; the `NoExecute` preemption path
  under DRA needs a check against KServe's restart behaviour.
