---
status: accepted
date: 2026-09-14
---

# spurstepd is a separate binary with one supervisor per job and per step

When `spurd` was the parent of user processes, its restart or upgrade killed
the work on the node. `spurd` now starts a separate `spurstepd` binary for
each job and numbered `srun` step, so work survives agent restarts without
running the agent's network RPC server or Kubernetes administration code
beside the job. Each supervisor owns the process tree, cgroup, and exit
status in a detached session that a restarted agent can adopt over a local,
versioned Unix socket.

## Consequences

- The isolation is at runtime only. `spurstepd` still links the `spurd`
  library crate, because the runtime code is not in its own crate yet.
- The Unix-socket protocol must stay compatible across versions, because a
  new `spurd` adopts supervisors that an old version started.
- Each PMIx server lives in the supervisor that owns its job and step. An
  agent restart must not destroy MPI rendezvous while ranks survive, which
  previously caused jobs to hang in `MPI_Finalize`. PMIx can communicate
  across nodes; only the supervisor's agent-control interface is local.

References: https://github.com/ROCm/spur/pull/754
