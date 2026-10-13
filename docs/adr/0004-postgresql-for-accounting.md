---
status: accepted
date: 2026-03-17
---

# PostgreSQL stores accounting, the Raft log stores cluster state

Accounting data (job history, usage, accounts, users, QOS, and the audit log)
grows without a limit, and `sacct`, `sreport`, and fair-share need SQL
aggregation over it. The controller keeps cluster state in memory and in the
Raft log, and it purges terminal jobs from memory after
`terminal_job_retention_secs`. Accounting goes to PostgreSQL through `sqlx`,
and accounting is off when `[accounting] database_url` is empty.

## Consequences

- A change to an account, a user, or a QOS writes its audit row in the same
  PostgreSQL transaction. A change to cluster state (nodes, jobs, partitions,
  reservations) cannot share a transaction with PostgreSQL, so its audit row
  is best-effort.
- An RPC layer in `spurctld` writes the audit row for every mutating RPC,
  not each handler. For an RPC that goes through Raft, only the leader
  writes the row, so each change has one row. A test fails when an RPC in
  the proto has no audit classification.
- Accounting commands fail when PostgreSQL is down.
- MySQL and MariaDB, which `slurmdbd` uses, are not supported.

References: https://github.com/ROCm/spur/pull/870
