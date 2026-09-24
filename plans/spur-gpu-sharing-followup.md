# Follow-up plan: open decisions for Spur GPU sharing with Kubernetes

Date 2026-09-24. Continues `plans/spur-schedules-kubernetes-gpu-pods.md`
section 10 (status and open points). Read that section first. This file lists
the decisions that are still open, and the work that follows from each one.

Rules for all work in this file:

- Work only in git worktrees under `/home/prepo/dev/silo/git-worktrees/`. Do
  not commit to main. Do not push. Do not open a PR, a GitHub issue or a
  GitHub comment. The user decides about each of them.
- Follow `AGENTS.md` of the spur repo. Before you call a step done, run
  `cargo fmt --all`, `cargo clippy --workspace --exclude spur-ffi
  --all-targets --locked -- -D warnings` and `cargo test --locked`.
- Test only on Kaytoo VMs that you create yourself, or on a GPU host that the
  user names. Do not use itg1. Log each change on a GPU host with its undo
  command.
- Write docs and plans in ASD-STE100 Simplified Technical English, with no
  em dashes.

## D1 Credential RPC in open admission mode (security)

Problem. With `admission.mode = open` (the default), `verify_node_identity`
accepts any caller. A client that reaches port 6817 can then get the
Kubernetes credential of any GPU worker. That credential can create pods in
`spur-system` (Pod Security `baseline`) and patch the Node object of that
worker. The docs warn about it.

Options:

- A. Refuse `GetGpuSharingKubeconfig` unless the controller enforces node
  identity (token admission). This is safer. GPU sharing on k0s workers then
  needs token admission, and the Kaytoo e2e (`test_gpu_sharing.py`) must
  start the cluster with token admission.
- B. Keep the current rule and the warning in the docs.

Recommendation: A. Decision: the user.

Work after A: change the check in `spurctld` `server.rs`
(`get_gpu_sharing_kubeconfig`), add a unit test for the refusal, update
`docs/deployment/gpu-sharing.rst` (section "The Kubernetes credential"),
and change the e2e fixture. Run the Kaytoo e2e again.

## D2 CPX card lookup check (WP8)

Problem. The check needs a GPU host that can switch to CPX. The MI325X host
`root@107.170.49.109` is a DigitalOcean VM guest; its `amd-smi set` has no
partition option.

Decision: the user names a bare-metal host where CPX is allowed.

Work: run `tests/native_host/e2e/test_gpu_sharing_gpu.py` in SPX, switch to
CPX, run it again, and switch back to SPX. The first test compares every DRA
name with the `ResourceSlice`. Log each change on the host. If the node has a
DRM card that is not AMD, see D5.

## D3 Kaytoo VMs

The three VMs `useocpm2m-silogen-petrus-jeqzzr`, `-v9rr3f` and `-w69s2m`
expire on their own about 8 hours after 2026-09-24 18:40. Delete them with
`mcp__kaytoo__delete_my_vm` only when the user agrees. Make new VMs for a new
e2e run (skill `spur-kaytoo-cluster`, and section "How the Kaytoo e2e ran"
below).

## D4 Pull requests

Do not open any PR until the user says so. Prepare these:

- Spur, the whole feature: branch `feat/gpu-sharing`. PR title in the
  conventional form of `AGENTS.md`, for example `feat(spurd): share the gpus
  of a node with kubernetes`. Do a code review first (skill `code-review`,
  against the plan).
- Spur, the CDI fix alone: commit `59cbb43` `fix(spur-devices): ignore the
  cdi specs of dra drivers`. The bug is also on `main`: spurd reads
  `/var/run/cdi`, and counts each CDI spec that a DRA driver writes there as
  node inventory. The fix does not depend on GPU sharing, so it can be a small
  PR to `main` before the feature. The user decides.
- cluster-forge: branch `EAI-8560-byok-gpu-sharing`. The title needs an EAI
  Jira ticket: `EAI-NNNN <Verb> ...`. Ask the user for the ticket.

## D5 Upstream bug in the AMD DRA driver

The AMD DRA driver v1.0.1 reads the driver version of the first DRM card. On a
VM with a virtio-gpu display the version is `1`, and the API server rejects
every `ResourceSlice`. This is upstream issue ROCm/k8s-gpu-dra-driver#56, with
the open fix PR #117. A draft comment with our reproduction is in
`/home/prepo/dev/silo/spur/plans/upstream-draft-dra-driver-issue-56-comment.md`.
Post it only with the user's permission, and use the skill `oss-contribution`
before you post.

## Open points that need no decision

Section 10.5 of the main plan lists them. Do them after D1:

- Delete the RBAC objects of a node when it leaves k0s or opts out.
- Give a job on a conflicting GPU a comment.
- After a `spurd` restart, check the presence of the placeholders of jobs that
  ran before the restart.
- e2e teardown: wait until `spurd` has reset k0s before the fixture kills the
  daemons. `spur k8s down --reset` reports `phase: down` too early.

## How the Kaytoo e2e ran

The VMs cannot reach each other over the public IPs, and your machine cannot
reach the private IPs. So run pytest on node 0:

1. Make three VMs. Open the firewall on each (skill `spur-kaytoo-cluster`,
   section 1). Install k0s on each: `sudo spur k8s install-k0s`.
2. On node 0 make an SSH key and add it to `~/.ssh/authorized_keys` of all
   three VMs. Install `python3-pytest python3-paramiko python3-tomli-w`.
3. Copy `target/release/{spurctld,spurd,spur,spurstepd,spurauthd}` and
   `tests/native_host/e2e/` to node 0.
4. On node 0: `SPUR_TEST_NODES=<3 private IPs> SPUR_TEST_SSH_USER=ubuntu
   SPUR_TEST_SSH_KEY=~/.ssh/id_ed25519 SPUR_TEST_BINARIES_DIR=<bin dir>
   python3 -m pytest -v test_gpu_sharing.py`.

The GPU test ran from the local machine over SSH as root, with no new key on
the host: `SPUR_TEST_NODES=<host> SPUR_TEST_SSH_USER=root
SPUR_TEST_REMOTE_BIN_DIR=/root/spur-e2e/bin uv run --no-project --with pytest
--with paramiko --with tomli-w python -m pytest -m gpu
test_gpu_sharing_gpu.py`. The harness uploads a binary only when its size
changed; delete the remote binary to force an upload.
