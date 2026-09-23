# Plan: Spur and Kubernetes share the GPUs of one node

Date 2026-09-23, revision 2. Replaces the revision of 2026-09-18. Based on
`spur` main at `eb02dd0`, cluster-forge worktree `EAI-8560-byok`, the
research in `plans/spur-k8s-gpu-coscheduling.md` and the decisions in
`plans/spur-k8s-gpu-sharing-grill.md`. Section 8 lists what changed and
why. Decisions marked "pending" have a recommendation but no confirmation
yet; the grill file names the open question.

## 1. Requirement

Spur jobs and Kubernetes GPU pods (aim-engine through KServe) run on the
same nodes. The requirement is:

- GPU granularity. A node with 8 GPUs can hold 3 GPUs of Spur jobs and 5
  GPUs of pods at the same time, and the split changes with the load.
- Opt-in per node. An administrator marks a node as shared. A node that is
  not marked keeps today's rule: an enrolled node is all Kubernetes.
- Shared visibility. Both sides show, per GPU, who holds it.
- No over-allocation. One GPU is never held by a Spur job and a pod at the
  same time.
- No change in aim-engine, KServe, the AMD GPU operator, the AMD device
  plugin or the AMD DRA driver. No fork, no patch. Changes go to Spur only.

Not required, decided out of scope: one queue for pods and jobs, and
eviction in either direction. Both sides keep their own queue.

## 2. The device conflict

Two allocators hand out the same GPUs and neither sees the other.

- Spur. `spurd` discovers the GPUs from the KFD topology and reports them
  at registration. `spurctld` allocates whole GPUs
  (`crates/spur-sched/src/cons_tres.rs`) and writes the allocation to the
  Raft log. At launch `spurd` installs a cgroup-v2 BPF device filter that
  permits only the allocated `/dev/dri/renderD*` and `/dev/kfd`.
- Kubernetes. The AMD driver publishes the GPUs to the kubelet. With the
  device plugin the scheduler sees a count and the kubelet picks the
  device. With the DRA driver the scheduler allocates a named device into
  a `ResourceClaim`.
- Neither filter bounds the other side. KFD lets any number of processes
  open one GPU, so both launches succeed. The failure appears later as
  VRAM out-of-memory or compute contention.

Today Spur avoids the conflict with a node-level rule. A node that `spur
k8s up` enrolled has a `k0s_role`, and `NodePlacement::matches()` excludes
it from Spur placement (`crates/spur-sched/src/node_match.rs`). The job
pends with "ReqNodeNotAvail, Reserved for Kubernetes cluster". PR
ROCm/spur#558 that added the rule says "Dual-use workers can be a future
opt-in". No issue tracks that follow-up.

## 3. What the roadmap covers

`plans/implementation-roadmap.md` Phase 11 gives GPU granularity only when
Spur executes every GPU workload itself (11.2, virtual kubelet, not
started). It has no item for a real kubelet and `spurd` sharing the GPUs
of one node. This plan adds that item. Items 11.2, 11.4, 11.5 and 13.x
stay untouched.

## 4. Design

### 4.1 Kubernetes is the ledger of record

With no change to the AMD driver, the only supported way to keep a GPU
away from pods is that Kubernetes allocates it to something Spur owns.
So the Kubernetes allocation ledger is the ledger of record for every GPU
on a shared node. Spur writes into it and reads from it.

Verified 2026-09-23 from Kubernetes 1.36 and AMD sources:

- Only kube-scheduler allocates a `ResourceClaim`, and only while it
  schedules a pod. A pod with `spec.nodeName` bypasses the scheduler and
  never gets its claim. There is no supported allocate call for a client.
- `DeviceTaintRule` is the other native mechanism. The KEP documents a
  race with the scheduler, `NoSchedule` has no status to wait for, the
  rule selects by device name only, and on 1.36 both the API version and
  the feature gate are off by default. A pod that wins the race keeps
  running, which breaks the no-over-allocation requirement. Rejected,
  pending confirmation (grill Q10b).
