# AV Tools

AV Tools runs the **SNMP/ping sweep**, plus the **EAM and LanDB reconciliation
syncs**, for the `itdcim/avtools` fleet. It is moving off a single OpenStack
monolith VM onto a **self-managed Kubernetes cluster on OpenStack (Magnum)**,
where each job is its own CronJob. Only the sweep — `snmp-timeseries` — is
**sharded**: N pods, each polling a `crc32`-hashed slice of the fleet, N threads
per pod. The reconciliation jobs stay single-node — see
[Architecture → Not everything shards](architecture.md#not-everything-shards).

!!! note "Scope of this site"
    This documents how to **deploy and operate** AV Tools on CERN Kubernetes
    (Magnum) — architecture, deployment, secrets, logging, and the operations
    runbook. Application/source docs live in the
    [`av-tools`](https://gitlab.cern.ch/itdcim/av-tools) repository.

## At a glance

| | |
|---|---|
| **Workload** | `snmp-timeseries` — sharded SNMP + ICMP ping sweep, every 5 minutes |
| **Shape** | Indexed `CronJob` — 8 shard-pods × 8 threads = 64 workers |
| **Fleet split** | `crc32(equipment_no) % SHARD_TOTAL == JOB_COMPLETION_INDEX` |
| **Also on the cluster** | `run-eam`, `run-landb` — single-node reconciliation syncs (not sharded) |
| **Hostgroup** | `itdcim/avtools` (shared; source of truth for secrets via tbag/Teigi) |
| **Secrets** | `DATABASE_URL` (DBoD), `MONIT_PASSWORD` (MONIT tenant) |
| **Metrics** | Push-based OTLP → MONIT |
| **Logs** | Container stdout → OpenSearch |

## Where to go next

- **[Getting Started](getting-started.md)** — what you need before deploying.
- **[Architecture](architecture.md)** — the sharded design and why Magnum.
- **[Repositories](repos.md)** — the four repos and how a change reaches QA/PROD without touching the monolith.
- **[Deployment](deployment/magnum.md)** — create the Magnum cluster and roll out the CronJobs.
- **[Secrets & Configuration](secrets.md)** — tbag stays the single writer; sync into the cluster.
- **[Operations Runbook](operations.md)** — scaling shards, triggering a run, known gotchas.
