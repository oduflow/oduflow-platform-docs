# Billing setup and acceptance

Architecture: [billing-architecture.md](billing-architecture.md).
Gateway runtime: [litellm-managed.md](litellm-managed.md).
Base subscriptions: [addon guide](https://github.com/oduflow/oduflow-platform/blob/main/addons/oduflow_billing/README.md).
Paddle: [merchant setup](https://github.com/oduflow/oduflow-platform/blob/main/addons/oduflow_billing_paddle/README.md).

## Initial setup

1. Configure the Odoo company's accounting, currency, service products, taxes
   and journals. Assign **Billing Manager** separately from infrastructure admin.
2. Create a published tariff version with explicit selling prices per million
   input/output tokens for each commercial model. Enable cached input only for
   a backend whose semantics have been verified, and define its separate rate.
3. Create a separate **Inference Gateway** for local inference and for external
   proxying as needed. Each has its own endpoint, model deployments, metering
   source and master credential. Clients connect to these endpoints directly.
4. Mark the intended infrastructure instance as a **LiteLLM Platform Host**
   before provisioning. Assign it as the gateway's node. This uses the normal
   provider ownership and verified volume workflow, without installing a
   customer's production Odoo stack on the gateway host.
5. Build the extended runtime image described in `litellm-managed.md`; configure
   its immutable digest, private bind address and backend deployments. Enter
   credentials through the encrypted credential wizard. The default managed
   PostgreSQL password is generated and encrypted by Odoo.
6. Configure the source's HTTPS ingestion URL and initialize its authentication.
   Prepare, review and apply the gateway configuration revision. Salt installs
   the gateway, managed PostgreSQL and aggregate exporter on the verified volume.
7. Explicitly create a customer's server subscription or mark a quotation line
   **Create Server Subscription**. Set the customer server and tariff. Activate
   billing only after the customer server has been verified active.
8. Create/verify the customer's gateway key bindings and assign their historical
   subscription. Publish billing contexts before using those keys for inference.
   The admission hook rejects requests without a unique published context.
9. Verify a real text and streaming request against source and Odoo aggregates;
   repeat batch delivery and on-demand sync. Verify key blocking and final drain.
10. Configure the desired payment adapter in sandbox, complete its acceptance
    below, and only then enable the subscription billing cron and selected
    collection options. Existing clients are never automatically subscribed.

The initial quote fixes the price of its first interval, even when partial.
Manual subscriptions without an initial quote prorate the first calendar month.
Later server months use the agreed monthly price, and token usage is settled in
arrears. Changing active commercial terms requires a new agreed subscription.

## Payment acceptance

Stripe reuses Odoo's payment provider, customer token and invoice machinery.
Configure a bank journal and payment method accounts, collect the customer's
recurring-payment authorization through the normal payment flow, and assign the
resulting token to the subscription. Enable automatic collection explicitly.
Verify success, authentication-required/declined payment, timeout reconciliation
and refund behavior in Stripe test mode before using live credentials.

Paddle uses standalone hosted checkouts and actual Paddle customer documents.
Follow its addon guide for customer/address ownership markers, approved product,
webhook destination and merchant accounting mapping. Customer completion is
required; this adapter does not create a separate Paddle recurring subscription.
Merchant settlement entries are explicit accountant actions.

## Operational checks

- A recent delivery timestamp does not certify completeness. Compare the
  source's **Complete Through** cutoff with the period being closed.
- Missing statistics leave usage closure pending. Investigate the source inbox,
  sequence gap, unresolved admission or correction; never reset usage counters.
- Lost payment responses require reconciliation. Absence from a bounded provider
  search does not establish that no payment exists and never permits blind retry.
- Cancellation blocks every assigned key and waits for all pre-cutoff admissions
  to settle. Infrastructure deletion remains a separate protected operation.
- Back up source PostgreSQL and SQLite provenance consistently. Restores preserve
  source identity and require reconciliation against Odoo's retained receipts.
- Portal customers see their own subscriptions, orders and billing documents;
  they have no direct access to internal usage receipts or gateway credentials.

A successful addon test suite proves the exercised software contracts. Record a
separate acceptance result for actual Salt provisioning, upstream token counts,
streaming, source failure, cancellation and each payment provider's sandbox flow.
