# Operations Runbook

`oc` on OpenShift, `kubectl` on Magnum — commands are otherwise identical.

## Trigger a sweep now

Don't wait 5 minutes — fire a manual Job from the CronJob:

```bash
oc create job --from=cronjob/avtools-snmp avtools-snmp-manual
oc get pods -w                        # expect 8 pods, indices 0..7
oc logs -l app=avtools --tail=50      # snmp_shard_applied + cycle_summary
```

Each pod's log shows its own `shard_index` and a `devices_in_shard` roughly equal to
`fleet / 8`.

## Scale the number of shards

Change **both** values and re-apply — they must match, or some devices are never
polled:

- `completions` (and `parallelism`) in `cronjob.yaml`
- the `SHARD_TOTAL` env in `cronjob.yaml`

Compute N from `ceil(device_count / target_devices_per_shard)`.

```bash
oc apply -f deploy/openshift/cronjob.yaml
```

## Rotate a secret

Update the value in tbag, then re-sync — no manifest change:

```bash
# update in tbag (hostgroup itdcim/avtools), then:
./deploy/openshift/sync-secret.sh
```

## Known gotchas

!!! warning "ICMP ping needs `CAP_NET_RAW`"
    Under OpenShift's restricted SCC, **SNMP works but ping may fail**. Check
    ping-status metrics after the first run. Fixes: unprivileged ICMP via
    `net.ipv4.ping_group_range`, an SCC granting `NET_RAW` (PaaS admin approval), or
    move to [Magnum](deployment/magnum.md) where you add `NET_RAW` directly.

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
