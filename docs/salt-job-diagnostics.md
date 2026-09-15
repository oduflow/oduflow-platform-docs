# Salt job diagnostics and reconciliation

The client executor writes optional `diagnostic-<request UUID>.json` files beside
its durable request/JID receipts. They use the same private directory, atomic
replacement, fsync and mode 0600. The envelope binds the minion, request, JID,
profile, state digest and client revision (or source digest). Schema 1 contains
only fixed phase/reason/error classifications, timestamps and bounded source
locations from the two client execution modules. Exception messages, locals,
state IDs, pillar, command output and raw Salt returns are excluded.

`oduflow_job.diagnostics` reads this separate surface without changing a receipt.
The master reads it during receipt polling and accepts only an exact binding and
schema. Old clients without this method or sidecar remain supported. Odoo applies
its own schema validation and displays accepted configuration diagnostics on the
Operations tab. Invalid optional evidence is ignored; unavailable diagnostics
never prevent adoption of a valid terminal receipt. Validator parity is checked
across the separately deployed client, master and control packages.

The phases distinguish executor setup, release preparation, state execution,
result reduction and result persistence. `invalid_result` means Salt returned a
shape the strict reducer cannot interpret. `states_failed` means a valid state
result contained failures. A lost worker can leave an `in_progress` diagnostic;
only the process lock and receipt determine whether it is still running.

## Reconciliation rules

1. Match the instance UUID, dispatch generation/request UUID, JID, profile and
   selected revision against the Odoo dispatch claim, master record and minion
   receipt. A different binding is never adopted.
2. If the request process still owns its lock, keep polling. A free lock does not
   prove that no changes occurred.
3. Adopt an exact terminal minion receipt through the normal status path. Preserve
   the original request and JID. Never issue another publication to retrieve a
   lost response.
4. If the result remains unknown, inspect the expected VM and owned block volume,
   mounts and application services with read-only checks. Do not format a volume,
   recreate resources, clear the claim or mark success based on a diagnostic.
5. An absent filesystem or service proves its current state, not the absence of
   all prior effects. Without adequate evidence, retain `uncertain`. A recovery
   workflow must explicitly account for possible partial effects and preserve the
   old receipt before it can authorize a new generation.

Legacy exceptions cannot be recovered retroactively from a newly installed
executor. Updates to the receipt executor do not replay old requests.

An administrator can now authorize a separate, linked configuration recovery
through the [deployment interface](deployment-recovery-ui.md). That workflow
retains the unknown receipt, verifies ownership and inactive execution, and
reapplies the guarded configuration role with a new request identity. It does
not convert diagnostics into proof of historical success.
