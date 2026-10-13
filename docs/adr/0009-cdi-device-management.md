---
status: accepted
date: 2026-06-11
---

# GPUs are discovered and injected through CDI

`spurd` first detected only AMD GPUs, with code specific to AMD. The
`spur-devices` crate now reads Container Device Interface (CDI) specs from
`/etc/cdi`, `/var/run/cdi`, and `[devices] cdi_spec_dirs`, and applies them
to container jobs and to native jobs. CDI is a vendor-neutral standard that
GPU vendors and Kubernetes device plugins already produce, but it is made for
containers, so a native job gets only a part of each spec.

## Consequences

- When no CDI spec exists, `[devices] auto_detect` discovers AMD GPUs from
  KFD sysfs. `[[devices.gres]]` entries configure devices by hand.
- A native job gets the environment variables and device nodes of a spec.
  Mounts, hooks, and additional groups apply only to container jobs.
- A new device type with a CDI spec needs no vendor code, but the GPU
  visibility variable (for example `ROCR_VISIBLE_DEVICES`) comes from a
  table for each vendor in `spur-devices`.

References: https://github.com/ROCm/spur/pull/262