- Writing `ResourceClaim.status.allocation` directly, or editing the AMD
  driver's `ResourceSlice`, is undocumented and the driver rewrites the
  slice. Rejected.

### 4.2 DRA only on shared nodes

A shared node runs the AMD DRA driver (`gpu.amd.com`), not the device
plugin. DRA names the device at scheduling time, so Spur keeps topology
placement and learns a hold before the pod starts. The device plugin
gives a count and Spur would learn the device only after the pod starts.

Facts that shape this:

- The driver names a device `gpu-<card>-<renderD>`, pool name is the node
  name, and publishes `resource.kubernetes.io/pciBusID` in extended BDF
  with the domain, `0000:19:00.0`. CPX partitions carry the parent's
  `pciBusID` and differ only by name.
- gpu-operator 1.5.0 is the first release with `spec.draDriver`; 1.5.1 is
  the latest and supports Kubernetes 1.29 to 1.36. One `DeviceConfig`
  with both the device plugin and the DRA driver is rejected by the
  reconciler. Two `DeviceConfig`s with disjoint node selectors are
  accepted: device plugin on ordinary nodes, DRA on shared nodes.
- The `amd.com/gpu` extended-resource mapping is only on the DRA driver's
  `develop` branch, not in v1.0.1. Until a release ships it, pods on a
  shared node must request a `ResourceClaim`. AIM pods request
  `amd.com/gpu` today, so they land only on ordinary nodes. Pending
  (grill Q18): implement and test with a `ResourceClaim`, announce the
  feature after the driver release.

### 4.3 Spur to Kubernetes: the placeholder pod

For each Spur job on a shared node, `spurd` creates one placeholder pod
before it launches the job.

- The pod runs the pause image that k0s already uses for sandboxes, has
  no CPU or memory request, and carries the Spur job id, user and account
  as labels.
- The pod references one `ResourceClaim` with one request per GPU. Each
  request selects the device with a CEL expression on `pciBusID` and, for
  a partition, on the device name.
- The pod pins the node with a node selector on the hostname, never with
  `spec.nodeName`.
- `spurd` waits for the claim to be allocated and the pod to be
  scheduled. Then it installs the cgroup filter and launches the job.
- If the scheduler cannot allocate the claim, a pod won the GPU in the
  window. `spurd` deletes the placeholder and answers the launch with the
  existing resources-unavailable reply. The job requeues. Kubernetes is
  never the loser.
- When the job ends, `spurd` deletes the placeholder. The pod's owner
  reference and a label let a cleanup pass remove orphans.

Spur's placement on a shared node is provisional until the placeholder is
scheduled.

### 4.4 Kubernetes to Spur: holds

`spurd` watches its own node's `ResourceClaim`s and `ResourceSlice`. Each
GPU that a claim of a non-placeholder pod names is a hold. `spurd`
reports holds to `spurctld` when they change, not on a timer. A hold
carries the inventory `generation` of PR 898, so the controller drops a
hold that belongs to a stale inventory.

- The scheduler treats a held GPU as allocated. Backfill treats it as
  unavailable for the whole planning window, because a pod has no end
  time.
- Holds live in the leader's memory, not in the Raft log. Every `spurd`
  re-reports on leader change and on its own restart. A shared node is
  unplaceable until its first report, with a pending reason that names
  the window. Pending (grill Q13).
- On restart `spurd` rebuilds holds from the claims and, as a cross
  check, from the kubelet Pod Resources API, which returns DRA devices
  as driver, pool and device name.

### 4.5 Where the Kubernetes client lives

`spurd` gets a Kubernetes client, `spurctld` does not. `spurd` already
mints an admin kubeconfig for the managed k0s. The translation from
Spur's device identity to the DRA device name is node-local: PR 898's
`stable_id` decodes to bus:dev.func, discovery adds the domain, and for
a partition the render minor gives `gpu-<card>-<renderD>`. Pending (grill
Q12).

