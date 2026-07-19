# Repositories

AV Tools splits across **four single-responsibility repositories**. Each owns one
concern and ships on its own release cadence — a dashboard tweak doesn't need to
drag through the app's test suite, and a Terraform change doesn't need a new
container image.

| Repo | Owns | Produces |
|---|---|---|
| [`av-tools`](https://gitlab.cern.ch/itdcim/av-tools) | the application (SNMP/EAM/LanDB/Postgres) **and its own container build** | an **RPM** (Puppet VMs), a **wheel** (ITDCIM PyPI), and the **image** `registry.cern.ch/avtools/avtools:{qa,prod}` — one codebase, three channels |
| [`av-tools-infra`](https://gitlab.cern.ch/itdcim/av-tools-infra) | Terraform (Magnum cluster), Helm chart, ArgoCD | the running deployment |
| [`av-tools-grafana`](https://gitlab.cern.ch/itdcim/av-tools-grafana) | dashboards + alert rulegroups | Grafana panels/alerts |

## One codebase, three outputs

`av-tools` ships the **same code** three ways:

- an **RPM**, installed by Puppet onto the `itdcim/avtools` monolith VMs. QA and
  PROD monoliths both run today — the QA box, **`avtools-faol`**, is the staging
  environment.
- a **wheel**, published to the ITDCIM PyPI index (how the monolith installs the package).
- a **container image**, built **from source** in `av-tools`'s own CI (kaniko) and
  pushed to `registry.cern.ch/avtools/avtools:{qa,prod}`. The Kubernetes Helm chart
  pulls this image.

```
                        ┌── RPM ──────────────────> Puppet ──> itdcim/avtools monolith VMs (QA: avtools-faol, PROD)
av-tools (one codebase) ┼── wheel ─────────────────> ITDCIM PyPI
                        └── image (kaniko, source) ─> registry.cern.ch/avtools/avtools:{qa,prod} ─> K8s Helm chart
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

## Why the image is built in `av-tools`, not a separate repo

The container image is built **from source** inside `av-tools`'s own CI — there is
no separate image repo. An earlier `av-tools-image` repo (which `pip install`ed
the published wheel) was tried and **removed**: it added a publish-first
dependency (wheel → PyPI → pinned version) for no real gain, because the image and
Puppet's wheel are **independent artifact streams**. Building from source means
the image carries exactly the branch's code — including sharding — with no
wheel-publish step, and Puppet is unaffected (it installs the wheel from PyPI,
never the image). The image `docker_build` job is decoupled from the heavy app
pipeline with `needs: []`, so it doesn't wait on koji/test/e2e. See
[Deployment → Build & publish the image](deployment/magnum.md).

## Why split `av-tools-grafana` out

Dashboard and alert changes used to drag through `av-tools`'s full pipeline — koji
RPM build, test suite, integration, e2e. `av-tools-grafana` is just **validate
JSON → push to Grafana**: a panel tweak ships in seconds, decoupled from app
releases, which change far less often than dashboards do.
