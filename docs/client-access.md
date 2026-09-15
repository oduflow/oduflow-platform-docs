# Client access and password rotation

Oduflow Admin can use **Access Link** and **Reset Password** beside Client
Addresses. Portal customers have the same two buttons for their own servers
under **Passwords and access** on their server page. Observers cannot issue
links or reset passwords. Neither surface ever renders a credential itself.

## One-use delivery

An access link expires after 15 minutes. The bearer token is placed in the URL
fragment, so it is not sent in the landing request or ordinary access logs.
The recipient explicitly selects **Reveal access** on a standalone page. A
CSRF-protected POST consumes the grant once and shows Oduflow's UI password,
Paseo's password, the production Odoo administrator login/password, and the
Oduflow MCP bearer token. A failed or lost response cannot replay a consumed link.
Generating another link revokes all older links for that client.

Copy buttons install a selection-based clipboard fallback, because browsers
expose the asynchronous Clipboard API only in secure contexts and a control
plane opened over plain HTTP would otherwise copy nothing.

The grant stores only a SHA256 token digest and the verified credential revision.
The requesting administrator's short-lived delivery wizard stores the token
encrypted with the control plane encryption key. Its computed URL is readable
only by that administrator in an allowed company. Consumption or revocation
erases the encrypted delivery token. Credentials never enter the operation
journal, queue arguments, notification payloads, or portal projection.

The demo sets **Public Access Base URL** to
`https://headscale.example.com` in Oduflow Settings.
Traefik forwards only `/oduflow/access` and `/oduflow/access/reveal` to the private
VPN gateway. That gateway forwards only the bounded landing/reveal requests to
Control Odoo; it does not expose the control interface or pillar endpoint.
Pages disable caching, referrers, framing, indexing, and third-party resources.

## Customer self-service

A portal customer requests delivery and rotation through CSRF-protected POST
routes on the server they own. The same ownership gate as SSH keys applies:
commercial-partner ownership, allowed operating company, a portal (not observer)
user, and the instance lock re-checked after a transfer. Both actions use the
identical model workflow as the administrator buttons, including the verified
credential revision, the 15-minute single-use grant, and revocation of older
links.

Customer delivery keeps nothing in plaintext. The link is rendered once, in the
response to the customer's own request, with no encrypted delivery wizard and no
stored token; reloading the page cannot replay it. Repeated requests within five
minutes are throttled. A customer-requested rotation records that customer as
the requester, so no administrator delivery wizard is prepared on completion and
the customer requests a new link once the revision is verified. States are
exposed coarsely: available, applying, or needs attention. Applying and
unresolved states hide both customer buttons and refuse both routes, so a failed
client-side rotation is retried by an administrator rather than by the customer.

## Credential authority

The existing encrypted `oduflow.secret` bundle is the desired configuration.
Its encryption key is supplied outside the database. Client services retain
their necessary runtime copies in root-only files. Fetching plaintext credentials
back through Salt for every link is intentionally unnecessary: an unavailable
client must not become the only source of recovery data, and subsequent Salt
applications must retain the current passwords rather than restore old values.

Production Odoo receives a generated administrator password during initial
provisioning. A link requires completed configuration and production operations.
After a reset begins, links remain unavailable until the new revision is applied
and verified. Provider credentials, database passwords, GitHub credentials, and
LiteLLM management/client keys are excluded from this delivery and reset.

## Reset workflow

The confirmation queues a revisioned Salt operation and revokes old links
immediately. Four new secrets are generated together and encrypted before the
worker dispatches Salt. Durable control and master receipts prevent accidental
republication after an unknown result. A terminal failed attempt may be explicitly
retried with the same desired passwords; a new revision cannot bypass a partial
or unknown revision.

The fixed `roles.client_credentials` state verifies the client UUID, attached
filesystem, runtime configuration, production receipt, and exact production
container. It changes only the application UI/MCP passwords and production
administrator password. Oduflow and Paseo restart only when their authentication
checks require it. Production Odoo is updated through its ORM without restarting
the production container. Database passwords and data are unchanged.

A private client receipt binds the revision, request ID, desired-value HMAC, and
production container. Retries reconcile partial changes before continuing. The
state reports only a reduced success/failure result. Once all access checks pass,
the original administrator receives a fresh encrypted delivery wizard; the open
instance form shows it when its status poll observes completion. No client email
is sent automatically.

## Delivery operation

**Deliver client access** is a manual handover step. It stays Pending after
provisioning and does not block activation. Request **Access Link** in the client
form, or request credentials from the owning customer's portal, to move it to
Running. It becomes Done only when the single-use link successfully reveals the
verified credentials. Issuance alone, expired or revoked links, and failed
reveals do not prove delivery. If a link expires, request a new one.

The journal records only the first consumed grant ID and completion time, never
credentials or a bearer token. Later password resets and link requests preserve
that initial handover receipt. Completion records the server-side reveal; it
cannot prove the recipient received the HTTP response. No email is sent by this
workflow.
