# IDE installation

Each instance selects Source or Prebuilt. Source uses the IDE source archive
included in its selected client tree. Prebuilt selects a tagged IDE release such
as v1.0 from Configuration → IDE Releases.

## Synchronization

Sync Releases and the hourly Odoo schedule enqueue the same synchronization job.
It imports the ten most recently published stable GitHub releases with v1.0-style
tags, excludes drafts and prereleases, and retains older referenced records.
Master downloads archives through a restricted runner; no Master minion or Master
cron is needed. The credentials used to read private GitHub assets never reach
client pillar or artifact files. Control stores metadata, not archive attachments.

A release has three separate archives for Ubuntu 22.04, 24.04 and 26.04 amd64,
plus manifest.json and SHA256SUMS. The manifest contains contract=1, release_tag,
commit, and variants. Each variant records asset, size, sha256, and metadata;
metadata includes contract, release_tag, commit, version, os, os_version,
architecture and node_version. OS package requirements and build image digests
may be recorded alongside these fields. The same tag and commit identify all
three variants. Published manifests cannot be silently replaced.

## Apply

Ready means the complete matrix is verified on Master. Save & Apply freezes the
selected release descriptors. The minion chooses its OS/architecture, verifies
SHA256, and installs the matching archive with the declared Node runtime.
Unsupported platforms fail explicitly. New releases do not update existing
clients automatically. Master caches archives under content hashes, separate from
client source archives addressed by platform commit SHA.

## Hostnames and compatibility

New client subdomains use `ide.<slug>.<domain>`. Existing provisioning snapshots
retain their original names. On a configured client, **Use IDE Hostname** records
an operational hostname override and queues a new configuration; it does not
rewrite the original provisioning snapshot. Resolve running or uncertain jobs
before migrating. The new route becomes available only after Salt applies it.
The internal `paseo` pillar key and package/service identifiers are retained for
compatibility; UI labels use IDE.
