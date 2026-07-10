# Deployment — Kubernetes on OpenStack (Magnum)

AV Tools runs on a self-managed **Magnum** cluster (so it can add `NET_RAW` for ICMP
ping — see [Architecture → Why Magnum](../architecture.md#why-magnum)), deployed by
**GitOps (Terraform + ArgoCD + Helm)**.

## Two repositories

| Repo | Owns | Produces |
|---|---|---|
| [`av-tools`](https://gitlab.cern.ch/itdcim/av-tools) | app code, `Dockerfile`, Grafana dashboards, Sentry SDK | the image `registry.cern.ch/itdcim/avtools:{qa,prod}` |
| [`av-tools-infra`](https://gitlab.cern.ch/itdcim/av-tools-infra) | `terraform/`, Helm `chart/`, `argocd/`, `scripts/sync-secret.sh` | the running deployment |

The contract between them is the **image tag**. Everything below runs from
`av-tools-infra` unless noted.

!!! info "Reference"
    Grounded in the CERN Kubernetes docs (`kubernetes.docs.cern.ch`). Template/flavor
    names change over time — re-check `openstack coe cluster template list` first.

## 1. Provision the cluster (Terraform)

From `av-tools-infra/terraform/`, with an OpenStack application credential sourced
(see `terraform/README.md`):

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

## 2. Build & publish the image (av-tools)

Magnum has no in-cluster build. The **`av-tools`** CI builds `Dockerfile` with kaniko
and pushes `registry.cern.ch/itdcim/avtools:qa` (on `qa`/`master`) and `:prod` (on
tags). For a private Harbor repo, create a robot account and a pull secret:

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

## Lifecycle

```bash
openstack coe cluster resize  avtools-k8s 5           # or terraform apply with node_count=5
openstack coe cluster upgrade avtools-k8s <template>  # upgrade k8s version
openstack coe cluster delete  avtools-k8s             # NB: LBs/volumes may persist - clean up
```
