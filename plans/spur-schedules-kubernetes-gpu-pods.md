# Plan: Spur and Kubernetes share the GPUs of one node

Date 2026-09-24, revision 3, with the implementation status in section 10. Replaces the revision of 2026-09-23. Based on
`spur` main at `0b19a99`, cluster-forge worktree `EAI-8560-byok`, the
research in `plans/spur-k8s-gpu-coscheduling.md` and the decisions in
`plans/spur-k8s-gpu-sharing-grill.md`, rounds 1 to 5. Every decision in
this revision is confirmed; nothing is pending. Section 9 lists what
changed since revision 2 and why.

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
- No required change in aim-engine, KServe, the AMD GPU operator, the AMD
  device plugin or the AMD DRA driver. No fork, no patch. Changes go to
  Spur only. Optional upstream contributions are future work (section 7).

Not required, decided out of scope: one queue for pods and jobs, and
eviction in either direction. Both sides keep their own queue.

## 2. The device conflict

Two allocators hand out the same GPUs and neither sees the other.

- Spur. `spurd` discovers the GPUs from the KFD topology and reports them
  at registration. `spurctld` allocates whole KFD devices
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
ROCm/spur#558 (merged 2026-08-04, release 0.7.0) added the rule and says
"Dual-use workers can be a future opt-in". No issue tracks that
follow-up; this plan is the design for it.

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
on a shared node. Spur writes into it and reads from it. Spur's placement
on a shared node is provisional until Kubernetes confirms it; Kubernetes
is never the loser.

Verified from Kubernetes 1.36 and AMD sources (grill file, sections 2.3
and 2.5):

- Only kube-scheduler allocates a `ResourceClaim`, and only while it
  schedules a pod. A pod with `spec.nodeName` bypasses the scheduler and
  never gets its claim. There is no supported allocate call for a client.
- `DeviceTaintRule` is the other native mechanism. The KEP documents a
  race with the scheduler, `NoSchedule` has no status to wait for, the
  rule selects by device name only, and on 1.36 both the API version and
  the feature gate are off by default. A pod that wins the race keeps
  running, which breaks the no-over-allocation requirement. Rejected.
- Writing `ResourceClaim.status.allocation` directly, or editing the AMD
  driver's `ResourceSlice`, is undocumented and the driver rewrites the
  slice. Rejected.

### 4.2 DRA only on shared nodes

A shared node runs the AMD DRA driver (`gpu.amd.com`), not the device
plugin. DRA names the device at scheduling time, so Spur keeps topology
placement and learns a hold before the pod starts. The device plugin
gives a count and Spur would learn the device only after the pod starts.

Spur's contract with the Kubernetes side is one sentence: a shared node
has a DRA driver `gpu.amd.com` that publishes a `ResourceSlice` for the
node with `resource.kubernetes.io/pciBusID` per device. Spur ships no
driver, DaemonSet or DeviceClass. Who installs the driver is the
installer's choice (section 4.10).

Facts that shape this:

- The driver names a device `gpu-<card>-<renderD>`, pool name is the node
  name, and publishes `pciBusID` in extended BDF with the domain,
  `0000:19:00.0`. CPX partitions carry the parent's `pciBusID` and differ
  only by name. No attribute carries the partition index; CEL selectors
  cannot see the device name.
- gpu-operator 1.5.0 is the first release with `spec.draDriver`; 1.5.1 is
  the latest. One `DeviceConfig` with both the device plugin and the DRA
  driver is rejected by the reconciler. Two `DeviceConfig`s with disjoint
  node selectors are accepted: device plugin on ordinary nodes, DRA on
  shared nodes.
- The `amd.com/gpu` extended-resource mapping is only on the DRA driver's
  `develop` branch, not in v1.0.1. Until a release ships it, pods on a
  shared node must request a `ResourceClaim`. AIM pods request
  `amd.com/gpu` today, so they land only on ordinary nodes. Spur's
  implementation and e2e use a plain pod with a `ResourceClaim`;
  aim-engine support is future work.

### 4.3 Spur to Kubernetes: the placeholder pod

For each Spur job on a shared node, `spurd` creates one `ResourceClaim`
and one placeholder pod before it launches the job.

