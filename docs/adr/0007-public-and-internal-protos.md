---
status: accepted
date: 2026-04-15
---

# Raft traffic sends openraft messages as JSON inside an internal proto

The FFI library, the REST API, and the CLI depend on `proto/slurm.proto`, so
Raft messages must not go into it. Controller-to-controller traffic uses
`proto/raft_internal.proto` on its own port (6821, `raft_listen_addr`), and
each message has one `bytes` field that contains an openraft request or
response serialized as JSON. This keeps openraft types out of all protos, but
proto field rules do not protect this traffic.

## Consequences

- Compatibility between controller versions depends on the JSON form of the
  openraft types and of the Spur types in them, not on proto field tags. An
  openraft upgrade that changes this form can break a rolling upgrade.
- `raft_internal.proto` has no external clients.

References: https://github.com/ROCm/spur/pull/93
