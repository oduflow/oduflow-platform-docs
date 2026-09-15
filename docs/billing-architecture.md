# Oduflow billing and managed LiteLLM architecture

Status: implementation architecture. All five addons are present in this repository.
See addon READMEs and `litellm-managed.md` for current setup and runtime limits;
installation and unit/integration tests do not prove a live gateway or payment trial.

## Design decisions

- Sell server service and local-model inference using our own versioned token prices.
- Manage LiteLLM through a dedicated `oduflow_litellm` addon and Salt.
- Support multiple logical gateways immediately and multiple replicas later.
- Keep request-level metering outside Odoo; deliver compact aggregate snapshots.
- Use Odoo 19 Community sales, invoices and payments with our own subscription
  scheduler. Do not depend on Enterprise or `sale_subscription`.
- Expose orders, subscriptions, usage and invoices in the customer portal.

## Modules

```text
oduflow
  ├── oduflow_litellm
  └── oduflow_billing + sale/account/payment/portal
        ├── oduflow_billing_litellm + oduflow_litellm
        ├── oduflow_billing_stripe + payment_stripe
        └── oduflow_billing_paddle
```

`oduflow_litellm` owns gateway deployment, model routing, configuration revisions,
keys and aggregate metering. It works independently of commercial billing.

`oduflow_billing` owns subscriptions, tariffs, service periods, renewal orders,
rated charges, invoice generation and entitlement policy. It reuses standard
Odoo payment and accounting objects rather than creating parallel financial models.

The small `oduflow_billing_litellm` bridge assigns usage to subscriptions, applies
sale rates and translates entitlements into gateway policy. This dependency
layout keeps fleet management usable without billing and avoids a cycle.

The payment addons implement collection and reconciliation. Each subscription
and charge has exactly one collection owner. Portal extensions can live in the
billing addons; a separate portal addon is optional packaging.

## Managed LiteLLM fleet

Use new models under `oduflow.litellm.*` and preserve existing model names.

| Model | Responsibility |
| --- | --- |
| `gateway` | Logical API endpoint, authentication/metering domain, stable UUID, purpose and pinned software version. |
| `node` | One runtime replica with managed host/minion reference, health and deployed revision. |
| `model` | Stable customer-visible model identity, capabilities and supported metering categories. |
| `deployment` | Mapping to a backend model/provider/URL, secret reference and routing/capacity settings. |
| `config.revision` | Validated immutable desired configuration, digest, rollout and verification state. |
| `key` / `key.binding` | Logical credential and historical per-gateway verified identity, ownership and access policy. |
| `usage.bucket` | Cumulative quantities for a bounded interval and dimensional key. |
| `usage.batch` | Compact import receipt: source, sequence, digest, processing status. |
| `usage.source` | Stable metering source identity, approved incarnation, cursor and completeness watermark. |

Start with separate `local` and `external` gateways, each with one node. LiteLLM
routes to local vLLM/Ollama-compatible inference services or external APIs; it
does not itself run model weights. GPU provisioning can use separate Salt roles.

Replicas of one gateway will share authentication and coordination state under
a verified deployment contract and serve a stable front endpoint. Independent
gateways remain separate security domains; initially issue separate physical
keys and associate them with one logical customer identity. Do not assume that
a key created on one gateway automatically works on another.

Distinguish balancing gateway replicas from routing a model among inference
backends. Clients connect directly to their assigned gateways. No common/front gateway
is planned. Each independent gateway reports its own customer usage exactly once.
Replica addition must not duplicate consumption by adding a second exporter
reading the same source ledger. Model fallback must preserve the sold product
semantics or follow an explicit alternative-pricing rule.

## Salt configuration and deployment

Odoo holds structured desired configuration. Render a reviewable `config.yaml`
with model_list, routing and runtime settings; Salt installs it. Keep a restricted
advanced-settings surface for supported options, not arbitrary executable callbacks.
Secrets are references in configuration previews/Git and protected files or
secret-backed environment values at runtime. Do not log configuration secrets.

Keep model/runtime configuration file-authoritative: disable competing LiteLLM
UI/API writes and verify database overlays do not override Salt's configuration.
Use `roles.litellm` to prepare verified storage, configure the gateway and
install its durable metering service. Pin images and dependencies. Provision
persistent database/exporter storage and backups; add Redis when the chosen
coordination/routing setup requires it.

Rollout: validate revision, render candidate, validate against the pinned runtime,
apply atomically, use its verified reload/restart mechanism, check readiness and
effective model catalogue, then record the deployed digest. Preserve the previous
working revision. Later, drain and update replicas one at a time for streaming.

