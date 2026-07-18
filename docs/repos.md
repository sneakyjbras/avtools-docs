# Repositories

AV Tools splits across **four single-responsibility repositories**. Each owns one
concern and ships on its own release cadence — a dashboard tweak doesn't need to
drag through the app's test suite, and a Terraform change doesn't need a new
container image.

| Repo | Owns | Produces |
|---|---|---|
| [`av-tools`](https://gitlab.cern.ch/itdcim/av-tools) | the application (SNMP/EAM/LanDB/Postgres) | an **RPM** (Puppet VMs) and a **wheel** (ITDCIM PyPI) — same code, two channels |
| [`av-tools-image`](https://gitlab.cern.ch/itdcim/av-tools-image) | the container build — **no application code** | `registry.cern.ch/itdcim/avtools:{qa,prod}` |
| [`av-tools-infra`](https://gitlab.cern.ch/itdcim/av-tools-infra) | Terraform (Magnum cluster), Helm chart, ArgoCD | the running deployment |
| [`av-tools-grafana`](https://gitlab.cern.ch/itdcim/av-tools-grafana) | dashboards + alert rulegroups | Grafana panels/alerts |

## One wheel, two homes

`av-tools` publishes the **same package** two ways:

- an **RPM**, installed by Puppet onto the `itdcim/avtools` monolith VMs. QA and
  PROD monoliths both run today — the QA box, **`avtools-faol`**, is the staging
  environment.
- a **wheel**, published to the ITDCIM PyPI index, which `av-tools-image`
  `pip install`s to build the container. No forked or duplicated app code — the
  container runs the exact package Puppet would install.

```
av-tools ──wheel──> ITDCIM PyPI ──installed by──> av-tools-image ──> registry.cern.ch/itdcim/avtools:{qa,prod}
        └───RPM───> Puppet ──installed on──> itdcim/avtools monolith VMs (QA: avtools-faol, PROD)
```

## Channel isolation keeps the monolith safe

Puppet installs `avtools=latest` from **PROD PyPI** (`pypi-itdcim.cern.ch`) and is
**pinned off `latest`** — it does not track a branch or a k8s-era build line. The
sharding-capable builds (the ones that understand `SHARD_TOTAL` /
`JOB_COMPLETION_INDEX`) are published to **QA PyPI** (`qa-pypi-itdcim.cern.ch`)
**only**. Consequences:

- The monolith never installs a sharding-capable build — Puppet's index doesn't
  carry one.
- Even if it somehow did, `SHARD_TOTAL=1` — implicit on the monolith, which never
  sets the env — is a no-op in the sharding formula. Behaviour is unchanged.

!!! tip "Two independent belts, not one"
    Puppet's PyPI channel pin and the `SHARD_TOTAL=1` no-op each protect the
    monolith on their own. Either alone is enough; together, the migration has no
    single point of failure for "did this touch QA/PROD".

## Why split `av-tools-image` out

Containerizing needed **zero changes to `av-tools`**: the image just installs the
already-published wheel. Keeping the `Dockerfile` in its own repo means a build
tweak (base image, layer caching) doesn't need an `av-tools` release, and an
`av-tools` release doesn't need a new image build to reach QA — ArgoCD just
re-pulls `:qa` once `av-tools-image`'s CI republishes it. See
[Deployment → Build & publish the image](deployment/magnum.md).

## Why split `av-tools-grafana` out

Dashboard and alert changes used to drag through `av-tools`'s full pipeline — koji
RPM build, test suite, integration, e2e. `av-tools-grafana` is just **validate
JSON → push to Grafana**: a panel tweak ships in seconds, decoupled from app
releases, which change far less often than dashboards do.
