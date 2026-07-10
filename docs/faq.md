# FAQ

### Why 8 shards × 8 threads and not 64 nodes?

Sharding gives partition-tolerance and HA through **independent shards + retry**, not
through many machines. 64 nodes would mean 64 failure domains and far more
coordination/partition surface. 8 shard-pods (the scheduler spreads them across the
cluster) × 8 threads gives the same 64 workers with a cluster you can reason about.

### Is this OpenShift or OpenStack?

Neither the PaaS nor a VM: the **workload runs on Kubernetes** — a self-managed
**Magnum** cluster on OpenStack. The single OpenStack **VM** is only the legacy
monolith, which runs alongside during migration and is then retired. See
[Architecture](architecture.md).

### Why does the deploy run under a personal account?

At CERN the acting identity for OpenStack is always personal — there's no
service-account alternative. Ownership lives in the shared **hostgroup**
`itdcim/avtools` (Foreman permissions + tbag secrets), so the person who runs the
deploy is not a single point of failure. Optionally add an e-group as project admin
to make that explicit.

### Where do the secrets come from?

tbag/Teigi (hostgroup `itdcim/avtools`) is the single source of truth; `sync-secret.sh`
pushes the live values into a K8s `Secret`. The sweep needs only `DATABASE_URL` and
`MONIT_PASSWORD`. See [Secrets](secrets.md).

### Do I need a ServiceMonitor / Prometheus scrape?

No. Metrics are **pushed** via OTLP to MONIT, not scraped. See
[Logging & Monitoring](logging-monitoring.md).

### How do I build the image for Magnum?

Magnum has no in-cluster build — build in GitLab CI and push to `registry.cern.ch`
(Harbor), then point the chart image at `registry.cern.ch/itdcim/avtools:{qa,prod}` (built by the `av-tools` CI). See
[Deployment](deployment/magnum.md).

### This site is for AV Tools — can we reuse it for timeseries-DIP?

Yes. This is a standard CERN MkDocs Material site (one repo → one
`*.docs.cern.ch`). Copy the structure into a `timeseries-dip-docs` repo, swap the
content, and register a new Web Services site.

### Why two repos — `av-tools` vs `av-tools-infra`?

App code and deployment config have different lifecycles. **`av-tools`** owns the
application and builds/publishes the container image; **`av-tools-infra`** owns the
Terraform, Helm chart, and ArgoCD manifests that deploy it. The contract between them
is the image tag (`registry.cern.ch/itdcim/avtools:{qa,prod}`). The `Dockerfile` stays
in `av-tools` because it *packages the app* (it needs `src/` + `pyproject.toml`) — the
same reason the PyPI wheel is built there, not in the infra repo.
