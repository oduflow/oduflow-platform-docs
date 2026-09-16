# Client application installation

Client software belongs to `oduflow-client`, mounted at `client/` in the platform
checkout. This guide describes its Salt states; use the control-plane
[release workflow](client-releases.md) to apply them to an enrolled client.
Package versions, URLs, checksums and the IDE source commit are declared in the
selected release's `salt/states/client_apps/artifacts.json`.

## Installation stages

| State | Effect |
| --- | --- |
| `client_apps.install` | Install verified application artifacts and prerequisites without client credentials or application startup |
| `client_apps.configure` | Prepare guarded Docker storage, application configuration and systemd units |
| `client_apps.start` | Validate configuration, start Client Oduflow and IDE services, register the client project and check local health |
| `roles.client_stack` | Compose the client configuration workflow, including storage and the application stages |

Production creation and public verification belong to the separate
[production role](client-production.md). A configuration update does not recreate
production. Package states install a list-only `needrestart` policy so Salt
controls restarts; an interrupted operation requires receipt reconciliation.

## Application artifacts

Oduflow is installed from a checksum-verified wheel. Node and uv use verified
artifacts. The installer copies its version manifest to
`/var/cache/oduflow-apps/artifacts.json`; prose version numbers do not override it.

The client release includes a checksum-pinned IDE source archive under
`salt/states/client_apps/artifacts/`. Installation needs no GitHub credential for
this archive or the public `oduflow/paseo` repository. Customer project access is
separate; see [repository access](github-download-access.md).

The source build verifies the archive, embedded commit, version and extraction
paths. It runs `npm ci`, packs the declared workspaces in dependency order, and
installs them into `/opt/oduflow/paseo/<version>+<commit>`. The source's contributor
Git hook is removed before installation. Build directories and the private npm
cache are removed afterward. Source builds run reviewed npm lifecycle scripts as
root; the source commit and checksum define the trusted input.

The build uses a 4 GiB V8 old-space limit. Small builders need sufficient memory,
swap and scratch storage. This is a compiler setting, not a runtime memory cap.
Use a verified [prebuilt IDE runtime](ide-installation.md) or
[golden image](golden-image.md) to avoid repeating compilation.

Installation receipts are written only after success. Paths, receipts and the
systemd unit identify both version and commit because a fork can retain its
upstream version string. Missing executables trigger installation again.
The source lockfile pins build dependencies; runtime tarball/wheel dependencies
and signed Ubuntu repositories are not all frozen by the client Git SHA.

For maintainers, run this from the **client repository root** to prepare a new
source archive, then review and commit the matching manifest and artifact:

```sh
python3 scripts/prepare-client-artifacts.py \
  --repository /path/to/reviewed/paseo --output /path/to/staging
```

## Configuration inputs

Pillar schema 1 must identify the exact `client-<UUID>` and verified storage.
Application configuration requires:

- `dns.oduflow`, `dns.paseo` and `ingress.mode=direct_tls`;
- `ingress.acme.email` for HTTP-01 certificates;
- distinct `oduflow.auth_token`, `oduflow.ui_password`,
  `oduflow.database_password` and `paseo.password`;
- bounded quotas and lifecycle values from the prepared plan.

The retained `paseo` keys configure the UI's **IDE**. See
[terminology](glossary.md#names-and-compatibility). Its generated password must be
at least 16 URL-safe characters. Credential files are mode `0600`, their diffs
are suppressed, and passwords are not command arguments.

The PostgreSQL password is mandatory and never falls back to an application
credential or upstream default. Editing TOML does not rotate an existing
PostgreSQL role. Production receives its own generated administrator password.
An omitted license key does not erase an already installed license.

Optional Git/LLM bundles must be complete before they are supplied. No LLM login
is needed simply to start the authenticated interfaces. Agent execution uses
[client agent configuration](client-agent.md). Storage formatting and UUID
recovery follow the [storage contract](salt-minion.md#storage-handoff-and-states).

## Project registration

GitHub CLI is installed during the credential-free package stage. Once repository
access is verified, `paseo.project` configures the project's Git access, clones its
confirmed branch into `/srv/paseo/projects/<repository>` and registers it with
the daemon. Git fetch/clone/push use the managed SSH identities for new clients;
GitHub API commands require a separate API credential.

Retries check the existing origin and preserve working copies. They do not pull,
reset or create duplicate projects. Registration must succeed before application
health completes. Keys and working copies stay on the client data volume and
never enter an image. See [repository access](github-download-access.md) for key
scope, legacy migration and revocation.

## Storage and listeners

| Component | Data / listener |
| --- | --- |
| Docker | `/srv/oduflow/data/docker` |
| containerd | `/srv/oduflow/data/containerd` |
| IDE | `/srv/paseo` points to its private directory on the verified volume; loopback listener `127.0.0.1:6767` |
| IDE proxy | `172.17.0.1:6768` forwards to the loopback listener |
| Client Oduflow | Docker-bridge listener `172.17.0.1:8000`; team data on the verified volume |

Storage guards are installed before packages can start Docker. Existing data in
`/var/lib/docker` or `/var/lib/containerd`, or incompatible configuration, causes
preflight refusal; automatic data migration is not provided. The services stop
when the verified mount disappears. Docker live-restore is disabled.

IDE runs as the unprivileged `paseo` user using
`paseo daemon run --home /srv/paseo`. Listen/relay/proxy settings live in
`/srv/paseo/config.json`; `/etc/paseo/credentials.env` supplies its password and
client MCP token. Client Oduflow runs as root to manage Docker and XFS quotas.
The IDE proxy lets containerized Traefik reach the host daemon without exposing
its private listener publicly.

Client Oduflow owns Traefik. The hostname and TLS contract lives in
[DNS and certificates](cloudflare.md). The managed IDE route retains the internal
name `[route.paseo]` even when its public hostname starts with `ide`.
Use the actual generated runtime to inspect upstream container tags and limits;
the client manifest does not digest-pin every image selected by Oduflow itself.

## Verification

From the **client repository root**, install `requirements-test.txt` and run the
relevant `test_client_apps.py` suite with Salt's rendering dependencies. Verify
candidate TOML using the Oduflow version pinned by that same release.

Local tests cover invalid configuration, private credentials, package/mount
ordering, installation receipts, source-pin checks and refusal to adopt existing
container data. A local build or healthy daemon does not establish public TLS,
production readiness or client capacity. Record real deployment checks separately.
