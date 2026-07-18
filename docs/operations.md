# Operations Runbook

All commands use `kubectl` against the Magnum cluster (`openstack coe cluster config
avtools-k8s`).

## Trigger a sweep now

Don't wait 5 minutes — fire a manual Job from the CronJob:

```bash
kubectl create job --from=cronjob/avtools-snmp avtools-snmp-manual
kubectl get pods -w                        # expect 8 pods, indices 0..7
kubectl logs -l app=avtools --tail=50      # snmp_shard_applied + cycle_summary
```

Each pod's log shows its own `shard_index` and a `devices_in_shard` roughly equal to
`fleet / 8`.

## Scale the number of shards

Change **both** values and re-apply — they must match, or some devices are never
polled:

- `completions` (and `parallelism`) via the chart values (`jobs.<name>.shards`)
- the `SHARD_TOTAL` env via the chart values (`jobs.<name>.shards`)

Compute N from `ceil(device_count / target_devices_per_shard)`.

```bash
helm template avtools chart -f chart/values.yaml -f chart/values-qa.yaml | kubectl apply -n avtools-qa -f -
```

!!! warning "Only scale `snmp-timeseries`"
    `run-eam` and `run-landb` are single-node reconciliation syncs — their
    `shards` must stay `1`. Raising it doesn't parallelize anything; it runs N
    redundant full syncs that fight over the same Postgres rows (write
    contention, wasted DB load, lost updates). See
    [Architecture → Not everything shards](architecture.md#not-everything-shards).

## Rotate a secret

Update the value in tbag, then re-sync — no manifest change:

```bash
# update in tbag (hostgroup itdcim/avtools), then:
./scripts/sync-secret.sh
```

## Known gotchas

!!! warning "ICMP ping needs `CAP_NET_RAW`"
    SNMP works without extra privileges, but the ICMP ping probes need `NET_RAW`. On
    Magnum you are cluster-admin, so add it to the CronJob container's
    `securityContext` (`capabilities.add: ["NET_RAW"]`). Check ping-status metrics
    after the first run to confirm it took.

!!! warning "completions must equal SHARD_TOTAL"
    A mismatch means some `crc32` buckets are never polled. The app refuses to start
    on an out-of-range shard, but a *too-small* `completions` silently drops devices.

!!! note "N-1 capacity on small clusters"
    With few nodes, losing one is a large fraction of capacity. Make sure a cycle
    still completes with one node down — shards just reschedule, but the remaining
    nodes must hold them.

## Health signals

- Per-shard metrics in Grafana (`avtools_snmp_devices_targeted{shard="k"}`).
- `avtools-k8s-slo` alerts (liveness, freshness, availability under 5 min).
- Logs in OpenSearch, filtered by `source=k8s` during the migration overlap.
