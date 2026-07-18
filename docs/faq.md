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

### Why four repos — `av-tools`, `av-tools-image`, `av-tools-infra`, `av-tools-grafana`?

Each concern has a different lifecycle and shouldn't share a pipeline.
**`av-tools`** owns the application and publishes it two ways — an RPM for
Puppet, a wheel for the ITDCIM PyPI index. **`av-tools-image`** owns the
`Dockerfile` and contains **no application code**: it `pip install`s the
published wheel into a container, so a container-build tweak never needs an app
release. **`av-tools-infra`** owns the Terraform, Helm chart, and ArgoCD
manifests that deploy it. **`av-tools-grafana`** owns dashboards and alerts,
split out so a panel edit ships in seconds instead of riding the app's full
koji/test/e2e pipeline. The contract between all of them is the **image tag**
(`registry.cern.ch/itdcim/avtools:{qa,prod}`) and the **wheel** published to
ITDCIM PyPI. See [Repositories](repos.md).

### Why aren't `run-eam` and `run-landb` sharded like the SNMP sweep?

They're bulk **reconciliation** syncs, not per-device polls: each fetches the
*entire* EAM/LanDB inventory and diffs it against Postgres to catch orphaned
rows. That diff needs the global view — a shard that only sees its own slice
can't safely detect what's missing. Sharding them would just run N redundant
full syncs that fight over the same rows. They run `shards: 1` on purpose. See
[Architecture → Not everything shards](architecture.md#not-everything-shards).

### Does a sharding-capable build ever reach the Puppet monolith?

No, by two independent mechanisms. Puppet installs `avtools=latest` from
**PROD PyPI** and is pinned off `latest`; sharding-capable builds are published
to **QA PyPI** only, so Puppet's index never has one to install. And even if it
did, `SHARD_TOTAL=1` — the monolith's implicit default, since it never sets the
env — is a no-op in the sharding formula. See [Repositories](repos.md).

### Why does cluster creation need Kerberos instead of my usual OpenStack app credential?

Magnum creates a Keystone **trust** so the cluster can call OpenStack back (load
balancers, volumes, the autoscaler). Keystone refuses trust creation from an
application credential — even an `unrestricted` one — so cluster *creation* has
to run under a personal Kerberos identity (`kinit`). Day-to-day `kubectl`
against an already-running cluster doesn't need this. See
[Deployment](deployment/magnum.md).
