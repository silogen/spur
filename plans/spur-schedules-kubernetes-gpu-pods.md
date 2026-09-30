# Plan: Spur and Kubernetes share the GPUs of one node

Date 2026-09-29, revision 4. Replaces revision 3 of 2026-09-24.
This revision corrects the AMD DRA extended-resource requirements. The
implementation status in section 10 remains a record of 2026-09-24; this
update does not report new implementation or test results. Based on `spur`
main at `0b19a99`, cluster-forge worktree `EAI-8560-byok`, the research in
`plans/spur-k8s-gpu-coscheduling.md` and the decisions in
`plans/spur-k8s-gpu-sharing-grill.md`, rounds 1 to 5. Section 9 records the
revision history. The AIM/KServe path described in 4.2 still needs an
end-to-end test. Other open points remain in section 10.5.

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
- No required source-code change in aim-engine, KServe, the AMD GPU
  operator, the AMD device plugin or the AMD DRA driver. No fork, no
  patch. The Spur integration needs code changes; the installer needs
  cluster configuration (section 4.10). Optional upstream contributions
  are future work (section 7).

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
- As checked on 2026-09-29, the latest official AMD GPU DRA Driver release
  is v1.0.1, published on 2026-07-22. No official beta or prerelease is
  listed. Development tags, such as `develop-36`, are not beta releases.
- AMD PR 73, merged on 2026-08-03, adds
  `DeviceClass.spec.extendedResourceName: amd.com/gpu` to the Helm
  defaults. It changes no driver code. Its description states that an
  administrator could already add the field manually. Thus v1.0.1 with
  an explicitly configured DeviceClass is a test path supported by the
  source review; a new driver binary is not a prerequisite for this mapping.
- The mapping also needs Kubernetes `DRAExtendedResource`, not only base
  DRA support. The current Kubernetes feature-gate reference lists it as
  alpha and disabled by default in 1.34 and 1.35, beta and enabled by
  default in 1.36, and stable from 1.37. Check the installed Kubernetes
  version and component settings before a test.
- With the mapping enabled, AIM/KServe can keep the existing
  `resources.requests` and `resources.limits` entries for `amd.com/gpu`.
  Kubernetes performs DRA allocation for those requests. AIM does not
  need to author `spec.resourceClaims` or `resources.claims`; this does
  not mean that Kubernetes uses no ResourceClaim internally.
- Spur continues to use explicit claims for its placeholder pods. Both
  paths must allocate from the same DRA device inventory. Without the
  mapping, GPU pods on a DRA-only node need explicit claims; ordinary
  `amd.com/gpu` requests do not provide this shared-node path.
- Existing Spur tests use a plain pod with an explicit ResourceClaim.
  AIM/KServe compatibility, including GPU discovery, node selection and
  Spur hold reporting for Kubernetes-generated claims, is not yet tested
  end to end. Do not report it as working until WP8 passes.

Sources checked on 2026-09-29:

