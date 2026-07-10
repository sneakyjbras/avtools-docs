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

## Why Magnum

The workload is a 5-minute CronJob, not a long-running service. It runs on a
**self-managed Kubernetes cluster on OpenStack (Magnum)**, and the deciding factor
is one technical driver: **ICMP ping**.

AV Tools does SNMP **and** ICMP ping. SNMP works on any Kubernetes; ping needs the
`NET_RAW` capability. On a managed PaaS the restricted security context strips that
capability, so ping fails unless an admin grants an exception. On **Magnum you are
cluster-admin**, so `NET_RAW` is a one-line `securityContext` addition and the
problem disappears.

Magnum is **semi-managed**: you provision and manage the cluster lifecycle with
`openstack coe cluster …`, so it is not "run a control plane by hand" — but you do
own upgrades, node health, and the quota footprint (control plane + workers consume
your OpenStack project's cores/instances).

| | Magnum (self-managed k8s on OpenStack) |
|---|---|
| Control plane | Magnum provisions; **you own upgrades/health** |
| Isolation | Your own cluster |
| ICMP ping (`NET_RAW`) | Cluster-admin: add `NET_RAW` freely |
| Quota | Control plane + workers consume your OpenStack quota |
| Image build | Build in CI → push to `registry.cern.ch` |

Everything else (the `CronJob`, `ConfigMap`, `Secret`, and the tbag sync) is plain
Kubernetes. See [Deployment](deployment/magnum.md).