- Namespace `spur-system`, created by `spurd`. Pod and claim are named
  `spur-job-<jobid>-<run_attempt>`. Both carry the Spur job id, user and
  account as labels.
- The pod runs the pause image k0s already pins as its sandbox image
  (`quay.io/k0sproject/pause:3.10.2-0` on the pinned k0s), so it is
  always present locally. It has no CPU or memory request.
- The pod references the claim with `resourceClaimName`. The claim uses
  `deviceClassName: gpu.amd.com` and holds one request per whole GPU
  (`count` 1, CEL `device.attributes["resource.kubernetes.io"].pciBusID
  == "<bdf>"`) and one request per partitioned parent GPU (`count` k,
  same selector on the parent's BDF).
- The pod pins the node with a node selector on the hostname, never with
  `spec.nodeName`.
- `spurd` waits for the claim to be allocated and the pod to be
  scheduled, then installs the cgroup filter and launches the job. The
  wait is bounded by the launch deadline only
  (`controller.dispatch_timeout_secs`, default 300). It is built for a
  busy API server that answers slowly: an API call that times out is
  retried inside the deadline, and a slow answer is never read as a
  loss. Only two things end the wait early: the placeholder carries
  `PodScheduled=False` with reason `Unschedulable`, which the DRA
  scheduler plugin sets at once when a claim cannot be allocated, or the
  deadline passes. In both cases `spurd` deletes the placeholder and
  answers `ResourceExhausted`; the job requeues through the existing
  dispatch confirmation path.
- On a partitioned (CPX) GPU kube-scheduler chooses which sibling
  partitions satisfy `count` k. `spurd` reads
  `claim.status.allocation.devices.results[].device`, maps each name to a
  render minor and launches on those devices. It returns the devices it
  used in a new optional field on `LaunchJobResponse`, set only when they
  differ from the request. The controller accepts the substitution only
  when the count per node is unchanged and every device shares the
  parent BDF of a requested device; otherwise the launch counts as failed
  and the job requeues. The accepted set is what `JobStart.per_node_alloc`
  records. SPX allocations stay exact.
- When the job ends, `spurd` deletes the pod and the claim beside the
  `NodeAllocation` release. The orphan reconcile deletes placeholders
  whose job is not live. No `activeDeadlineSeconds`.

### 4.4 Kubernetes to Spur: holds

`spurd` watches its own node's `ResourceClaim`s and `ResourceSlice`. Each
GPU that an allocated claim of a non-placeholder pod names is a hold. A
hold is effective from claim allocation, before the pod starts.

- Report path: an optional field on `HeartbeatRequest`, precedent
  `k0s_status`. Every heartbeat carries the full hold state of the node
  plus the inventory `generation` of PR 898; a change triggers one
  immediate extra heartbeat. The controller drops a report whose
  generation is not the current inventory.
- Report content per GPU: `stable_id` and a state, one of free, Spur job
  id, held with pod namespace, pod name and claim name, or conflict.
- Holds live in the leader's memory, like heartbeat telemetry, not in the
  Raft log. Every `spurd` re-reports on leader change and on its own
  restart. A shared node is unplaceable until its first report, and again
  when its last report is older than `controller.heartbeat_timeout_secs`
  (default 90, three heartbeats at the 30 second interval). No new
  configuration field.
- The scheduler treats a held GPU as allocated. Backfill treats it as
  busy to the far future and never plans on it, because a pod has no end
  time. Holds affect GPU picks only; CPU-only jobs land on a shared node
  as on any node.
- Free GPUs on the controller are `total` minus `alloc` minus `held`, in
  the same computation as today. The hold set is empty on a non-shared
  node, so no second code path appears.
- On restart `spurd` rebuilds holds from the claim watch only.

### 4.5 Where the Kubernetes client lives

`spurd` gets a Kubernetes client, `spurctld` does not. Every k0s member
runs `spurd` and nothing else joins a node, so each `spurd` watches its
own node, creates the placeholders at launch and reports holds over the
agent protocol. `spurd` already mints an admin kubeconfig for the managed
k0s. `kube` 4.0 and `k8s-openapi` 0.28 with `v1_36/api/resource/v1` are
already in the lockfile; no dependency bump.

### 4.6 Identity

PR 898 is merged as `0b19a99`; `GpuResource.stable_id` is a `u64` with the
PCI domain in bits 24..39. The sharing code never decodes it. The DRA
device name is `gpu-<card_id>-<render_minor>`, both values `spurd` already
reads from sysfs, and the lookup path `/sys/class/drm/card*/device/drm`
is the same tree the DRA driver reads for whole GPUs and `amdgpu_xcp`
partitions alike. A GPU with no `card_id` is unshareable; `spur show
node` says "no DRM card for renderD<M>", and the CDI fallback to the
device index stays untouched.

The claim selector uses the BDF from one helper that masks the PCI
function to 0 for a partitioned device, as the DRA driver does, because
the kernel ORs the partition `node_id` into the function bits. Issue 920
reports this with values measured on an MI300X in CPX. After the
upstream fix the helper collapses to `bdf_from_location_id`; the unit
test with the measured values stays as the guard. Work does not wait for
the fix.

### 4.7 Opt-in and opt-out

A flag on `spur k8s up` marks a node as shared at enrolment, and `spur
update node` toggles it later. The flag persists beside the k0s role in
the Raft log with `#[serde(default)]`.

At opt-in `spurd` does three things on the node, in this order: it
creates the symlink `/var/lib/kubelet` to `/var/lib/k0s/kubelet`, only
when the path is absent or is already its own link; it sets the label
`spur.amd.com/gpu-sharing=true` on its own Node object; and it checks
that a `ResourceSlice` from driver `gpu.amd.com` exists for the node.
Until the slice exists the node stays unshareable with that reason in
`spur show node`. The symlink is what lets the gpu-operator's DRA
DaemonSet, which hard-codes `/var/lib/kubelet`, work on k0s: the kubelet
finds registrations by inotify on the registrar directory and dials the
endpoint path the plugin reports, with no check that the path lies under
the kubelet root (verified on release-1.36). `spur k8s down --reset`
removes the link.

Opt-out is accepted at once. The node is Kubernetes-reserved for new
placements, running Spur jobs finish and their placeholders go at their
own teardown, the label is removed at once, and the symlink stays until
the last placeholder is gone. This mirrors drain.

### 4.8 Scope rules on a shared node

- Holds cover GPUs only. CPU and memory can be oversubscribed by the two
  sides. Documented limitation.
- Advance reservations are not allowed on a shared node. Creation fails
  with a clear error.
- The partition mode is fixed while a node is shared. The unit is one KFD
  device.
- One Spur agent per node. The `spur-k8s` operator mode does not also
  register the node.

### 4.9 Failure cases

- An administrator deletes a placeholder while its job runs: `spurd`
  recreates it. If the recreate loses, the GPU is reported as conflict.
- Conflict: the GPU shows `conflict` in `spur show node`, the job gets a
  comment, the controller places nothing new on that GPU, no job is
  killed. The administrator decides.
- The API server is unreachable from `spurd`: new launches on the node
  are refused, and a stale report equals no report, so the node becomes
  unplaceable after the heartbeat timeout.
- `spurd` restarts: it rebuilds holds from the claim watch, then reports.
- A claim is allocated but the pod never starts: the hold is effective
  from allocation, so nothing changes for Spur.

### 4.10 The installer's side

Spur documents, and does not code, how the DRA driver gets onto a shared
node. Two supported ways:

- The driver's own Helm chart (`helm-charts-k8s` in
  `ROCm/k8s-gpu-dra-driver`, v1.0.1) with the k0s values
  `kubeletPlugin.kubeletRegistrarDirectoryPath=/var/lib/k0s/kubelet/plugins_registry`,
  `kubeletPlugin.kubeletPluginsDirectoryPath=/var/lib/k0s/kubelet/plugins`,
  `cdi.dynamicPath=/var/run/cdi`, `image.tag=v1.0.1` and a node selector
  on `spur.amd.com/gpu-sharing=true`. With the symlink the default paths
  also work.
