# 0001 — Odoo 19 control plane and deployment preparation

Accepted for the initial foundation on 2026-09-11. Later decisions extend this
stage; statements below describe its boundaries, not the entire current system.

The user selected a separate Odoo 19.0 with a clean database, deployed through
megaflow MCP using `template_name="none"`. This concerns the control plane;
client production is a separate installation and also starts without a template.

Preparation ends in `planned`: one transaction freezes quotas, reserves DNS,
generates encrypted credentials and creates the operation journal. It performs
no network operations. An instance-row lock makes repeated preparation safe.
External operations run separately through `queue_job`.

- `minion_id = client-<UUID>` keeps machine identity independent of client naming.
- DNS ownership is global and persists after destruction. The initial `flat_v1`
  design reserved explicit service slots. The current `nested_v2` design reserves
  the whole client subtree and no longer allocates individual `devN`/`svcN` slots.
- Plan parameters are copied into an immutable snapshot. Editing a plan does not
  update existing clients.
- `planned`, `suspending` and `decommissioning` distinguish lifecycle transitions.
  A demo period begins after readiness, not when the database record is created.
- Until activation, pillar `client.expires_at` is `null`; consumers must accept it
  rather than independently calculating an expiry date.
- Secrets live in a separate model without user ACLs. The encryption key is supplied
  only through Odoo's environment. Pillar remains unavailable until its network
  restriction and separate master token are configured.

An idempotency key alone does not make an external operation idempotent. The
executor records intent before a network request, reconciles uncertain responses
and never blindly repeats creation. `uncertain` represents this condition.
The implemented protocol is described in [decision 0002](0002-provider-dispatch.md).

Backups must include the control-plane database and separately preserved encryption
key. Salt Master is a trusted secret processor: external pillar does not eliminate
secrets from memory or possible caches/results. Salt configuration must explicitly
restrict these surfaces.
