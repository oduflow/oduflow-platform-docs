# Infrastructure master automation

`oduflow_master` manages a logical infrastructure master independently of client
instances and cloud providers. It installs onto an explicitly selected SSH host;
it does not purchase a VM or replace the currently configured master implicitly.
Only Oduflow Admin can change nodes, submit operations or enter credentials.
Observers can read sanitized node, operation and backup records. Global company
rules also apply to searches, counts and individual records.

## Configure a node

Open **Oduflow → Infrastructure Masters** and create a node with its SSH address,
port, login and independently verified OpenSSH `SHA256:...` host-key fingerprint.
Unknown or mismatched host keys are not accepted. Select password, private key,
or certificate authentication. An SSH certificate is a separate input and still
requires its matching private key. A sudo password is optional for accounts that
cannot use noninteractive sudo.

Enter the public master hostname, distinct master/control VPN addresses, HTTPS
control gateway origin and certificate contact email. The deployment revision
defaults to the control checkout's current Git commit and is visible under
Deployment Details. The node UUID identifies the logical master throughout
backup and recovery; it is not an editable cloud-resource identifier.

Before deployment, create the public DNS records for the master hostname and
point them at the intended host. This workflow does not create or switch DNS
records. Public reachability and DNS must be ready for certificate issuance.
A later failover also requires an explicit DNS change at the appropriate point
after fencing the old master.

A new Headscale installation requires the control gateway to join its VPN.
Retrieve the private enrollment handoff with the standalone CLI's `handoff`
command and apply it through the control gateway's enrollment process. Do not
copy the handoff into operation comments or ordinary Odoo fields. Until this
connection exists, `control_network_pending` is an expected prerequisite failure.

Use **Store Credentials** to enter SSH and control credentials. Inputs are
removed before ordinary ORM storage; the transient wizard persists only an
encrypted payload and clears it after use. The permanent secret model has no
access grants, views or chatter. Fernet encryption uses the existing deployment
`ODUFLOW_ENCRYPTION_KEY` and binds ciphertext to its node or immutable operation.
The encryption key is never stored in Odoo.

For a new Deploy operation, omitted pillar/API credentials can be taken privately
from `ODUFLOW_PILLAR_TOKEN` and `ODUFLOW_SALT_API_PASSWORD`. They are frozen in
the encrypted operation bundle and are not displayed in the wizard. The gateway
CA certificate must be supplied explicitly. Changing node credentials does not
change credentials already frozen for an operation.

## Operations and ambiguous outcomes

- **Deploy** installs and checks the master stack on the selected host.
- **Check** checks the existing master and records a reduced verified result.
- **Backup** creates and verifies an encrypted repository snapshot. Services may
  pause briefly while a consistent local copy is captured.
- **Restore** opens a separate recovery dialog for a new empty target.

Every operation has a unique request UUID, immutable configuration, its digest,
connection snapshot and encrypted secret snapshot. Local bundle/credential
preflight runs before durable dispatch intent. A deterministic local failure is
reported as failed with a fixed error category; no remote request is sent.

Dispatch intent is committed independently of the queue transaction. After an
ambiguous send or worker death, the queue polls the same UUID. It never converts
an unknown result into a successful operation. A missing remote request may
resume its interrupted upload through the transport's exact-request protocol:
the remote executor checks immutable inputs under its lock and commits its claim
before spawning. An accepted executor is never spawned again, including after
its death. A missing local/remote response alone is not proof that execution did
not happen.

Operations block configuration changes and other operations while queued,
running or unresolved. **Reconcile** continues the existing request; it does not
create a new operation. Polling is bounded. Transport error strings and raw
command output are not placed in queue descriptions, operation messages or
chatter. The UI displays fixed error categories and reduced counts. A terminal
receipt is matched to node UUID, request UUID, action and configuration digest.

The master executor invokes fixed Salt phases in an isolated process pinned to
Salt 3006.27. It reuses the reviewed state cache/event guards: automatic `sls.p`
and compiled state-cache writes are discarded in memory, per-state events are
suppressed, and parallel state execution is refused before spawning. The local
Salt cache directory is private tmpfs storage under `/dev/shm`; normal worker
cleanup removes it. Raw Salt JSON is captured only for reduction, not written to
operation receipts or logs. Global job/pillar caching stays disabled.

