# IDE installation per instance

Open **Client Instances → instance → Configuration → IDE**. **IDE Installation**
selects **Source** (the default) or **Prebuilt**. For Prebuilt, select a ready
**IDE Build**. The build's compatibility metadata and SHA256 are shown below it.
Save the instance before provisioning; for an already configured client, use
**Save & Apply** or **Apply Updated Configuration**. Editing settings alone does
not launch Salt. Existing matching installations are reused regardless of the
selected method; selecting a build is not a forced reinstall.

Builds are managed under **Configuration → IDE Builds**. Create a build with its
GitHub release asset ID from `oduflow/paseo` and independently obtained SHA256,
then click **Verify Build**. A queue job downloads the public archive without
credentials and checks its SHA256 and runtime metadata. Odoo stores the verified
URL and metadata, not the binary. Verified entries are immutable; create another
build for a different archive.

The selected client revision pins the IDE commit, version and Node version.
Control verifies that the build matches these values. The installer additionally
requires an exact OS version and architecture match. The Ubuntu 26.04 amd64 build
requires Node 22.20.0; selecting it for Ubuntu 24.04 is unsupported.

Immediately before dispatch, control freezes the installation method, public URL
and checksum for that Salt request. Retries use the same selection. Preparation
commits before another DB connection supplies pillar to Salt. The client downloads
directly from GitHub over HTTPS; no download token, repository key or control
proxy is involved. GitHub App authentication is not required for public software
repositories. Client-owned project repositories retain their separate write access.

## Hostnames and compatibility

New client subdomains use `ide.<slug>.<domain>`. Existing provisioning snapshots
retain their original names. On a configured client, **Use IDE Hostname** records
an operational hostname override and queues a new configuration; it does not
rewrite the original provisioning snapshot. Resolve running or uncertain jobs
before migrating. The new route becomes available only after Salt applies it.
The internal `paseo` pillar key and package/service identifiers are retained for
compatibility; UI labels use IDE.

## Runtime archives for maintainers

The selected client's `salt/states/client_apps/artifacts.json` pins the IDE source
commit, version and Node version. From the **client repository root**,
`scripts/build-paseo-runtime.py` builds the runtime; its CI workflow produces an
archive and metadata for review. Publish verified archives as public release
assets in `oduflow/paseo` before registering an IDE Build in Odoo.

Salt receives the frozen selection through the retained technical keys:

```yaml
paseo:
  install_method: prebuilt
  runtime:
    source: https://github.com/oduflow/paseo/releases/download/RELEASE/ARCHIVE.tar.gz
    hash: sha256=REVIEWED_ARCHIVE_SHA256
```

These are placeholders, not a usable build. Use the URL and checksum from the
verified build record. Source mode is `install_method: source`. A `salt://` URL
must resolve within the selected client's local release file roots.

The archive contains installed workspaces and npm dependencies; Node is installed
separately. Before extraction, the installer checks the archive hash, OS version,
architecture, Node/IDE versions and full source commit. A mismatch fails without
silently compiling from source. A matching installed release can be reused.

An IDE archive is distinct from a [client VM image](golden-image.md). Verifying
an archive does not establish a complete client deployment or public readiness.
