---
status: accepted
date: 2026-03-17
---

# Configuration uses TOML, not slurm.conf

Spur preserves user-facing Slurm interfaces, but not Slurm's configuration
format (ADR 0006). It uses `/etc/spur/spur.conf` as a TOML file for daemon and
client settings, parsed into typed structs with defaults and validation in
`crates/spur-core/src/config.rs`. This trades configuration-file compatibility
for one shared configuration model.

## Consequences

- An administrator who migrates from Slurm rewrites the configuration. Spur
  does not read `slurm.conf`.
- To remove, rename, or retype a field is a breaking change for deployed
  configurations.
- Runtime partition changes go through Raft and do not rewrite `spur.conf`.
  Persisted partition state overrides the startup configuration seed, and
  deletion records prevent removed partitions from returning on restart.
  This avoids configuration-file write ordering and partial-write problems
  and keeps runtime partition changes consistent across controllers.
- Explicit `reconfigure` re-reads the file and applies partition changes
  through Raft. This authority rule is specific to partitions: other live
  settings can remain leader-local, and some settings require a restart.

References: https://github.com/ROCm/spur/pull/425