## Backup and recovery

Set a dedicated Restic S3 repository URL and enter its repository password and
S3 credentials through Store Credentials. Enable **Initialize Empty Backup
Repository** only for a new repository; a successful backup clears that option.
Keep these recovery credentials in an independent private location. A backup
must remain usable if the Odoo database or its encryption key is unavailable.

A registered backup contains a verified snapshot ID, manifest digest, source
revision, creation time, file/byte counts and pinned service versions. Operators
cannot manufacture or edit a verified backup record through ordinary RPC.
Its source operation retains the configuration and encrypted credentials used
for that snapshot.

Restore requires all of the following:

1. A verified snapshot belonging to the logical master.
2. A different target SSH address and separately verified host key.
3. Fresh target SSH credentials; a current Odoo password recheck by the same
   administrator who opened the dialog.
4. An explicit statement that the old master is powered off or network-isolated
   and will remain fenced, plus confirmation that the target is empty.

Fencing is an **operator attestation**, not an automatic inspection of the old
host. The queued operation stores its method, user, timestamp and snapshot.
Remote checks enforce the empty-target and identity constraints. The selected
backup supplies the master identity, network configuration and source revision;
current editable node settings do not silently replace its hostname or VPN
identity. The new target SSH port is applied explicitly.

The node continues to show the original connection until the restore's terminal
receipt is verified. Only then does it switch to the new connection and restored
configuration. Health checks must complete as part of the remote restore;
verifying archive integrity alone does not mean the master is operational.

## Standalone recovery CLI

The same transport is available without a running Odoo database through
`ops.master_bootstrap.cli`. Run it from the reviewed source checkout with a
private, owner-only JSON input file:

```sh
python3 -m ops.master_bootstrap.cli --input /private/master.json --request REQUEST_UUID start --action install
python3 -m ops.master_bootstrap.cli --input /private/master.json --request REQUEST_UUID status
python3 -m ops.master_bootstrap.cli --input /private/master.json --request REQUEST_UUID resume --action install
python3 -m ops.master_bootstrap.cli --input /private/master.json --request REQUEST_UUID handoff --output /private/enrollment.json
```

The input has this shape; secret-file values are absolute paths to separate
private files, not their contents:

```json
{
  "connection": {
    "host": "192.0.2.10",
    "port": 22,
    "username": "root",
    "host_key_sha256": "SHA256:<verified fingerprint>",
    "private_key": "<private key material>"
  },
  "config": {
    "node_uuid": "<logical master UUID>",
    "hostname": "master.example.org",
    "master_vpn_ip": "100.64.0.1",
    "control_vpn_ip": "100.64.0.2",
    "pillar_url": "https://control.example.org",
    "acme_email": "admin@example.org",
    "ssh_port": 22,
    "source_revision": "<full reviewed Git revision>"
  },
  "secret_files": {
    "pillar-token": "/private/pillar-token",
    "api-password": "/private/api-password",
    "gateway-ca.crt": "/private/gateway-ca.crt",
    "master-backup.json": "/private/master-backup.json"
  }
}
```

The entire input file is private because SSH credentials can be inline. The CLI
also accepts the transport's password/certificate/private-key-passphrase and
sudo-password inputs. It does not read credentials from environment variables.
For restore, include the selected `restore.snapshot_id` and
`restore.manifest_digest` in configuration and the `source_fencing` attestation.
Stdout contains only a reduced receipt/error. Enrollment handoff material is
written only to the explicit private output file, never stdout.

Standalone means that the CLI can run and restore the master's identity files
without using an Odoo database to launch the operation. Final operational health
still depends on the control VPN, gateway trust and reachable Odoo pillar service.
If those services are unavailable, identity recovery and an operationally healthy
master are separate outcomes; the CLI does not bypass the final health checks.

## Verification limits

Odoo tests cover role/company enforcement, encrypted wizard storage, immutable
snapshots, preflight failures, lost responses, strict receipt adoption and
password-confirmed recovery. Transport and backup tests cover their own SSH,
filesystem, process, repository and identity contracts. These tests do not prove
that a new production host has been provisioned. Report source revision,
deployed modules and any real host/recovery checks separately.