- gpu-operator 1.5.x with `draDriver.enable`, `draDriver.image` pinned
  (the default tag is `latest`), `draDriver.selector` on the shared-node
  label, and the device-plugin `DeviceConfig` excluding shared nodes by
  selector.

CDI does not collide: `spurd` writes kind `amd.com/gpu` into
`/etc/cdi/amd.json`, the DRA driver writes kind `k8s.gpu.amd.com/gpu`
into `/var/run/cdi`, and containerd 2.3.2 in k0s scans both.

## 5. Work packages

In dependency order. Each package is one or more PRs. WP1 and WP2 run in
parallel; WP3 and WP4 depend on both; WP5 to WP8 follow.

### WP1 Identity

- `spurd` builds the DRA name `gpu-<card_id>-<render_minor>` per device
  and marks a device with no `card_id` unshareable.
- One helper yields the claim selector BDF and masks the function to 0
  for a partitioned device.
- Test: unit tests of the DRA name and the selector BDF for an SPX node
  and a CPX node, with the values measured in issue 920; a fixture from a
  real `ResourceSlice`.

### WP2 Shared-node opt-in

- Flag on `spur k8s up` and a `spur update node` toggle, persisted beside
  the k0s role.
- At opt-in: guarded symlink, node label, `ResourceSlice` check with the
  unshareable reason. At opt-out: label off at once, symlink kept until
  the last placeholder is gone. `spur k8s down --reset` removes the link.
