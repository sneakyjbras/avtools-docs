# Getting Started

## Prerequisites

- **CERN account** with access to the `itdcim/avtools` hostgroup (for tbag secrets)
  and to the target platform:
    - **OpenShift PaaS** — access to the `avtools` project on `paas.cern.ch`, or
    - **Magnum** — an OpenStack project with container-cluster (Magnum) quota, plus
      [LXPLUS](https://abpcomputing.web.cern.ch/guides/lxplus/) access.
- `oc` (OpenShift) **or** `kubectl` + the `openstack` CLI (Magnum).
- `tbag` (Teigi CLI) to read the shared secrets — see [Secrets](secrets.md).

!!! info "Acting identity is personal; ownership is the hostgroup"
    At CERN the account that deploys is always **personal** (e.g. `jsapinat`) — there
    is no service-account alternative for OpenShift/OpenStack. The durable, shared
    thing is the **hostgroup** `itdcim/avtools`: its Foreman permissions and tbag
    secrets are what the team shares. Any teammate with hostgroup access can run the
    same deploy, regardless of whose account does the `apply`.

## Pick your platform

| | OpenShift PaaS | Magnum (k8s on OpenStack) |
|---|---|---|
| Cluster ops | None — you get a namespace | You own the cluster lifecycle |
| ICMP ping (`NET_RAW`) | Fights the restricted SCC | Just works (you are cluster-admin) |
| Quota | Doesn't touch your OpenStack quota | Control plane + workers use it |
| Best when | You want zero cluster maintenance | You need `NET_RAW`/isolation/control |

See **[Architecture → Choosing a platform](architecture.md#choosing-a-platform)**
for the full trade-off. The decision hinges on one question: **does ICMP ping need
`NET_RAW`, and can PaaS admins grant it?** If yes-and-yes, stay on PaaS; if not,
Magnum removes the problem.

## Fastest path

=== "OpenShift"

    ```bash
    oc login --token=<token> --server=https://api.paas.cern.ch:6443
    oc project avtools
    # then follow Deployment -> OpenShift (PaaS)
    ```

=== "Magnum"

    ```bash
    # from lxplus, with your OpenStack project sourced
    openstack coe cluster create avtools-k8s --keypair <mykey> \
      --cluster-template kubernetes-1.33.3-1 --node-count 3
    # then follow Deployment -> Kubernetes on OpenStack (Magnum)
    ```
