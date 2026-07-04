# Deployment — Kubernetes on OpenStack (Magnum)

Use this path if AV Tools needs `NET_RAW` for ICMP ping and PaaS can't grant it, or
you want an isolated, self-owned cluster. See
[Architecture → Choosing a platform](../architecture.md#choosing-a-platform).

!!! info "Reference"
    Grounded in the CERN Kubernetes docs (`kubernetes.docs.cern.ch`) — *Getting
    Started* and *Registry → Quickstart*. Template/flavor names change over time;
    re-check `openstack coe cluster template list` before creating.

## 1. Create the cluster

From LXPLUS, with your OpenStack project sourced:

```bash
# See available templates (versions change frequently)
openstack coe cluster template list

# 1 master + 3 workers = 4 instances (fits a 10-instance / 10-core project)
openstack coe cluster create avtools-k8s --keypair <mykey> \
  --cluster-template kubernetes-1.33.3-1 --node-count 3

openstack coe cluster list           # wait for CREATE_COMPLETE
```

!!! tip "Sizing for a 10-core / 10-instance project"
    `--node-count` is the **worker** count; `master_count` defaults to 1. The 8
    shard-pods request 250m CPU each (2 cores total, bursting to 8), so **1 master +
    3 workers** at a modest flavor holds a full 8-shard cycle with headroom. Losing
    one of three workers just reschedules its shards.

## 2. Get kubectl access

```bash
$(openstack coe cluster config avtools-k8s)     # or: openstack coe cluster config avtools-k8s > env.sh && . env.sh
kubectl get nodes
```

## 3. Image via registry.cern.ch (Harbor)

Magnum has no in-cluster build. Build in GitLab CI and push to
[`registry.cern.ch`](https://registry.cern.ch):

```bash
docker login registry.cern.ch -u <username>     # CLI secret from your Harbor profile
docker build -f deploy/openshift/Dockerfile -t registry.cern.ch/itdcim/avtools:latest .
docker push registry.cern.ch/itdcim/avtools:latest
```

For a **private** repo, create a robot account and a pull secret:

```bash
kubectl create secret docker-registry harbor-avtools \
  --docker-server=registry.cern.ch \
  --docker-username='<robot-account>' \
  --docker-password='<robot-password>' \
  --docker-email=no-reply@cern.ch
```

Then reference it in the pod spec (`imagePullSecrets: [{name: harbor-avtools}]`) and
point the CronJob's `image:` at `registry.cern.ch/itdcim/avtools:latest`.

## 4. Config, secret, CronJob

The plain-Kubernetes manifests port over unchanged:

```bash
kubectl apply -f deploy/openshift/configmap.yaml
./deploy/openshift/sync-secret.sh                # uses `oc`? swap for `kubectl` — see note
kubectl apply -f deploy/openshift/cronjob.yaml
```

!!! note "Two edits vs the OpenShift manifests"
    1. **Image** — point `cronjob.yaml` at `registry.cern.ch/itdcim/avtools:latest`
       (not the OpenShift internal ImageStream) and add `imagePullSecrets` if private.
    2. **`NET_RAW`** — you are cluster-admin here, so add it directly and the ICMP
       gotcha disappears:
       ```yaml
       securityContext:
         capabilities:
           add: ["NET_RAW"]
       ```
    `sync-secret.sh` uses `oc`; on a Magnum cluster either alias `oc=kubectl` or
    change the two `oc` calls to `kubectl` (the `create secret … --dry-run | apply`
    shape is identical).

## 5. Trigger one run and watch

```bash
kubectl create job --from=cronjob/avtools-snmp avtools-snmp-manual
kubectl get pods -w
kubectl logs -l app=avtools --tail=50
```

## Lifecycle

```bash
openstack coe cluster resize  avtools-k8s 5      # scale workers
openstack coe cluster upgrade avtools-k8s <template>   # upgrade k8s version
openstack coe cluster delete  avtools-k8s        # NB: LBs/volumes may persist — clean up
```
