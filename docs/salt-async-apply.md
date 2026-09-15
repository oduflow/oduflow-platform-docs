# Asynchronous client configuration

The controller first calls `oduflow.ping(minion_id)`. Once the exact accepted
client is reachable and its full pillar is ready, it generates a canonical request
UUID and calls `oduflow.apply_start(minion_id, request_id)`. Both identifiers are
required on every subsequent `oduflow.apply_status` call.

The only published command is `state.apply roles.client_stack`, targeting one
canonical `client-UUID` with Salt's list matcher. No state name, function,
arguments, pillar override, or cache path comes from the API caller.

Production uses separate `oduflow.production_start(minion_id, request_id)` and
`oduflow.production_status(minion_id, request_id)` endpoints with the same
response contract. Their only role is `roles.client_production`. New claims record
both a profile (`configure` or `production`) and its fixed role before starting
the worker. A request UUID already bound to one profile cannot be reused or read
through the other profile's endpoint. Legacy receipts without profile fields
remain configuration jobs exclusively. The worker validates this persisted pair
against its internal whitelist; its command line still accepts identities only.

`control_master` installs `/usr/local/libexec/oduflow-apply-worker` and creates
`/var/cache/salt/oduflow-apply`, owned by the Salt service account with mode 0700.
The worker runs as that account. Its arguments contain only the two identities.
The runner durably writes a mode 0600 claim before launching the worker; the worker
records dispatch before publishing. Repeated start calls never publish again.
The master retains `job_cache: false`, `pillar_cache: false` and no external job
cache. Only the sanitized receipt is persisted, not raw Salt returns or pillar.

The response fields are `minion_id`, `request_id`, `jid`, `status`, `passed`,
`total`. `jid` is a preassigned Salt job identifier, or null for `not_started`.
States:

- `not_started`: no master claim exists.
- `running`: a claim is launching or its worker holds the execution lease.
- `succeeded`: a nonempty Salt state return has every result exactly true.
- `failed`: a nonempty state return contains at least one result that is not true.
- `unknown`: launch/transport failed, no trustworthy result returned, or the
  worker disappeared. Do not create a new request automatically.

Worker execution waits up to 7200 seconds, with a process alarm at 7250 seconds.
Status detects an abandoned lease after a 30-second launch grace period; after 7260
seconds it reports unknown even if the process still holds a lease. A late
trustworthy return can replace unknown with its final summary. A service/master
restart can kill the detached process while the minion continues applying; this
intentionally leaves an unknown receipt requiring explicit reconciliation.
A per-client worker lock prevents concurrent publication by different requests.

Apply the updated `control_master` state before deploying controller code that
calls the new endpoints. The eauth allowlist explicitly includes only these
fixed runners. Configuration does not create production or publish its HTTP
ports. The separate production role owns guarded creation, hardening and
publication; both profiles retain the same at-most-once dispatch behavior.
