# Independent managed LiteLLM gateways

The source implementation provides one independent gateway per managed host and
one exporter per source database. It does not install a common/front gateway.
`roles.litellm` uses the existing exact `client-<UUID>` identity, provider-verified
storage device and storage ownership checks. Docker/containerd data and the
SQLite ledger stay on `/srv/oduflow/data`. By default the role provisions PostgreSQL 16.15 using a pinned Docker Official
Image digest, an internal Docker network and loopback-only port 5433. PostgreSQL
data stays under `/srv/oduflow/data/litellm/postgres`. Source ownership labels and
a protected volume marker prevent adopting unknown containers, networks or data.
The addon generates and encrypts its password; no secret enters command arguments.
An explicitly selected external PostgreSQL service is also supported.

## Runtime contract

The initial adapter contract is LiteLLM **v1.100.1**, upstream commit
`1dba17b10ded12ad0021edb453ba2c54e4637928`. This release was published on 2026-09-10 and includes the patched version ranges
for the reviewed upstream host-header authentication and request-routing SSRF
advisories. Review future upstream advisories before upgrading; each supported
runtime change needs the schema/admission adapter contract tests.
Use the supplied `salt/states/litellm/files/Dockerfile` with a reviewed immutable
upstream base digest, then set the resulting image digest in Odoo. Runtime
validation requires `litellm==1.100.1` and `psycopg==3.2.10`; the exporter also pins
`psycopg[binary]==3.2.10` and `typing_extensions==4.15.0`.

Pillar `oduflow:litellm` contains:

| Field | Meaning |
| --- | --- |
| `image`, `software_version` | Immutable extended runtime image and the adapter version. |
| `config`, `desired_digest` | Authoritative structured config and SHA256 of canonical JSON with sorted keys and compact separators. |
| `environment` | Master key and only backend secrets referenced by model configuration. |
| `managed_postgres`, `postgres_password` | Default managed database with an encrypted generated password of at least 32 characters. |
| `database_url` | Managed `postgresql://oduflow:<encoded-password>@127.0.0.1:5433/litellm` or an explicitly external durable PostgreSQL DSN. |
| `source_id`, `incarnation` | Control-plane approved UUIDs, stable across process restart. |
| `ingest_url`, `ingest_secret` | Explicit HTTP(S) aggregate endpoint and independent per-source HMAC secret. Use HTTP only inside a trusted private network or VPN; use HTTPS over public networks. |
| `rating_contexts` | Historical key hash/model/context intervals, each with `key_hash`, `model_code`, `rating_context`, `period_start`, `period_end`. |
| `bind_address` | Private IPv4 listener, including Tailscale addresses; default loopback, port 4000. |
| `push_interval_minutes` | 15 (default), 30 or 60. |
| `sync_cutoff`, `sync_request_id` | Optional bounded on-demand synchronization request. |

Configuration disables database model overlays, prompt/response/error logging,
retries and fallbacks. Runtime Docker/systemd output logging is disabled so
upstream error formatting cannot persist prompts or credentials; reduced health
and metering reports provide diagnostics. Only the trusted durable admission callback is allowed.
Backend credentials in the reviewable config use `os.environ/NAME`; root-owned
mode-0600 runtime files provide their values. Configuration, environment and
export credentials are not written to command arguments or Salt results.

The supported initial surface is text chat completions through `openai/` or
`ollama_chat/` backends. Each model has explicit operational
`model_info.input_cost_per_token` and `output_cost_per_token` with positive
enforcement values, including for local models with zero incremental provider cost. The pinned upstream writer skips spend
persistence when it cannot compute a response cost. These operational costs are
separate from Odoo's own customer tariffs. The runtime refuses zero enforcement costs so local models cannot bypass
the gateway budget checks.

The helper validates configuration before activation and parses the candidate
with the actual pinned container SDK. JSON is emitted as valid YAML. A protected
revision directory snapshots configuration, secret environment, metering contexts,
listener settings and admission/exporter code. The atomic `active` symlink restores
the entire previous revision;
failed model-catalogue readiness rolls back. Both the runtime and exporter read
only active revision snapshots; on-demand sync overlays only request identity and
cutoff after validating source identity. Settings are file-authoritative and
mounted read-only. Readiness produces `/etc/oduflow-litellm/deployed.json` with
only desired digest and model count. A separately managed private ingress may
expose this independent endpoint; this role creates no public ingress.

## Durable admission and aggregation

The trusted `CustomLogger.async_pre_call_hook` writes an admission transaction
before allowing the request to proceed. It uses a new server UUID, database clock,
authenticated key hash and exactly one historical rating context. Customer-supplied
`spend_logs_metadata` is replaced. Unsupported modalities and requests without a
unique trusted context fail before forwarding. Admission errors fail closed.
Streaming admission explicitly requests the provider's terminal usage chunk;
text-only token estimates cannot account for hidden reasoning tokens.

