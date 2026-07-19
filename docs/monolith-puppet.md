# The OpenStack Monolith & Puppet — why it's still here

AV Tools' **current production** deployment is a single Puppet-managed
OpenStack VM — not the new Kubernetes deployment. This page explains why that
VM still carries production traffic, what Puppet does for it, and what has to
be true before it can be retired.

## Current production, not a leftover

It's tempting to read "monolith" as legacy cruft left over from before
Kubernetes existed. It isn't — not yet. As of today, the OpenStack monolith VM
is where `snmp-timeseries`, `run-eam`, and `run-landb` actually run against
real devices in production. The Kubernetes deployment (see
[Architecture](architecture.md) and [Deployment](deployment/magnum.md)) is the
new path, being proven out environment-by-environment before it takes over.

## Managed by Puppet (`it-puppet-hostgroup-itdcim`)

The [`it-puppet-hostgroup-itdcim`](https://gitlab.cern.ch/it-puppet/it-puppet-hostgroup-itdcim)
repository holds the Puppet configuration for the `itdcim` hostgroup, which
includes the AV Tools monolith VM. Puppet is responsible for:

| Puppet manages | Detail |
|---|---|
| **Install** | `avtools`, version **pinned to `1.8.10`** (not `latest`), from the **PROD** ITDCIM PyPI index (`pypi-itdcim.cern.ch`) |
| **Scheduling** | systemd timers, roughly every 5 minutes, for all three jobs: `snmp-timeseries`, `run-eam`, `run-landb` |
| **Service user** | the OS account AV Tools runs as |
| **Secrets** | pulled from tbag/Teigi (hostgroup `itdcim/avtools`) — see [Secrets & Configuration](secrets.md) |

Because it's a single host running all three jobs with `SHARD_TOTAL` unset,
`snmp-timeseries` runs **unsharded** here — one process sweeps the whole fleet
every cycle, the same as it always has. See
[Architecture → Sharding model](architecture.md#the-sharded-sweep).

## Why the pin matters — and why it's `1.8.10`, not `latest`

Puppet installs an **exact, pinned version** (`1.8.10`), not `latest`. That
one line does two jobs at once:

1. It stops an ordinary `av-tools` release from reaching production
   unreviewed — someone has to deliberately bump the pin.
2. It **decouples the monolith from the Kubernetes sharding line entirely**.
   Builds that support sharding are published as **pre-release** versions
   (e.g. `1.8.11.dev1`) to the **QA** PyPI index (`qa-pypi-itdcim.cern.ch`)
   only. Even setting the pin aside, `pip` does not install a pre-release
   version when a stable version satisfies the request — so the monolith,
   which installs a fixed version number from the **PROD** index, was never
   going to resolve to a `.devN` build either way.

The pin and the pre-release/QA-only channel are two independent belts;
either alone keeps the sharding code off the monolith. See
[Release Channels & Compatibility](release-channels.md) for the full picture.

## Phase-out plan

The monolith is the **legacy, stable path** — kept running until the
Kubernetes deployment is proven out, then retired. Until that happens:

!!! warning "Run one or the other per environment — never both"
    The monolith and the Kubernetes deployment for a given environment (QA or
    PROD) share the **same Postgres database**. Running both against the same
    environment at the same time means two independent processes polling and
    writing the same device rows — double-polling, double-writes, and
    conflicting `run-eam`/`run-landb` reconciliation runs. It's a **cutover,
    not a parallel run**: pick one deployment mode per environment.

    It's fine for *different* environments to be on different modes at the
    same time — e.g. QA cut over to Kubernetes while PROD is still on the
    monolith. That staged rollout is the intended path. See
    [Architecture → The monolith → Kubernetes cutover](architecture.md#migration-shape).

Retiring the monolith ultimately means: decommissioning the Puppet-managed
VM, removing AV Tools' bits from the `itdcim` hostgroup in
`it-puppet-hostgroup-itdcim`, and moving secret ownership off tbag (see
[Secrets → At cutover](secrets.md#at-cutover)) — none of which has happened
yet.
