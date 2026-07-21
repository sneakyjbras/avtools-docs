# Rebuilding the Cluster

The Magnum cluster is **disposable**. Everything on it is either in git (GitOps via
ArgoCD) or in Teigi/tbag (secrets), so it can be destroyed and rebuilt from scratch —
for a flavor change, a version bump, or disaster recovery.

!!! info "Canonical runbook"
    This page is the **overview**. The authoritative, command-by-command runbook lives
    next to the scripts it drives, in the infra repo:
    [`av-tools-infra/docs/deployment/bootstrap.md`](https://gitlab.cern.ch/itdcim/av-tools-infra/-/blob/master/docs/deployment/bootstrap.md).
    Keeping the exact commands in one place (versioned with `bootstrap.sh`) avoids drift.

!!! note "Prod is unaffected"
    Production runs on the **Puppet monolith**, not on k8s. A rebuild only blips the
    **QA** k8s monitoring for ~15–25 min. See
    [The Monolith & Puppet](../monolith-puppet.md).

## The entry point

`scripts/bootstrap.sh` orchestrates the whole rebuild — it does not reinvent anything,
just chains the preflight checks, `os-auth.sh`, Terraform, the `argocd/` manifests, and
`sync-secret.sh`.

```bash
./scripts/bootstrap.sh --check     # read-only orientation — what's ready, what's missing
./scripts/bootstrap.sh             # rebuild end-to-end (type REBUILD to confirm)
```

`start-here.sh` is now a thin shim for `bootstrap.sh --check`.

## ⚠️ The rebuild is two-host

The rebuild needs two things that live on **different hosts**, and no single CERN host
has both — so in practice it runs as a two-host procedure:

| Need | Runs on | Why |
| --- | --- | --- |
| **Terraform + OpenStack auth** | your **workstation / lxplus** | terraform + the kerberos python libs `os-auth.sh` needs |
| **tbag secrets** | **aiadm** (an `itdcim/avtools` host) | tbag is hostgroup-scoped; the workstation isn't a member, aiadm has no terraform |

High-level flow:

1. **Workstation** — set the tfvars target, `terraform apply` (destroy + rebuild), fetch
   the kubeconfig, install ArgoCD, label the nodes.
2. **aiadm** — create the three Secrets from tbag (app secrets via `sync-secret.sh`, the
   Harbor pull secret, and the ArgoCD repo deploy token).
3. **Workstation** — apply the app-of-apps; ArgoCD reconciles and the app comes back.

The exact commands for each step (including the aiadm gotchas — `OS_PROJECT_NAME`,
`mkdir /tmp/kube`, and pinning `KUBECONFIG` before `kubectl create secret`) are in the
[infra runbook](https://gitlab.cern.ch/itdcim/av-tools-infra/-/blob/master/docs/deployment/bootstrap.md).

## Prerequisites

- `kinit <you>@CERN.CH`
- `terraform/terraform.tfvars` set to the target (gitignored; `--check` verifies it):
  `master_flavor` and `flavor` = `m2.large`, `node_count` = 4, `autoscale_min/max` = 4
  (4 + 4×4 = 20 cores, the exact quota).
- `scripts/env.sh` filled in.
- The three tbag keys stored once, on an `itdcim/avtools` host:
  ```bash
  tbag set --hg itdcim/avtools harbor_robot_token   # prompts for the value
  tbag set --hg itdcim/avtools argocd_repo_user
  tbag set --hg itdcim/avtools argocd_repo_token
  ```

## Node names

Nodes are labelled `wow=<name>` (World of Warcraft; workers alphabetical). Labels are
**lost on every rebuild** and reapplied by the runbook. View: `kubectl get nodes -L wow`.

| Magnum node | `wow=` label | Role |
| --- | --- | --- |
| `…-master-0` | **medivh** | control plane |
| `…-node-0` | **arthas** | worker |
| `…-node-1` | **bolvar** | worker |
| `…-node-2` | **cairne** | worker |
| `…-node-3` | **draka** | worker |

Extend alphabetically as workers are added (`draka, eitrigg, grommash, …`).

## Current topology

1 × `m2.large` master + 4 × `m2.large` workers = **20 cores** (the exact project quota).
The move to `m2.large` (from `m2.medium`) was a **memory** fix — the smaller nodes ran
94–96% memory-allocated, starving pod scheduling and jittering the metric sweep. See
[Deployment](magnum.md) for the cluster architecture.
