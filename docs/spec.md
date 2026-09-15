# Oduflow Platform — technical specification

This document describes the full intended
workflow; a requirement here is not evidence of a completed deployment. See the
[README](https://github.com/oduflow/oduflow-platform/blob/main/README.md) for implementation boundaries and the
[deployment journal](https://github.com/oduflow/oduflow-platform/blob/main/reports/deployments/deployed-control-plane.md) for live checks.

## 1. Objective

An action in the control-plane Odoo creates a dedicated client VM, installs Paseo
and Oduflow, and launches the client's standard Odoo production stack. The result
is working HTTPS at `company.oduflow.sh` and access to client management panels,
with configured quotas and a demo lifecycle.

The Odoo 19 control plane in Megaflow and the client's Odoo are separate instances.
The control plane creates external resources and tracks lifecycle. Client Oduflow
manages Docker and launches production inside the client's dedicated VM.

## 2. Components and ownership

| Component | Location | Responsibility |
| --- | --- | --- |
| Odoo 19 control plane and addons | An Oduflow-managed Odoo environment | Clients, plans, queue, provider APIs, Cloudflare DNS, Salt dispatch and lifecycle |
| `addons/oduflow` | Control-plane Odoo | Provider-independent core and bootstrap contract |
| `addons/oduflow_vultr` | Control-plane Odoo | Vultr catalogue/API, VM creation, block storage and attachment |
| `addons/oduflow_cloudflare` | Control-plane Odoo | Per-client R2 bucket, retention locks and bucket-scoped credentials |
| Headscale and Salt master/API | Dedicated Oduflow services or a separate control host | VPN coordination, minion trust and Salt execution |
| Odoo VPN gateway | Persistent Megaflow service | Odoo-to-Salt access and private pillar delivery to the master |
| Tailscale and Salt Minion | Each client VM | Headscale enrollment and client configuration |
| Paseo and Oduflow | Each client VM | Client management; Oduflow runs the production stack |
| Traefik and Let's Encrypt | Each client VM | Application HTTPS |
| Cloudflare `oduflow.sh` zone | Cloudflare | Authoritative DNS, managed by the control plane |
| GitHub, LLM gateway and email | External services | Repository, model budget and access delivery through separate adapters |

The former addon is merged into `addons/oduflow` without a compatibility wrapper.
Existing `oduflow.*` models, tables and records are retained; migration changes
metadata/XML-ID ownership rather than creating replacement client records.

## 3. Scope

The current workflow includes Odoo-driven Vultr VM creation, volume attachment,
VPN/DNS, trusted Salt enrollment, app installation, production launch and readiness
verification. It does not depend on manually supplying a client server.

Repository provisioning, budgeted LLM credentials, access delivery, expiry,
suspension, full client deletion and updates are also part of the platform's
lifecycle scope. Their implementation and live validation are tracked separately.

Optional golden-image acceleration through Packer/Vultr is now authorized. Public
signup, billing and additional infrastructure providers remain separate work.
Client production hosting is already part of the primary workflow.

## 4. Architectural decisions

1. One VM per client. Provider parameters and API calls belong in the provider
   addon; the core owns shared lifecycle behavior.
2. Clean Ubuntu 24.04/26.04 amd64 is supported by the bootstrap installer. Optional
   images preinstall dependencies, while per-client identity and real configuration
   are generated at first boot. Salt remains authoritative after image creation.
3. Odoo keeps immutable preparation snapshots, external resource IDs and a journal.
   Retries begin with reconciliation. Unknown creation outcomes never authorize
   blind POST retries. See the [dispatch contract](decisions/0002-provider-dispatch.md).
4. Salt manages installed software and client configuration. Initial bootstrap
   establishes Tailscale/Minion identity; subsequent states configure the same VM.
5. A minion uses `client-<UUID>`, a pre-registered public key and a pinned master.
   Arbitrary pending keys are never automatically accepted.
6. Headscale and Salt master run as dedicated services or on a separate control host. Clients run
   Tailscale, not their own Headscale server.
7. Client publication uses DNS-only Cloudflare records and direct Traefik HTTPS. Let's Encrypt
   HTTP-01 issues certificates for concrete application hostnames; the wildcard
   DNS record does not require a wildcard certificate. No Cloudflare DNS token is
   distributed to clients and Advanced Certificate Manager is not purchased.
8. Production creation uses the tested Oduflow 1.75.0 API contract, an empty template
   name and `odoo:19.0`. Publication is blocked until the administrator credential
   is hardened and the immutable container identity is verified.

## 5. Provisioning workflow

Preconditions: the control plane has a provider plan, credentials, Headscale/Salt
configuration and required application settings. No client VM exists yet.

1. **Prepare:** validate naming/quotas, freeze the plan snapshot, reserve the DNS
   subtree, generate UUID/secrets and create journal entries. `planned` creates no
   cloud resources.
2. **Prepare bootstrap:** generate the client's unique RSA identity. Before VM
   creation, register its exact Salt public key and obtain a non-reusable Headscale
   enrollment key. Validate renderable user-data; never log or bake in secrets.
3. **Create infrastructure:** create and reconcile the VM and volume, preserve
   provider IDs and confirm attachment. The instance remains `provisioning`.
4. **Connect management:** first boot installs/enrolls Tailscale and Salt Minion,
   checks the master pin and becomes reachable as the expected `client-<UUID>`.
5. **Publish DNS:** create DNS-only `company.oduflow.sh` and
   `*.company.oduflow.sh` records pointing to the verified client IPv4.
6. **Configure the client:** Salt verifies and mounts owned XFS storage, installs
   dependencies and configures Paseo/Oduflow. Data and services depend on the
   verified volume; an unknown disk must never be formatted.
7. **Create production:** close host ingress before Docker DNAT for both IP families,
   record a publication guard, call client Oduflow, set the Odoo administrator
   password and verify the resulting immutable container ID. Then restore only
   the guard's own rules and retry ACME when necessary.
8. **Back up client data:** the control plane creates a dedicated R2 bucket and
   exact-bucket credential. Salt stops data-owning services, backs up the verified
   block-volume mount, restarts services and restores an identity-bound probe from
   the resulting Restic snapshot. It enables the daily timer only after the
   byte-identical restore and repository check succeed.
9. **Verify readiness:** check valid HTTPS, Odoo 19 administrator login/logout and
   unauthenticated/authenticated panel behavior. Only the appropriate lifecycle
   completion step may activate the client and start its demo period.

GitHub and LLM credentials are prepared before the steps that need them. Missing
required credentials cause an explicit failure, never a fabricated working value.
Basic app installation does not require an invented LLM key or license.

`infra_state=ready` confirms the VM and disk; Salt ping confirms connectivity.
Neither substitutes for production checks. Retry applies to the existing VM,
not a replacement created for every Salt-state correction.

## 6. Network and names

| Purpose | Hostname |
| --- | --- |
| Headscale / Salt control host | `headscale.example.com` |
| Client production Odoo | `<company>.oduflow.sh` |
| Client Oduflow panel | `oduflow.<company>.oduflow.sh` |
| Client Paseo panel | `paseo.<company>.oduflow.sh` |
| Intended branch route | `<branch>.<company>.oduflow.sh` |
| Additional service | `<service>.<company>.oduflow.sh` |

The client owns its entire DNS subtree. Panel names cannot be reused by branch
routes. Control-plane quotas limit resource counts, not a static collection of
`devN`/`svcN` names. Prepared `flat_v1` instances keep their legacy addresses.
Oduflow 1.75 requires explicit route hostnames for branch routes directly below
the client domain; implicit branch naming can add the panel's extra label.

Ports 80/443 reach client Traefik for HTTP-01 and HTTPS. Cloudflare proxy/Tunnel is
not enabled automatically: an origin certificate does not provide Cloudflare edge
coverage for deep hostnames. Salt 4505/4506, Salt API and pillar are VPN-restricted.
Clients cannot access Salt API, other clients or the pillar gateway.

## 7. Pillar and secrets

`GET /oduflow/pillar/<minion_id>` returns JSON `schema=1` only for eligible
instances with required configuration. It checks the original gateway socket peer
and Bearer token; forwarded headers cannot bypass this. Salt external pillar
validates the UUID and schema.

The contract includes `client`, `dns`, `naming_version`, `ingress`, `oduflow`,
`paseo` and `storage`, plus a complete `backup` bundle only for opted-in snapshots.
Direct TLS uses HTTP-01 and the prepared ACME contact
configured for the installation, without `tunnel_token`, `cloudflared` or a broad
DNS API token. Separately generated credentials protect Oduflow UI/MCP, PostgreSQL,
Paseo and the production administrator. Complete optional Git/LLM bundles are
validated before use.

Secrets are encrypted using an external environment key stored separately from
DB backups. Secret files have restrictive permissions and Salt suppresses their
content in changes. Golden images contain no client PKI, enrollment credentials,
VPN state, customer data or certificates.

## 8. Client deletion, updates and storage growth

Full client deletion accounts for client workloads, provider VM/volume, DNS,
Salt/Headscale identities, LLM access and repository policy. Preserve journal IDs;
only report `destroyed` after the relevant cleanup has been confirmed. VM deletion
alone does not establish all external cleanup is complete.

Salt applies pinned application changes to existing VMs. Test an update on one
client before wider rollout; changing a running client does not require rebuilding
its golden image.

Block storage can grow online through an explicit target-size request. Keep the
original preparation snapshot unchanged and record verified current capacity
separately. Reconcile provider resize before growing the existing owned XFS mount;
never shrink, format, detach or stop services as part of the online grow path.
No paid resize is inferred when the operator has not supplied a target.

## 9. Acceptance criteria

Verify Odoo-driven client creation, production HTTPS and authentication, no duplicate
resources on retry, reboot recovery, service shutdown on loss of the data volume,
and confirmed client deletion. Exercise an opted-in backup and restore, then a
separate replacement-VM recovery rehearsal. Billing and additional provider
integrations require their own end-to-end checks. Record observed results, source
revision and deployment identities in a separate deployment report; unit tests
alone do not establish live readiness.