- [AMD DRA releases](https://github.com/ROCm/k8s-gpu-dra-driver/releases).
- [AMD PR 73: Helm extended-resource mapping](https://github.com/ROCm/k8s-gpu-dra-driver/pull/73).
- [Kubernetes DRAExtendedResource feature gate](https://github.com/kubernetes/website/blob/main/content/en/docs/reference/command-line-tools-reference/feature-gates/DRAExtendedResource.md).
- [Kubernetes extended-resource allocation by DRA](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources-allocation-by-dra).

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

For AIM/KServe extended-resource requests, add these installer steps:

1. Check the Kubernetes version and the effective `DRAExtendedResource`
   settings (section 4.2). For 1.34 and 1.35, enable the gate on the
   components that require it, using the documentation for that release.
   Verify the settings on every relevant control-plane and worker node.
2. Install the released AMD GPU DRA Driver v1.0.1 on shared nodes. Keep
   the device plugin on ordinary nodes only. The shared nodes need a
   loaded `amdgpu` kernel driver and a CDI-enabled container runtime.
3. Manage this DeviceClass in the installer's GitOps configuration:

   ```yaml
   apiVersion: resource.k8s.io/v1
   kind: DeviceClass
   metadata:
     name: gpu.amd.com
   spec:
     extendedResourceName: amd.com/gpu
     selectors:
       - cel:
           expression: "device.driver == 'gpu.amd.com'"
   ```

   The v1.0.1 driver chart creates the same class without the mapping by
   default. If GitOps manages the full object, set `deviceClass.create=false`
   in the driver chart. For the operator chart, use
   `draDriver.deviceClass.create=false`. One owner must manage the class;
   a manual patch alone is not a persistent install configuration.
4. Verify the ResourceSlices, the mapping and an unchanged `amd.com/gpu`
   pod request before testing AIM/KServe. Keep the explicit-claim test as
   a separate check of the Spur integration. See WP8 for acceptance.

This mapping requires no development image or driver source change.
The install path is based on source inspection, not a completed AIM test.
See the [v1.0.1 chart defaults](https://github.com/ROCm/k8s-gpu-dra-driver/blob/v1.0.1/helm-charts-k8s/values.yaml),
[DeviceClass template](https://github.com/ROCm/k8s-gpu-dra-driver/blob/v1.0.1/helm-charts-k8s/templates/deviceclass.yaml)
and [operator DRA instructions](https://github.com/ROCm/gpu-operator/blob/v1.5.1/docs/dra/dra-driver.md).

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
  the limitations of 4.8, and the two pod request paths in 4.2. Include
  the Kubernetes gate and DeviceClass configuration from 4.10. Distinguish
  the tested explicit-claim path from the untested AIM/KServe path.
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
- Configure the DeviceClass mapping and Kubernetes gate checks from 4.10.
  Manage the class with one GitOps owner, separate from chart defaults.
- Docs in `byok/docs`: describe unchanged `amd.com/gpu` requests through
  DRA and the explicit-claim alternative. State the Kubernetes version
  requirements and the AIM/KServe test status.

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

Additional acceptance checks for revision 4, not yet run:

- Configuration: render the install manifests. Verify the pinned v1.0.1
  image, disjoint device-plugin and DRA selectors, one DeviceClass owner,
  and `extendedResourceName: amd.com/gpu`. A second reconcile must retain
  the mapping. Verify the effective Kubernetes gate settings.
- Plain pod: request `amd.com/gpu` without authored claims on a shared
  node. Verify that Kubernetes allocates through DRA and that the pod can
  use the allocated GPU. Record the generated claim and device identity.
- AIM/KServe: deploy an AIM workload with its existing GPU request format.
  Verify discovery, profile selection, placement on the shared node and
  a successful inference request. No aim-engine or KServe source change.
- Coexistence: run a native Spur job and the AIM workload concurrently on
  different GPUs of the same node. Check the actual device identities,
  Kubernetes claims and Spur holds. Repeat with each workload starting
  first. When no GPU is free, an additional request must wait rather than
  reuse an allocated device. On completion, both paths must release their
  allocations and permit the waiting workload to run.
- Run hardware checks only on a host explicitly approved by the user for
  this task. Keep unrun checks marked as pending. If the unchanged AIM
  path fails, report the failing stage before proposing source changes.

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
- The v1.0.1 Helm defaults omit the extended-resource mapping. Configure
  the DeviceClass explicitly and verify the Kubernetes gate. Source
  inspection supports this path, but WP8 must verify the complete
  AIM/KServe flow and Spur holds before it is reported as working.
- Base DRA support alone is insufficient. Kubernetes 1.34 and 1.35 need
  explicit alpha feature enablement; 1.36 enables the beta by default.
  Verify the installed release rather than infer support from the driver.
- A chart reconciliation can remove a manually added mapping. Manage the
  DeviceClass with one owner as described in 4.10.
- The gpu-operator DRA image defaults to `latest`. Pin it.
- CPX card lookup is not yet confirmed on hardware; WP8 covers it.
- Scheduling latency. The placeholder adds one kube-scheduler round trip
  to each Spur job launch on a shared node, and a busy API server
  lengthens it up to the launch deadline.
- CPU and memory oversubscription on a shared node is not prevented.
- A conflict is visible but not resolved by Spur.

## 7. Future work

Not part of this plan. Each item has an owner or a trigger.

- Native aim-engine ResourceClaim support is optional, not a prerequisite
  for this plan. Consider a separate project only if explicit DRA requests
  are needed beyond the extended-resource mapping, or if WP8 identifies
  a requirement that cluster configuration cannot satisfy. Scope and
  approve any aim-engine changes separately.
- An upstream PR to `ROCm/gpu-operator` that adds the kubelet registrar
  and plugins directories to `DRADriverSpec`, with the device plugin's
  `kubeletSocketPath` as the precedent. It removes the need for the
  `/var/lib/kubelet` symlink on the operator path. Consider after the
  feature works.
- After issue 920 is fixed upstream: collapse the selector-BDF helper to
  `bdf_from_location_id` and keep the unit test with the measured values.
- When a released chart includes the mapping default from AMD PR 73,
  review DeviceClass ownership before an upgrade. This is a configuration
  maintenance step, not a prerequisite for unchanged AIM GPU requests.
  No future release number or date is confirmed by this plan.
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

## 9. Revision history

### Revision 5, 2026-09-30

- Added section 11: what is implemented and where, the results of the
  revision 4 checks on one MI325X node (the configuration, the plain pod,
  the unchanged AIM, both start orders, exhaustion and release), the
  blockers, the security points and other findings. Items not run stay
  pending (11.9).
- Found the stale `amd.com/gpu` node field and its effect on aim-engine
  (11.6).

### Revision 4, 2026-09-29

- Corrected the future-release requirement: AMD PR 73 changes Helm
  defaults only. Test released driver v1.0.1 with an explicitly managed
  DeviceClass mapping instead of waiting for a new binary.
- Added Kubernetes version and feature-gate requirements, installer
  configuration, source links and the official release status.
- Kept AIM/KServe GPU requests unchanged. Native claim support in
  aim-engine is optional future work.
- Extended WP6, WP7 and WP8 with configuration, documentation and
  acceptance checks. These additions are pending; the historical status
  below does not mark them complete.
- No cluster was changed, no hardware test was run and no existing
  implementation status was reverified for this revision.

### Revision 3, changes from 2026-09-23

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

## 10. Implementation status (2026-09-24, evening)

Historical record, not reverified on 2026-09-29. Section 11 has the status
of 2026-09-30. Revision 4 replaces the
old release-wait requirement in sections 4.2 and 4.10, WP6 to WP8, and
sections 6 and 7. This section's overrides for other design details still
apply. The added extended-resource and AIM/KServe checks remain pending.
The host references below are records, not permission to run new tests.

Steps 1 to 5 of the old list are done, and step 6 is done for SPX. This
section tells the next agent what is done, what changed from sections 4 and
5, and what to do next. All work is local. Nothing is pushed and no PR
exists. The user decides about PRs.

### 10.1 Where the work is

| Item | Location |
|---|---|
| Spur integration branch | `feat/gpu-sharing` in `/home/prepo/dev/silo/git-worktrees/spur-gpu-sharing`, based on `origin/main` `1d17654`, head `332d7c1` |
| Contract and lead decisions for sub-agents | `/home/prepo/dev/silo/git-worktrees/gs-common.md` |
| Merged agent branches, can be removed | `gs-ctl`, `gs-spurd`, `gs-place`, `gs-cli`, `gs-cred`, `gs-wire` and their worktrees `git-worktrees/spur-gs-*` |
| Backup of the branch before the reword of step 4 | `refs/original/refs/heads/feat/gpu-sharing` (`f96eb6b`), can be removed |
| WP7 cluster-forge | branch `EAI-8560-byok-gpu-sharing` in `/home/prepo/dev/silo/git-worktrees/cluster-forge-gpu-sharing`, based on `origin/EAI-8560-byok`, 4 commits |
| e2e tests | `tests/native_host/e2e/test_gpu_sharing.py` (3 nodes, no GPU), `tests/native_host/e2e/test_gpu_sharing_gpu.py` (1 GPU node, marker `gpu`) |
| GPU host change log | `/home/prepo/dev/silo/spur/plans/do-mi325x-change-log.md` (host `root@107.170.49.109`) |
| itg1 change log | `/home/prepo/dev/silo/spur/plans/itg1-cleanup-log.md`. Do not use itg1. |
| Captured ResourceSlice (MI300X, SPX) | `crates/spur-devices/tests/fixtures/resourceslice-mi300x-spx.json` |

On the head `332d7c1`: `cargo fmt --all --check` is clean, `cargo clippy
--workspace --exclude spur-ffi --all-targets --locked -- -D warnings` is
clean, and `cargo test --locked` passes (4481 tests in 40 binaries). One
PTY test in `spur-cli` (`interactive_pty_retries_past_a_hung_reconnect_attempt`)
failed once under load and passed alone. The file is not changed by this work.

### 10.2 Status per work package

| WP | Status |
|---|---|
| WP1 Identity | Done. |
| WP2 Opt-in | Done, with the worker credential. |
| WP3 Holds | Done, with the worker credential. |
| WP4 Placeholder | Done. Launch wiring, release on failure and teardown, a 30 s check loop for orphans and presence, CPX substitution. |
| WP5 Visibility | Done. A job on a conflicting GPU does not get a comment (see 10.5). |
| WP6 Docs | Done: `gpu-sharing.rst` also covers the credential, the placeholder deadline, the CDI rule and the DRA driver version bug. |
| WP7 cluster-forge | Done, local. No change in this round. |
| WP8 e2e | Done on Kaytoo (3 VMs) and on one MI325X node in SPX. The CPX card lookup check is not done (10.4, step 1). |
| WP9 | No change. |

### 10.3 Changes from sections 4 and 5

These decisions replace the text above. Items from the first round stay:

- Kubelet paths (4.7): two links, `/var/lib/kubelet/plugins_registry` and
  `/var/lib/kubelet/plugins`, to the k0s paths. `pod-resources` is not linked.
- Placeholder name (4.3): `spur-job-<jobid>-<run_attempt>-<node>`, with the
  labels `spur.amd.com/node` and `spur.amd.com/run-attempt`.
- Node label (4.7): `true` on a shared node, `false` on every other
  k0s-enrolled GPU node. After a label change the gpu-operator needs a restart.
- Toggle: `UpdateNodeRequest.gpu_sharing`, used by `spur node gpu-sharing`
  and `scontrol update ... GpuSharing=`. There is no `spur update node`.
- Freshness (4.4): with no fresh valid report, all GPUs of a shared node are
  held. CPU-only jobs still land on the node.
- Parent key for CPX substitution: `gpu_parent_key`, `stable_id >> 11`.

New in this round:

- Credential (4.5). A worker has no admin kubeconfig. The controller RPC
  `GetGpuSharingKubeconfig {hostname, node_token}` authenticates the node
  like `Heartbeat`, forwards to the leader, and allows a node with k0s role
  Worker or Single and at least one GPU. The node does not have to be
  shared, because a non-shared GPU node also writes its label. The
  controller asks a control-plane `spurd` with the new field
  `GetKubeconfigRequest.gpu_sharing_node` (tag 4). That `spurd` applies the
  namespace `spur-system` (Pod Security `baseline`), the ServiceAccount
  `spurd-gpu-sharing-<node>` and minimum RBAC, and returns a kubeconfig with
  a 24 h bound token. The worker rebuilds its client after 12 h and on HTTP
  401. A control-plane or Single node uses the local admin kubeconfig.
- Launch deadline (4.3). The wire has no deadline from the controller. So
  `spurd` reads `controller.dispatch_timeout_secs` from its own `spur.conf`,
  and stops 10 s before it.
- Placeholder checks run in their own 30 s loop, not in the 2 s monitor loop,
  because a slow API call must not stop job reaping.
- CDI (4.10). The statement "CDI does not collide" was wrong. The AMD DRA
  driver writes one CDI spec for each allocated claim, and an env-only
  `common` device, into `/var/run/cdi`. `spurd` read them as inventory: one
  GPU more, and the GPU of the pod twice. The fix: the CDI cache skips each
  kind whose vendor starts with `k8s.`, which is the DRA driver convention.
- DRA driver v1.0.1 bug (found on the MI325X VM). The driver reads the
  version of the first card in `/sys/class/drm`. On a VM whose `card0` is
  virtio-gpu, the version is `1`, which is not semver, and the API server
  refuses every `ResourceSlice`. The docs name the symptom and the work-around
  `modprobe -r virtio_gpu`. An upstream issue to `ROCm/k8s-gpu-dra-driver`
  is possible, only with the user's permission.

### 10.4 Next steps, in order

1. CPX card lookup check. The MI325X host is a DigitalOcean VM guest, and
   its `amd-smi set` has no partition option, so it cannot go to CPX. Ask
   the user for a bare-metal host where CPX is allowed. Then run
   `test_gpu_sharing_gpu.py` in CPX (the first test compares every DRA name
   with the slice). Restore SPX after the test.
2. Decide the open security point in 10.5 (credential RPC in open admission).
3. Code review of the whole branch against this plan (skill `code-review`).
4. Ask the user about PRs. The Spur PR title needs the conventional form of
   AGENTS.md. The cluster-forge PR needs an EAI Jira ticket.
5. Remove the merged `gs-*` branches and worktrees, and the backup ref.

### 10.5 Open points found during the work

- Security. In `admission.mode = open` (the default), `verify_node_identity`
  accepts any caller. Any client that reaches port 6817 can then get the
  credential of any GPU worker: pod create in `spur-system` (baseline Pod
  Security) and patch on that Node. The docs warn about it. Option: refuse the
  RPC unless the controller enforces node identity. Then GPU sharing on
  workers needs token admission. The user decides.
- RBAC objects of a node are not deleted when the node leaves k0s or opts out.
- A job on a conflicting GPU gets no comment: there is no path from `spurd` to
  the job comment.
- After a `spurd` restart, placeholders of jobs that ran before the restart
  are not checked for presence; only the orphan check covers them.
- e2e teardown: `spur k8s down --reset` reports `phase: down` before `spurd`
  has reset k0s. A fixture that kills the daemons at once leaves k0s running.
  The same pattern is in the older k0s fixtures. Wait for the reset before
  the kill.
- If enrolment fails before a role is assigned, the flag is set but has no
  effect (`Node::shares_gpus()` needs a role).
- Hold reports live only on the leader. A follower that answers `get_node`
  itself shows no `gpu_holds`.
- The claim watch is cluster-wide, because claims have no node field.
- `placeholder::acquire` polls every 500 ms instead of a watch (`ponytail:`).
- A node name longer than 63 characters cannot be a label value, so a
  placeholder on such a node fails.

## 11. Extended-resource implementation and results (2026-09-30)

This section records the work of revision 4. For the items that it marks as
passed, it replaces the "pending" status of the revision 4 checks in
sections 4.2, 4.10, WP6, WP7 and WP8. All other items stay pending. The user
approved the host `root@107.170.49.109` for this task only.

Nothing is pushed. No PR exists. The user allowed local commits only.

### 11.1 Branches and commits

| Repository | Branch and worktree | Base | Commits of this work |
|---|---|---|---|
| Spur | `feat/gpu-sharing-extres` in `git-worktrees/spur-gpu-sharing-extres` | `feat/gpu-sharing` rebased with `--rebase-merges` onto `origin/main` `18937f8` | `164c064` rebase fix, `e0e0428` e2e, `9403269` docs |
| cluster-forge | `gpu-sharing-dra-extres` in `git-worktrees/cluster-forge-dra-extres` | `origin/main` `e601e899` | `65de9897` |
| Plan | `docs/spur-schedules-k8s-gpu-pods` in `git-worktrees/spur-plan-k8s-gpu` | not changed | `37efae1` revision 4, then revision 5 |

The cluster-forge branch replaces the byok branch `EAI-8560-byok-gpu-sharing`
of WP7 and 10.1. Operator 1.5.1 and `spur-aims` are on cluster-forge main
now, so byok is not necessary. The old byok branch is not changed and can be
removed after a decision about the PRs.

Other locations:

| Item | Location |
|---|---|
| Host change log | `spur/plans/do-mi325x-change-log.md`, items 9 to 19 |
| Host logs | `/root/dra-extres` on the host: `smoke-a-pass.log`, `smoke-b.log`, `smoke-wait.log`, `smoke-migrated.log`, `aims-install.log`, `spurd.log`, `spurctld.log` |
| Release binaries for Ubuntu jammy | `target-jammy/release` in the Spur worktree, built with `spur/plans/build-jammy.sh` (and `--bin spurauthd` for the e2e harness) |

### 11.2 What is implemented, and where

cluster-forge (`65de9897`):

| File | Change |
|---|---|
| `sources/amd-gpu-operator-config/v1.5.1/templates/deviceclass-gpu-amd.yaml` | New. DeviceClass `gpu.amd.com`, `extendedResourceName: amd.com/gpu`, selector `device.driver == 'gpu.amd.com'`, only when `gpuSharing.enabled`. ArgoCD option `SkipDryRunOnMissingResource`. |
| `sources/amd-gpu-operator-config/v1.5.1/templates/deviceconfig-dra.yaml` | New. DeviceConfig `gpu-operator-dra`: DRA driver with the pinned image, no device plugin, metrics exporter on node port `gpuSharing.metricsNodePort`, selector `amd-gpu=true` and `spur.amd.com/gpu-sharing=true`. |
| `sources/amd-gpu-operator-config/v1.5.1/templates/deviceconfig-example.yaml` | The device-plugin DeviceConfig adds the selector `spur.amd.com/gpu-sharing: "false"` when GPU sharing is on. No change when it is off. |
| `sources/amd-gpu-operator-config/v1.5.1/values.yaml`, `README.md` | Values `gpuSharing.enabled` (false), `gpuSharing.draDriverImage` (`docker.io/rocm/k8s-gpu-dra-driver:v1.0.1`), `gpuSharing.metricsNodePort` (32501). |
| `spur/packages/amd-gpu-operator/values.yaml` | `draDriver.deviceClass.create: false`. |
| `root/values.yaml` | `draDriver.deviceClass.create: false` in the `amd-gpu-operator` `valuesObject`, for the ArgoCD path. No version change, so `sbom/components.yaml` does not change. |
| `spur/profiles/gpu-sharing.yaml` | New profile. Extends `default`, sets `gpuSharing.enabled: true` and the k0s `podResourceAPISocketPath`. |
| `spur/capabilities.yaml`, `spur/spur-aims/probes.go` | Capability `gpu.spur-sharing`. The probe passes when a node with `spur.amd.com/gpu-sharing=true` has a `gpu.amd.com` ResourceSlice. Ported from the byok branch. |
| `spur/tests/check-gpu-sharing-render.sh`, `spur/justfile`, `.github/workflows/helm-chart-checks.yaml` | New render check (`just gpu-sharing-render`) and a CI step. |
| `spur/docs/spur-gpu-sharing.md`, `spur/README.md`, `docs/configuration-reference.md` | New operator guide, profile list entry, document link, values table. |

Spur:

| Commit | File | Change |
|---|---|---|
| `164c064` | `crates/spurctld/src/server.rs` | `get_gpu_sharing_kubeconfig` uses the new `forward_request` signature of main. Test `UpdateNodeRequest` gets `gpu_sharing: None`. |
| `164c064` | `crates/spurctld/src/audit/registry.rs` | `GetGpuSharingKubeconfig` is in the internal daemon-to-daemon RPC list. The audit test of main requires each RPC in one list. |
| `164c064` | `crates/spurctld/src/scheduler_loop.rs` | `start_borrowed_job` of idle-fill gets the CPX-substituted `per_node_alloc`. Tests follow the new signatures of main. |
| `e0e0428` | `tests/native_host/e2e/test_gpu_sharing_gpu.py` | The fixture installs the driver chart with `deviceClass.create=false` and applies the class with the mapping. New test `test_extended_resource_pod_is_held`. |
| `9403269` | `docs/deployment/gpu-sharing.rst` | New subsection "The DeviceClass" (mapping, one owner, feature gate). The Helm install sets `deviceClass.create=false`. "Run a pod on a shared node" describes `amd.com/gpu` requests and keeps the explicit-claim example. New failure case for the stale node field. |

No change to Spur daemon logic was necessary for the extended resource. The
claim watch of `spurd` counts every allocated `gpu.amd.com` claim of the pool
of the node, so a generated claim is a hold as any other claim.

Checks on the Spur branch: `cargo fmt --all --check` and `cargo clippy
--workspace --exclude spur-ffi --all-targets --locked -- -D warnings` are
clean. `cargo test --locked` passes 4654 tests in 40 binaries.

Checks on the cluster-forge branch: `spur/tests/check-gpu-sharing-render.sh`,
`just test`, `helm lint ./root`, `helm template` with the small, medium and
large values, and `sbom/validate-sync.sh` pass.

### 11.3 Status per work package

| WP | Status of revision 4 |
|---|---|
| WP6 Docs | Done. `gpu-sharing.rst` in Spur, `spur/docs/spur-gpu-sharing.md` in cluster-forge. |
| WP7 cluster-forge | Done on cluster-forge main (not byok). One DeviceClass owner, pinned image, disjoint selectors, render check in CI. |
| WP8 e2e | Configuration, plain pod, AIM, coexistence in both orders, exhaustion and release: passed on one SPX node (11.5). CPX and multi-node checks stay pending (11.9). |

### 11.4 Versions

| Component | Version |
|---|---|
| Kubernetes | k0s v1.36.2 (single node) |
| `DRAExtendedResource` | beta, on by default. The metric `kubernetes_feature_enabled` is `1` in kube-apiserver, kube-scheduler, kube-controller-manager and kubelet. No component sets `--feature-gates`. |
| AMD GPU operator | 1.5.1 |
| AMD DRA driver | `docker.io/rocm/k8s-gpu-dra-driver:v1.0.1` |
| aim-engine | v0.2.6 |
| AIM | `docker.io/amdenterpriseai/aim-meta-llama-llama-3-1-8b-instruct:0.11.1`, the smallest 1-GPU model of the catalog with a profile for this GPU |
| GPU | 8 x MI325X virtual function, SPX, NPS1, DigitalOcean VM |

### 11.5 Results on hardware

| Check | Result |
|---|---|
| Rendered manifests | Passed. One class with the mapping, pinned image, disjoint selectors. The render check fails, as it must, when the operator chart makes the class, when the mapping is wrong and when the profile does not turn on GPU sharing. |
| Reconcile keeps the mapping | Passed. A reinstall of the profile and an operator restart keep the same object (same UID and resourceVersion). After a manual removal of the field, `spur-aims install gpu-sharing` writes it again. |
| Effective gate settings | Passed, see 11.4. |
| Plain pod, `amd.com/gpu: 1`, no claims | Passed. Kubernetes makes `<pod>-extended-resources-<suffix>`, the container sees only the allocated card, Spur shows `held <ns>/<pod> (claim ...)`, the delete frees the GPU. |
| Unchanged AIM serves inference | Passed, after the workaround of 11.6. The predictor has `amd.com/gpu` requests and limits and no authored claims. |
| AIM first, then Spur `gpu:7` | Passed. The job got the 7 other GPUs. The AIM answered while all 8 GPUs were in use. |
| Spur `gpu:7` first, then AIM | Passed. The AIM got the free GPU and answered. |
| Exhaustion | Passed in both orders. A new pod was `Unschedulable` and a new Spur job was `PD (Resources)`. |
| Release | Passed in both orders. When a GPU became free, the next workload started. |
| AIM waits for Spur | Passed. With a Spur `gpu:8` job, the AIM predictor was `Unschedulable`. After `scancel` the AIM started and answered. |
| Move from device plugin to DRA | Passed with the workaround of 11.6. |
| Spur e2e module `test_gpu_sharing_gpu.py` | Passed, two runs, 3 of 3 tests each, including `test_extended_resource_pod_is_held`. |

### 11.6 Blockers

1. Stale `amd.com/gpu` node field stops aim-engine.

   When a node moves from the device plugin to DRA, the kubelet keeps
   `amd.com/gpu` in the node status: capacity 8 and allocatable 0, then 0
   and 0 after approximately 5 minutes. The kubelet checkpoint
   `/var/lib/kubelet/device-plugins/kubelet_internal_checkpoint` keeps it
   after `spur k8s down --reset`.

   kube-scheduler ignores the field, because the class maps `amd.com/gpu` to
   DRA. aim-engine v0.2.6 does not: `resourceMismatchReasons` in
   `internal/v1alpha2/aimprofile/node_match.go` refuses a node where the key
   exists and is less than the request. The AIMModel is then `NotAvailable`
   (`NoSupportedProfiles`), and aim-engine makes no InferenceService. A node
   with no key passes. A node that never had the device plugin is not
   affected.

   Workaround, documented in both repositories: after the 5 minutes, remove
   `/status/capacity/amd.com~1gpu` and `/status/allocatable/amd.com~1gpu`
   with a JSON patch on the node status. The kubelet does not write them
   again.

   Options for a permanent fix, not done, the user decides:

   - `spurd` removes the two fields at opt-in, after the grace period. This
     needs `patch` on `nodes/status` in the RBAC of the worker credential,
     which is a larger permission (see 11.7).
   - aim-engine ignores an extended resource that a DeviceClass maps. This
     is an AIM source change, so it needs a separate approval.

   No change to AIM or KServe source was made.

2. The cluster-forge PR needs an EAI Jira ticket. The Spur PR needs a
   decision about the base: `feat/gpu-sharing` is not on Spur main.

### 11.7 Security

The points of 10.5 are unchanged. None is fixed by this work:

- In `admission.mode = open`, the default, `verify_node_identity` accepts
  any caller. Any client that reaches port 6817 can get the credential of
  any GPU worker through `GetGpuSharingKubeconfig`: pod create in
  `spur-system` (Pod Security `baseline`) and patch on that Node. The docs
  warn about it. Option: refuse the RPC unless the controller enforces node
  identity. The user decides.
- RBAC objects of a node are not deleted when the node leaves k0s or opts
  out. A removed node keeps a valid path to a credential until an
  administrator deletes them.

New points from this work:

- The rebase puts `GetGpuSharingKubeconfig` in the audit registry list of
  internal daemon-to-daemon RPCs. The RPC returns a credential, so an audit
  record for it could be useful. Review this classification.
- The permanent fix of blocker 1 in `spurd` needs `patch` on
  `nodes/status`. With the open admission above, this permission would also
  go to any client that reaches the controller.
- The test host firewall allows the Spur ports only from `10.0.0.0/8` and
  `127.0.0.0/8`. This limits the risk of open admission on that host. It is
  a host setting, not a Spur setting.

### 11.8 Other findings

- The AMD operator controller (`handleDeviceClass`) makes the class only on
  OpenShift, only when it is absent, and never patches it. On k0s the Helm
  owner is the only writer, so the mapping stays after a reconcile.
- The operator chart makes its class only when the API server has
  `resource.k8s.io/v1`. A render check must give helm this API
  (`--api-versions`), or it does not see the second class.
- A generated claim name is `<pod>-extended-resources-<suffix>`, but it is
  shorter for a long pod name, for example
  `...-predictor-5f6d967f48-8zglq-extenfrh9d`. Read the name from
  `pod.status.extendedResourceClaimStatus.resourceClaimName`, not from a
  name prefix.
- The generated claim has the annotation
  `resource.kubernetes.io/extended-resource-claim: "true"`.
- A VF host has no Node Feature Discovery label. The node needs
  `feature.node.kubernetes.io/amd-gpu=true` by hand, or no DeviceConfig
  selects it.
- The DRA driver v1.0.1 version bug with `virtio_gpu` (10.3) still applies.
  `modprobe -r virtio_gpu` was necessary again after the reinstall.
- `spurd` refuses root jobs (`allow_root_jobs = false`). The manual tests
  used the user `spurtest` (groups `render`, `video`). Root cannot
  `scancel` the jobs of that user.
- The e2e harness needs `spurauthd` in the binaries directory. The
  `build-jammy.sh` script did not build it.
- The e2e harness connects to the public IP of the node. On the test host
  this needed a temporary firewall rule (change log item 17, undone in 19).
- The teardown problem of 10.5 happened again: `spur k8s down --reset`
  reports `phase: down` before the reset is done. A daemon stop at once left
  k0s running, and `k0s stop; k0s reset` by hand was necessary. After the
  first e2e run, `k0scontroller` was still active for a short time.
- The rebase onto main needed `--rebase-merges`, because the branch has
  merge commits. main added idle-fill, the audit registry and
  `dispatch_timeout_secs`, and `164c064` adapts the branch to them.
- Section 4.10 still says "CDI does not collide". Section 10.3 corrects it,
  and `gpu-sharing.rst` describes the `k8s.` vendor rule.
- The `/unslop` skill is not available, so it did not run on the commits
  and documents.

### 11.9 Pending

- CPX partitions (10.4, step 1). The host cannot go to CPX.
- More than one node, and a worker node with the worker credential.
- The ArgoCD path of cluster-forge on a real cluster.
- The change of an existing `default` cluster to `gpu-sharing`, where the
  operator chart made the class before. Helm must give the class to the new
  owner. Not tested.
- Kubernetes 1.34 and 1.35 with the alpha gate.
- The interaction of idle-fill reclaim (from `origin/main`) with GPU holds.
  The rebase compiles and the unit tests pass, but no test covers both.
- A code review of the rebased Spur branch and the cluster-forge branch.

### 11.10 Host state at the end

No Spur daemons run, and k0s is reset. `virtio_gpu` stays unloaded. The user
`spurtest` and `/root/dra-extres` stay. To restore the state before this
work, follow item 9 of the change log.
