# Oduflow namespace and control repository

Oduflow uses `oduflow.sh` for client hosting. Configure the delivery repository
and branch for the installation.

Core version `19.0.6.0.1` discovers the previous model namespace from installed
XML ownership, then uses the pinned official Odoo upgrade utilities to rename
models, tables, many-to-many relations, fields and indirect references. It
retains record IDs, foreign keys, ACLs, XML ownership and encrypted values.
The queue channel and serialized recordsets migrate with the models. All
installed provider and integration modules must be upgraded in the same run.

Before deployment, take a database/filestore backup, drain running queue jobs
and supply the unchanged encryption key under `ODUFLOW_ENCRYPTION_KEY`.
Rename other deployment environment keys to the `ODUFLOW_` prefix while
preserving their values. Apply through Megaflow, including module upgrades.
Verify record IDs/counts, secret decryption, the effective queue configuration,
and a harmless real queued operation.

Immutable provisioning snapshots, provider IDs and dispatch receipts are audit
records, not branding configuration. Do not rewrite them with a text replace:
external address changes require reconciliation and fresh verification of the
same owned resources. New and unprepared client instances use `oduflow.sh`.
Existing domain cutovers require DNS access, certificates and coordinated VPN
endpoint configuration.

## Control master's repository

The master runs a root-owned Git checkout at `/srv/oduflow`. Credentials,
local pillar and durable operation caches stay outside Git, under `/etc/oduflow`,
`/etc/salt` and `/var/cache/salt`. A private repository uses a read-only deploy
key owned by root; do not store a personal GitHub token in the checkout.

For the first installation, clone the repository into `/srv/oduflow`, check out
the delivery branch, and run `scripts/bootstrap-control.sh`. For an existing
master, preserve its identities, secrets and operation caches before moving
paths and applying the new control configuration.

Subsequent updates use:

```sh
sudo /srv/oduflow/scripts/update-control.sh --revision COMMIT_SHA
sudo /srv/oduflow/scripts/update-control.sh --revision COMMIT_SHA --apply control_config,control_master
```

The updater refuses dirty checkouts and non-fast-forward history. It fetches
only the selected branch and verifies the requested commit belongs to it.
Updating sources does not apply states until `--apply` is supplied. Salt remains
the configuration authority. Keep local configuration and pillar out of the
checkout so updates never overwrite deployment secrets.
