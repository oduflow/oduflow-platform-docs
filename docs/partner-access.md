# Partner companies and employee access

An `oduflow.partner` agreement references a top-level `res.partner` company:
`is_company=True`, no `parent_id`, and `type=contact`. Individuals, subsidiary
companies and invoice/delivery addresses cannot serve as the partner company.
The form filters eligible records and defaults newly created records to a
company. ORM validation also covers imports, RPC, and later changes to the
company's type or parent. One company can have one active partner agreement.

Employees are personal contacts beneath this company, including nested personal
contacts resolved through `commercial_partner_id`. A subsidiary company starts
its own commercial boundary and does not inherit the parent's agreement.
Users need the explicit Oduflow Partner Portal role; creating a contact alone
does not grant a login or permissions. Only active contact records of type
`contact` qualify. The selected operating company must also be allowed for the
user. The current release retains the portal interface while the internal
partner workspace is being designed.

User membership is computed from the contact hierarchy and cannot be assigned
manually. Members share the company's assigned customers, leads, instance
summaries and repository bundle. Customer contacts inherit the customer
company's partner assignment through Odoo's commercial fields. Ownership rules
resolve membership through current database relations instead of embedding
company IDs in cached rule domains. Additional group rules cannot expand
access past the global partner boundary.

An Oduflow administrator manages the hierarchy. Moving a contact changes its
access to the new company's records; detaching it removes partner access.
Archiving the agreement or its company prevents further workspace access.
Archiving the agreement also queues repository key revocation. Archiving just
the company does not revoke previously downloaded GitHub keys. A downloaded
key cannot be recalled by changing a user's company: the current keys belong
to the partner organization. Revoke the agreement's keys when those credentials
must stop working. Personal service accounts and per-specialist revocation
remain separate from this company membership change.

Upgrade validation rejects existing agreements referencing individuals,
duplicate active company agreements, and manual user assignments inconsistent
with the contact hierarchy. It never silently reparents users or changes
contact types. Correct those records before upgrading another installation.

## Downloading platform source access

Creating an Oduflow partner agreement generates a distinct Ed25519 key pair for
`oduflow/oduflow-platform` in the creation transaction, encrypts its private key,
and queues registration as a read-only GitHub deploy key. Each partner company
has its own key; downloads reuse it instead of generating another credential.

After GitHub confirms the platform key, **Download platform SSH key** appears
in the partner workspace (`/my/partner`) and the administrator's partner form.
It downloads `oduflow-platform-access.zip`, containing the private key, a scoped
SSH config, pinned GitHub host keys and a README with clone/checkout commands.
Extract the included directory into `~/.ssh/` and follow the README. No shared
GitHub token is included. IDE or Client repository-grant failures do not block downloading
an already-active platform key. The full three-repository archive remains
available when all grants are active.

Portal downloads require current company membership and a CSRF-protected POST.
The administrator download checks the Oduflow Admin role, current record access,
company scope and key readiness again when serving the file. Responses use
attachment disposition and `private, no-store` caching headers.
