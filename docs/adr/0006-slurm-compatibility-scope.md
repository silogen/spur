---
status: accepted
date: 2026-06-23
---

# Slurm compatibility applies only to the CLI, REST API, and FFI

Spur must run existing Slurm job scripts, wrappers, and clients without
copying Slurm's internal complexity. Slurm compatibility applies to the CLI
(including command names and `SLURM_*` job environment variables), the REST
API, and the C FFI. Internal design, configuration, storage, and daemon
layout use simple, modern defaults, with compatibility as a migration
bridge rather than a constraint on those choices.

## Consequences

- A change that breaks a Slurm-compatible flag, command, response field, or
  FFI symbol is a breaking change.
- Freedom from Slurm's internal design does not remove Spur's own upgrade
  contracts. Incompatible changes to persisted state, wire formats, or
  deployed configuration are also breaking changes.
- ADRs 0002, 0003, 0004, and 0008 are examples of internal choices that differ
  from Slurm.
