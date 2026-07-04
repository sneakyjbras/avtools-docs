# Logging & Monitoring

The container world inverts the VM model: the pod **emits**, the platform
**collects**. Don't configure per-pod log shippers the way Puppet configured Fluent
Bit on the VM.

## Logs → OpenSearch

- **Containers** write structured logs to **stdout/stderr**. That's it — the pod
  doesn't know about OpenSearch.
- A cluster collector picks them up:
    - **OpenShift** — very likely a built-in cluster-logging path to central
      OpenSearch/MONIT; confirm what your instance offers before deploying your own.
    - **Magnum** — run **Fluent Bit as a DaemonSet** (one per node), which scrapes
      every pod's stdout, enriches with Kubernetes metadata, and ships to OpenSearch.
- **Monolith** (during overlap) keeps its Puppet `fluentbit::pipeline` → OpenSearch,
  untouched until decommission.

!!! tip "Tag the streams during migration"
    Both the monolith and the containers land in OpenSearch during the overlap. Add a
    label/index that tells them apart (`source=monolith` vs `source=k8s`) so you can
    compare cycles and cleanly drop the monolith stream at cutover.

## Metrics → MONIT (push-based OTLP)

AV Tools **pushes** metrics via OTLP to MONIT — it is *not* scraped. So:

- No `ServiceMonitor` / Prometheus scrape config is needed for the sweep.
- Config comes from `configmap.yaml`: `MONIT_TENANT`, `MONIT_OTLP_ENDPOINT`,
  `MONIT_OTLP_INSECURE`, `OTEL_SERVICE_NAME`.
- The MONIT tenant password is the `MONIT_PASSWORD` secret — see [Secrets](secrets.md).

After a run, per-shard series appear in Grafana, e.g.
`avtools_snmp_devices_targeted{shard="k"}` (one series per shard), and the
`avtools-k8s-slo` alerts activate.
