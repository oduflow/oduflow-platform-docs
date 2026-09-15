# Client software releases

Client installation sources live in the public repository
`https://github.com/oduflow/oduflow-client.git`. The `client` Git submodule pins
the dependency used by platform builds and tests. Initialize it with
`git submodule update --init client` in a source checkout without credentials.
Odoo runtime delivery does not require a populated submodule: cloud-init uses the
SHA in `addons/oduflow/data/client-release.json`, and master bundle creation can
fetch the pinned Git object over public HTTPS.

## Boundaries

The client repository owns client Salt states, bootstrap/minion, application
artifacts, Packer client recipes and client tests. The platform retains Odoo
addons, provider operations, master orchestration and infrastructure states.
The master fileserver publishes only `/srv/oduflow/client/salt/` roots. It does
not publish platform states, even under a separate named Salt environment.
Gateway operations receive a bounded, digest-bound source bundle addressed to
the expected enrolled minion through the authenticated Salt job transport.

Infrastructure images and master bundles consume shared installation helpers
from the pinned client repository. Local infrastructure states may use a merged
tree, which is separate from the master fileserver's published roots.

## Release and installation

1. Commit and test the client repository; use a full lowercase commit SHA.
2. Update the platform submodule and `addons/oduflow/data/client-release.json`
   together. The latter is the default for new deployment plans.
3. Client source and IDE release downloads need no GitHub credentials.
4. Deliver and upgrade `oduflow`. Rebuild the master when its executor or source
   transport changes; ordinary client state changes need only the pinned checkout.
   Container CI checks out the exact public client submodule SHA over HTTPS.
5. Test one client before selecting the new SHA on additional clients.

Preparing an instance freezes the initial client SHA in its provisioning
snapshot. Cloud-init receives that SHA and private enrollment
inputs; it checks out the client release and runs that release's `bootstrap.sh`.
Client secrets and machine identity remain separate from the checkout. Prepared
historical snapshots are not rewritten during migration.

## Update an existing client

An Oduflow administrator selects **Desired Client Revision** and clicks
**Apply Updated Configuration**. The existing queue, per-target execution lease
and durable dispatch protocol apply. Selection is blocked while an operation
is running or uncertain. A request binds its SHA before publication; a missing
response is reconciled against that exact receipt, never retried as a new apply.

Release-aware clients keep clean detached checkouts under
`/opt/oduflow/client/releases/<SHA>`. Every managed client role uses local states
from the selected release. Fresh authenticated pillar stays in memory. Dirty
checkouts are rejected rather than overwritten. Application health checks are
part of successful configuration; only a reconciled success records
**Applied Client Revision**, **Verified Client Revision** and its timestamp.
These fields certify application configuration, not public production readiness.

Client release checkout uses public HTTPS with no repository credentials in
pillar, cloud-init, or images. `salt-call oduflow_job.version` returns reduced
local metadata without secrets.

Version 1 receipts remain readable. Version 2 binds a client SHA; version 3 binds
an addressed infrastructure bundle digest. The master refuses to dispatch these
jobs to minions that do not advertise the matching protocol.

Changing a SHA is not a transactional OS/database rollback. A failed apply can
leave partial changes; the last verified revision is historical evidence, not a
claim that every current file still matches it. Ubuntu package repositories are
not snapshotted by a source commit. Golden images remain optional accelerators.

## Checks

Run platform Ruff and local tests, the client repository's own checks, and
relevant Odoo tests through the target Oduflow MCP. A source or mocked test result
does not certify live provisioning. Keep deployment reports explicit about
source revisions, image digests, module upgrades, queued operations and limits.

Client releases never include platform repository access. The old platform download
default is removed on upgrade. Existing installation snapshots remain immutable;
queued configuration reconciles and revokes recorded read-only platform deploy keys.
Legacy client HTTPS tokens are replaced by a verified deploy key for the client's own
repository before Salt receives the new pillar. Salt removes the managed token file
and GitHub entries from the team credential store while preserving other hosts.

Public `oduflow/paseo` and `oduflow/oduflow-client` repositories do not receive
customer deploy keys. Additional private repositories can still use scoped
read-only grants. The client's own project repository retains scoped write access.

## Selecting the version for a new client

The plan's **Client Version** defaults to `main`. During preparation the control
plane resolves `oduflow/oduflow-client`'s main branch through the configured
GitHub API credential and saves the full SHA in the instance and immutable
provisioning snapshot. The same field accepts a full lowercase commit SHA to
pin a plan. A nonempty **Desired Client Revision** on a draft instance takes
precedence over the plan. Preparation fails if main cannot be resolved.

Already prepared instances never follow moving branches automatically. Their
queued configuration and update operations continue to use an immutable SHA.
`addons/oduflow/data/client-release.json` and the client submodule remain the
platform's bundled compatibility reference and fallback for historical release
adoption; they no longer choose the version for a newly prepared unpinned client.