- `NodePlacement::matches()` keeps a shared node eligible. The
  pending-reason classifier reports resources, not `K8sReserved`, for
  such a node. Reservation creation on a shared node fails with a clear
  error.
- Show the flag in `spur show node` and `sinfo`.
- Test: unit tests of the placement rule and the reservation check; the
  symlink guard against an absent path, an own link and a foreign
  directory.

### WP3 spurd Kubernetes client and holds

- New module in `spurd`, active only on a shared node. `kube` client from
  the admin kubeconfig, tolerant of a slow API server.
- Watch the node's `ResourceClaim`s and `ResourceSlice`. Build the hold
  set. Report it in the optional heartbeat field with the inventory
  generation; one extra heartbeat on change.
- `spurctld` keeps holds per node in the leader's memory, drops stale
  generations, and marks a shared node unplaceable until its first
  report and after the heartbeat timeout. Free GPUs subtract holds.
  Backfill treats a held GPU as busy to the far future.
- Test: a fake API server in unit tests; a controller test in which a
  hold makes a pending job wait and a release lets it run; a leader
  change that clears holds and a re-report that restores them; a stale
  generation that is dropped.

### WP4 Placeholder at launch

- Namespace `spur-system`. Before launch on a shared node, create the
  claim and the pod, wait as in 4.3, then launch. On give-up delete both
  and answer `ResourceExhausted`.
- CPX substitution: read the allocated device names, launch on them,
  return them in the new optional `LaunchJobResponse` field; the
  controller validates count and parent BDF and records the accepted set
  in `JobStart`.
- Delete pod and claim at teardown. Orphan reconcile. Recreate a deleted
  placeholder; report conflict on loss.
- Test: unit tests with a fake API server for the wait, the
  `Unschedulable` early exit, the slow-answer path, the substitution
  validation and the orphan reconcile.

### WP5 Visibility

- `spur show node` lists every GPU as free, a Spur job id, held with the
  pod namespace and name, conflict, or unshareable with the reason.
  `sinfo` GRES columns show counts only.
- A job on a conflicting GPU gets a comment.
- On Kubernetes the placeholder pod and its claim carry the Spur job id,
  user and account as labels.
- Test: golden output tests.

### WP6 Docs

- New page `docs/deployment/gpu-sharing.rst`: the contract, opt-in and
  opt-out, the installer's two ways (4.10), the placeholder and holds,
  the limitations of 4.8, and the `ResourceClaim` requirement for pods
  until the DRA driver ships `extendedResourceName`.
- `docs/deployment/managed-kubernetes.rst` and the configuration
  reference for the new flag and the heartbeat field.

### WP7 cluster-forge and byok

Prerequisite: byok is not on cluster-forge main. It is PR 836 on branch
`EAI-8560-byok`, and it must be installed from that branch. That byok
needs a Spur built from the `feat/cli-plugins` branch (worktree
`git-worktrees/spur-plugins`, pushed to the silogen fork), because the
`spur-aims` plugin uses the `spur <name>` plugin mechanism that main does
not have yet. Any test of WP7 on a real cluster starts from those two
branches.

