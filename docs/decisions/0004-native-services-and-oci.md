# Native application services and OCI distribution

Status: Implemented; image builds and basic container lifecycle qualified in CI.

Salt remains the configuration authority. LiteLLM and its durable exporter are
one application release, installed into a versioned Python environment and
supervised by systemd. The same Salt application states run on a verified VM
or inside an OCI image. VM storage and network preparation remain host concerns;
container startup verifies a mounted volume against the metering source identity.

The Oduflow deployment uses its existing managed service database, with a separate
non-superuser PostgreSQL role. No additional PostgreSQL server is deployed by
the stack. The local SQLite ledger remains persistent and is backed up consistently
with that database. Existing Docker gateway revisions and histories remain valid;
upgrading the addon does not migrate a running gateway to native execution.

Master/API and LiteLLM/exporter are separate systemd containers. A third,
userspace Tailscale gateway connects to Headscale and forwards only configured
service ports. It also offers SOCKS access and the existing restricted pillar
proxy. The Docker subnet is not advertised to client machines.

Oduflow exposes explicit container lifecycle settings and preserves them through
create, update, inspection, presets and declarative stack reconciliation. Container
systemd is not assumed portable merely because an image builds: cgroup access,
shutdown, service failure and persistent-state recovery need live qualification.
The initial qualification recipe explicitly uses privileged systemd containers
on a dedicated test host; a least-privilege production profile is not yet proven.

See [deployment and verification](../container-services.md).
