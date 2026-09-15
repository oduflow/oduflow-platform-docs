# Manage private network nodes from Odoo

`addons/oduflow_headscale` extends the control plane independently of the VM
provider. Install it alongside `oduflow`. The **Oduflow → Private Network** menu
contains Nodes, Server Enrollments, and Operations.

## Inventory and access

**Synchronize Nodes** queues a complete inventory fetch. A scheduled job also
requests synchronization every five minutes. Each row records the Headscale ID,
identity fingerprint, VPN addresses, tags, observed online state, last activity,
expiration, and related client server. A missing node remains in history with
Registered cleared. Existing client nodes are associated by their canonical
`client-<UUID>` hostname. Nodes using an Odoo-issued enrollment inherit its company.

**Oduflow Observer** can read permitted metadata. **Oduflow Admin** can request
synchronization, issue and revoke enrollment keys, and confirm node expiration.
Company record rules apply to client and enrollment-owned nodes. Shared
infrastructure is visible across allowed companies. Raw Headscale machine, node,
and discovery keys are never imported.

Operations run through `queue_job` and retain their requester, immutable request
UUID, status, timestamps, and a sanitized error code. Open forms poll for changes
every five seconds and notify on completion or a result needing attention. The
poller defers form reload while the user has unsaved changes. Browser notifications
are local UI feedback; the module sends no external email or chat messages.

## Connect a LiteLLM server

1. Install and start Tailscale on the target server.
2. Open **Server Enrollments**, create a record with its hostname, choose the
   owning company, and keep the default **30-minute** registration window.
3. Click **Issue Enrollment Key**. Wait for Key Ready, then open
   **Show Connection Instructions** as an administrator.
4. Save the key to the indicated private file on the target host and run the
   displayed command. Remove that file after successful registration.
5. Synchronize Nodes to see the registered server and mark the enrollment Used.

The manual enrollment UI currently issues only `tag:llm`. Client VM enrollment
continues through the existing provisioning/bootstrap workflow. The master and
Control Plane tags cannot be issued by this management bridge. Keys are
single-use and non-ephemeral, with a supported registration window of 5–60 minutes.
The key expiration limits when a new node may register; it does not define the
lifetime of that registered node. Hostnames are requested by the joining client,
not cryptographically bound to the enrollment key.

The enrollment key is encrypted in a separate Odoo model without ordinary access
grants. The administrator's temporary instructions dialog computes it on demand;
it is not stored in the dialog, operation log, or queue arguments. Used, locally
expired, and revoked enrollments erase the Odoo copy. The master retains protected
request receipts for safe reconciliation; receipt access is restricted to Salt.

See [LiteLLM host setup](litellm-vpn.md) for private HTTP access, client inference
access, and the distinction between VPN enrollment and LiteLLM API credentials.

## Revoke a key or expire a node

**Revoke Unused Key** prevents a still-unused credential from registering a node.
It does not disconnect a node that has already joined. If a key was consumed since
the last sync, refresh inventory and manage the actual node separately.

**Disconnect Node** opens a warning and requires the operator's current personal
Odoo password. The dialog freezes the target's external ID and fingerprint;
changing the target or replacing that identity invalidates the confirmation.
The queue receives no password. The bridge verifies the same identity before
requesting expiration. Master and Control Plane nodes are protected in both the
addon and the bridge.

Disconnection uses Headscale's node expiration API. It preserves the node record,
VM, disk, and application data; rejoining requires registration again. Expiration
is verified by reading Headscale's stored expiry. The observed Online field comes
from inventory and can lag this operation. Immediate teardown of every existing
connection is not an asserted guarantee. Do not use this action as a substitute
for a host firewall rule when immediate traffic interruption is required.

## Reconciliation and idempotency

The bridge writes a durable request claim before any creating or mutating POST.
Repeating the same request UUID with different parameters is rejected. Repeating
a successful enrollment returns the same still-usable key, never creates a new
one. If it was used, expired, or removed, the response carries that state without
the key. Revocation is bound to the original enrollment receipt; node expiration
is bound to its identity fingerprint.

**Reconcile Operation** preserves the original request UUID. Revocation and
expiration can converge from a subsequent read after a lost HTTP response. If an
enrollment POST might have succeeded but its response was never saved, the
operation remains Needs Review; no automatic second key is created. An operator
must reconcile Headscale state privately or let the uncertain key expire. There
is no public action to discard such a claim and repeat its POST.

## Transport and deployment

The addon reuses the existing Salt HTTPS transport and its configured SOCKS
gateway. It needs no Headscale API token inside Odoo. The master alone reads
`/etc/salt/oduflow-headscale-api.key` and calls the fixed loopback API at
`127.0.0.1:8080`. Salt eauth exposes only these additional runner methods:

- `oduflow.headscale_nodes`
- `oduflow.headscale_enroll`
- `oduflow.headscale_revoke`
- `oduflow.headscale_expire`
- `oduflow.headscale_delete`

Inventory is bounded and fails rather than silently truncating. Request receipts
live under `/var/cache/salt/oduflow-headscale`, owned by Salt, directory mode
0700 and file mode 0600. Global Salt job and pillar caches stay disabled. Deploy
the matching Salt runner/configuration before installing the addon, then use
Megaflow `pull_and_apply(install="oduflow_headscale")` and verify a real queued
inventory operation.

API contract references:
[Headscale registration](https://headscale.net/stable/ref/registration/),
[v0.29.3 node expiration handler](https://github.com/juanfont/headscale/blob/v0.29.3/hscontrol/grpcv1.go#L451-L503),
[v0.29.3 node expiration storage](https://github.com/juanfont/headscale/blob/v0.29.3/hscontrol/state/state.go#L900-L930).