- Raise the AMD GPU operator pin from 1.4.1 to 1.5.1. Pin the DRA driver
  image tag.
- Two `DeviceConfig`s: device plugin on ordinary nodes, DRA driver on
  shared nodes, disjoint node selectors on `spur.amd.com/gpu-sharing`.
- A byok capability `gpu.spur-sharing` with a probe that reads the
  `ResourceSlice` of a shared node.
- The `podResourceAPISocketPath` override becomes unnecessary on shared
  nodes through the symlink; keep it for the others.
- Docs in `byok/docs`: shared-node pods need a `ResourceClaim` until the
  DRA driver release with `extendedResourceName`.

### WP8 e2e

- Kaytoo CI cluster, no GPU: opt-in refused without a `ResourceSlice`,
  symlink and label lifecycle, placeholder cleanup with a fake
  `ResourceSlice`.
- GPU test behind a pytest marker, run manually: one pod with a
  `ResourceClaim` and one Spur job on the same shared node, reading the
  GPU each one got; the CPX `card_id` lookup check. On the byok path the
  cluster comes from the two branches named in WP7. The host is created by
  the user before testing and named then. Nothing runs on any GPU node
  without the user's explicit permission.

### WP9 Upstream housekeeping

- Done 2026-09-23: comment on PR 898 about the PCI domain; addressed in
  the merged PR.
- Done 2026-09-24: issue 920 on the CPX `location_id` decode, measured on
  an MI300X, https://github.com/ROCm/spur/issues/920.
- Done 2026-09-24: the PR 558 follow-up issue,
  https://github.com/ROCm/spur/issues/923.
- The roadmap entry (section 7) is posted only with explicit permission.

## 6. Risks and open points

- Issue 920 is open. The selector BDF helper is the workaround; WP1 does
  not wait.
- No released AMD DRA driver maps `amd.com/gpu`. Shared nodes are usable
  by pods with a `ResourceClaim` only. The feature is announced after the
  driver release.
- The gpu-operator DRA image defaults to `latest`. Pin it.
- CPX card lookup is not yet confirmed on hardware; WP8 covers it.
- Scheduling latency. The placeholder adds one kube-scheduler round trip
  to each Spur job launch on a shared node, and a busy API server
  lengthens it up to the launch deadline.
- CPU and memory oversubscription on a shared node is not prevented.
- A conflict is visible but not resolved by Spur.

## 7. Future work

Not part of this plan. Each item has an owner or a trigger.

- aim-engine `ResourceClaim` support, a separate Silo project with its own
  ticket: `spec.resourceClaims` with a `ResourceClaimTemplate` on
  `gpu.amd.com`, `resources.claims` instead of the `amd.com/gpu` extended
  resource, node capacity read from `ResourceSlice`s. Until then AIM pods
  do not land on shared nodes.
- An upstream PR to `ROCm/gpu-operator` that adds the kubelet registrar
  and plugins directories to `DRADriverSpec`, with the device plugin's
  `kubeletSocketPath` as the precedent. It removes the need for the
  `/var/lib/kubelet` symlink on the operator path. Consider after the
  feature works.
- After issue 920 is fixed upstream: collapse the selector-BDF helper to
  `bdf_from_location_id` and keep the unit test with the measured values.
- When a DRA driver release ships `extendedResourceName` (merged to
  `develop` 2026-08-03): pods that request `amd.com/gpu` land on shared
  nodes without a claim. Re-check the byok docs and the aim-engine item.
- The follow-up issue that PR 558 promised is filed as issue 923.
  Propose a roadmap entry between 11.3 and 11.4, "Shared nodes: GPU-level sharing with a
  Kubernetes DRA driver", and a note on 11.2 that it covers clusters where
  Spur executes the pods itself. Only with explicit permission.
- GPU accounting for fair-share (upstream issue 439) can read the same
  per-device state once holds exist.

## 8. Out of scope

- One queue for pods and jobs: `schedulerName`, a scheduler extender, and
  placeholder jobs for pods.
- Eviction in either direction, including an automatic requeue on a
  conflict.
