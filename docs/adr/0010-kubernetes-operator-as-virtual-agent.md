---
status: accepted
date: 2026-03-18
---

# In Kubernetes operator mode, Spur places jobs and Kubernetes runs them

Kubernetes operator mode uses the existing kubelet to run Spur workloads,
without requiring a native `spurd` for those workloads. The
`spur-k8s-operator` registers nodes that match a label selector with
`spurctld` and serves the `SlurmAgent` gRPC API as one virtual agent, so
`spurctld` keeps the scheduling while the operator creates workload pods on
the allocated nodes. Users can also submit jobs as `SpurJob` custom
resources.

## Consequences

- The kubelet owns the pods, so the virtual agent does not supervise steps
  and has no `spurstepd`.
- The operator refuses a pod outside the job's recorded allocation.
- The virtual agent does not support `RunCommand`, `CancelStep`, or
  `RegisterJobAllocation`, so `srun` steps inside an allocation and `salloc`
  workflows do not work in operator mode.
