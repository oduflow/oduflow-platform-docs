# Client application states

Client source paths and commands in this guide are relative to the
`oduflow-client` repository (the platform checkout mounts it at `client/`).
See [client releases](client-releases.md) for the managed checkout/update workflow.

These states install the published Oduflow 1.76.0 package and build Paseo 0.8.0
from the pinned `oduflow/paseo` commit on Ubuntu amd64.
`roles.client_stack` now includes the authenticated application bootstrap.
Production creation remains a separate step after the publication checks.

The combined command is `salt-call state.apply roles.client_stack`. For staged
installation and debugging, apply explicitly in order:

```sh
salt-call state.apply client_apps.install
salt-call state.apply client_apps.configure
salt-call state.apply client_apps.start
```

`install` needs no application credentials and starts no services. It installs
signed Ubuntu prerequisites and checksum-verified uv 0.12.13, Node 22.20.0 and
the Oduflow wheel, then builds Paseo from source. Artifact URLs, upstream
checksums and the Paseo repository, commit and workspace list are recorded in
`salt/states/client_apps/artifacts.json`. The same file is copied to
`/var/cache/oduflow-apps/artifacts.json` and is the installer's only source of
versions, so a pin is declared once.

The pinned IDE source archive is included in the client repository at
`salt/states/client_apps/artifacts/`. Clean installations read it from the selected
client checkout without a GitHub credential. New client plans also provide a
separate read-only deploy key for `oduflow/paseo`, so clients can read its source.
This access is independent of the checksum-pinned installation archive.
To prepare a replacement artifact from an authorized local Paseo checkout, run
`scripts/prepare-client-artifacts.py --repository /path/to/paseo --output /path/to/staging`
and review the manifest checksum before committing the artifact to the client
repository. The installer verifies its checksum, embedded commit, version and
extraction paths. The legacy `docker/master-artifacts/Dockerfile` remains an
optional platform packaging utility.

The Paseo build follows the upstream container recipe: fetch the pinned commit,
`npm ci` from the committed lockfile, `npm pack` each pinned workspace in
dependency order, then install those tarballs into
`/opt/oduflow/paseo/<version>+<commit>`. The checkout is refused unless the
resolved `HEAD` is the pinned commit and the source carries the pinned version;
the contributor Git-hook script is dropped before installing, as upstream's own
image build does. The build tree and its private npm cache are removed
afterwards, so a rebuild is a full rebuild rather than an incremental one.
A source build needs far more time, memory and scratch disk than a package
install did — the golden image performs it once for all clients.
The npm build uses a 4 GiB V8 old-space limit: Node's automatic limit on a
2 GiB client is insufficient for the server TypeScript build. Small clients
need sufficient swap during compilation. This setting applies only to build
commands and does not change the running Paseo service's memory settings.

Building runs the pinned source's own npm lifecycle scripts as root, as
installing the published tarball already did. Review a commit before pinning it;
the pinned commit and archive checksum define the trusted source.

Installation receipts are written only after installation succeeds; a missing
executable causes installation to run again. Because the fork keeps the upstream
`0.8.0` version string, the installed directory, the receipt and the systemd unit
are all keyed by version **and** commit. Python/npm transitive dependencies are
resolved from their registries at installation time: `package-lock.json` pins
Paseo's own dependency tree, but the tarballs' runtime dependency resolution and
the Oduflow wheel are not fully locked. Ubuntu package versions follow the
host's signed apt repositories.

`configure` installs/starts Docker with verified storage guards, writes app
configuration and systemd units, but does not start the application services.
`start` validates the configuration with the installed Oduflow package, then
starts Paseo, its private proxy, and Oduflow. Reapplying does not create a
production environment. A running service is not production readiness; the
separate production workflow must create and verify Odoo.

## Required pillar

The schema1 identity must match the minion ID `client-<UUID>`, and storage must
point at a stable `/dev/disk/by-id/...` device mounted at `/srv/oduflow/data`.
The existing storage state still requires explicit authorization before it can
format a blank disk. App configuration additionally needs:

- `dns.oduflow`, `dns.paseo`, and `ingress.mode=direct_tls`;
- `ingress.acme.email` for Let's Encrypt HTTP-01;
- generated `oduflow.auth_token`, `oduflow.ui_password`,
  `oduflow.database_password`, and `paseo.password`;
- existing Oduflow quotas/lifecycle values, if overriding the bounded defaults.