- Time-sharing one GPU between a pod and a Spur job.
- Roadmap items 11.2, 11.4, 11.5 and 13.x.
- A required change in aim-engine, KServe, the AMD GPU operator or the AMD
  drivers. Optional upstream contributions are listed in section 7.

## 9. Changes from the 2026-09-23 revision

| Was | Now | Why |
|---|---|---|
| Identity decodes PR 898's `stable_id` to a BDF | DRA name from `card_id` and `render_minor`; selector BDF from one helper that masks the function for a partition | The DRA name carries no BDF or rank; issue 920 shows the decode is wrong for CPX partitions. |
| Placeholder selects a partition by device name | One request per parent with `count` k; kube-scheduler picks the siblings; `spurd` returns the devices used in a new `LaunchJobResponse` field | CEL cannot see the device name and siblings share every attribute. |
| Hold report on change, path undecided | Optional heartbeat field, full state each heartbeat, extra heartbeat on change, staleness from the heartbeat timeout | One path, no new RPC, no new configuration. |
| Loser wait undecided | Wait to the launch deadline, early exit on `Unschedulable` only, built for a slow API server | A busy API server must not cause a false loss. |
| Restart rebuilds holds from claims plus the Pod Resources API | Claim watch only | One source; the hold is effective from allocation. |
| DRA driver comes from the gpu-operator, kubelet paths unaddressed | Spur ships only the contract and a guarded symlink; the installer uses the driver chart or the operator | The operator hard-codes `/var/lib/kubelet` with no field to change it; Spur must not carry AMD components. |
| Failure policies undecided | Section 4.9: recreate, conflict visible only, refuse launches without the API server | Grill rounds 3 and 5. |
| Opt-out unspecified | Accepted at once, running jobs finish, mirrors drain | No eviction. |
| Placeholder namespace, name, cleanup undecided | `spur-system`, `spur-job-<jobid>-<run_attempt>`, deleted at teardown plus orphan reconcile | Grill round 3. |
| e2e on Kaytoo with a GPU | Kaytoo covers the non-GPU paths; the GPU test runs manually on a host the user provides | Kaytoo VMs have no GPU. |
| WP7 files the PR 558 issue | Future work, after this revision, with explicit permission | The user decides what is posted upstream. |

## 10. Implementation status (2026-09-24)

Work is paused. This section tells the next agent what is done, what
changed from sections 4 and 5, and what to do next. All work is local.
Nothing is pushed and no PR exists. The user decides about PRs.

### 10.1 Where the work is

| Item | Location |
|---|---|
| Spur integration branch | `feat/gpu-sharing` in `/home/prepo/dev/silo/git-worktrees/spur-gpu-sharing`, based on `origin/main` `1d17654` |
| Contract and lead decisions for sub-agents | `/home/prepo/dev/silo/git-worktrees/gs-common.md` |
| Empty branches for the next two steps | `gs-cred` in `git-worktrees/spur-gs-cred`, `gs-wire` in `git-worktrees/spur-gs-wire`, both at the integration head `64b1f72` |
| Merged agent branches, can be removed | `gs-ctl`, `gs-spurd`, `gs-place`, `gs-cli` and their worktrees `git-worktrees/spur-gs-*` |
| WP7 cluster-forge | branch `EAI-8560-byok-gpu-sharing` in `/home/prepo/dev/silo/git-worktrees/cluster-forge-gpu-sharing`, based on `origin/EAI-8560-byok`, 4 commits |
| itg1 change log | `/home/prepo/dev/silo/spur/plans/itg1-cleanup-log.md` |
| Captured ResourceSlice (MI300X, SPX) | `crates/spur-devices/tests/fixtures/resourceslice-mi300x-spx.json` on `feat/gpu-sharing` |

The integration branch builds, and `cargo clippy --workspace --exclude
spur-ffi --all-targets --locked -- -D warnings` is clean. Each agent ran
the tests of its crates and they passed. A full `cargo test --locked` on
the merged branch was not run yet.

### 10.2 Status per work package

