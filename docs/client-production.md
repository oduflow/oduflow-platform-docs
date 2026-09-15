# Client production bootstrap

`client_production` creates one fresh Odoo 19 production through the **client's**
Oduflow 1.75 REST API. The production is named `main`, uses the verified GitHub
branch, and sets `template_name` to an empty string. Its repository URL never
contains a credential. This is separate from the control-plane Odoo in Megaflow.

## Automatic control-plane queue

After `configure` succeeds, Odoo queues `production` through the fixed
`oduflow.production_start/status` endpoints. The worker runs only
`roles.client_production`. The manager's **Start / Retry Production** button
also supports previously configured instances. It creates only a missing
journal operation; existing infrastructure is not reallocated.

The provider verifies VM/volume ownership and attachment and supplies canonical
`network.public_ipv4` in committed pillar. Git credentials must already exist in
the encrypted bundle or come from explicit `ODUFLOW_GITHUB_CLIENT_TOKEN`;
the management token is never used as an implicit fallback. The demo uses the
user-provided temporary development credential in that explicit setting.

The role prepares configuration, checks authenticated application health, closes
only the host's public HTTP/HTTPS ingress, creates and hardens production, verifies
its immutable container ID, restores its own host rules, retries ACME through
Traefik when needed, verifies real HTTPS login/logout, and, when the immutable
snapshot opts in, takes the first cold R2 backup and verifies a scoped restore.
It does not edit
provider firewall groups. The standalone provider guard below remains useful
for operator-driven debugging.

An interrupted bootstrap keeps ingress closed. A Docker prerequisite service
restores an unfinished host guard after reboot before containers start. Failed
creation or an unknown POST outcome never opens ingress or authorizes another
POST. Previously hardened, published production is verified without resetting
its password, closing ingress, recreating resources or restarting containers.

Odoo retains separate durable dispatch receipts for configuration and production;
250 polls at 30-second intervals cover the worker's two-hour execution window.
A completed production operation is not repeated after configuration updates.
Actual coding/LLM workflow verification remains a separate final step.

## Inputs

The existing client application pillar must include `dns.production`,
`dns.oduflow`, `oduflow.ui_password`, and:

```yaml
oduflow:
  production_admin_password: <generated encrypted instance secret>
  git:
    host: github.com
    username: <GitHub credential username>
    token: <authorized GitHub token>
    repo: oduflow/oduflow-demo
    branch: <actual repository branch recorded by the control plane>
```

Salt writes configuration and the three credential files under `/etc/oduflow`
as root-owned mode `0600`, with changes hidden. The helper checks the single
configured team against `/etc/oduflow/oduflow.toml`: team ID, hostname, API bind
address/port, UI credential, and mounted data directory must agree. The current
client configuration uses team `1` and `172.17.0.1:8000`.

Before creation, the helper stores the Git credential in
`/srv/oduflow/data/team_1/.git-credentials` using `git credential approve` with
stdin. The token is absent from process arguments, environment, repository URL,
Docker labels and diagnostic output. It verifies read access to the configured
branch with `git ls-remote`; errors are sanitized. Git's store is plaintext on
disk, root-owned mode `0600`, in the client team's data directory. A temporary
token must be replaced with the intended long-lived credential before future
fetches or deployment updates can work after revocation.

## First deployment order

1. Keep the client's provider firewall closed to public inbound TCP `80` and
   `443`, for both IPv4 and IPv6. SSH and VPN remain independently permitted.
   Oduflow starts a routable production container before its API call returns;
   therefore this external publication guard is required throughout bootstrap.
   Use `scripts/guard-client-production.py` to preserve and close the exact
   provider rules. A successful API response is insufficient: verify fresh
   external connections actually fail. In the live demo, provider rules were
   removed while the public endpoints still responded. The additional
   `client/scripts/client-publication-guard.py close --instance-uuid <UUID> --public-ip <IP>`
   blocks only inbound TCP 80/443 on the identified public interface, for IPv4
   and IPv6, before Docker DNAT. Its root-only receipt records the current boot.
   These temporary host rules do not survive a reboot; keep the same boot
   throughout initial creation and hardening.
2. On that client, write `/etc/oduflow/production-publication-guard.json`, owned
   by root and mode `0600`, after checking the actual firewall rules:

   ```json
   {
     "instance_uuid": "<exact client UUID>",
     "domain": "oduflow-demo.oduflow.sh",
     "firewall_group_id": "<attached and verified provider firewall ID>",
     "public_ingress_closed": true
   }
   ```

   The guard is the controller's attestation. The helper does not call Vultr or
   independently inspect the provider firewall, and never opens public ports.
3. Refresh the minion's pillar and apply `client_production` through the master:

   ```sh
   salt 'client-<UUID>' saltutil.refresh_pillar
   salt 'client-<UUID>' state.apply client_production
   ```

   The state starts the configured client applications if needed. Its creation
   command only runs when the publication guard exists. Read its stateful result:
   publication is allowed only after it reports `status: hardened`.
4. The helper writes the Odoo administrator login (`admin` by default) and strong
   password through an Odoo ORM shell over Docker stdin, then explicitly commits.
   No admin or PostgreSQL password is placed in argv. It validates the production
   metadata and Docker labels and targets the inspected immutable container ID.
5. Open public TCP `80`/`443` only after hardening, remove the guard, and verify
   HTTPS plus Odoo login. HTTP-01 issuance cannot complete while the ports are
   closed; allow Traefik to retry after publication. A hardened receipt confirms
   the password transaction, not completion of certificate issuance or the
   application's public health check.
   Restore the provider's saved rules using the hardened container attestation,
   then run `client-publication-guard.py restore` with the same identity to remove
   only its own host rules. Run `/usr/local/libexec/oduflow-verify-production` as root on
   the client (or the repository wrapper `client/scripts/verify-client-production.py`): it checks the immutable container identity, performs a real Odoo
   19 HTTPS login/logout, and checks both panels reject anonymous API requests
   and accept their configured credentials. Its output contains only statuses.

## Recovery and repeatability

The root-only receipt at `/var/lib/oduflow/production/receipt.json` is fsynced
before the creation POST. An interrupted or failed POST is never blindly repeated.
A subsequent run may adopt a matching production only when that receipt exists;
name, domain, clean repository URL, branch and image must match. An existing
production without a receipt is not modified.

If the outcome is unknown and no production exists, the helper stops for explicit
reconciliation. Do not delete the receipt as an automatic retry mechanism: the
original API request could still be executing. A completed receipt makes reruns
read-only and does not reset a password changed subsequently by the administrator.
The Salt state deliberately skips its command after the publication guard is
removed. Credential rotation is a separate operation.