Paseo's generated password must be at least 16 URL-safe characters. Credential
files are 0600, Salt file diffs are suppressed, and passwords are not command
arguments. `oduflow.database_password` is a distinct, mandatory PostgreSQL credential
generated and encrypted by the control plane. It never falls back to the MCP
token or the upstream default `odoo`. New prepared instances also receive a
distinct `production_admin_password`, exposed as
`oduflow.production_admin_password` for the separate production workflow.
Treat database credentials as persistent: changing TOML does not rotate an
existing PostgreSQL role password. A supplied `oduflow.license_key` is installed
as `/etc/oduflow/license.key`; an omitted key does not delete an existing license.

No LLM key, provider CLI login, or license is needed merely to install/start
these authenticated web interfaces. GitHub CLI is installed on the host, including during credential-free image
installation. When the verified client GitHub bundle is present, `paseo.project`
writes private GitHub CLI credentials for the `paseo` user, configures Git's
GitHub credential helper, clones the confirmed repository branch into
`/srv/paseo/projects/<repository>`, and registers it with the running daemon.
Project registration must succeed before application health completes. Retries
verify the existing origin and preserve user work; they do not pull, reset, or
create duplicate projects. This state can also be applied to an existing client
without rebuilding Paseo. GitHub credentials and working copies stay on the
verified client volume and are never included in images.
Agent execution still requires the separate `client_agent` configuration.
The control plane omits absent optional integrations and refuses partially
provided credential bundles. Git credentials additionally require the confirmed
repository branch; no branch is guessed. Old prepared records must explicitly
receive the new database/production credentials before deployment.

Encrypted `storage_allow_format` maps to `storage.allow_format` and defaults to
false; only an actual boolean is accepted. Optional
`storage_filesystem_uuid` maps to `storage.filesystem_uuid` after canonical UUID
validation. The client storage helper also verifies volume ownership. Direct
TLS pillar declares `provider=letsencrypt`, `challenge=http-01`; it carries no
Cloudflare credential.

## Storage and private routing

Docker's data root is `/srv/oduflow/data/docker`; containerd's persistent root is
`/srv/oduflow/data/containerd`. Config and systemd mount guards are installed
before apt can start Docker. Existing data in `/var/lib/docker` or
`/var/lib/containerd`, or an incompatible existing daemon configuration, causes
preflight refusal. Automatic migration is intentionally absent.

Paseo is launched with `paseo daemon run --home /srv/paseo`. The source build
removed the former launch flags, so the listen address, relay state, trusted
proxies, hostnames and web UI come from the managed `/srv/paseo/config.json`, and
`PASEO_PASSWORD` from `/etc/paseo/credentials.env` still overrides the stored
password hash. Passing the removed flags is a hard startup error, not a warning.

Docker, containerd, Oduflow and Paseo all depend on the verified mount and stop
when its mount unit disappears. Docker live-restore is disabled. Paseo runs as
its own unprivileged user, with `/srv/paseo` pointing to its private directory on
the volume. Oduflow runs as root because it manages Docker and XFS quotas, as in
the published host installation contract.

A clean client reserves Docker's `172.17.0.1/16` bridge. Oduflow listens on
`172.17.0.1:8000`; Paseo listens on `127.0.0.1:6767`. A systemd socket proxy
listens only on `172.17.0.1:6768` and forwards to Paseo loopback. This is needed
because Oduflow's containerized Traefik cannot reach a host loopback listener.
The proxy does not bind public interfaces; do not expose/forward its private
ports. The host firewall must permit HTTP/HTTPS publicly and Docker-bridge
traffic to 8000/6768, while keeping administrative ports private. Firewall
provisioning and external probes belong to the surrounding client workflow.

Oduflow owns the only Traefik instance, with `routing.mode=traefik`, `tls=true`,
and `[production].enabled=true`. Its exact-host HTTP-01 certificates need public
DNS-only records and inbound port 80. No client Cloudflare API token is used.
The published package hardcodes the rolling `traefik:v3` image and defaults to
`postgres:15`; the states do not pretend those upstream image tags are digest
pinned. Pinning that part of the upstream runtime is follow-up work.

## Hostname contract

The listener uses `[server].bind`; the legacy `server.host` key is no longer
generated. Each team sets its explicit `hostname`, which also determines its
OAuth issuer. No `[oauth]`, `oauth_base_url` or `routing.hostname` override is
generated. This follows upstream commits `6546486` and `58db708`: service and
environment secret values may be expressed as `secret:<name>` references to the
operator-managed team vault rather than expanded into generated manifests.

