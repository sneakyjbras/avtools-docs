# Deployment — Kubernetes on OpenStack (Magnum)

AV Tools runs on a self-managed **Magnum** cluster (so it can add `NET_RAW` for ICMP
ping — see [Architecture → Why Magnum](../architecture.md#why-magnum)), deployed by
**GitOps (Terraform + ArgoCD + Helm)**.

## Four repositories

| Repo | Owns | Produces |
|---|---|---|
| [`av-tools`](https://gitlab.cern.ch/itdcim/av-tools) | app code (SNMP/EAM/LanDB/Postgres) | RPM (Puppet) + wheel (ITDCIM PyPI) |
| [`av-tools-image`](https://gitlab.cern.ch/itdcim/av-tools-image) | `Dockerfile` — no app code, `pip install`s the published wheel | the image `registry.cern.ch/itdcim/avtools:{qa,prod}` |
| [`av-tools-infra`](https://gitlab.cern.ch/itdcim/av-tools-infra) | `terraform/`, Helm `chart/`, `argocd/`, `scripts/sync-secret.sh` | the running deployment |
| [`av-tools-grafana`](https://gitlab.cern.ch/itdcim/av-tools-grafana) | dashboards + alert rulegroups | Grafana panels/alerts |

The contract between them is the **image tag**. Everything below runs from
`av-tools-infra` unless noted. See [Repositories](../repos.md) for the full
publish chain (RPM vs wheel, QA vs PROD PyPI) and why each split exists.

!!! info "Reference"
    Grounded in the CERN Kubernetes docs (`kubernetes.docs.cern.ch`). Template/flavor
    names change over time — re-check `openstack coe cluster template list` first.

## 1. Provision the cluster (Terraform)

From `av-tools-infra/terraform/`, with a **Kerberos**-derived OpenStack token
sourced (`kinit` + `scripts/os-auth.sh` — see `terraform/README.md`; an
application credential works for `plan` but **not** for cluster creation, see
below):

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars   # set keypair, sizing
terraform init   # GitLab-managed state (address in versions.tf / CI)
terraform plan
terraform apply  # 1 master + 3 workers, cluster-autoscaler 3-6
```

Then get a kubeconfig:

```bash
$(openstack coe cluster config avtools-k8s)
kubectl get nodes
```

!!! danger "Authenticate with Kerberos, not an application credential"
    Magnum creates a Keystone **trust** so the cluster can call OpenStack back
    (load balancers, volumes, the autoscaler). Keystone refuses trust creation
    from an application credential — **even `unrestricted = True`** — so
    `terraform apply` for cluster *creation* must run under a personal
    **Kerberos** identity (`kinit <you>@CERN.CH`), not an app-cred `clouds.yaml`.
    Symptom if you get this wrong: a clean plan, then `CREATE_FAILED: Failed to
    create trustee or trust for Cluster` about 30 seconds into `apply`.

!!! warning "Master flavor must be `m2.large`, not the template default `m2.medium`"
    CERN installs roughly 15 heavy addons on the master at boot (Falco,
    Prometheus, Velero, cert-manager, Cilium, four CSI drivers, the autoscaler,
    node-feature-discovery, fluentd…) via one Helm job. `m2.medium` (3.75 GB)
    starves under that weight — the control plane runs low on headroom, addon
    probes time out, pods liveness-restart in a loop, and the Helm install never
    converges: `CREATE_FAILED`, with every VM deceptively `ACTIVE` in Horizon.
    `m2.large` (7.5 GB) has the headroom and builds cleanly.

!!! note "Cluster templates get retired"
    Run `openstack coe cluster template list` before every apply — a pinned
    template name can vanish between builds. Avoid `-argo` variants (they set
    `cern_chart_enabled: false` and expect CERN's newer addon delivery).

## 2. Build & publish the image (av-tools-image)

Magnum has no in-cluster build, and **`av-tools-image`** contains **no
application code**: its CI `pip install`s the `avtools` wheel that `av-tools`
already published to the ITDCIM PyPI index, and wraps it in a container with
kaniko — QA builds pull from QA PyPI, PROD builds pull from PROD PyPI (see
[Repositories](../repos.md)). It pushes `registry.cern.ch/itdcim/avtools:qa` (on
`qa`/`master`) and `:prod` (on tags). For a private Harbor repo, create a robot
account and a pull secret:

```bash
kubectl -n avtools-qa create secret docker-registry harbor-avtools \
  --docker-server=registry.cern.ch \
  --docker-username='<robot-account>' --docker-password='<robot-password>' \
  --docker-email=no-reply@cern.ch
```

The chart references it via `imagePullSecrets`.

## 3. Deploy via ArgoCD (GitOps)

Install ArgoCD once, register `av-tools-infra`, then apply the root app-of-apps
(details in `av-tools-infra/argocd/README.md`):

```bash
kubectl apply -n argocd -f argocd/app-of-apps.yaml
```

ArgoCD then reconciles the AppProject, the ApplicationSet (`avtools-qa` tracking
`master`, `avtools-prod` tracking `prod`), and kube-prometheus-stack. To render
locally without ArgoCD:

```bash
helm template avtools chart -n avtools-qa \
  -f chart/values.yaml -f chart/values-qa.yaml | kubectl apply -n avtools-qa -f -
```

## 4. Secrets (tbag to K8s Secret)

The chart never contains secret values. Bridge them from tbag on an `itdcim/avtools`
host (the monolith during the overlap):

```bash
AVTOOLS_ENVIRONMENT=qa ./scripts/sync-secret.sh   # upserts secret/avtools-secrets
```

`NET_RAW` for ICMP is already set on the `snmp-timeseries` CronJob (you are
cluster-admin on Magnum), so ping works out of the box.

## 5. Trigger one run and watch

```bash
kubectl -n avtools-qa create job --from=cronjob/avtools-snmp-timeseries manual
kubectl -n avtools-qa get pods -w                                  # 8 pods, indices 0..7
kubectl -n avtools-qa logs -l app.kubernetes.io/component=snmp-timeseries --tail=50
```

!!! note "`run-eam` / `run-landb` are single-pod"
    Only `snmp-timeseries` fans out to 8 indexed pods. Triggering
    `avtools-run-eam` or `avtools-run-landb` manually yields exactly **one**
    pod — that's by design, not a misconfiguration. See
    [Architecture → Not everything shards](../architecture.md#not-everything-shards).

## Lifecycle

```bash
openstack coe cluster resize  avtools-k8s 5           # or terraform apply with node_count=5
openstack coe cluster upgrade avtools-k8s <template>  # upgrade k8s version
openstack coe cluster delete  avtools-k8s             # NB: LBs/volumes may persist - clean up
```
