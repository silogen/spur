---
status: accepted
date: 2026-07-08
---

# Accounting and the REST API run inside spurctld

Slurm uses separate daemons for accounting (`slurmdbd`) and for REST
(`slurmrestd`), and Spur first copied this with `spurdbd` and `spurrestd`.
Each daemon was one more binary to build, ship, configure, and monitor, and
each talked to `spurctld` over gRPC, which added a network hop to every job
start and end record. `spurctld` now serves the accounting gRPC service on
its client port (6817) when `[accounting] database_url` is set, and the REST
API on its own port (6820, `[rest_api] enabled`).

## Consequences

- REST and accounting share the failure domain of the controller, and REST
  adds an HTTP surface to the process that holds the cluster state.
- The REST layer keeps no state of its own. REST writes (submit, cancel) go
  through the in-process controller service, so they get the same
  authorization, validation, leader forwarding, and audit as gRPC. REST reads
  come from the local controller state and are not forwarded to the leader.
- The REST API is off by default (`[rest_api] enabled = false`). When it is
  on, it authenticates Bearer credentials with the same `[auth]` plugin and
  mode as gRPC. REST has no handshake to mint native credentials, so with
  `plugin = "spur"` and `mode = "required"` the controller refuses REST on a
  non-loopback address unless `[rest_api] allow_non_loopback = true`.
- CLI accounting commands use the controller address. There is no separate
  accounting address.
- Separate binaries can still serve a different lifetime or privilege
  boundary. `spurstepd` uses a local socket for agent control, but its PMIx
  server also communicates across nodes for MPI jobs (see ADR 0005).

References: https://github.com/ROCm/spur/pull/319,
https://github.com/ROCm/spur/pull/376, https://github.com/ROCm/spur/pull/637,
https://github.com/ROCm/spur/pull/640