Use `team.1.hostname=oduflow.<client>.<platform>` for the dashboard and the
explicit `[route.paseo]` for `paseo.<client>.<platform>`. The production workflow
must explicitly supply `<client>.<platform>` as the production domain.

Published Oduflow 1.75.0 accepts `environment_hostname_mode="branch"`, but its
implicit branch hostname appends the entire team hostname. To obtain
`feature.<client>.<platform>`, creation must pass an explicit short
`hostname="feature"`. There are no reserved `devN`/`svcN` records in these states.
Do not claim implicit branch naming already uses the client-wide namespace.

## Validation and sources

`PYTHONPATH=/tmp/oduflow-salt-minion-deps python3 -m unittest discover -s tests
-p test_client_apps.py` exercises fail-closed configuration, private secret
files, mount/package ordering, include/requisite resolution, installer receipts,
refusal of a source that does not match its pin, build-tree cleanup after a
failure, and refusal to migrate existing container data. Salt 3006.25 compiled
the state files offline and rendered the templated unit, and the actual
checksum-verified Oduflow 1.75.0 wheel parsed/validated the rendered TOML.

The updated TOML was also parsed and validated with the checksum-verified
Oduflow 1.76.0 wheel, confirming `bind_host` and automatic OAuth. The platform
Stack and manual service specifications passed its Pydantic models. The
corresponding local suite ran 401 tests (13 skipped), with Ruff lint/format checks.

The Paseo source build was executed locally with the pinned Node 22.20.0 and the
pinned commit: the installer produced all seven workspace tarballs, installed
them, and the resulting daemon answered `/api/health` 200, `/api/status` 401
unauthenticated and 200 with the `PASEO_PASSWORD` bearer token on the address
taken from a copy of the managed `config.json`. That local run used no client
identity, no Docker and no client volume; installation on a real enrolled client,
its public TLS and the production stack remain separate verification steps.

Package and build contracts were checked against
[Oduflow PyPI 1.75.0 metadata](https://pypi.org/pypi/oduflow/1.75.0/json),
[uv tool installation](https://docs.astral.sh/uv/guides/tools/),
[uv release checksums](https://github.com/astral-sh/uv/releases/tag/0.12.13), and
[Node 22.20.0 checksums](https://nodejs.org/dist/v22.20.0/SHASUMS256.txt).
Node 22 is the version the fork declares in `.tool-versions` and builds with in
`docker/base/Dockerfile`; that Dockerfile is also the source of the workspace
pack order used here. The source pin is a full commit. Reviewing a newer commit means updating `artifacts.json` alone: the
install path, receipt and systemd unit follow it.

Client package states install a list-only `needrestart` policy before package
operations. Salt owns service restarts; an automatic package-triggered restart
of Salt Minion can interrupt a durable provisioning operation before its result
is recorded. Interrupted operations require reconciliation before another dispatch.

## Prebuilt IDE runtime

`paseo.install_method` selects `source` (the default) or `prebuilt`.
The first verified Ubuntu 26.04 amd64 build is published in
[IDE Releases](https://github.com/oduflow/paseo/releases/tag/ide-0.8.0-ubuntu26.04-amd64).
Both the source repository and release assets are public.

```yaml
paseo:
  install_method: prebuilt
  runtime:
    source: https://github.com/oduflow/paseo/releases/download/ide-0.8.0-ubuntu26.04-amd64/ide-0.8.0-ubuntu26.04-amd64.tar.gz
    hash: sha256=6e60d34730f769790d138860dbd5148366f779583b2bfecfd5fd727a0d1582fa
```

Salt downloads the archive directly over HTTPS without authentication. Odoo stores
its URL, checksum and compatibility metadata; see [instance settings](ide-installation.md).
A `salt://` source must exist in the selected client's local release file roots.

The current build recipe is `client/scripts/build-paseo-runtime.py`. Its workflow
builds in `ubuntu:26.04` on amd64 and uploads a temporary Actions artifact. Publish
verified runtime archives as release assets in `oduflow/paseo` for permanent URLs.

The runtime includes all seven installed workspaces and their npm dependencies.
Node is installed separately from its pinned artifact. Before extraction, the
installer checks SHA256, Ubuntu version, architecture, Node version, IDE version
and full source commit. A mismatch fails without falling back to a source build.
An already installed matching release is reused.

A runtime build is not a client VM image or proof of full Ubuntu 26.04 provisioning.
