# AV Tools

AV Tools runs the **SNMP/ping sweep** for the `itdcim/avtools` fleet. It is
moving off a single OpenStack monolith VM onto a **self-managed Kubernetes cluster
on OpenStack (Magnum)**, where it runs as a **sharded, every-5-minutes job**: N
pods, each polling a `crc32`-hashed slice of the fleet, N threads per pod.

!!! note "Scope of this site"
    This documents how to **deploy and operate** AV Tools on CERN Kubernetes
    (Magnum) — architecture, deployment, secrets, logging, and the operations
    runbook. Application/source docs live in the
    [`av-tools`](https://gitlab.cern.ch/itdcim/av-tools) repository.

## At a glance

| | |
|---|---|
| **Workload** | Sharded SNMP + ICMP ping sweep, every 5 minutes |
| **Shape** | Indexed `CronJob` — 8 shard-pods × 8 threads = 64 workers |
| **Fleet split** | `crc32(equipment_no) % SHARD_TOTAL == JOB_COMPLETION_INDEX` |
| **Hostgroup** | `itdcim/avtools` (shared; source of truth for secrets via tbag/Teigi) |
| **Secrets** | `DATABASE_URL` (DBoD), `MONIT_PASSWORD` (MONIT tenant) |
| **Metrics** | Push-based OTLP → MONIT |
| **Logs** | Container stdout → OpenSearch |

## Where to go next

- **[Getting Started](getting-started.md)** — what you need before deploying.
- **[Architecture](architecture.md)** — the sharded design and why Magnum.
- **[Deployment](deployment/magnum.md)** — create the Magnum cluster and roll out the CronJob.
- **[Secrets & Configuration](secrets.md)** — tbag stays the single writer; sync into the cluster.
- **[Operations Runbook](operations.md)** — scaling shards, triggering a run, known gotchas.
