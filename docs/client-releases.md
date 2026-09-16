# Client releases

Client VM software lives under `client/` in the platform repository. There is no
separate client repository dependency; `client/` is an ordinary directory. Run
client checks from that directory; platform and client changes ship in one commit.

## Selecting the version for a new client

A plan defaults to main. Preparation resolves the platform main branch to a full
commit SHA, or uses the explicit SHA selected on the plan/instance. That SHA is
frozen in the provisioning snapshot and each configuration operation. The bundled
client-release manifest references a platform commit containing its client tree.

## Delivery through Master

Control requests preparation through Salt API's restricted runner. Master reads
the selected private platform archive in temporary storage and exports only its
client subtree. The immutable client archive is addressed by commit and checked
by SHA256. Platform addons and credentials are not published through file_roots.
Cloud-init contains only enrollment scripts and private enrollment inputs; after
joining Salt, the client receives its release over the authenticated fileserver.

Minions extract into `/opt/oduflow/client/releases/<SHA>` and reject modified local
releases. Salt applies those local states with fresh in-memory pillar. IDE runtime
archives are fetched from Master before switching to local state rendering.

## IDE releases

See [IDE installation](ide-installation.md). Build and source repository access
belongs to infrastructure. Clients retain separate credentials for their own
private project repositories.

## Publish a client change

1. Change and test `client/` together with any affected platform code.
2. Commit and push the platform changes. Select that full platform SHA for the
   client release; there is no separate client commit or gitlink to advance.
3. Deliver the control/master changes required by that release before applying
   it to a client. Keep the bundled release manifest consistent with the selected
   platform commit and its `client/release.json` compatibility metadata.
4. Apply and verify one client before selecting the SHA for additional clients.

## Update an existing client

An Oduflow administrator selects **Desired Client Revision** and clicks
**Apply Updated Configuration**. The existing queue, per-target execution lease
and durable dispatch protocol apply. Selection is blocked while an operation
is running or uncertain. A request binds its SHA before publication; a missing
response is reconciled against that exact receipt, never retried as a new apply.

Release-aware clients keep verified extracted releases under
`/opt/oduflow/client/releases/<SHA>`. Every managed client role uses local states
from the selected release. Fresh authenticated pillar stays in memory. Modified
release trees are rejected rather than overwritten. Application health checks are
part of successful configuration; only a reconciled success records
**Applied Client Revision**, **Verified Client Revision** and its timestamp.
These fields certify application configuration, not public production readiness.

Master delivers the client archive over authenticated Salt transport. Platform
Git credentials never enter client pillar, cloud-init or images.
`salt-call oduflow_job.version` returns reduced local metadata without secrets.

Version 1 receipts remain readable. Version 2 binds the selected platform SHA;
version 3 binds an addressed infrastructure bundle digest. The master refuses to dispatch these
jobs to minions that do not advertise the matching protocol.

Changing a SHA is not a transactional OS/database rollback. A failed apply can
leave partial changes; the last verified revision is historical evidence, not a
claim that every current file still matches it. Ubuntu package repositories are
not snapshotted by a source commit. Golden images remain optional accelerators.

## Checks

Run client checks from `client/`, platform checks from the repository root, and
relevant Odoo tests through the target Oduflow MCP. Record selected, applied and
verified revisions separately. See [repository access](github-download-access.md)
for customer project keys and [IDE installation](ide-installation.md) for artifacts.