| WP | Status |
|---|---|
| WP1 Identity | Done. `SharingIdentity` in `spur-devices` (`discover_sharing_identities`, `dra_device_name`, selector BDF with the function set to 0 for a partitioned device). Tests: SPX, CPX with the issue 920 values, the captured slice, a synthetic CPX slice. |
| WP2 Opt-in | Done except the worker credential (10.4, step 1). Controller: `Node.gpu_sharing`, `WalOperation::NodeGpuSharingSet`, `update_node`, `cluster_up`/`cluster_add_nodes` flags, reservation checks. spurd: kubelet links, node label, `ResourceSlice` check. CLI: `spur k8s up --gpu-sharing-nodes`, `spur k8s add-nodes --gpu-sharing`, `spur node gpu-sharing <nodes> on|off`, `scontrol update NodeName=<n> GpuSharing=yes|no`. |
| WP3 Holds | Done except the worker credential. spurd watches claims and the slice, reports `GpuHoldReport` on the heartbeat, sends one extra heartbeat on change. Controller keeps holds in leader memory, drops a wrong generation, holds all GPUs of a shared node without a fresh report, reserves held GPUs to the far future in backfill. |
| WP4 Placeholder | Core module done (`crates/spurd/src/gpu_sharing/placeholder.rs`). Controller accepts `LaunchJobResponse.substituted_alloc`. Not wired into the launch path (10.4, step 2). |
| WP5 Visibility | Done. `scontrol show node` shows `GpuSharing=` and one line per GPU. REST `node_to_json` has `gpu_sharing`. |
| WP6 Docs | Done: `docs/deployment/gpu-sharing.rst`, `managed-kubernetes.rst`, configuration and monitoring pages. Update after steps 1 and 2. |
| WP7 cluster-forge | Done, local. gpu-operator 1.5.1 on all paths, remediation off, a second `DeviceConfig` for the DRA driver behind `gpuSharing.enabled` (default false), capability `gpu.spur-sharing`, `byok/docs/spur-gpu-sharing.md`. |
| WP8 e2e | Not started. |
| WP9 | No change. |

### 10.3 Changes from sections 4 and 5

These decisions replace the text above. Tests on itg1 (MI300X, SPX,
k0s v1.36.2, DRA driver chart v1.0.1) confirmed the first two.

- Kubelet paths (4.7). On k0s the kubelet always makes a real directory
  `/var/lib/kubelet/device-plugins/`. Thus Spur does not link all of
  `/var/lib/kubelet`. It makes two links:
  `/var/lib/kubelet/plugins_registry` to
  `/var/lib/k0s/kubelet/plugins_registry` and `/var/lib/kubelet/plugins`
  to `/var/lib/k0s/kubelet/plugins`. Each link is made only when the path
  is absent or is already the Spur link. The DRA driver chart then works at
  its default paths. `pod-resources` is not linked, so the metrics exporter
  keeps its `podResourceAPISocketPath` override.
- Verified on hardware: the device names are `gpu-<card>-<renderD>` (for
  example `gpu-9-136` with `pciBusID` `0000:2f:00.0`); a claim with the CEL
  selector on `pciBusID` gets that exact device; a second claim for the
  same GPU gives `PodScheduled=False` with reason `Unschedulable` at once;
  the pause image `quay.io/k0sproject/pause:3.10.2-0` is present in k0s.
- Placeholder name (4.3). The name is
  `spur-job-<jobid>-<run_attempt>-<node>`, because a job on many nodes
  has one placeholder on each node. Added labels `spur.amd.com/node` and
  `spur.amd.com/run-attempt`. A user or account name that is not a valid
  label value is left out.
- Node label (4.7). gpu-operator 1.5.1 selectors are equality-only, and a
  node can belong to one `DeviceConfig` only. So spurd sets
  `spur.amd.com/gpu-sharing=true` on a shared node and `=false` on every
  other k0s-enrolled GPU node. It never removes the label while the node
  is enrolled. After a label change the gpu-operator needs a restart.
- Toggle command. There is no `spur update node`. The toggle is
  `UpdateNodeRequest.gpu_sharing`, used by `spur node gpu-sharing` and
  `scontrol update ... GpuSharing=`.
- Freshness (4.4). With no fresh valid report, all GPUs of a shared node
  are held. This makes the node GPU-unplaceable, but CPU-only jobs still
  land on it.
- Parent key for CPX substitution: `spur_core::resource::gpu_parent_key`,
  which is `stable_id >> 11`.
