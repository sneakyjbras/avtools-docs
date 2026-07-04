# Deployment — OpenShift (PaaS)

Manifests live in the [`av-tools`](https://gitlab.cern.ch/itdcim/av-tools) repo
under `deploy/openshift/`:

| File | What it is |
|---|---|
| `Dockerfile` | Two-stage build of the avtools image (context = repo root) |
| `configmap.yaml` | Non-secret config (MONIT endpoint, environment, labels) |
| `secret.example.yaml` | Template for the two runtime secrets |
| `sync-secret.sh` | Pushes live secret values from tbag into the cluster |
| `cronjob.yaml` | The sharded Indexed-Job CronJob |

## 1. Point `oc` at the project

```bash
oc login --token=<token> --server=https://api.paas.cern.ch:6443
oc project avtools
```

## 2. Build the image in-cluster

The Dockerfile lives in `deploy/openshift/` but the build **context is the repo
root**:

```bash
oc new-build --name=avtools --binary --strategy=docker
oc patch bc/avtools --type=merge \
  -p '{"spec":{"strategy":{"dockerStrategy":{"dockerfilePath":"deploy/openshift/Dockerfile"}}}}'
oc start-build avtools --from-dir=. --follow
```

## 3. Config + secret

```bash
oc apply -f deploy/openshift/configmap.yaml
./deploy/openshift/sync-secret.sh      # pulls DATABASE_URL + MONIT_PASSWORD from tbag
```

`sync-secret.sh` keeps **tbag as the single source of truth** — see [Secrets](../secrets.md).

## 4. Deploy the CronJob

```bash
oc apply -f deploy/openshift/cronjob.yaml
```

## 5. Trigger one run and watch

```bash
oc create job --from=cronjob/avtools-snmp avtools-snmp-manual
oc get pods -w                        # expect 8 pods, indices 0..7
oc logs -l app=avtools --tail=50      # look for snmp_shard_applied + cycle_summary
```

!!! warning "ICMP ping needs `CAP_NET_RAW`"
    Under the default restricted SCC (all capabilities dropped) **SNMP works but the
    ICMP ping probes may fail**. Check the ping-status metrics after the first run.
    If ping fails, either confirm the node allows unprivileged ICMP
    (`net.ipv4.ping_group_range`), or request an SCC that grants `NET_RAW` (needs
    PaaS admin approval) and add `securityContext.capabilities.add: ["NET_RAW"]`.
    If neither is possible, consider the [Magnum path](magnum.md), where you control
    this directly.
