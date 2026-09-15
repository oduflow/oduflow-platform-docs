# Salt result recovery with durable minion receipts

The repository implements protocol 1 in the `oduflow_job` minion execution
module and the `oduflow` master runner. Deployment and live verification must
be reported separately: local protocol tests do not prove that an existing
client has received the module or survived a real transport failure.

This protocol addresses a completed state result lost during Salt publication,
including master IPC failures. It does not recreate receipts for older jobs.
Existing ambiguous legacy dispatches still require explicit reconciliation and
must never trigger another automatic apply.

## Execution and identity

The master persists its request UUID, exact client identity, role/profile and
20-digit Salt JID before publication. New requests require the minion's protocol
capability; failure to verify it stops dispatch. The master publishes
`oduflow_job.run(request_id, profile, state_data_json)` once, using that JID.
It records the dispatch intent before calling Salt, so a lost acknowledgement
cannot authorize another publication.

The wrapper receives the actual JID through Salt's `__pub_jid` metadata. In
pinned Salt **3006.27**, `load_args_and_kwargs` injects publication metadata
*after* parsing caller arguments, replacing any caller-supplied `__pub_jid`.
[Upstream argument binding](https://github.com/saltstack/salt/blob/v3006.27/salt/minion.py#L379).

The minion validates a canonical `client-<UUID>` identity, canonical request UUID,
exact JID and profile. Supported profiles map to fixed roles:

| Profile | Role |
| --- | --- |
| `configure` | `roles.client_stack` |
| `production` | `roles.client_production` |
| `volume_resize` | `roles.client_storage_resize` |
| `credentials` | `roles.client_credentials` |
| `custom` | Canonical bounded high data, bound by SHA-256 digest |

The wrapper rejects caller execution options. Fixed-role calls set `test=False`,
`queue=False`, `concurrent=False` and `state_events=False`. Salt's `state.apply`
with a role delegates synchronously to `state.sls`; `state.high` also returns its
result synchronously. A compiler/conflict list, exception, empty mapping or
non-boolean state result cannot become a successful receipt.
[Upstream state execution](https://github.com/saltstack/salt/blob/v3006.27/salt/modules/state.py#L637).

## Durable state machine

Receipts live under `/var/lib/oduflow/job-receipts`, owned by root with mode
`0700`; each file has mode `0600`. Directory traversal and file opens refuse
symlinks, unsafe ownership/permissions and multiply linked files. JSON writes
use an exclusive temporary file, file `fsync`, atomic rename and directory
`fsync`.

A nonblocking execution lock serializes managed applies on one minion. A
separate request lock distinguishes a living worker from an abandoned claim.
The JID index and running request record are committed before calling the state
function. These are two separate atomic writes: an interruption between them
leaves a JID index without a request record and occurs before any state work. An incomplete claim for the same publication remains unknown and
cannot execute again. A request is never recovered by assigning a new JID. The minion does not scan
unrelated JID indexes to discover this incomplete binding; the master must
retain its original JID even when only the first write survived.

After state execution, the wrapper reduces its result and commits this schema
before returning to Salt:

```json
{
  "protocol": 1,
  "minion_id": "client-<canonical UUID>",
  "request_id": "<canonical UUID>",
  "jid": "<exact 20-digit Salt JID>",
  "profile": "configure",
  "state_digest": "",
  "status": "succeeded",
  "passed": 60,
  "total": 60
}
```

Success requires a nonempty state mapping whose results are all exactly `True`.
A valid mapping containing `False` produces `failed`; unstructured results and
exceptions produce `unknown`. No state names, comments, changes, commands,
stdout/stderr, pillar or credentials enter the reduced receipt or return.

| Failure boundary | Recovery |
| --- | --- |
| No matching local evidence | `not_started`; this does not authorize redispatch of a published master claim |
| Partial claim or worker killed before terminal commit | `unknown`; no automatic replay |
| Worker running with matching request lock | `running` |
| Terminal commit completed, publication lost | Exact terminal receipt remains readable |
| Identity, request, JID, profile or digest conflicts | Reject the receipt |
| Receipt inaccessible or malformed | Remain unresolved |

The master polls `oduflow_job.status` only on the expected authenticated
minion, validating the complete schema and exact immutable binding. Network I/O
occurs outside the master record lock; after reacquiring it, a poll preserves
any terminal result already committed by another worker. A target with an
unresolved published request blocks another managed dispatch.

## Why a returner is insufficient

Salt 3006.27 calls `_return_pub` before normal custom returners in both its
single-function and multiple-function paths. An unexpected publication exception
can prevent a returner from running. This wrapper instead commits inside the
execution function, before control reaches publication.
[Single-function ordering](https://github.com/saltstack/salt/blob/v3006.27/salt/minion.py#L2871),
[multiple-function ordering](https://github.com/saltstack/salt/blob/v3006.27/salt/minion.py#L3023).

## Raw-result suppression and runtime boundary

Global job and pillar caches remain disabled. These settings alone are
insufficient: pinned `state.sls` writes `sls.p` and compiled `*.cache.p` files
regardless of `cache_jobs=False` or the call's cache-read option. The wrapper
intercepts those writes into disposable in-memory streams within its isolated
minion worker and refuses reads through that cache path.
[Unconditional state caches](https://github.com/saltstack/salt/blob/v3006.27/salt/modules/state.py#L1632).

The wrapper also suppresses `State.event`, since individual `fire_event` options
can otherwise publish raw state returns even with `state_events=False`.
These shims require exactly Salt 3006.27 and `multiprocessing=True`; they are
restored before returning and do not affect other minion workers.
[Per-state event behavior](https://github.com/saltstack/salt/blob/v3006.27/salt/state.py#L3124).

Per-state parallel execution is refused. Salt otherwise writes raw parallel
returns below `<cachedir>/<invocation_id>/<state-tag-hash>`, outside the two
ordinary state cache files. Custom payloads reject any explicit `parallel`
option, and the isolated worker guards `State.call_parallel` before it can spawn.
[Parallel result persistence](https://github.com/saltstack/salt/blob/v3006.27/salt/state.py#L2265).

The receipt is evidence about the synchronous state result, not current service
health. Administrator-authored custom states can start background work or call
external services; a successful outer state does not prove that such work has
finished. Application and infrastructure verification remains separate.

## Bootstrap and verification

Cloud-init carries the module as a reviewed bootstrap asset and installs it in
the minion extension-module directory before starting the managed minion. Salt's
`client_receipts` state keeps that file managed afterward. Capability checking
prevents a new request from silently falling back to unreceipted execution on an
older client. Legacy master records retain their original interpretation.

`tests/test_minion_receipts.py` covers reduced schema, identity/role restrictions,
private storage, duplicate execution and pinned-Salt loader/state integration.
`tests/test_salt_receipt_protocol.py` independently uses real process forks,
`SIGKILL`, file locks and a local side-effect marker to verify interruption before
and after terminal commit, lost publication, concurrent polls and duplicate
workers. These tests do not publish network jobs or perform live provisioning.
Master runner and bootstrap tests verify their respective delivery contracts.

Salt's older mutable `sls.p` artifact is not an immutable JID-bound receipt.
Never adopt it automatically or expose its raw contents in an operation journal.
Missing evidence remains unknown even when timestamps or state counts appear
plausible.
