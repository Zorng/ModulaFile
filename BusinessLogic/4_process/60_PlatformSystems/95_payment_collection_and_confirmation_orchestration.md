# 95 — Payment Collection & Confirmation Orchestration (March Billing)

## Purpose

This process defines **how an owner/admin pays** Modula invoices and how the backend confirms payment
so entitlements can be activated deterministically.

It exists to prevent "hand-wavy billing":
- a dev/AI must be able to implement `Pay Now -> Confirm -> Activate` without inventing steps,
- and without coupling operational modules to payment mechanics.

March baseline stance:
- invoices are USD-only
- primary rail: KHQR self-serve "Pay Now" with backend verification
- fallback: bank transfer with manual verification (no proof upload requirement)

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/saas_governance/activating_first_branch_and_starting_subscription.md`
- `BusinessLogic/1_stories/saas_governance/upgrading_branch_capabilities_mid_cycle.md`
- `BusinessLogic/1_stories/saas_governance/handling_overdue_subscription_and_freeze.md`

---

## Domains Involved

- Subscription & Entitlements (invoice + state transitions + entitlement activation)
- Access Control (who may pay/trigger pay-now views)
- Audit (invoice + payment evidence)
- Idempotency (duplicate suppression)

Related process entry points that call into this payment step:
- `BusinessLogic/4_process/60_PlatformSystems/90_first_branch_activation_orchestration.md`
- `BusinessLogic/4_process/60_PlatformSystems/91_enable_branch_module_or_seats_proration_process.md`
- `BusinessLogic/4_process/60_PlatformSystems/94_additional_branch_activation_orchestration.md`
- `BusinessLogic/4_process/60_PlatformSystems/93_subscription_renewal_grace_and_freeze_orchestration.md` (recovery)

---

## When This Process Runs

Triggered when an owner/admin chooses "Pay Now" for a USD invoice issued by Subscription/Entitlements:
- first-branch activation invoice
- additional branch activation invoice
- upgrade invoice (modules/add-ons/seats)
- renewal invoice (recovery from `PAST_DUE`/`FROZEN`)

---

## Inputs (Minimum)

- `tenant_id`
- `actor_id` (owner/admin)
- `invoice_id`

Optional:
- `branch_id` if the invoice is branch-scoped (activation/upgrade)

---

## Canonical Concepts

### Payment Reference Code (Locked)

Every invoice must have a stable `payment_reference_code` used to match the external payment to the invoice.

Rules:
- reference code is derived from invoice identity (stable, unique, human-copyable)
- KHQR payload must include this reference code
- bank transfer fallback requires this reference code in the transfer note/description

This avoids "upload proof" workflows for March.

---

### Denial-Code Direction (Locked)

Payment/subscription denials must use billing-oriented reason codes, for example:
- `BRANCH_ACTIVATION_PAYMENT_REQUIRED`
- `SUBSCRIPTION_UPGRADE_REQUIRED`

Do not introduce slot-capacity denial codes for branch monetization (for example `BRANCH_SLOT_LIMIT_REACHED`).

---

## Orchestration Steps

### Step 0 — Idempotency Gate (Pay Now view)

`action_key = subscription.payNow`

Pay-now must be idempotent:
- repeated taps must return the same payment instructions for the same invoice
- do not issue duplicate invoices here (invoice issuance belongs to the caller processes)

---

### Step 1 — Validate invoice is payable

Validate:
- invoice exists and belongs to `tenant_id`
- invoice status is payable (not already paid/void)
- invoice currency is USD (March baseline)

If not payable:
- return a clear status (already paid / void / not found)
- record a rejected audit event only if meaningful (avoid noise)

---

### Step 2 — Present payment instructions (KHQR Pay Now)

System returns a payment instructions payload that includes:
- amount (USD)
- receiver identity (Modula billing receiver)
- `payment_reference_code`
- KHQR payload / QR image data (rendering is UI-level)

Notes:
- Pay-now display must be accessible even when subscription state is `FROZEN` so tenants can recover.
- This is achieved by treating `subscription.payNow` as a governance read action (it does not mutate business truth).

---

### Step 3 — Backend confirms payment (authoritative)

Client confirmation is not trusted.

Backend confirmation may occur via:
- webhook (preferred; shared gateway), or
- polling/query on the payment network (degradation)

Webhook ingestion reference:
- `BusinessLogic/4_process/60_PlatformSystems/96_webhook_ingestion_and_dispatch_process.md`

Reconciliation mechanism (March):
- When webhook is missed/delayed, polling confirmation is run by the Job Scheduler tick:
  - `BusinessLogic/4_process/60_PlatformSystems/97_job_scheduler_tick_and_execution_process.md`

Correlation rule (March):
- invoice `payment_reference_code` must be embedded in the KHQR payload so the webhook can resolve the target invoice deterministically.

Confirmation outcome:
- If payment is confirmed for `payment_reference_code` and expected amount:
  - mark invoice `PAID`
  - emit audit event: `SUBSCRIPTION_INVOICE_PAID`
  - return confirmation result to client
- If payment cannot be confirmed yet:
  - return "PENDING" safely (no entitlements activated)

---

### Step 4 — Apply the paid invoice effect (caller-owned)

After payment is confirmed, the *caller process* applies business truth:

- First-branch activation (Process 90):
  - provision branch
  - set billing anchor
  - emit `BRANCH_ACTIVATED`, `BILLING_ANCHOR_SET`

- Additional branch activation (Process 94):
  - provision branch
  - emit `BRANCH_ACTIVATED`

- Upgrade (Process 91):
  - activate entitlements/seats immediately
  - emit `BRANCH_PLAN_UPGRADE_APPLIED`

- Renewal recovery (Process 93):
  - restore subscription state to `ACTIVE`
  - emit `SUBSCRIPTION_ACTIVE_RESTORED`

This split prevents this payment process from owning operational orchestration.

---

## Bank Transfer Fallback (March)

If KHQR pay-now verification is not usable for a tenant, the system may offer a fallback:

### Fallback Steps (Minimal)

1. Show Modula bank account details (USD only) and `payment_reference_code`.
2. Tenant performs a transfer and includes the reference code in the transfer description.
3. Admin/support performs manual verification outside the app (operational responsibility).
4. System marks the invoice `PAID` and then executes the same "Apply invoice effect" step as above.

Rules:
- No proof upload is required in March baseline.
- Manual verification must be auditable:
  - record `SUBSCRIPTION_INVOICE_PAID` with metadata indicating `payment_method = BANK_TRANSFER_MANUAL`.

---

## Failure / Degradation Rules (March)

- Never activate entitlements before payment is confirmed.
- If verification cannot be completed due to network instability, keep invoice pending and allow retry.
- If a tenant is `PAST_DUE`:
  - operations continue with warnings,
  - pay-now should remain available.
- If a tenant is `FROZEN`:
  - operational writes remain blocked,
  - pay-now and invoice viewing remain available to recover.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/subscription_billing_edge_case_sweep.md`
- Audit event catalog: `BusinessLogic/5_modSpec/60_PlatformSystems/subscriptionEntitlements_module.md`
