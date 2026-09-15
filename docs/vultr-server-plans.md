# Vultr server plans

In Odoo, open **Settings → Oduflow → Fetch Server Plans** to refresh the Vultr
catalogue. Platform managers can also refresh it from the **Server Plans** list.
The API credential stays on the control plane in `ODUFLOW_VULTR_API_KEY`.

The catalogue contains Shared CPU and Dedicated CPU plans, including vCPU,
memory in MB, local disk in GB, bandwidth in GB, hourly and monthly USD prices,
storage type and regional pricing. GPU and bare-metal products are excluded.
Prices shown in the catalogue are base prices; regional overrides are retained.
Only plans with local storage can currently be selected for client provisioning.

For VX1 plans with `storage_type=block_storage`, the API currently returns
`disk: 1` without a unit field. This is not an included 1 TB volume: these plans
have no local disk, and the bootable block volume is provisioned separately with
its own size. The catalogue displays **Separate block volume**, records zero
included local GB and preserves the original `disk` value in `api_disk` for
inspection. Plans with local NVMe retain their reported GB capacity (for example,
`vx1-g-4c-16g-240s` has 240 GB). Existing client snapshots are not rewritten.
See [Vultr's VX1 boot disk instructions](https://docs.vultr.com/products/compute/instances/vx1-cloud-compute/provisioning).

Refresh updates existing catalogue rows by Vultr plan ID. Plans removed from a
complete API response are archived, preserving historical references. An API
failure or incomplete response must not partially replace the catalogue.

When a client is prepared, the selected plan's ID, characteristics, regional
prices, catalogue timestamp and backup choice are copied into its immutable
provisioning snapshot. Later catalogue or platform-plan changes do not rewrite
that client's recorded configuration. Existing snapshots without these details
remain historical records; current catalogue values are not backfilled as if
they were the original prices.

**Server Level Backup** defaults to enabled and can be changed before preparing
the client. The Vultr create-instance request explicitly sends `backups=enabled`
or `backups=disabled` from the frozen choice. The catalogue prices exclude
backups; Vultr currently adds 20% of the VM price for automatic backups.

Vultr automatic backups cover the VM filesystem, **not attached block storage**.
Application databases and files on the separate client data volume therefore
use the dedicated R2 policy described in [Client data backup and restore
verification](client-backup.md).

References:

- [Vultr API](https://www.vultr.com/api/)
- [Official create-instance request fields](https://github.com/vultr/govultr/blob/master/instance.go)
- [Automatic backup coverage and pricing](https://docs.vultr.com/vps-automatic-backups)
