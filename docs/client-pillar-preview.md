# Client pillar preview

Open an instance in Odoo and select the **Pillar** tab. On a draft, enable
**Advanced Settings** to reveal the tabs. **Refresh** regenerates the preview.

The YAML comes from the complete `_pillar()` method chain used by the Odoo Salt
gateway, including installed provider/integration extensions and the selected
configuration revision. It shows the current Odoo-generated input, not a readback
from Salt or proof that the client applied it. Salt state defaults which are not
pillar values do not appear in this document.

Only explicitly classified public scalar paths are displayed. Passwords, tokens,
private keys, arbitrary environment variable values and unclassified new values
are replaced with `<redacted>`. Mapping and list structure is preserved. Even a
public URL is hidden if it contains user information, a query or fragment.
Integrations can extend `_pillar_preview_public_paths()` for reviewed public
values. No unredacted reveal or editing is provided.

The preview is available to Oduflow observers and administrators within their
allowed companies. It is computed on demand and is not stored in the database,
attachments or chatter. Refresh does not change configuration or dispatch Salt.

The real builder requires provisioning, active, suspending or resuming state and
complete provider configuration. In other states, or when generation fails, the
tab displays an availability message and no partial YAML. Provider exception
details are not displayed. The ordinary pillar endpoint and provisioning checks
remain unchanged.
