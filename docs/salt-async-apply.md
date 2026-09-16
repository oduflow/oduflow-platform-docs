# Asynchronous client configuration

Control Odoo dispatches managed client work through restricted start/status
runners. Configuration, production, resize, credential and custom operations have
separate profiles. Each request binds an exact `client-<UUID>`, canonical request
UUID, profile and, for release-aware clients, the selected immutable client SHA.
The profile-to-role table and receipt formats live in the
[durable execution protocol](salt-result-recovery.md).

## Dispatch and status

The controller first checks `oduflow.ping(minion_id)` and complete authenticated
pillar. For configuration it calls
`oduflow.apply_start(minion_id, request_id, client_revision)`; production uses
`oduflow.production_start` with the same identities. Subsequent status calls use
the original minion/request pair and matching profile endpoint.

The master commits a private claim before launching its worker and records the
Salt JID and dispatch intent before publication. The client receives the reviewed
`oduflow_job.run` wrapper, which applies the fixed profile from the selected
release and commits a reduced result before returning it. It does not accept an
arbitrary RPC function, target pattern, role or pillar override. Validated custom
high data uses its own bounded, digest-bound path.

`control_master` installs `/usr/local/libexec/oduflow-apply-worker`. Master claims
live in `/var/cache/salt/oduflow-apply`, owned by the Salt account with directory
mode `0700` and file mode `0600`. Global job/pillar caches stay disabled. Raw
state returns and secret pillar are not the operation history.

The public summary includes minion/request/JID, status and state counts, plus the
revision or source digest when required by the protocol:

| Status | Interpretation |
| --- | --- |
| `not_started` | No matching claim/result at that boundary; not permission to republish an already recorded dispatch |
| `running` | The request is launching or still has matching live execution evidence |
| `succeeded` | A valid, nonempty state mapping has every result exactly `True` |
| `failed` | A valid state mapping includes `False`, or a confirmed preflight failure prevented dispatch |
| `unknown` | Transport/execution evidence is missing, invalid or interrupted; possible effects must be reconciled |

Empty mappings, compiler-error lists and non-boolean results are not successful
or trustworthy failed-state receipts. Optional diagnostics explain classifications
but do not override the durable result.

## Concurrency and time limits

Profiles share a per-client execution lease. A worker can wait up to 60 seconds
for it; a busy or quarantined target fails preflight without publishing Salt work.
Execution waits up to 7,200 seconds, with a 7,250-second hard deadline and bounded
status polling. The dispatch timestamp separates lease waiting from execution.

A worker/master restart can lose the response while the minion continues applying.
Keep the original request and JID and retrieve its exact terminal receipt.
A trustworthy late result may resolve an unknown status. Exiting the master
worker does not establish that the client stopped executing.

## Recovery and rollout

Update the master executor and allowlist before a controller uses a new protocol;
capability checks reject clients that cannot produce the required receipt. Old
receipts retain their original format and interpretation. Client software updates
follow [client releases](client-releases.md).

For a failed or unknown configuration, use [deployment recovery](deployment-recovery-ui.md).
A repeated status read does not run configuration again. The explicit recovery
workflow can link a new configuration generation after ownership and inactive
execution checks, preserving the old result. General job retry or receipt deletion
cannot substitute for that workflow.

Configuration checks local application health. Production separately guards
creation, hardens the administrator account and verifies public access. Neither
profile proves that a real AI coding task completed.
