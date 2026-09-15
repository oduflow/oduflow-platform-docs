# Oduflow access roles

**Oduflow Observer** can read instances, plans, operation journals, reserved
hostnames and the Vultr server-plan catalogue within allowed companies.
The role grants no create, write or delete permission. Mutating public actions
also enforce the Admin role on the server, including direct RPC calls.
Observers cannot read encrypted secret records, durable dispatch records or
deletion-confirmation wizards.

**Oduflow Admin** includes Observer access and retains the existing management
permissions. The existing XML identifier `oduflow.group_manager` is preserved
so current administrators and references remain valid. Odoo groups are additive:
granting Admin together with Observer gives Admin permissions.

## Client client deletion

**Delete Client** opens a separate confirmation dialog. Opening the dialog
does not schedule or perform deletion. The administrator must enter their own
current Odoo login password for every confirmation. Another administrator's
password, a client Odoo database password, an API key or a context flag is not
accepted as confirmation.

The wizard follows Odoo 19's nonstored password-field pattern. The button passes
the entered password as an untrusted request input; the server always verifies
it with `res.users._check_credentials` in interactive password mode. The returned
user ID and authentication method must match the current user and `password`.
Odoo's authentication cooldown is applied. The password has no database column
and is removed from context before entering deletion or creating a queue job.

The wizard is owned by its creator and its instance target cannot be changed.
The actual deletion entrypoint is private and cannot be called through Odoo RPC.
All provider ownership and company-access checks still run after password
verification. The queued workflow revokes access links, the LLM key, public DNS,
Headscale and Salt identities before deleting the VM, volume and GitHub repository.
Every external identity is re-read and matched to the immutable instance UUID or
recorded resource ID before mutation. A later read must confirm absence or expiry;
only then are encrypted client credentials erased and the client marked destroyed.
Operation journals and hostname reservations remain available for audit.

The API contract was checked against installed Odoo 19 source:
`odoo/addons/base/models/res_users.py` (`_check_credentials`, `_assert_can_auth`,
`ResUsersIdentitycheck`) and `odoo/service/model.py` (`get_public_method`).