### 4.6 Identity

PR ROCm/spur#898 (open, changes requested 2026-09-23, fix pushed) makes
`GpuResource.stable_id` a `uint32` that encodes
`(bus << 16) | (dev << 11) | (func << 8) | partition_index`. The PCI
domain is not encoded. A comment on the PR asks whether the intent is
"domain 0000 only" or where the domain goes. Until that is answered,
`spurd` assumes domain 0000 for the decode and keeps the full address from
discovery beside it.

### 4.7 Opt-in

A flag on `spur k8s up` marks a node as shared at enrolment, and `spur
update node` toggles it later. The flag persists beside the k0s role in
the Raft log with `#[serde(default)]`. Kubernetes needs no signal, because
the DRA driver publishes every GPU regardless.

### 4.8 Scope rules on a shared node

- Holds cover GPUs only. CPU and memory can be oversubscribed by the two
  sides. Documented limitation.
- Advance reservations are not allowed on a shared node. Creation fails
  with a clear error.
- The partition mode is fixed while a node is shared. The unit is one KFD
  device.
- One Spur agent per node. The `spur-k8s` operator mode does not also
  register the node.

## 5. Work packages

In dependency order. Each package is one or more PRs. WP1 and WP2 run in
parallel; WP3 and WP4 depend on both.

### WP1 Identity, on top of PR 898

- Rebase on PR 898 when it lands. Do not branch from it now.
- Keep the full PCI address on `DeviceEntry` and add the render minor to
  what `spurd` keeps per device, so the DRA name can be built. Whether the
  domain enters the proto depends on the answer on PR 898.
- Test: unit test of `stable_id` to DRA name for an SPX node and a CPX
  node; a fixture from a real MI300X `ResourceSlice`.

### WP2 Shared-node opt-in

- Flag on `spur k8s up` and a `spur update node` toggle, persisted beside
  the k0s role. `NodePlacement::matches()` keeps a shared node eligible.
  The pending-reason classifier reports resources, not `K8sReserved`, for
  such a node.
- Show the flag in `spur show node` and `sinfo`.
- Docs: `docs/deployment/managed-kubernetes.rst`, the configuration
  reference, and a new page on GPU sharing.
- Test: unit tests of the placement rule; the e2e `test_k8s_scheduling.py`
  gets a case in which a job runs on a shared node with no pods.

### WP3 spurd Kubernetes client and holds

- New module in `spurd`, active only on a shared node. `kube` client from
  the admin kubeconfig.
- Watch the node's `ResourceClaim`s and `ResourceSlice`. Build the hold
  set. Report on change over the agent protocol: a new optional field on
  the heartbeat or a dedicated RPC, decided in grill round 3. The report
  carries the inventory generation.
- `spurctld` keeps holds per node in the leader's memory. The scheduler
  and backfill treat a held GPU as allocated with no end time.
- Restart: rebuild from claims, cross check with the Pod Resources API.
- Test: a fake API server in unit tests; a controller test in which a
  hold makes a pending job wait and a release lets it run; a leader
  change that clears holds and a re-report that restores them.

### WP4 Placeholder pod at launch

- Before launch on a shared node, create the placeholder pod and claim,
  wait for scheduling, then launch. On failure delete the placeholder and
  reply resources-unavailable.
- Delete the placeholder on job end. Cleanup pass for orphans.
- Failure cases: an administrator deletes a placeholder while its job
  runs; the API server is unreachable; a claim is allocated but the pod
  never starts. Policies decided in grill round 3.
- Test: unit tests with a fake API server; an e2e on a Kaytoo VM cluster
  with one pod that uses a `ResourceClaim` and one Spur job on the same
  node, reading the GPU each one got.

### WP5 Visibility

- `spur show node` and `sinfo` GRES columns show free, Spur-allocated and
  held, with the pod name for a held GPU.
