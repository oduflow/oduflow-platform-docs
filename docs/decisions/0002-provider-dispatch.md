# 0002 — Queue execution and uncertain Vultr outcomes

Accepted for the infrastructure stage on 2026-09-11.

`queue_job` performs external operations after intent is committed in Odoo.
An instance-row lock serializes changes. Jobs carry a generation; once the target
changes to deletion, stale creation jobs become no-ops. A queue identity key does
not replace external resource reconciliation.

## Dispatch and rollback

A worker transaction can roll back after a successful HTTP POST. Before creation,
a separate cursor commits an `oduflow.dispatch` receipt with a unique operation
key. The returned resource ID is committed there before further ORM changes.
The job's main cursor is never manually committed. Receipts have no foreign keys
or secrets and no user ACLs; only private executor methods write them.

This deliberately crosses normal transaction boundaries so the receipt survives
rollback and worker death. A subsequent execution finds the resource by recorded
ID or unique UUID label, then verifies its region and disk or VM configuration.
Duplicate matches and foreign attachments require intervention.

A crash between claiming and sending cannot be distinguished automatically from
an accepted request whose response was lost. The executor stops sending POSTs and
polls the provider, eventually reporting an uncertain outcome. Reconciliation
repeats observations without deleting the receipt. This can leave a request
blocked without a resource rather than risking duplicate paid allocation.

The adapter may record a definitive create rejection only around the actual
creation call. Confirmed HTTP 400/401/403/422 rejections permit an explicit
reconciliation to allocate a new dispatch generation. Old receipts remain intact;
transport errors, ambiguous responses and mutable journal error text do not grant
that permission.

Volume attachment is also sent at most once and confirmed by reading attachment
state. DELETE can be repeated, but completion requires a subsequent GET 404.
A temporary 404 never replaces a known ID. After an uncertain create, absence from
a listing does not establish that deletion is complete.

## Provider binding and limits

The API key's SHA256 fingerprint is captured when infrastructure provisioning
starts. A changed key blocks creation, reconciliation and deletion. This is a
conservative binding to the exact credential configuration because `/account`
does not expose an immutable account identifier. Rotation needs a separate
procedure that proves ownership of existing resources.

Include receipts in database backups. A restored older database may lag behind
the cloud: reconcile resources before resuming its queue. The protocol does not
promise exactly-once execution after journal loss, credential changes or external
label changes. Automatic receipt retention/cleanup is not implemented.

Infrastructure readiness covers the VM and disk. Full client deletion separately
accounts for DNS, VPN/Salt identities and other adapters; it must not infer those
resources were removed merely because the VM disappeared.

## Validation

ORM tests cover lost responses, retries, target changes, credentials, partial
deletion and polling limits. HTTP adapter tests use mocked transport. Actual
queue execution and independent receipt survival after main-cursor rollback were
also verified in the Megaflow Odoo 19 environment. Unit tests do not allocate cloud
resources.
