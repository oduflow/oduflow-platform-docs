# Client demo and service lifecycle

New client activations enter the automatic trial policy. The demo deadline and
the suspension deadline are separate: applications remain available through the
prepared plan's grace period (zero days by default). New demos and extensions
have no grace period unless an administrator explicitly configures one. With zero
days, the suspension deadline equals the demo deadline. The hourly scheduler
only enqueues expiry; the job locks and rechecks the current policy, deadline,
commercial subscription and active operations before requesting suspension.
An extension committed before expiry dispatch takes precedence. Existing clients
remain on manual lifecycle after upgrade until an administrator explicitly uses
**Extend Demo**. Archiving a record does not exempt an automatic trial from expiry.

Only Oduflow administrators can extend demos, suspend, resume or reconcile these
operations. **Extend Demo** takes a future deadline, grace days and an audit
reason; it never shortens the existing term or rewrites the provisioning snapshot.
An expired suspended demo needs an extension before **Resume**. Extending alone
does not resume services. No operation sends customer mail or deletes resources.

Suspension stops Oduflow, IDE and its proxy, gracefully stops running Docker
containers, then stops Docker/socket activation and containerd. A durable receipt
preserves the service/container inventory. Systemd conditions prevent their
restart while the suspension marker exists, including after a reboot. Salt Minion,
Tailscale, SSH and the backup timer stay available. Backup and lifecycle share an
exclusive host lock so neither can restart services from a stale inventory.
VM and block-volume allocation and their provider charges continue unchanged.

The provisioning inference key and verified managed gateway bindings are blocked
with identity-checked read/write/read reconciliation. Preexisting blocks and later
billing cancellation blocks survive resumption. Customer-supplied external API
credentials are outside the platform's revocation authority.

Resume verifies the existing owned filesystem, starts the captured services and
containers, and reruns authenticated application health and public production
HTTPS/login/logout checks against the existing production receipt. It neither creates
production nor reinstalls applications, reformats storage, or resets demo dates.
A failed start/health check leaves the control state `resuming` for reconciliation.

Each suspend/resume pair has a monotonically increasing cycle. Each Salt dispatch
uses the existing authenticated custom-state transport, a fixed generated payload,
an immutable client Git revision and durable master/minion/Odoo receipts. Unknown
outcomes retain the same request. A new attempt is allowed only after an explicit
reconciliation observes a terminal failed Salt result. Old jobs cannot alter a
later lifecycle or decommissioning state. Journals retain completed cycles.

Activating an explicit billing subscription on an active server changes the policy
to commercial and removes automatic demo suspension. Historical demo dates remain.
Billing cancellation and infrastructure deletion are separate operations; this
release does not introduce automatic deletion or payment-delinquency suspension.
Concurrent billing activation and expiry serialize on the instance row.

Commit platform and client changes together, select the platform SHA in the
client-release manifest, and upgrade `oduflow,oduflow_billing,oduflow_litellm,oduflow_portal`.
Client job transport must already support immutable client revisions. No new master
runner, minion profile, Packer image or provider resource is required.

Run client lifecycle/backup tests with the client's pinned Salt dependencies,
and Odoo lifecycle, credentials, verification, pillar, billing, LiteLLM and portal
tests through the selected control Oduflow MCP. Unit tests simulate failures; a
real service interruption and reboot rehearsal must be recorded separately.