- On Kubernetes the placeholder pod and its claim show the Spur job id,
  user and account as labels.
- Test: golden output tests for both commands.

### WP6 cluster-forge and byok

- Raise the AMD GPU operator pin from 1.4.1 to 1.5.1. Pin the DRA driver
  image tag; the operator default is `latest`.
- Two `DeviceConfig`s: device plugin on ordinary nodes, DRA driver on
  shared nodes, disjoint node selectors.
- A byok capability `gpu.spur-sharing` with a probe that reads the
  `ResourceSlice` of a shared node.
- Docs in `byok/docs`: shared-node pods need a `ResourceClaim` until the
  DRA driver release with `extendedResourceName`.

### WP7 Upstream housekeeping

- Done 2026-09-23: comment on PR 898 about the PCI domain.
- File the follow-up issue that PR 558 promised, with this plan as the
  design. Problem statement only, no prescribed fix.
- Propose a roadmap entry between 11.3 and 11.4, "Shared nodes: GPU-level
  sharing with a Kubernetes DRA driver", and a note on 11.2 that it covers
  clusters where Spur executes the pods itself.

## 6. Risks and open points

- PR 898 is not merged. Its identity encoding may still change. WP1 waits;
  WP2 to WP4 do not depend on the encoding, only on the decode helper.
- No released AMD DRA driver maps `amd.com/gpu`. Shared nodes are usable
  by pods with a `ResourceClaim` only. The feature is announced after the
  driver release.
- The gpu-operator DRA image defaults to `latest`. Pin it.
- CPX order. PR 898 ranks partitions by render minor inside a BDF group;
  the DRA driver names them by card and render index. Confirm on a CPX
  node that both orders agree before WP1 closes.
- Scheduling latency. The placeholder adds one kube-scheduler round trip
  to each Spur job launch on a shared node.
- CPU and memory oversubscription on a shared node is not prevented.
- Grill round 3 is open: placeholder namespace and naming, hold report
  path, loser-path timing, failure policies.

## 7. Out of scope

- One queue for pods and jobs: `schedulerName`, a scheduler extender, and
  placeholder jobs for pods. Removed from this plan, see section 8.
- Eviction in either direction.
- Time-sharing one GPU between a pod and a Spur job.
- Roadmap items 11.2, 11.4, 11.5 and 13.x.
- A change in aim-engine, KServe, the AMD GPU operator or the AMD drivers.

## 8. Changes from the 2026-09-18 revision

| Was | Now | Why |
|---|---|---|
| Layer 1: `spurd` replaces the AMD driver as the kubelet's device driver | Kubernetes allocation is the ledger; `spurd` writes placeholders and reads holds | Decision: no fork, patch or replacement of AMD's driver. A Spur driver would be a fork to keep in step with ROCm releases. |
| Layer 2: Spur places the pods through `schedulerName: spur` and an extender | Out of scope | Decision: both sides keep their own queue. Goal is visibility and no over-allocation. |
| DRA target, device plugin fallback | DRA only, documented requirement | The device plugin names the device only after the pod starts. |
| Device taints hide and evict | Rejected | Documented race with no signal to wait for; no eviction. |
| WP1 adds `pci_bdf` and `render_minor` to the proto | PR 898 adds `stable_id` with the BDF encoded; domain open | Upstream moved first. |
| WP2 adds a `NodeDeviceHold` WAL operation | Holds in leader memory, re-reported | Kubernetes is the ledger; a Raft copy is a second truth. |
| WP3 `[cluster] dual_use` config field plus node label | Per-node flag on `spur k8s up` and `spur update node` | Per node, no restart, persisted with the role. |
| WP6 turns the device plugin off and adds a Kyverno policy for `schedulerName` | Operator 1.5.1, two `DeviceConfig`s, no Kyverno | DRA on shared nodes only; no scheduler name. |
| Reservations withdraw GPUs before start | Reservations not allowed on shared nodes | A reservation without a guarantee is not a reservation. |
