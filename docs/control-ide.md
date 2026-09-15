# Oduflow dashboard and shared IDE

Oduflow opens **Dashboard**, with **Customers**, **Instances**, and **IDE** tiles.
Customers opens the contact directory in customer mode. Instances opens the existing
client instance action. IDE opens a shared development environment containing one
project per client instance.

## Access and projects

The shared IDE is an administrative environment spanning the control inventory.
Opening it requires both **Oduflow Admin** and **Settings / Administration**
(`base.group_system`). Observer users retain access to the dashboard, contacts and
instance views, but cannot enter the shared IDE.

The tile starts a signed, single-use login with a 30-second lifetime. The browser
posts the ticket to the IDE; tickets are never placed in URL query parameters.
The gateway sets a host-only, Secure, HttpOnly session cookie lasting one hour.
Launch nonces and the gateway signing identity survive container restarts. The
Odoo signing key is encrypted using the existing control-plane encryption key;
only its public verification key is supplied to the IDE container. Opening the
gateway directly without a session asks the user to enter through Odoo.

Every minute the IDE service requests the inventory using its own signed request.
Odoo trusts an explicitly registered service public key. The inventory supplies
only each instance's own repository deploy key, never platform-wide GitHub tokens.
Repositories are cloned at their confirmed branch, using pinned GitHub host keys.
Instances whose repository grant is not ready receive a project directory until
the repository can be imported. An existing nonempty directory is never replaced
by a clone. Removed instances' repository keys are removed from the service; their
working copies are retained for administrator review.

Each project lives at `/data/home/projects/<slug>`. Registration is idempotent;
existing origins and ownership receipts are checked. Synchronization never pulls,
resets or cleans a working copy, and preserves uncommitted changes. These are
central working copies of client repositories. Client VM files and running client
services are managed separately through the existing instance operations.

The IDE runtime is Oduflow IDE 0.8.0, built on the target server from
`oduflow/paseo` commit `3f3555c5a64e9bbffa139c1482c691757a2a5a25`. Agent provider
authentication is configured in the IDE separately; a repository deploy key does not authorize an AI provider.

## Deployment

Use the configured Oduflow MCP for the selected control environment:

1. Commit and push the code, then `pull_and_apply(upgrade="oduflow")` for the first
   installation. Later Python-only changes require restart, and new fields/data
   require upgrade.
2. Create `control-ide-config` and `control-ide-data` named volumes. Copy the
   committed `docker/control-ide/` and `salt/states/control_ide/` trees to the
   configuration volume, preserving their repository-relative paths.
3. In Odoo shell, call `odoo.addons.oduflow.ide.initialize_signing_key(env)` and
   set `oduflow.ide_url` to the HTTPS origin. The initializer returns only the
   public key. Supply its base64-encoded PEM as `ODUFLOW_CONTROL_PUBLIC_KEY`.
4. Create the `control-ide` service using the pinned Node image below, port 8080,
   command `sh /config/docker/control-ide/start.sh`, and mounts
   `control-ide-config:/config:ro,control-ide-data:/data`.
5. Supply `ODUFLOW_IDE_URL`, `ODUFLOW_CONTROL_PUBLIC_URL`,
   `ODUFLOW_CONTROL_URL` (the trusted internal Odoo HTTP endpoint), and
   `ODUFLOW_CONTROL_HOST` (its public virtual host). Do not expose Odoo's internal
   endpoint outside the team network.
6. Read `/data/home/service-public-key.pem` from the service and register it in
   Odoo as `oduflow.ide_service_public_key`. This file is public key material.
7. Wait for `/health` to report `ready: true`, then verify actual dashboard
   navigation and both project names in the IDE browser interface.

Pinned image:

```text
node@sha256:915acd9e9b885ead0c620e27e37c81b74c226e0e1c8177f37a60217b6eabb0d7
```

The container installs Salt 3006.27 with the resolved dependency versions in
`docker/control-ide/salt-requirements.txt`. The `control_ide` Salt state manages
runtime files, builds the pinned source and its workspace packages using the source
lockfile, and pins GitHub SSH host keys. Salt's job and pillar caches are disabled. The daemon listens only on loopback and has
relay connectivity disabled; all browser HTTP and WebSocket traffic passes
through the authenticated gateway. It runs as the unprivileged `node` user.

For updates, deliver the committed configuration tree and restart `control-ide`.
The bootstrap reapplies Salt and installs changed dependency locks. The source
build lives in `/data/releases/<version>-<commit>`; `/data/current` selects the
release only after the build succeeds. Its `build.json` records the full commit,
source lockfile checksum and Node version. Existing releases are retained.

To build ahead of a restart, run `python3 /config/docker/control-ide/build.py`
inside the service. This builds alongside the running release. Then deliver the
updated Salt/gateway files and restart the service to select the completed build.
Keep the data volume: it contains working copies, daemon state and the trusted service identity.
Recreating that volume requires registering a new service public key.

## Verification

- `ruff check addons salt scripts tests`
- `ruff format --check addons salt scripts tests`
- `node --test tests/test_control_ide.cjs`
- Odoo tests: `/oduflow:TestOduflowDashboard` (post-install).
- Browser: all three tiles, desktop/mobile layout, SSO and project navigation.
- Service: unsigned HTTP/WebSocket denial, signed inventory, project registration,
  idempotent Salt application, and preservation of uncommitted files after restart.
