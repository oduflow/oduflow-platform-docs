# Platform architecture

Oduflow Platform uses an **Odoo 19 control plane** to provision and operate one
VM per client. Each VM runs **Client Oduflow**, which creates that customer's
production Odoo and development stacks. These are separate installations with
separate data and credentials. See [terminology](glossary.md) and the
[deployment runbook](client-debugging.md) for the reading and operating paths.

This page describes source contracts. Deployed versions and completed checks
belong in dated [deployment reports](https://github.com/oduflow/oduflow-platform/blob/main/reports/deployments/README.md).

## Components and ownership

| Component | Runs in | Owns |
| --- | --- | --- |
| Control Odoo and `addons/oduflow` | Oduflow-managed Odoo environment | Customers, plans, lifecycle, secrets, queues, immutable snapshots and operation journals |
| Provider adapters, including `oduflow_vultr` | Control Odoo | Provider-specific catalogues, VM, volume and firewall APIs |
| DNS and backup integrations | Control Odoo | Client DNS ownership, R2 buckets and scoped backup credentials |
| Headscale | Platform service or separate control host | VPN coordination and access policy |
| Salt Master/API and VPN gateway | Platform services or control host | Trusted enrollment, authenticated pillar and restricted job transport |
| Salt Minion and Tailscale | Each client VM | Client identity, connectivity and configuration execution |
| Client Oduflow and IDE | Each client VM | Application lifecycle and project development |
| Production Odoo, PostgreSQL and Traefik | Client containers | Customer application, database and HTTPS |
| LiteLLM and exporter | Independent gateway services/hosts | Inference authorization and durable usage metering |

The core does not depend on Vultr. Existing `oduflow.*` models and records are
preserved across platform upgrades. For code ownership and release delivery, see
[client releases](client-releases.md); for the control deployment, see
[platform stack](container-services.md).

## Architectural contracts

- **Salt owns configuration.** A supported clean Ubuntu host or an optional
  sanitized image receives fresh identity and configuration after boot.
  [Images](golden-image.md) accelerate installation; they contain no client keys,
  enrollment state, data or certificates.
- **Identity is independent of naming.** The minion ID is `client-<UUID>`, with a
  pre-registered public key and pinned master. Names and grains do not authorize
  enrollment. See [Salt master](salt-master.md).
- **Plans are immutable evidence.** Preparation freezes resource choices, quotas,
  initial naming and client release. Configuration revisions and operational
  overrides preserve that snapshot.
- **External work is queued.** HTTP actions enqueue work and return. Dispatch
  intent is persisted independently before resource-creation requests; ambiguous
  responses require reconciliation. A queue identity key alone does not make a
  provider POST safe to retry. See [dispatch design](decisions/0002-provider-dispatch.md).
- **Client data stays on verified storage.** Docker/containerd and application
  data use the owned block volume. Unknown filesystems are never formatted.
  Mount guards prevent services from starting on the wrong disk. See
  [storage](salt-minion.md).
- **Public access follows verification.** Production creation runs behind an
  ingress guard until the administrator password is hardened and immutable
  container identity is checked. See [production bootstrap](client-production.md).

## Provisioning and readiness

| Stage | Result | Detailed procedure |
| --- | --- | --- |
| Prepare | Resolve the selected client release, freeze the plan, reserve the DNS subtree and prepare secrets/journal; no VM allocation | [Client releases](client-releases.md) |
| Allocate and enroll | Reconcile provider resources, verify attached storage, establish the expected VPN/minion identity | [Deployment runbook](client-debugging.md), [Salt minion](salt-minion.md) |
| Configure | Apply the selected client release, storage and applications; record verified configuration only after health checks | [Applications](client-apps.md) |
| Publish production | Create and harden production, verify HTTPS, login/logout and panel authentication; complete opted-in backup checks | [Production](client-production.md), [backup](client-backup.md) |
| Activate and hand over | Apply the service lifecycle and offer one-use access delivery | [Lifecycle](client-lifecycle.md), [access](client-access.md) |

`infra_state=ready` establishes VM/disk readiness. It does not certify
applications, production or an AI coding session. Activation and credential
handover are separate: the delivery operation completes when a grant is consumed.
Billing requires an explicit subscription; installation never retroactively
subscribes or charges existing clients.

## Network and names

New instances use a common slug for their project repository and DNS subtree:

| Purpose | Default address |
| --- | --- |
| Production | `<slug>.<domain>` |
| Client Oduflow | `oduflow.<slug>.<domain>` |
| Client IDE | `ide.<slug>.<domain>` |
| Development/service route | `<name>.<slug>.<domain>` |

The client owns the entire subtree. The control plane does not reserve fixed
`devN`/`svcN` slots; actual managed application routes cannot be replaced by a
custom route. Older snapshots retain their addresses, including `paseo` names.
The authoritative naming/TLS contract is [DNS and certificates](cloudflare.md).

Administrative Salt traffic stays on the private network. Public application
traffic uses DNS-only records and direct Traefik HTTPS. Headscale's coordination
endpoint is a separate direct HTTPS service. See [network policy](headscale.md)
and [administrative SSH](administrative-ssh.md) for their distinct access paths.

## Configuration and secrets

Control Odoo keeps deployment credentials encrypted, with its encryption key
outside the database. The master retrieves authenticated schema-1 pillar for
one expected client; global job and pillar caches stay disabled. Public summaries
contain reduced statuses and counts. The [pillar preview](client-pillar-preview.md)
redacts secrets and does not prove that a client applied those values.

[Control settings](control-settings.md) define platform configuration.
[Customer settings](customer-configuration.md) describe editable preferences and
frozen application revisions. [Repository access](github-download-access.md)
separates public software downloads, customer project keys and partner access.

## Continuing operations

- [Client releases](client-releases.md) cover controlled updates and their limits.
- [Online storage growth](salt-minion.md#online-storage-growth) preserves the owned
  volume and original plan; shrinking is unsupported.
- [Lifecycle](client-lifecycle.md) covers trials, suspension and resumption.
- [Recovery](deployment-recovery-ui.md) covers failed or unknown configuration.
- [Access roles](access-control.md) cover password-confirmed client deletion.
- [Billing architecture](billing-architecture.md) covers subscriptions and metering.
- [Master automation](master-automation.md) covers infrastructure recovery.

Provider allocation, running services, a backup upload and passing unit tests each
prove different things. Record actual deployment and verification evidence for
the selected target; historical reports are not proof of its current state.
