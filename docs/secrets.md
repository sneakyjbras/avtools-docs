# Secrets & Configuration

## Principle: tbag is the single writer

The monolith reads its secrets via Puppet/Teigi today. Rather than fork those values
into git or a second store, **tbag/Teigi (hostgroup `itdcim/avtools`) stays the
single source of truth**, and we *sync* the live values into a native Kubernetes
`Secret`. Rotate a key = update it in tbag, re-run the sync. No drift.

!!! warning "Don't fork shared keys"
    Putting the same credential in both tbag and a hand-edited K8s Secret means the
    day you rotate it, one copy gets missed. One writer (tbag), everywhere else syncs.

## What this workload needs

The SNMP sweep consumes exactly **two** secrets — nothing more:

| Key | Source (tbag) |
|---|---|
| `DATABASE_URL` | `tbag show database_url_qa --hg itdcim/avtools --plain` (DBoD URL) |
| `MONIT_PASSWORD` | `tbag show monit_pwd --hg itdcim/avtools --plain` (MONIT tenant) |

!!! note "Not every avtools key belongs here"
    The wider avtools project also uses the EAM service account and LanDB OAuth — but
    the **SNMP sweep does not**. Keep them out of this Secret (least privilege). A
    future component that needs them gets its own Secret.

## Non-secret config

Lives in the chart ConfigMap (`avtools-config`) in `av-tools-infra`: `MONIT_TENANT`, `MONIT_OTLP_ENDPOINT`,
`AVTOOLS_ENVIRONMENT` (start on `qa`), `AVTOOLS_HOSTGROUP=itdcim/avtools`, labels.

## Sync the secret

In the `av-tools-infra` repo, `scripts/sync-secret.sh` reads both values from tbag and upserts the
`avtools-secrets` Secret idempotently (values never hit stdout):

```bash
./scripts/sync-secret.sh                     # qa, hostgroup itdcim/avtools
AVTOOLS_ENVIRONMENT=prod ./scripts/sync-secret.sh
```

Manual fallback (no tbag access from where you deploy): copy `secret.example.yaml`
to `secret.yaml`, fill it, `apply`. `secret.yaml` is gitignored so a filled copy
can't be committed.

## At cutover

When the monolith is retired, promote the source of truth off tbag (a Puppet/VM
construct) to a K8s-native home — **Vault + External Secrets** if your platform
offers it, otherwise sealed-secrets/SOPS in git. The container's consumption path
(env/file from a `Secret`) doesn't change, so decide the endpoint now and the cutover
touches nothing in the pod.