Use a platform-host provisioning profile without customer production setup or
automatic billing. Preserve exact `client-<UUID>` Salt targeting for managed
hosts and verify assigned platform role and ownership. Do not relax runner
validation to globs for service hosts. Provider-independent infrastructure
interfaces create the host; dedicated Salt roles configure it. Management traffic
uses the private network and existing durable queued operation patterns.

## Own prices for local models

Token rates are required in the first release, independently of upstream cost:

```text
charge = sum(category_tokens * category_price_per_million / 1_000_000)
```

Version prices by commercial model and supported category: uncached input,
output, cached input and cache writes as available. Normalize overlapping totals:
reasoning tokens may already belong to output; cached input may already belong
to total input. Unsupported modalities must be excluded or explicitly priced.
A technical alias maps to a stable commercial model identity; changing its
physical backend must not silently change the quoted price.

Use exact integer counters (64-bit where necessary), decimal rates and precise
intermediate amounts. Round once per invoice pricing group with currency rules.
Test Odoo quantity/UoM and product-price precision for small token charges; keep
exact counts and rates on linked billing lines even if invoice quantities are
shown in millions. Two-decimal defaults must not turn small usage into zero.

A local model with no upstream per-token invoice can still have a nonzero selling
price. Track hardware/provider cost separately. A cost-multiplier tariff remains
an optional method for external models, not the local-resale default.

## Aggregate metering protocol

Run a small durable metering/exporter service beside LiteLLM. It consumes
persisted, version-tested usage records or a durable metering hook. Keep normalized
request identities locally for deduplication and billing audit without prompts
or completions. In-memory counters and best-effort callbacks alone are insufficient.
For replicas sharing one ledger, consume it once using transactional cursor and
leader coordination. Each independent ledger has its own source identity.

Separate sending frequency from Odoo storage granularity:

- Default push interval: 15 minutes, configurable to 30 or 60.
- Send changed cumulative daily UTC buckets, updating existing open Odoo rows.
- Split daily buckets at tariff, subscription-period, ownership and metering-schema
  boundaries. Do not combine different rate contexts into an irreversible total.
- Skip empty buckets. Keep fine-grained diagnostics at the source.

Bucket identity includes source/incarnation, key binding, model, interval bounds,
rating context and metering schema. Store categorized token counts, request count,
optional source cost/provenance, revision and completeness state. Include any
nonlinear rate dimension, such as context-length tier, before aggregation.

For example, 1,000 keys using four models daily generate approximately 120,000
buckets per 30-day month before extra split dimensions. Sending 96 updates per
day does not create 96 times as many usage rows. This still requires measured
capacity, indexes and archival with preserved invoice audit references.

Private endpoint: `POST /oduflow/litellm/usage`. This exporter protocol
is custom work, not a promised built-in LiteLLM reporting feature. Payloads carry
protocol version, source UUID/incarnation, batch UUID/sequence, source cursor,
bucket snapshots/revisions and completeness watermark. Authenticate the original
body with timestamped per-source HMAC (or equivalent), use bounded sizes, and
validate source ownership. Customer inference keys cannot submit reports.

HTTP verifies and durably stores an inbox batch, then enqueues processing.
An initial acknowledgment means durable receipt; a receipt-status endpoint
confirms processing/rejection. Retain the exporter outbox until processing is
confirmed. Rejected batches never advance completeness.

Import rules:

1. Repeating the same batch identity and digest is safe; the same identity with
   different content is rejected.
2. A newer revision replaces an open cumulative bucket under a lock. Never add
   the new cumulative total to the old one. Older revisions cannot overwrite it.
3. Decreasing totals and changed facts require explicit corrections, not resets.
4. Import and progress updates are atomic. Track gaps; a high sequence or recent
   timestamp alone does not prove that all earlier usage arrived.
5. Process restarts preserve source identity. Restores/new incarnations require
   reconciliation so old usage is not billed again under a new identity.
6. Once a bucket is billed, its invoice inputs are frozen. Later corrections
   create linked adjustment allocations rather than rewriting issued documents.

Retain detailed inbox payloads only for bounded processing/recovery periods;
keep compact digests, receipts and final bucket provenance for audit. Avoid
permanently storing a full repeated snapshot every 15 minutes.

## On-demand sync and closing periods

Provide queued `request_usage_sync(source, cutoff, request_id)` with progress
tracking. It requests publication through a defined cutoff. Use the authenticated
metering API or a bounded Salt operation; never block an invoice HTTP action.
Flush means synchronizing durable facts, not clearing counters or deleting logs.

