# Deployment — Kubernetes on OpenStack (Magnum)

AV Tools runs on a self-managed **Magnum** cluster (so it can add `NET_RAW` for ICMP
ping — see [Architecture → Why Magnum](../architecture.md#why-magnum)), deployed by
**GitOps (Terraform + ArgoCD + Helm)**.

!!! tip "Rebuilding from scratch?"
    The cluster is disposable — to destroy and recreate it (flavor change, recovery),
    see [Rebuilding the Cluster](rebuild.md) and the one-command `scripts/bootstrap.sh`.
    This page covers the underlying manual steps.

## Four repositories

| Repo | Owns | Produces |
|---|---|---|
| [`av-tools`](https://gitlab.cern.ch/itdcim/av-tools) | app code (SNMP/EAM/LanDB/Postgres) **+ the container build** | RPM (Puppet), wheel (ITDCIM PyPI), image `registry.cern.ch/avtools/avtools:{qa,prod}` |
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

## 2. Build & publish the image (in `av-tools`)

Magnum has no in-cluster build. The image is built **from source** by `av-tools`'s
own CI: a kaniko job (`docker_build_qa`/`docker_build_prod`) runs `poetry build`
and wraps the result in a container (see [Repositories](../repos.md)). It pushes
`registry.cern.ch/avtools/avtools:qa` (on `qa`/`master`/the k8s image branch) and
`:prod` (on tags). Auth uses a Harbor robot account (`robot-avtools+avtools-ci`;
its token is the `REGISTRY_PASSWORD` CI variable). For the cluster to *pull* the
private image, create a pull secret:

```bash
kubectl -n avtools-qa create secret docker-registry harbor-avtools \
  --docker-server=registry.cern.ch \
  --docker-username='<robot-account>' --docker-password='<robot-password>' \
  --docker-email=no-reply@cern.ch
```

The chart references it via `imagePullSecrets`.

## 3. Deploy via ArgoCD (GitOps)

Install ArgoCD once, give it a read token for the (private) repo, then apply the
root app-of-apps.

```bash
# 3a. Install ArgoCD. Use --server-side: the ApplicationSet CRD exceeds the
#     262144-byte client-side apply annotation limit and fails otherwise.
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3b. av-tools-infra is PRIVATE. Register a read token (GitLab Deploy Token,
#     scope read_repository) so ArgoCD can pull the chart — else the app shows
#     "authentication required: HTTP Basic: Access denied".
kubectl apply -f - <<'YAML'
apiVersion: v1
kind: Secret
metadata:
  name: repo-av-tools-infra
  namespace: argocd
  labels: { argocd.argoproj.io/secret-type: repository }
stringData:
  type: git
  url: https://gitlab.cern.ch/itdcim/av-tools-infra.git
  username: <deploy-token-username>
  password: <deploy-token>
YAML

# 3c. Apply the root app — it reconciles the AppProject, the ApplicationSet
#     (avtools-qa tracks master, avtools-prod tracks prod), and kube-prometheus-stack.
kubectl apply -n argocd -f argocd/app-of-apps.yaml
```

Admin password + UI: `kubectl -n argocd get secret argocd-initial-admin-secret
-o jsonpath='{.data.password}' | base64 -d`, then
`kubectl port-forward svc/argocd-server -n argocd 8080:443`.

## 4. Secrets

Two secrets per environment namespace; neither is managed by the chart.

**Image pull secret** — so the cluster can pull the private Harbor image (else
pods sit in `ImagePullBackOff`):

```bash
kubectl -n avtools-qa create secret docker-registry harbor-avtools \
  --docker-server=registry.cern.ch \
  --docker-username='robot-avtools+avtools-ci' \
  --docker-password='<harbor-robot-token>'
```

**App secrets** — `DATABASE_URL`, `MONIT_PASSWORD`, `LANDB_CLIENT_SECRET`, etc.,
bridged from tbag on an `itdcim/avtools` host (aiadm works if you're a hostgroup
admin; write the kubeconfig to `/tmp` if AFS home is full):

```bash
AVTOOLS_ENVIRONMENT=qa ./scripts/sync-secret.sh   # upserts secret/avtools-secrets
```

Without the app secret the pods fail with `CreateContainerConfigError`.

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

!!! warning "Empty fleet? (`devices_fleet=0` / `skipped_no_targets`)"
    If `snmp-timeseries` reports `devices_fleet=0` even though the environment has
    devices, check `run-landb`'s logs. `cern_oauthlib` writes its OAuth token
    cache under `~/.cache`, and with `readOnlyRootFilesystem: true` that write
    fails (`OSError: [Errno 30] Read-only file system: '/home/avtools/.cache'`) —
    so **LanDB auth fails, no IPs are cached, and the SNMP fleet comes back
    empty** (a failed sync even *deletes* the previously-cached IPs). The chart
    mounts a writable `cachedir` emptyDir at `/home/avtools/.cache` to fix this;
    confirm it's present on the CronJob pod spec. (`run-eam` auths differently, so
    it keeps working and masks the problem.)

## Lifecycle

```bash
openstack coe cluster resize  avtools-k8s 5           # or terraform apply with node_count=5
openstack coe cluster upgrade avtools-k8s <template>  # upgrade k8s version
openstack coe cluster delete  avtools-k8s             # NB: LBs/volumes may persist - clean up
```
