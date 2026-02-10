# 96 — Webhook Ingestion & Dispatch (Bakong Payment Confirmations)

## Purpose

This process defines a single, reusable webhook ingestion pipeline so external confirmations (Bakong/KHQR)
can be handled once and reused by:
- Sale KHQR payment confirmation, and
- Subscription invoice pay-now confirmation.

The webhook is treated as a **signal**:
- it is never trusted without authenticity verification,
- it is always safe under duplicates (at-least-once delivery),
- and business truth remains owned by Payment/Sale/Subscription domains.

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/selling_and_checkout/accepting_khqr_payment.md`
- `BusinessLogic/1_stories/saas_governance/activating_first_branch_and_starting_subscription.md`
- `BusinessLogic/1_stories/correcting_mistakes/avoiding_duplicate_actions_due_to_retries.md`

---

## Domains Involved

- Webhook Gateway (authenticity, normalization, dispatch)
- Idempotency (dedupe under retries)
- Payment (proof validation rules)
- Sale (KHQR attempt state -> allow manual finalize)
- Subscription & Entitlements (invoice paid -> unlock/provision deterministically)
- Audit (evidence of meaningful outcomes)
- Operational Notification (signals for anomalies: unmatched/mismatch/superseded)
- Fair-Use Limits (rate limiting / abuse control; recommended)

References:
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/webhook_gateway_domain.md`
- Contract: `BusinessLogic/3_contract/10_edgecases/webhook_ingestion_edge_case_sweep.md`
- Idempotency gate: `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`
- Fair-use gate: `BusinessLogic/4_process/60_PlatformSystems/85_fair_use_limit_gate_process.md`

---

## When This Process Runs

Triggered when an external provider delivers a webhook to Modula’s backend webhook endpoint.

March baseline:
- provider: Bakong (KHQR payment confirmations)

---

## Inputs (Minimum, Conceptual)

- `provider = BAKONG`
- request headers (signature/secret verification material)
- request body (provider payload)
- `received_at` (server time)

---

## Orchestration Steps

### Step 0 — Apply fair-use protections (recommended)

Before doing expensive work:
- apply a fair-use gate for webhook ingestion (rate limit / burst limit).
- reject or degrade safely if limits are exceeded (avoid provider retry storms).

This is a platform safety rule, not a billing rule.

---

### Step 1 — Verify authenticity (non-negotiable)

Verify the webhook is authentic using provider-recommended mechanisms:
- signature/HMAC validation and/or shared secret token validation.

If authenticity fails:
- return a rejection (HTTP 401/403),
- do not apply any domain effects.

---

### Step 2 — Parse + schema validate

Parse payload and validate minimum required fields exist.

If invalid:
- return a rejection (HTTP 400),
- do not apply any domain effects.

---

### Step 3 — Normalize into an internal event

Create a normalized internal event with:
- `provider`
- `event_kind` (March baseline: payment confirmation signal)
- correlation key(s) present (examples: `md5`, `payment_reference_code`)
- proof fields (amount/currency/receiver/tx_hash/time), if provided

Normalization is required so downstream handlers do not parse provider-specific payloads.

---

### Step 4 — Resolve correlation -> internal dispatch target

Resolve the event to a Modula-owned target before any write effects:

Examples (March):
- If correlation key is `md5`:
  - resolve to a registered Sale KHQR payment attempt:
    - `(tenant_id, branch_id, sale_id, expected_amount, expected_currency, expected_receiver)`
- If correlation key is `payment_reference_code`:
  - resolve to the invoice that owns that reference code:
    - `(tenant_id, invoice_id, expected_amount_usd, receiver = Modula)`

If no target can be resolved:
- store as `UNMATCHED` evidence (implementation detail),
- emit operational notification for investigation,
- return a response that avoids infinite provider retries.

---

### Step 5 — Idempotency gate (dedupe)

Webhook delivery is at-least-once. The system must treat duplicates as normal.

Apply the Idempotency Gate with an idempotency key derived from the provider event:
- prefer `provider_event_id` if present,
- otherwise use a stable fingerprint (hash of canonicalized payload + critical headers),
- for payment confirmations, also acceptable to anchor by the correlation key (`md5` / `payment_reference_code`) if uniqueness is guaranteed.

If `DUPLICATE`:
- return the already-processed result (no new effects).

If `CONFLICT`:
- reject deterministically and emit an anomaly signal.

---

### Step 6 — Dispatch to domain handler (business interpretation)

Dispatch to the correct handler based on the resolved target:

#### 6A) Sale KHQR confirmation handler
- Validate proof matches expected (Payment domain rules).
- Update payment attempt state:
  - `PAID_CONFIRMED` for match
  - `MISMATCHED/REQUIRES_REVIEW` for mismatch
  - `SUPERSEDED_PAID` signal path when applicable (see POS edge cases)
- Make confirmation visible to the POS client (poll/push is implementation).

#### 6B) Subscription invoice payment handler
- Validate proof matches expected (amount + receiver; USD-only for March).
- Mark invoice as `PAID` (idempotent).
- Emit `SUBSCRIPTION_INVOICE_PAID` (auditable).
- Caller orchestration (activation/upgrade/renewal) applies the paid invoice effect deterministically.

This step is where business truth is decided; the gateway itself remains a transport layer.

---

### Step 7 — Respond safely

Return an acknowledgment response with:
- stable success response for duplicates (idempotent),
- clear rejection for authenticity/schema failures,
- and a safe response for unmatched events (no retry storm).

---

## Failure / Degradation Rules (March)

- Webhook ingestion must not be the only confirmation path:
  - reconciliation via polling/query still exists (see payment processes) if webhook is missed.
- Never allow webhook ingestion to bypass Payment proof validation.
- Never allow webhook ingestion to directly "finalize sale" in March (sale finalize remains manual).

