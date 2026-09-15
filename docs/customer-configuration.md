# Customer configuration

Customers edit **My Account → Oduflow → Server → Configuration**. Administrators
use the same fields on the instance's **Configuration** tab. Observers are
read-only. The portal never receives access to internal instance, revision or
configuration-line models: a private bridge checks current commercial ownership
and the allowed company before elevating access to an explicit field allowlist.

The editor contains:

- Environments: automatic stop and deletion, hours, and explicit deletion acknowledgement.
- AI Agents: enablement, default CLI, model overrides, provider credentials and
  a table of additional encrypted environment variables.
- Backups: own S3 bucket, HTTPS endpoint, region/prefix, encrypted credentials,
  daily UTC schedules, snapshot retention rows and WAL-G full-backup retention.
- Resource Limits: plan maximums and optional smaller positive client limits.
  Zero never bypasses a paid quota; database quota cannot exceed disk quota.
- Advanced: overlay threshold, bounded production worker cap, telemetry,
  diagnostic tracing and local-path development.
- Custom Routes: named HTTP(S) upstreams using unused hostnames within the
  client's namespace. DNS must already resolve the hostname to the client.
  Existing platform hostnames and the managed Paseo route cannot be replaced.

The platform still owns listener addresses, ports, routing/TLS mode, storage
paths, database credentials, component images, production enablement and OAuth
routing. Administrators retain the existing draft-only deployment fields.
Password/access-link and GitHub SSH-key actions remain separate from this editor.

## Saving and applying

Backend edits are drafts until **Save & Apply** is pressed. The portal saves and
queues application in a single transaction. Forms carry an edit version, so a
stale portal submission cannot overwrite newer settings. The immutable plan
snapshot is never rewritten to store customer preferences.

Each application creates a separate `oduflow.client.settings.revision`: an
immutable non-secret payload and an independently encrypted credential snapshot.
Only that frozen request contributes to pillar. Later draft edits cannot alter a
previously requested or successfully applied configuration. Pending, uncertain
or failed applications must be finished/reconciled before a new revision;
administrators use the existing configuration reconciliation workflow.

The existing configuration queue and durable dispatch receipts perform Salt
application against the expected client UUID. A missing response never licenses
blind redispatch. Credential rotations and custom Salt operations block settings
changes while in progress. A successful `configure` result and client health
checks mark the requested revision applied and record its time. Failed work
retains the last successfully applied revision; it may have made partial changes,
so that revision is historical evidence, not a claim of complete rollback.

Salt validates a candidate TOML with the pinned Oduflow package **before**
replacing the live secret file, validates it again before service startup, and
runs application health checks. Oduflow's systemd environment fixes `TZ=UTC` for
backup scheduling. Oduflow/agent restarts may interrupt their sessions; existing
Odoo application containers are not intentionally stopped by this workflow.

An applied backup configuration does not prove a successful backup or restore,
and an applied route does not establish DNS ownership/resolution or upstream
availability. Agent model/API availability depends on the customer's provider
credentials. Component package versions remain platform-managed.

## Secrets and validation

Credential replacement inputs and custom variable values have no plaintext
columns. Draft secrets use `oduflow.secret`; revision secrets are encrypted with
the instance-bound encryption helper. Empty inputs preserve saved values;
explicit removal clears the draft. Portal projections show presence only and
never echo secret submissions, including validation errors. Password resets
preserve the configuration secret bundle.

Backend child models enforce manager permissions, company rules, instance
locking and immutable parent ownership, including direct RPC. Portal table IDs
must belong to the current instance. Custom variable names cannot replace
platform-owned or dedicated provider variables. Routes cannot replace the
platform's actual hosts or publish a cloud metadata endpoint.

## Verification

Run Odoo configuration, credential and portal suites through Megaflow. Run local
`test_customer_settings.py`, `test_client*.py` and `test_salt_apply.py` with the
Salt test dependencies. The complete rendered TOML also passes parsing and
validation by the checksum-verified Oduflow 1.75.0 package. DOM checks cover
conditional sections, adding/removing table rows, submission serialization,
lifecycle switches and editable quota controls.
