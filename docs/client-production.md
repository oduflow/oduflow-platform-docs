# Client production bootstrap

`roles.client_production` creates and verifies one Odoo 19 production through
**Client Oduflow**. Production is named `main`, uses the confirmed project branch
and an empty template name. The repository URL contains no credential.
The Oduflow package is pinned by the selected [client release](client-releases.md),
separately from the control-plane Odoo.

## Automatic control-plane workflow

After configuration succeeds, Odoo queues `production` through the restricted
`oduflow.production_start/status` endpoints. **Start / Retry Production** also
supports previously configured instances. It reconciles the production operation
without reallocating infrastructure.

1. Verify provider VM/volume identity and attachment, the client UUID, application
   configuration and authenticated local health.
2. Close public HTTP/HTTPS ingress on the host before Docker DNAT for IPv4 and
   IPv6. Persist the publication guard before calling the creation API.
3. Record creation intent, create the missing production and set its generated
   administrator password through an Odoo ORM shell over Docker stdin.
4. Verify the immutable production container ID, metadata and Docker labels.
   Only a matching, hardened production authorizes reopening ingress.
5. Remove only the guard's own rules, retry ACME as needed, then verify trusted
   HTTPS, real Odoo login/logout and both panels' authentication.
6. If the frozen policy enables data backup, run its first cold backup and scoped
   restore check. See [backup coverage and limits](client-backup.md).

The managed role changes host ingress; it does not edit provider firewall groups.
A Docker prerequisite restores an unfinished host guard after reboot before
containers start. An interrupted bootstrap keeps ingress closed. A failed or
ambiguous creation request does not authorize another creation POST.

Configuration and production have separate durable receipts. Polling is bounded
by the [Salt execution protocol](salt-async-apply.md). A completed production
operation is not repeated after an ordinary configuration update. A real AI coding
session remains a separate check.

## Required inputs and Git access

The application pillar includes the recorded production/panel hostnames,
`network.public_ipv4`, `oduflow.ui_password`, a distinct
`oduflow.production_admin_password`, and the confirmed Git repository/branch.
New clients use their verified SSH deploy key for their own project. The control
GitHub management token is never a fallback client credential.

The helper checks team ID, hostname, bind address, UI credential and mounted data
against `/etc/oduflow/oduflow.toml`. The managed team is `1`, with its private API
on `172.17.0.1:8000`. Configuration and credential files under `/etc/oduflow` are
root-owned mode `0600`, with secret diffs suppressed.

The client also supports the historical explicit HTTPS Git credential bundle.
In that mode it supplies the token through Git's stdin helper into the team's
private credential store. Tokens do not enter repository URLs, command arguments,
Docker labels or diagnostic output. Current credential migration and removal
are described in [repository access](github-download-access.md); an old token
example is not the setup path for a new client.

## Receipts and recovery

`/var/lib/oduflow/production/receipt.json` is root-only and fsynced before the
creation POST. A subsequent run may adopt an existing production only when the
receipt and name, domain, clean repository URL, branch and image match.
An unrelated existing production is never modified.

If the request outcome is unknown and no matching production is visible, stop
for reconciliation. Do not delete the receipt to retry: the original request
could still be executing. A previously hardened production is checked without
resetting its password, recreating resources or restarting its containers.
Password rotation is a [separate operation](client-access.md#reset-workflow).

A hardened receipt establishes the password transaction and container binding;
it does not prove that ACME or public login succeeded. Diagnose certificate,
DNS and authentication failures before marking production verified.

## Operator-only component debugging

The low-level `client_production` state is a creation/hardening component. It is
**not** the complete `roles.client_production` controller workflow. It requires
an externally verified guard at `/etc/oduflow/production-publication-guard.json`
and does not itself reopen ingress or perform the complete publication sequence.

For a reviewed manual debugging procedure, the platform helper
`scripts/guard-client-production.py` can preserve/close provider firewall rules;
`client/scripts/client-publication-guard.py` implements the host guard. Verify
fresh external connections are actually denied before creating production.
Restore rules only against the exact hardened production identity. Keep the
original operation receipts and account for reboot behavior; an old provider-only
firewall checklist is not a substitute for the managed host guard.

After publication, run `/usr/local/libexec/oduflow-verify-production` as root on
the expected client when an independent check is needed. It verifies immutable
container identity, trusted HTTPS, Odoo login/logout and panel authentication,
returning statuses rather than credentials. Record the target, revision and actual
result in the deployment report.
