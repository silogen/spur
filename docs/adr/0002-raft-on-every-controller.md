---
status: accepted
date: 2026-04-16
---

# Every spurctld runs Raft, also a single controller

Kubernetes Lease election excluded bare-metal HA, and a later design kept
separate persistence paths for single-controller and Raft deployments that
drifted apart. Every `spurctld` now runs openraft, with a single controller
forming a one-member cluster. Raft logs and snapshots provide one durable
cluster-state store and one HA path for both bare metal and Kubernetes.

## Consequences

- `spurctld` does not call the Kubernetes API for election.
- Raft log entries contain `WalOperation` values; snapshots contain
  `ClusterSnapshot` state. Both formats and the types they contain must stay
  readable when a new controller restores old state.
- Controller reads that use leader forwarding prefer the leader for
  freshness but fall back to local applied state on any forwarding failure.
  This keeps those reads available during leader loss, but they can return
  stale data. Raft-backed writes still require the leader.
- Recovery is strict. A vote, log entry, or snapshot that `spurctld` cannot
  read stops the startup, because a skipped record is lost committed state.
  `--allow-partial-wal-recovery` skips such records only when an operator
  asks for it. A follower always rejects a snapshot that it cannot apply.
- Clients and agents accept a list of controller addresses and connect to
  the first one that answers. Any live controller is sufficient, because it
  forwards writes to the leader.

References: https://github.com/ROCm/spur/issues/82,
https://github.com/ROCm/spur/pull/84, https://github.com/ROCm/spur/pull/93,
https://github.com/ROCm/spur/pull/384, https://github.com/ROCm/spur/pull/414,
https://github.com/ROCm/spur/pull/506