Attribute usage by request admission time and its trusted rating context. Schedule
tariff boundaries explicitly and distribute their contexts to gateways. In-flight
requests keep their original context and publish quantities once available.
A completeness watermark requires source reconciliation and accounted-for
in-flight work; a successful push alone cannot establish it.

At monthly closure, allow a configurable settlement delay, for example one hour,
request a sync and verify every relevant source through the period boundary.
An unavailable source leaves AI closure pending, not zero-valued. Server renewal
can proceed separately when necessary. A known late record after posting becomes
an explicit adjustment on the next invoice with its original usage period.
A cancelled subscription receives a final adjustment invoice instead.

At cancellation, block new requests on every gateway key binding, verify this,
drain or explicitly terminate in-flight requests under policy, then sync and
close through the confirmed cutoff. Revoking just one of several keys is not
sufficient. Carry-forward is a fallback; synchronization is the normal close path.

## Odoo CE sales, own subscriptions and invoicing

The addons use the CE `sale`, `account`, `account_payment`, `payment` and
`payment_stripe` infrastructure, including standard order and invoice portals.

| Existing model | Role |
| --- | --- |
| res.partner | Customer, company and billing address. |
| product.template/product.product | Server offers and AI model/category products. |
| product.pricelist | Quote defaults and negotiated rates, frozen into tariff versions. |
| sale.order/sale.order.line | Initial accepted order and subsequent renewal/adjustment orders. |
| account.move/account.move.line | Direct-sale customer invoices and credit notes. |
| payment.provider/payment.token/payment.transaction | Online collection and token references. |
| account.payment and reconciliation | Accounting payments, partial amounts, refunds and settlement. |

Own `oduflow.billing.*` models cover subscription, tariff
versions, service/settlement periods, rated charge allocations and durable
operation receipts. Do not recreate invoices or accounting-payment balances.

An initial order creates the relevant server subscriptions exactly once.
Each subscription references its origin order, server, historical billing
partner/company/currency, rates, collection owner, dates and lifecycle state.
At renewal the scheduler creates a new order for each server or settled usage
period and uses standard invoice machinery and sale-to-invoice links. Server
and usage periods are separate so unavailable inference statistics do not
prevent a server renewal invoice. Renewal orders must not recursively
create new subscriptions. Do not repeatedly invoice an already-fully-invoiced
origin line or grow its ordered quantity without limit.

Unique period/order and charge/invoice allocations prevent duplicates under
concurrent cron jobs. Preserve taxes, fiscal positions, payment terms, discounts
and currency rounding. Posted documents are corrected with credit notes or
adjustment invoices, not regenerated on metering retries.

Commercial defaults are server prepayment and AI in arrears, using UTC calendar
months. Verified readiness permits explicit billing activation. An accepted
initial quotation fixes the first interval price; a manually created subscription
prorates the first partial month using actual seconds. Later server months use
the agreed monthly price. Active terms are immutable; changed terms require a
new agreement rather than modifying historical charges.
Suspension alone does not stop infrastructure reservation charges.

## Stripe, Paddle and document ownership

For direct sales, Odoo owns invoices and the recurring schedule.
`oduflow_billing_stripe` reuses CE `payment_stripe`, adding subscription collection
orchestration, durable intents and reconciliation. Use Odoo payment transactions,
authorized token use and interactive authentication recovery. Do not also create
a Stripe Billing subscription or second invoice scheduler for the same charge.

Paddle is a Merchant of Record and issues customer documents. Its adapter declares
external document ownership, shows the genuine Paddle invoice/credit note in
our portal, and maps merchant settlement/fees to Odoo using the actual company
relationship. Do not post and send a second ordinary seller-to-customer invoice
for that same Paddle sale, or imitate Paddle's seller on a normal Odoo invoice.
The precise settlement/journal mapping depends on the merchant setup and must
be configured explicitly before posting settlement entries. Direct/Stripe sales still use
ordinary Odoo invoices; the portal can show both document types with clear issuers.

The implemented Paddle adapter uses standalone transactions and hosted checkout,
with no Paddle recurring subscription. Odoo owns the commercial schedule. A
customer completes each checkout; this implementation does not promise arbitrary
off-session Paddle debits. Final usage creates another standalone transaction.
Refunds use explicit adjustments linked to the original Paddle transaction.
Sandbox end-to-end collection still requires configured merchant credentials.

