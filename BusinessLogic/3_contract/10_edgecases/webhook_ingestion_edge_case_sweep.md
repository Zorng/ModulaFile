# Edge Case Contract — Webhook Ingestion Gateway

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: external webhook ingestion (authenticity, dedupe, dispatch) for payment confirmations
- **Primary Audience**: Backend, QA
- **Owner(s)**: PlatformSystems (Webhook Gateway), Idempotency, Payment, Sale, Subscription/Entitlements, Audit, Operational Notification
- **Last Updated**: 2026-02-11
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred
- **Related Domain**:
  - `BusinessLogic/2_domain/60_PlatformSystems/webhook_gateway_domain.md`
- **Related Processes**:
  - `BusinessLogic/4_process/60_PlatformSystems/96_webhook_ingestion_and_dispatch_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`

---

## Purpose

Lock the minimum webhook ingestion behaviors so:
- payment confirmations are safe under retries,
- security is not "best effort",
- and downstream modules do not implement ad-hoc webhook handlers.

---

## Definitions / Legend

- **Delivery**: one HTTP request attempt from provider to Modula (may repeat).
- **Normalized event**: internal representation after verification + parsing.
- **Correlation key**: internal lookup key that ties a provider event to a Modula intent (e.g., `md5`, `payment_reference_code`).

---

## Edge Case Catalog (Locked)

### EC-WH-01 — Duplicate webhook deliveries (at-least-once)
- **Scenario**: Provider retries and delivers the same webhook multiple times.
- **Trigger**: network retry, timeout, provider delivery policy.
- **Expected Behavior**:
  - The gateway dedupes idempotently (same event -> same outcome).
  - Downstream handlers (invoice paid, KHQR paid) do not execute duplicate writes.
- **Owner**: Webhook Gateway + Idempotency + downstream handler
- **March**: Yes

### EC-WH-02 — Webhook authenticity verification fails
- **Scenario**: Request is missing required verification data or signature does not match.
- **Trigger**: wrong secret, spoofed request, replay without valid signature.
- **Expected Behavior**:
  - Reject without applying any domain effect.
  - Record an observational signal for security debugging (audit/telemetry).
- **Owner**: Webhook Gateway + Audit
- **March**: Yes

### EC-WH-03 — Correlation key not found (unmatched payment event)
- **Scenario**: A payment-confirmed webhook arrives but Modula cannot map it to a sale attempt or invoice.
- **Trigger**: missing attempt registration, wrong reference code, eventual-consistency race, operator used wrong receiver.
- **Expected Behavior (March baseline)**:
  - Do not apply any operational truth automatically.
  - Store the event as `UNMATCHED` evidence and emit an operational notification for investigation.
  - Respond in a way that does not create an infinite retry storm (implementation choice; but must avoid repeated effects).
- **Owner**: Webhook Gateway + Operational Notification
- **March**: Yes (signal-only)

### EC-WH-04 — Payment proof mismatch (amount/currency/receiver)
- **Scenario**: Provider reports "paid" but proof does not match what Modula expects for that intent.
- **Trigger**: wrong amount, wrong currency, wrong receiver account id, or stale/superseded intent.
- **Expected Behavior**:
  - Do not auto-finalize or auto-activate anything.
  - Mark the intent as `MISMATCHED` / `REQUIRES_REVIEW` (implementation naming may differ).
  - Emit audit + operational notification for investigation.
- **Owner**: Payment domain validation + Webhook Gateway dispatch + Audit/OperationalNotification
- **March**: Yes (reject + signal)

### EC-WH-05 — Superseded KHQR attempt is paid later
- **Scenario**: A cashier regenerates KHQR, but an older/superseded QR is paid later.
- **Trigger**: webhook confirms payment for a correlation key that is not the "latest attempt".
- **Expected Behavior**:
  - Do not finalize a sale automatically.
  - Emit operational notification + audit signal (refund/settlement workflows deferred).
- **Owner**: KHQR confirmation handler + Audit + Operational Notification
- **March**: Yes (signal-only)

### EC-WH-06 — Webhook arrives after manual confirmation already occurred
- **Scenario**: The system already confirmed payment via polling/query, then webhook arrives later.
- **Trigger**: delayed webhook delivery.
- **Expected Behavior**:
  - Treat as duplicate; no state oscillation.
  - Keep the same paid/confirmed result.
- **Owner**: Webhook Gateway + downstream handler idempotency
- **March**: Yes

---

## Summary

For March, webhook ingestion must be:
- authenticated,
- safe under duplicates,
- deterministic under mismatches,
- and able to signal unmatched/suspicious payments without breaking operations.