- Credential (4.5). The plan says `spurd` already mints an admin
  kubeconfig. That is true only on a control-plane node: `k0s kubeconfig
  admin` fails on a worker. Step 1 below fixes this.

### 10.4 Next steps, in order

1. Worker credential, in `spur-gs-cred`.
   - Add a controller RPC `GetGpuSharingKubeconfig {hostname, node_token}`.
     Authenticate it like `Heartbeat`, forward it to the leader, and
     allow it only for a node with k0s role Worker or Single.
   - Add a field `gpu_sharing_node` to `GetKubeconfigRequest`.
   - The control-plane `spurd` applies the namespace `spur-system`, a
     ServiceAccount `spurd-gpu-sharing-<node>` and minimum RBAC:
     - cluster-wide `resourceclaims` and `resourceslices` get/list/watch;
     - in `spur-system`, `resourceclaims` and `pods` create, get, list,
       watch, delete and patch;
     - `nodes` get/patch with `resourceNames: [<node>]`.
   - Then it mints a bound token and returns the kubeconfig. Validate the
     node name as a DNS-1123 label. Never log the token.
   - `GpuSharing::client()` uses this path on a worker, keeps the local
     admin kubeconfig on a control-plane node, and rebuilds the client on
     HTTP 401 and every 12 hours.
   - spurd writes `gpu-sharing=false` on every enrolled, non-shared GPU
     node, without the watches.
   - Update the docs.
2. Launch wiring, in `spur-gs-wire`.
   - In `agent_server.rs::launch_job`, only on a shared node, call
     `placeholder::acquire` between `allocate_local_resources` and
     `build_job_injection_plans`. Then call `map_allocation` with
     `GpuSharing::identity`.
   - Rebind the substituted devices under the allocation lock and set
     `substituted_alloc`.
   - Release the placeholder on a later launch failure and in the
     `LaunchReservationGuard` drop path.
   - Call `release` beside `release_job_if` in `release_stepd_tracking`.
   - In the monitor loop, at a slower cadence, call `reconcile_orphans`
     (live = running jobs plus launching reservations) and `ensure_present`.
     Report a `Conflict` as CONFLICT.
   - Replace the `live_placeholders()` stub in `gpu_sharing/mod.rs`, which
     returns 0.
   - `build_claim` must put `app.kubernetes.io/managed-by=spurd` and
     `spur.amd.com/job-id` on the ResourceClaim, because `holds.rs` finds
     Spur claims by these labels.
3. Merge `gs-cred` and `gs-wire` into `feat/gpu-sharing`. Run `cargo fmt`,
   the full clippy command and the full `cargo test --locked`.
4. Reword the commit subjects. AGENTS.md wants a lowercase message after
   the type (for example `feat(spurd): add ...`). Some commits start with
   a capital letter. Rewrite the local history before any push.
5. WP8 on Kaytoo VMs (no GPU). Use the skill `spur-kaytoo-cluster` for a
   multi-node Spur cluster with `spur k8s up`. Add a test in
   `tests/native_host/e2e/` (see `test_k8s_scheduling.py`) for:
   - opt-in with no `ResourceSlice`, which gives an unshareable reason and
     no GPU placement;
   - the kubelet link and label lifecycle, including `=false` on
     non-shared nodes;
   - the credential RPC on a worker;
   - placeholder cleanup with a fake `ResourceSlice`, applied with
     `kubectl` and made to look like the captured fixture.
6. GPU test (WP8, GPU part) and the CPX card lookup check. They need a GPU
   host. itg1 was given back to the team on 2026-09-24 and must not be used
   again without the user's permission. Ask the user for a host.
7. Code review of the whole branch against this plan, then ask the user
   about PRs. cluster-forge needs an EAI Jira ticket for its PR title.

### 10.5 Open points found during the work

- If enrolment fails before a role is assigned, the flag is set but has no
  effect (`Node::shares_gpus()` needs a role). The node still cannot join
  a reservation.
- Hold reports live only on the leader. A follower that answers
  `get_node` itself shows no `gpu_holds`.
- The claim watch is cluster-wide, because claims have no node field.
- `placeholder::acquire` polls every 500 ms instead of watching (marked
  `ponytail:`).
- A node name longer than 63 characters cannot be a label value, so a
  placeholder on such a node fails.
