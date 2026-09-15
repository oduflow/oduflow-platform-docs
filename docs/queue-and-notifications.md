# Asynchronous operations and notifications

The OCA `queue_job` addon is pinned in `.oduflow/requirements.txt` and declared
as a core addon dependency. `.oduflow/odoo.conf` enables `base,web,queue_job`,
two HTTP workers, one cron thread, and the `root:2,root.oduflow:1` channels.
The serial provisioning channel limits concurrent control-plane dispatches;
long Salt operations run in detached workers and are polled by short jobs.

This file is a configuration overlay. Preserve database connection settings,
addon paths, and encryption/provider environment variables when applying it.
Use Megaflow `update_environment`, inspect the effective `/etc/odoo/odoo.conf`,
and merge this overlay into that file if the environment does not load it.
Restart and verify a real queued task reaches `done`; installing the addon alone
does not prove the runner is active. Never manually mark a failed job as done to
make the client appear ready.

The Client Instance form checks operation state every five seconds while it is
visible. It shows progress and a notification when an observed operation finishes
or requires attention. Refresh preserves unsaved edits and resumes after saving
or discarding them. Reopening a form does not replay old notifications. The
persistent operation journal is the completion/error history when the browser
was closed; browser notifications are not an email delivery guarantee.

`oduflow.mail_template_client_ready` is an editable client handover email template.
It includes the three client application URLs and no passwords or tokens. No
automatic email dispatch is configured. Review actual readiness and recipients
before explicitly sending it; deliver credentials separately.

External actions retain their durable dispatch receipts and reconciliation rules.
A browser refresh, worker retry, or repeated click must not create duplicate cloud
resources. A terminal queue job is not automatically equivalent to a successfully
completed provisioning operation: inspect the journal and provider/Salt evidence.

Reference: [OCA queue_job 19 configuration](https://github.com/OCA/queue/tree/19.0/queue_job#configuration).
