# Getting Started

## Prerequisites

- **CERN account** with access to the `itdcim/avtools` hostgroup (for tbag secrets)
  and to an **OpenStack project** with container-cluster (Magnum) quota.
- [LXPLUS](https://abpcomputing.web.cern.ch/guides/lxplus/) access, with your
  OpenStack project sourced.
- A **Kerberos ticket** (`kinit <you>@CERN.CH`) for cluster **creation** —
  Magnum needs a Keystone trust that an OpenStack application credential cannot
  create. Day-to-day `kubectl` against an already-running cluster only needs a
  kubeconfig, not a fresh ticket.
- `kubectl` and the `openstack` CLI.
- `tbag` (Teigi CLI) to read the shared secrets — see [Secrets](secrets.md).

!!! info "Acting identity is personal; ownership is the hostgroup"
    At CERN the account that deploys is always **personal** (e.g. `jsapinat`) — there
    is no service-account alternative for OpenStack. The durable, shared thing is the
    **hostgroup** `itdcim/avtools`: its Foreman permissions and tbag secrets are what
    the team shares. Any teammate with hostgroup access can run the same deploy,
    regardless of whose account does the `apply`.

## Why Magnum

AV Tools does SNMP **and** ICMP ping. Ping needs the `NET_RAW` capability. On a
self-managed Magnum cluster you are cluster-admin, so `NET_RAW` is a one-line
`securityContext` addition — no restricted-SCC fight. See
[Architecture → Why Magnum](architecture.md#why-magnum).

## Fastest path

```bash
# from lxplus, with your OpenStack project sourced
kinit <you>@CERN.CH                              # cluster creation needs Kerberos, not an app credential
openstack coe cluster template list              # templates get retired -- confirm the name below still exists
openstack coe cluster create avtools-k8s --keypair <mykey> \
  --cluster-template kubernetes-1.33.3-1 --node-count 3
# then follow Deployment -> Kubernetes on OpenStack (Magnum)
```

!!! tip "Template name above is illustrative, not a pin"
    Cluster templates rotate; `kubernetes-1.33.3-1` may already be gone by the
    time you read this. Always take the name from
    `openstack coe cluster template list`, not from a doc or an old `tfvars`.
