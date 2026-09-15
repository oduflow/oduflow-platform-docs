# Client data backup and restore verification

Vultr server backups do not cover the attached client block volume. Oduflow
therefore backs up the verified `/srv/oduflow/data` mount to a dedicated
Cloudflare R2 bucket with Restic. This path contains Docker/containerd data and
the Oduflow team and database data.

## Responsibility split

The control-plane Odoo owns external identity and lifecycle:

- it freezes backup jurisdiction and retention in the client snapshot;
- creates `oduflow-<instance UUID>` and reconciles it before any retry;
- applies R2 bucket locks to Restic `data/` and `snapshots/` objects;
- creates one account-owned API token scoped to that exact bucket;
- encrypts the derived S3 credentials and Restic repository password;
- journals the bucket and token identifiers without recording secret values;
- releases production only after the Salt production job, including its first
  backup and restore check, succeeds.

Salt remains the configuration authority on the client VM. It installs Restic,
writes the root-only environment file, installs the systemd service and daily
timer, stops the data-owning services for a cold backup, restarts the services,
and restores a client-bound probe from the newly created snapshot. It also runs
`restic check`. A successful VM snapshot, R2 upload, or mocked test alone is not
accepted as restore evidence.

The restore probe proves that the encrypted repository can be read with the
client's bucket-scoped credentials and that a selected file is byte-identical.
It is not a full disaster-recovery rehearsal that boots a replacement VM from a
complete restored volume.

## Control-plane credentials

Set the provisioner token only on the control-plane runtime:

```text
ODUFLOW_CLOUDFLARE_R2_TOKEN=<account-owned provisioner token>
```

Set the 32-character Cloudflare account identifier in **Oduflow → Configuration
→ Settings → Cloudflare R2 Backup**. It is not a credential and is stored as
`oduflow.cloudflare_account_id`.

The provisioner token needs these account permissions, scoped to the one account
used for client backups:

- **Workers R2 Storage: Edit**
- **Account API Tokens: Edit**

Do not distribute that token to clients. Each client receives only the S3 key
derived from its individually scoped account token. Cloudflare returns an
account-token value only at creation time; its token ID is the S3 access key and
the SHA-256 digest of the value is the S3 secret access key.

References:

- [R2 API token authentication](https://developers.cloudflare.com/r2/api/tokens/)
- [Create an account-owned API token](https://developers.cloudflare.com/api/resources/accounts/subresources/tokens/methods/create/)
- [R2 bucket locks](https://developers.cloudflare.com/r2/buckets/bucket-locks/)

## Runtime behavior

The initial backup runs after production publication. The helper verifies the
storage ownership receipt and actual mount before touching data. It initializes
only the assigned repository, stops active Oduflow, Paseo and Docker services,
backs up the entire mount, and restarts exactly the services that were active.
Service restart is attempted even if Restic fails.

Only after the first restore and repository check succeed does Salt enable
`oduflow-backup.timer`. The timer runs daily at 02:00 UTC with up to one hour
of randomized delay. Restic keeps snapshots within the plan retention window;
R2 bucket locks prevent early deletion of protected data and snapshot objects.

Inspect status on a client without printing the secret environment file:

```sh
systemctl status oduflow-backup.timer
systemctl status oduflow-backup.service
journalctl -u oduflow-backup.service --since today
```

Existing immutable client snapshots created before the Cloudflare backup addon
was installed remain backup-disabled until a manager explicitly selects
**Enable / Retry Data Backup**. That action records a separate immutable policy
and queues the same guarded backup workflow; it does not rewrite the historical
provisioning snapshot or recreate production.

During full client client deletion, the control plane verifies and revokes the
exact bucket-scoped client token before erasing its encrypted secrets. Locked
backup objects and their bucket remain for the configured retention window;
client deletion does not claim that retained recovery data was deleted.