A PostgreSQL trigger on the pinned `LiteLLM_SpendLogs` table copies only reduced
usage fields into `oduflow_metering_spool` in the same transaction as the spend
record. Installation locks the source table while backfilling and enabling the
trigger, so there is no trigger/backfill capture gap. The adapter handles Prisma's
JSON-encoded metadata strings and camel-case timestamp columns. Correlation uses
the trusted admission ID carried in the documented spend metadata field; key and
model must also match the admission row. Uncorrelated historical usage requires
explicit reconciliation; it is not guessed into a customer tariff.

The exporter polls **unacknowledged rows**, with bounded pages. It does not use a
high-water sequence cursor: PostgreSQL sequence assignment happens before commit,
so a low sequence can become visible after a higher one. Each normalized request
is committed to SQLite before the source row is acknowledged. A crash in between
replays safely through durable request deduplication. Multiple changed records for
one admission are rejected as an explicit correction, not counted twice.

SQLite uses WAL and synchronous FULL commits. It keeps source identity, immutable
normalized request digests/admission times, daily cumulative buckets and an outbox.
Bucket dimensions include source/incarnation, key hash, model, UTC interval and
rating context. Intervals split at daily and trusted context boundaries. Cached
input is subtracted from total prompt tokens; reasoning already belongs to output.
Text Responses spend records are accepted when their persisted normalized
prompt/completion counters match the usage object. Reasoning is never added a
second time; a reasoning count greater than inclusive output is retained for
provider reconciliation. Native cache-write, audio, image and video token
modalities are currently rejected. The optional
operational USD cost is rounded to 18 decimal places; customer charges use exact
integer token categories and separately versioned selling prices.

An ambiguous, still-unacknowledged OpenRouter generation can be reconciled from
its authenticated generation-statistics GET response. Publish a new root-owned
mode-0600 JSON receipt at `reconciliations/<admission-uuid>.json` beside the ledger,
using exclusive creation and retaining the receipt with the ledger backup.
The version-1 receipt contains `source_id`, `incarnation`, `admission_id`, the
SHA256 of the original canonical spool payload (`source_payload_sha256`),
`provider: "openrouter"`, `reviewed_by`, `review_reason`, `evidence`, and the SHA256
of canonical evidence (`evidence_sha256`). Set `schema` to `1`. Evidence must
identify the exact provider generation in `id` and carry its native prompt,
inclusive completion, cached and reasoning counters, zero completion-image
tokens, and `total_cost`. These values come from the provider, never token/cost
estimates or the current tariff.

The exporter verifies the original payload and all identities before overlaying
an in-memory copy. It preserves the PostgreSQL spool and admission records and
stores the full proof and its digest in SQLite `reconciliations` in the same
transaction as the normalized request and bucket. Replaying identical evidence
is idempotent; changing evidence after consumption is rejected. This facility
does not rewrite an already-consumed request or bypass unsupported modalities.

Changed snapshots are sent in pages of 200 buckets. Sending never resets counters.
The exporter persists exact batch bytes and sequence before POST. An initial
receipt does not remove the outbox: signed GET
`/oduflow/litellm/usage/receipt/<batch_uuid>` must return matching `batch_id`,
`state=processed` and the SHA256 `digest` of those bytes. Errors, lost responses,
rejections and pending receipts retain the batch. HTTPS redirects are refused.

## Completeness and cancellation

`litellm_metering.sync` executes the fixed exporter CLI using a protected request
file; it accepts no arbitrary command. Odoo's queued operation supplies the cutoff
and waits for its processed source watermark. A Salt success means the bounded
sync invocation finished, not that a customer can already be invoiced.

Admission time assignment and cutoff checking share a PostgreSQL advisory lock.
A cutoff is certifiable only when it is in the past, every pre-cutoff admission is
terminal and every captured source row is ingested. A terminal admission requires
a persisted spend row that was durably ingested locally. The watermark is attached
only after every changed bucket page that precedes it. Streaming/in-flight calls,
source write loss, ambiguous provider failures and process crashes remain pending;
the exporter never calls them zero usage. Normal successful requests close
automatically; abnormal pending admissions require reviewed source reconciliation.

Every cumulative bucket also reports `last_request_at`, the maximum trusted
admission time. Cancellation can preserve the immutable daily bucket interval
while allocating a shorter final billing period only when this timestamp is
before the cutoff, all affected key bindings are revoked/drained, and all sources
are complete through that cutoff. A bucket containing later usage requires explicit
reconciliation; aggregate totals are not arbitrarily divided.

