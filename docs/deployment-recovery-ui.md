# Deployment errors and recovery in the interface

Open the instance and select **Operations**. Each row represents a deployment
step. **Details** opens its current error classification, execution stage,
timestamps and **Attempt History**. Open an attempt for its software revision,
request/JID, requester, recovery reason and safe executor diagnostics.

The history keeps the earlier uncertain attempt when recovery creates a new one.
Diagnostics exclude exception messages, command output, pillar and raw Salt
returns. An older executor can have no detailed diagnostic record; the interface
states that limitation instead of inventing an error. This is an execution
journal, not a raw terminal log viewer.

## Retry a confirmed configuration failure

Use **Retry** on a failed **Apply client configuration** step. The button queues
checks and returns immediately. The worker verifies current infrastructure
ownership, credentials, volume attachment and the exact previous dispatch result.
A running job, missing receipt after dispatch or conflicting identity blocks the
retry. If a previously lost successful result has appeared, it is adopted rather
than executing configuration again.

## Recover an unknown configuration result

Use **Recover** when the configuration step says **Reconciliation required**.
Enter a reason, confirm that partial configuration may already exist and retain
the selected revision or provide a reviewed repair commit SHA. **Queue Recovery**
records the requester and enqueues the checks; a second click cannot create a
parallel recovery request.

Odoo creates a new generation only after checking the previous receipt and live
provider ownership. The new request points to the earlier request; the earlier
receipt remains unchanged. The master updates the two reviewed executor modules,
checks that the earlier configure attempt is inactive, and blocks unrelated
unresolved jobs. Before publishing a pinned configuration release, it refreshes
the minion's in-memory pillar and verifies the instance UUID and absence of pillar
errors. This prevents an earlier enrollment error from surviving into the new
execution worker. The minion repeats the old-receipt check while holding its
execution lease and persists the recovery link before running the fixed
configuration role. Replaying the new publication returns its recorded result.

Recovery reuses the existing VM, volume, repository and DNS records. The fixed
configuration role keeps its normal storage guards: an owned existing filesystem
must match its recorded UUID, and a blank volume requires the original verified
formatting authorization. Recovery does not bypass those guards or pretend the
old attempt succeeded. Installation/configuration can be reapplied and services
may restart. Production creation remains its separate deployment step.

**Check Status** queues another status read. It does not restart a job. A failed
recovery check appears in **Recovery Check Error**; the original attempt remains
available. If a new attempt fails, open its details before retrying again.

Only Oduflow Admin can retry, recover or queue status checks. Observers can read
details and history. These controls currently cover client configuration; other
steps retain their existing provider-specific actions, including the existing
repository retry. Arbitrary provider creation and deletion are not replayed by a
generic queue-job retry button.