Persist intent and immutable request identity before external POSTs. Paddle does
not provide universal client-supplied idempotency keys: reconcile ambiguous
creates before retrying and stop automatic retries if the result is unknown.
Authenticate webhooks against the original body, deduplicate durable inboxes,
process asynchronously and reconcile out-of-order events. Check merchant,
customer, amount, currency and document; redirects are not proof of payment.

## Portal, security and budgets

Reuse standard order/direct-invoice portal pages. Add subscription pages with
server, tariff, next renewal, model usage, last complete sync, unbilled estimate
and linked orders/invoices. External Paddle documents use an authenticated
projection. Enforce historical billing-customer/company ownership for pages,
RPC, exports and links. Keep cost, key hashes, raw receipts and secrets private.

Observers remain read-only. Commercial permissions do not grant deployment
mutation rights. Infrastructure actions retain Oduflow Admin authority and
exact ownership checks; destructive removal requires fresh personal-password
verification. Module installation does not send messages or start collection.

A 15-minute delivery interval cannot enforce a strict monetary cap. Enforce
near-request policy at the gateway and allocate bounded credit across independent
gateways; copying the full limit to each multiplies exposure. Shared replicas
need coordinated limits. A single LiteLLM deployment cost does not represent
all negotiated customer tariffs: use explicit enforcement rates or, later,
a retail-aware reservation mechanism for strict custom limits.

Do not mark a sold local model free merely because its upstream cost is zero:
LiteLLM documents that explicit zero-cost settings can bypass budget checks.
Verify behavior on the pinned version and distinguish enforcement rates,
customer prices and operational cost.

## Migration, delivery and verification

1. Introduce gateway/configuration/key models and the Salt role. Import the
   existing endpoint as a legacy gateway while preserving encrypted keys,
   operation hashes, model names, ownership and immutable provisioning snapshots.
   Keep a compatibility facade for core callers during extraction to the addon.
2. Implement durable metering and aggregate push/sync. Validate source totals
   against Odoo under retries, restart, streaming and replica failover.
3. Implement CE subscription scheduling, renewal orders/invoices, the usage
   pricing bridge and portal; run shadow billing before collection.
4. Add Stripe using CE payments, then Paddle with explicit document/settlement
   ownership. Verify provider behavior in sandbox.
5. Add multi-node rollout while retaining direct gateway access and the initial
   multi-gateway schema.

Core key verification now separates immutable ownership from mutable budget and
blocked-state policy. Managed bindings verify identity before policy changes. Verify deletion
and recovery of billing-blocked keys. Old snapshots remain evidence; mutable
endpoints and policy resolve through current explicit bindings. The old demo
budget is not a customer balance or implicit free allowance.

Existing clients require explicit subscriptions and billable start dates; addon
installation must not retroactively charge them. Tests must cover aggregate
replay/reordering, lost ack, source restore/gaps, tariff splits, key transfer,
proxy-chain duplicates, token semantics, final drain, unavailable-source closure,
concurrent renewals, tiny amounts, taxes/credit notes, refunds, access isolation
and blocked-key deletion. Run Odoo tests via Megaflow and relevant local Salt
and Ruff checks. Report source, deployed and live verified states separately.

The current fleet supports independent gateways with one node each; replica
rollout, provider fallback, audio/image billing and automated strict retail credit
reservations are future work. Positive inference enforcement costs are separate
from customer selling prices. Production activation requires a built runtime
image matching the pinned contract, gateway/backend configuration, and a real
streaming/metering trial. Payment activation requires provider credentials,
customer authorization and configured accounting journals. New installations
leave subscription renewal and Paddle collection scheduling disabled.

## Primary references

- [LiteLLM config and overlays](https://docs.litellm.ai/docs/proxy/configs)
- [LiteLLM model balancing](https://docs.litellm.ai/docs/proxy/load_balancing)
- [LiteLLM custom pricing and zero-cost budgets](https://docs.litellm.ai/docs/proxy/custom_pricing)
- [LiteLLM spend tracking](https://docs.litellm.ai/docs/proxy/cost_tracking)
- [Odoo CE sales](https://github.com/odoo/odoo/tree/19.0/addons/sale)
- [Odoo CE invoicing](https://github.com/odoo/odoo/tree/19.0/addons/account)
- [Odoo CE Stripe](https://github.com/odoo/odoo/blob/19.0/addons/payment_stripe/models/payment_transaction.py)
- [Paddle Merchant of Record](https://developer.paddle.com/get-started/how-paddle-works/)
- [Paddle usage billing](https://developer.paddle.com/get-started/how-paddle-works/ai-companies/)
- [Paddle retry limitations](https://developer.paddle.com/sdks/libraries/)