An optional root-owned `/srv/oduflow/data/litellm/reconciliation-seal.json` permits
an externally audited seal: source/incarnation, `complete_through`, evidence SHA256
and reconciled request count must agree with the local ledger. Creating that proof
requires actual admission/provider reconciliation; a timer or successful POST is
not evidence. Keep its evidence with the billing audit.

Back up PostgreSQL admissions/spool and SQLite database/WAL/identity together with
a consistent checkpoint, preserving source incarnation. Missing local ledgers,
identity changes and count regressions fail closed. A simultaneous database/ledger
restore must be reconciled against control-plane batch receipts before restarting;
source incarnation approval is an operator action, not automatic deduplication.
Do not delete local request provenance or source spool records before the applicable
billing audit/reconciliation retention period. There is no automatic pruning in
this initial implementation.

## Verification and remaining deployment work

Local tests exercise actual SQLite crash/replay/outbox behavior and optionally a
real isolated PostgreSQL server through `LITELLM_TEST_POSTGRES_DSN`. PostgreSQL
tests cover the pinned schema's metadata representation, trigger capture,
admission context, pagination, late lower-sequence commits and cutoff blockers.
The callback test executes its real SQL with a lightweight upstream class boundary;
it does not establish a live LiteLLM streaming integration. Salt templates are
rendered with the repository's existing state-contract harness.

Before marking a new gateway verified, build/review its pinned image, apply the
Salt role including managed PostgreSQL, issue a real managed key and compare
a text and streaming request against the provider response, source admission/spool,
SQLite bucket, processed Odoo receipt and cutoff. Also test cancellation and a lost
source-write scenario against that image. Local fixtures do not prove provisioning,
public readiness, provider token semantics or live upstream callback propagation.

Pinned upstream references:

- [Spend schema](https://github.com/BerriAI/litellm/blob/1dba17b10ded12ad0021edb453ba2c54e4637928/schema.prisma)
- [Spend payload construction](https://github.com/BerriAI/litellm/blob/1dba17b10ded12ad0021edb453ba2c54e4637928/litellm/proxy/spend_tracking/spend_tracking_utils.py)
- [Admission hook and exception propagation](https://github.com/BerriAI/litellm/blob/1dba17b10ded12ad0021edb453ba2c54e4637928/litellm/proxy/utils.py)
- [Custom logger interface](https://github.com/BerriAI/litellm/blob/1dba17b10ded12ad0021edb453ba2c54e4637928/litellm/integrations/custom_logger.py)
- [Cost writer's persistence and error-log conditions](https://github.com/BerriAI/litellm/blob/1dba17b10ded12ad0021edb453ba2c54e4637928/litellm/proxy/hooks/proxy_track_cost_callback.py)

Current release/security review references:

- [LiteLLM v1.100.1 release and image signing instructions](https://github.com/BerriAI/litellm/releases/tag/v1.100.1)
- [Host-header authentication fix](https://github.com/BerriAI/litellm/security/advisories/GHSA-4xpc-pv4p-pm3w)
- [Request-routing credential exfiltration fixes](https://github.com/BerriAI/litellm/security/advisories/GHSA-3cv6-jpf6-8222)
- [Docker Official PostgreSQL image definitions](https://github.com/docker-library/official-images/blob/master/library/postgres)

## Server administration

Open **Infrastructure / Inference / Servers**. Each existing gateway record is an
independent LiteLLM server; model names and historical database identities remain
unchanged. Configure its management URL, optional client URL, allowed model aliases,
default model, reasoning effort, API mode, encrypted credentials and pinned image.
An empty allowed-model list uses the server's active backend model aliases.

Assign one platform host under Nodes. **Provision Host** prepares that host and
queues the existing provider lifecycle. After enrollment and volume verification,
**Apply with Salt** captures an immutable configuration revision and queues its
application. Metering source authentication and backend costs remain prerequisites.
The configuration history retains the reduced Salt receipts and reconciliation state.
Independent servers have independent hosts, secrets and configuration histories.

New clients use the default server from Settings, scoped to its company. The client
form can select another server before plan preparation; the selection is fixed
thereafter. The upgrade imports the old singleton endpoint and management credential
without recreating client keys or modifying provisioning snapshots and dispatch receipts.
The old configuration parameter names remain for backward compatibility.

**Odoo VPN SOCKS Gateway** is shared outbound access from Odoo to inference servers
in the Headscale-managed network. **Trusted Pillar Proxy Host** is the inbound
proxy's internal DNS name: Odoo resolves it and checks the direct connection peer,
in addition to the pillar bearer token. The deployed hostname
`oduflow-1-svc-oduflow-vpn` identifies the Megaflow `oduflow-vpn` service, not an
inference server or a public pillar URL.
