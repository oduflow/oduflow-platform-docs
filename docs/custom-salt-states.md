# Custom Salt states

Platform administrators can create **Oduflow → Salt States**, edit plain YAML in the highlighted editor, and validate it without executing commands. Assign a state on a server's **Custom Salt States** tab, then click **Apply**. The server must have completed its initial Salt configuration. Observers can read assignments and execution status within their allowed companies, but cannot read source, edit definitions, or execute states.

```yaml
check_custom_state:
  test.nop:
    - name: Custom Salt state is ready
```

This first version accepts self-contained YAML with full Salt function names and lists of single-key arguments. Jinja, renderer directives, YAML anchors/aliases, duplicate keys, and include/extend/exclude are rejected. Use separate state IDs for functions from the same Salt module. Source is limited to 64 KiB, 100 state IDs, and bounded nesting. YAML syntax validation does not establish that a particular Salt module, argument, package, or path exists on a client.

Administrators intentionally have Salt execution privileges through this feature. Keep passwords and tokens out of source: definitions, immutable execution snapshots, and private master receipts persist it. The execution journal and status API retain sanitized counts and error codes instead of raw Salt output. State functions themselves may produce files or logs on the client according to their arguments.

Saving changed source increments its revision. Applying freezes the source, revision, semantic SHA-256 digest, server, and request UUID. Later edits do not change a queued or running execution. Repeated **Apply** clicks reuse the current execution; applying an already successful identical digest also reuses its result. **Apply Again** explicitly requests another execution after a known terminal result. After failure, a new attempt requires the master to confirm the previous failed receipt. Old receipts and run records remain intact.

Jobs use the existing asynchronous queue and private authenticated Salt transport. `oduflow.custom_start` binds a canonical highstate payload to the request before starting a worker; `custom_status` exposes only target/request/job identity, status, and counts. The worker converts dotted declarations structurally and invokes fixed `state.high`; no template renderer runs in Odoo or as part of this source delivery. No arbitrary RPC function or role parameter is accepted. Configure, production, resize, and custom profiles share a per-client execution lease. A worker waits at most 60 seconds for this lease. A busy target or unresolved earlier dispatch produces a failed preflight receipt (zero passed out of one preflight check), with no Salt command published. An explicit retry creates a new request after the obstruction is resolved. At actual dispatch the worker records the dispatch timestamp and resets its 7,250-second hard deadline; Salt execution waits at most 7,200 seconds. Status expiry uses the dispatch timestamp, and the 250-poll budget covers waiting plus execution.

A lost response is reconciled by the same request UUID. An ambiguous durable claim never causes automatic re-execution. **Resume Status Check** continues polling the existing request; it does not create a replacement. If the master cannot prove the outcome, the run remains uncertain and blocks a new custom execution on that server until the outcome is reconciled operationally.

An unknown dispatched receipt quarantines the entire target across all profiles, even if the worker process has exited and its OS lease is free. A still-running minion job can outlive its worker. Do not delete that receipt or manufacture success to release the quarantine. An operator must establish the original job's actual terminal outcome and reconcile the private master receipt; no public reset or arbitrary Salt RPC is exposed. Confirmed successful and failed receipts do not quarantine a target.
