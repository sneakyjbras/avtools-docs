# Release Channels & Compatibility

AV Tools ships continuously to Kubernetes and only occasionally to the
monolith, from the same codebase. Keeping those two speeds from colliding is
a matter of **channels** (where a build is allowed to land) and a small set
of **compatibility rules** (what must never break, no matter which channel a
given deployment is on).

## Version channels

| Channel | Index / registry | What lands there | Consumed by |
|---|---|---|---|
| PROD PyPI | `pypi-itdcim.cern.ch` | stable releases | Puppet (pinned to `1.8.10`) |
| QA PyPI | `qa-pypi-itdcim.cern.ch` | stable releases **and** pre-releases (e.g. `1.8.11.dev1`) | manual/QA installs, container image builds |
| Container `:qa` | `registry.cern.ch/avtools/avtools:qa` | built from source on `qa`/`master` (job `docker_build_qa`) | ArgoCD, `avtools-qa` namespace |
| Container `:prod` | `registry.cern.ch/avtools/avtools:prod` | built from source on release tags (job `docker_build_prod`) | ArgoCD, `avtools-prod` namespace |

The Kubernetes-sharding line is developed and published as **pre-release**
versions (`1.8.11.dev1`, `.dev2`, …) to **QA PyPI only**. A build only
becomes an ordinary stable version — eligible for the PROD index and a
Puppet re-pin — once it's proven out on Kubernetes in QA.

## Two independent belts keep the monolith safe

1. **Puppet's version pin.** The monolith installs an exact version
   (`1.8.10`), not `latest` — a pre-release build can't reach it just because
   it exists somewhere, since Puppet isn't tracking "whatever's newest."
2. **pip's own pre-release rule.** Even setting the pin aside, `pip` does not
   install a pre-release version when a stable version satisfies the
   request. Since pre-release sharding builds only ever land on the QA index,
   and the monolith installs from the PROD index by exact version anyway,
   this belt mostly backs up the first — but it means a *human* slip (e.g.
   dropping the pin, or installing manually without one) still wouldn't pull
   a `.devN` build.

See [The OpenStack Monolith & Puppet](monolith-puppet.md) and
[Architecture → The monolith → Kubernetes cutover](architecture.md#migration-shape).

## Moving tags on Kubernetes

The container image uses **moving tags** — `:qa` and `:prod` — not immutable
per-commit tags, combined with `imagePullPolicy: Always`. That means:

- Every CronJob run re-pulls whatever currently sits behind the tag, so a new
  `av-tools` build reaches Kubernetes automatically, with no manifest change
  and no manual rollout step.
- This is safe here specifically *because* the jobs are **stateless and
  short-lived**: a CronJob run starts fresh, does its work, and exits. There's
  no long-running Deployment/rolling-update to reason about, so there's no
  "old pod on version N, new pod on version N+1" skew window to manage —
  every run is internally consistent even if the tag moved since the last one.

!!! note "Immutable tags would be the wrong trade here"
    Pinning a CronJob to a commit-sha tag is the right call for a workload
    where you need to freeze behaviour deliberately. AV Tools' CronJobs lean
    the other way — they want to always run current QA/PROD — so a moving tag
    plus `Always` is the simpler, correct choice for this workload shape, not
    a shortcut someone forgot to clean up.

## No database migrations

`av-tools` has no schema-migration tool (no Alembic) in the application.
In practice that means:

- There's no "migration must run before the new version's pods start"
  ordering hazard to manage under a moving-tag rollout — a real risk that
  `imagePullPolicy: Always` on stateless jobs would otherwise introduce.
- Schema changes, when they happen, are a manual/DBA-coordinated step outside
  the app's own deploy path. Plan for that separately — it isn't automated by
  either channel.

## Backwards-compatibility discipline

Because the monolith and the Kubernetes deployment run against the **same
shared database** per environment (never simultaneously — see
[cutover](architecture.md#migration-shape)), and because
Kubernetes pulls whatever the moving tag currently points to, a handful of
contracts have to stay stable across ordinary releases:

- **CLI contract** — the subcommands `snmp-timeseries`, `run-eam`, `run-landb`,
  and the `--logs` flag stay stable. Both Puppet's systemd timers and the
  Helm chart's CronJob commands invoke these directly.
- **Sharding env contract** — `SHARD_TOTAL` and `JOB_COMPLETION_INDEX` keep
  their names and semantics. The Helm chart, Kubernetes' Indexed Job
  mechanism, and the monolith's "just don't set them" all depend on this not
  moving.
- **Env/secret key names** — `DATABASE_URL`, `MONIT_PASSWORD`, etc. stay
  named the same across tbag, the Puppet-templated environment, and the K8s
  `Secret`. See [Secrets & Configuration](secrets.md).
- **Shared DB shape** — since there's no migration tooling, any schema change
  has to stay compatible with whatever version is currently deployed on the
  *other* mode, for as long as both modes exist across the fleet's
  environments.

Break any of these and a routine release can silently break the monolith, or
a Kubernetes CronJob run can silently break against Postgres — with no
version pin or QA-only channel to catch it, because these rules aren't about
*which build* is running, they're about what any build is allowed to assume.
