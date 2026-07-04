# Architecture

## The sharded sweep

AV Tools polls the whole `itdcim/avtools` device fleet every 5 minutes. To keep
each cycle fast and partition-tolerant, the fleet is **split N ways** and run as a
Kubernetes **Indexed Job**:

- Each pod receives `JOB_COMPLETION_INDEX = 0 … N-1` (set automatically by Indexed
  mode), which the app reads as `--shard-index`.
- A pod keeps only the devices where
  `crc32(equipment_no) % SHARD_TOTAL == JOB_COMPLETION_INDEX`.
- `completions` (in the CronJob) **must equal** `SHARD_TOTAL` (env), or some devices
  are never polled. The app refuses to start on a mismatched/out-of-range shard.

Current sizing: **8 shards × 8 threads = 64 concurrent workers**.

```
CronJob (*/5 * * * *)
└── Indexed Job  completions=8, parallelism=8
    ├── pod shard 0  (THREADS=8) ─┐
    ├── pod shard 1  (THREADS=8)  │  each polls crc32(dev) % 8 == index
    ├── ...                       │  spread across nodes by the scheduler
    └── pod shard 7  (THREADS=8) ─┘
```

!!! tip "Why sharding gives HA, not extra VMs"
    Partition-tolerance and high availability here come from **independent shards +
    retry**, not from running many VMs. A failed shard is retried (`backoffLimit`)
    and, worst case, re-run on the next 5-minute cycle. Losing a node reschedules its
    shards elsewhere. "8 nodes × 8 threads" means 8 *shard-pods* the scheduler places
    across the cluster — not 8 machines you manage.

## Migration shape

The legacy **monolith VM** (Puppet-managed, `itdcim/avtools`) runs **alongside** the
Kubernetes deployment during migration, then is retired at cutover
(strangler-fig). During the overlap:

- Both emit to OpenSearch — tag the streams (`source=monolith` vs `source=k8s`) so
  they can be compared and the monolith stream dropped cleanly at the end.
- **tbag/Teigi stays the single source of truth** for the shared secrets, feeding
  both worlds. See [Secrets](secrets.md).

## Choosing a platform

The workload is a 5-minute CronJob, not a long-running service — so the platform
choice is about **operational cost vs. control**, and one technical driver: ICMP ping.

| | OpenShift PaaS | Magnum (self-managed k8s) |
|---|---|---|
| Control plane | CERN operates it | Magnum provisions; **you own upgrades/health** |
| Isolation | Shared, multi-tenant namespace | Your own cluster |
| ICMP ping (`NET_RAW`) | Restricted SCC drops caps — must request an SCC | Cluster-admin: add `NET_RAW` freely |
| Quota | None of your OpenStack quota | Control plane + workers consume it |
| Image build | In-cluster `oc new-build` / ImageStream | Build in CI → push to `registry.cern.ch` |

**Decision rule.** AV Tools does SNMP **and** ICMP ping. SNMP works everywhere;
ping needs `NET_RAW`, which OpenShift's restricted SCC strips.

- If ping works unprivileged, **or** PaaS admins grant a `NET_RAW` SCC → **stay on
  OpenShift PaaS** (zero cluster maintenance).
- If neither → **Magnum**, where you are cluster-admin and `NET_RAW` is a one-line
  `securityContext` addition. Magnum is semi-managed (lifecycle via
  `openstack coe cluster …`), so this is not "run a control plane by hand" — but you
  do own upgrades, node health, and the quota footprint.

Almost everything else (the `CronJob`, `ConfigMap`, `Secret`, and the tbag sync)
is plain Kubernetes and ports between the two paths unchanged.
