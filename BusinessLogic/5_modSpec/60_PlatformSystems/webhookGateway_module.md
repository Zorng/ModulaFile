# Webhook Gateway Module — Platform Module (External Event Ingestion)

**Version:** 1.0  
**Status:** Draft (March baseline: Bakong payment confirmation webhooks)  
**Module Type:** Platform / Supporting Module  
**Depends on:** Idempotency (dedupe), Audit (evidence), Operational Notification (anomalies), Payment (proof validation), Subscription/Entitlements (invoice state), Sale (KHQR attempt state), Fair-Use Limits (rate limiting)  

---

## 1. Purpose

This module provides a single implementation surface for **ingesting external webhook deliveries**
and dispatching them as safe internal events.

It exists to avoid:
- per-feature webhook code paths,
- duplicate deliveries causing duplicate business effects,
- and insecure "accept everything" webhook endpoints.

Webhook ingestion is a **platform primitive**, similar to Idempotency:
it is not a user-facing feature, but it prevents painful operational failures.

Authoritative domain:
- `BusinessLogic/2_domain/60_PlatformSystems/webhook_gateway_domain.md`

Authoritative contract:
- `BusinessLogic/3_contract/10_edgecases/webhook_ingestion_edge_case_sweep.md`

Authoritative process:
- `BusinessLogic/4_process/60_PlatformSystems/96_webhook_ingestion_and_dispatch_process.md`

---

## 2. Scope (March Baseline)

Included:
- Bakong/KHQR payment confirmation webhook ingestion (sale payments and subscription invoices)
- authenticity verification (provider secret/signature)
- normalization into an internal `PaymentEvent`
- idempotent dedupe under retries
- dispatch to:
  - Sale KHQR confirmation handler
  - Subscription invoice payment handler
- anomaly signaling for:
  - unmatched payments
  - mismatched payment proofs
  - superseded attempts paid later

Excluded (explicit):
- general-purpose event bus for arbitrary product events
- webhooks for non-payment providers
- telemetry dashboards (tracked separately as `PS-OBS-01`)
- background reconciliation jobs (tracked separately as `PS-JOB-01`)

---

## 3. Interfaces (Conceptual)

### 3.1 Ingest endpoint (provider-specific wrapper)

- `IngestWebhook(provider, headers, body) -> IngestionResult`

Where `IngestionResult` includes:
- `accepted` (boolean)
- `reason_code` when rejected
- `normalized_event_ref` (if accepted)
- `dispatch_outcome` (success / no-op / signaled)

This interface is intentionally abstract: actual HTTP paths and payload schemas are implementation details.

---

## 4. Security (Authenticity Verification)

Webhook ingestion must verify authenticity before processing:
- provider signature/HMAC and/or shared secret token
- reject if verification fails

Important separation:
- Bakong API access token (used for polling/query endpoints) is not the same as webhook authenticity.
- Webhook authenticity must be enforced even if the backend can also poll Bakong.

---

## 5. Normalization & Dispatch (PaymentEvent)

Webhook payloads must be normalized into an internal event shape:
- `provider`
- `event_kind = PAYMENT_CONFIRMED` (March baseline)
- correlation key(s):
  - `md5` for sale KHQR attempts
  - `payment_reference_code` for subscription invoices
- proof fields if present: amount/currency/receiver/hash/time

Dispatch targets (March):
- Sale KHQR confirmation handler:
  - updates KHQR attempt state and makes it visible to POS client
- Subscription invoice payment handler:
  - marks invoice as `PAID` and emits `SUBSCRIPTION_INVOICE_PAID`

The gateway must never bypass Payment proof validation.

---

## 6. Idempotency (Non-Negotiable)

Webhook delivery is at-least-once; duplicates are expected.

The module must apply idempotency using the Idempotency Gate:
- `idempotency_key` derived from provider event id when available
- otherwise derived from a stable delivery fingerprint
- `action_key` should be stable for the provider + event kind (example: `webhook.ingest.bakong.payment_confirmed`)

Rules:
- duplicates return the same outcome (no double effects)
- conflicts (same key, different payload) are rejected and signaled

---

## 7. Observability (March)

Webhook ingestion should produce observable signals for operational confidence:
- unmatched payment event -> operational notification
- mismatched proof -> operational notification + audit signal via the owning domain

Security-only failures (invalid signature) are not tenant actions:
- do not leak them into tenant audit logs
- record them in system logs/telemetry (implementation; telemetry is post-pilot)

---

## 8. Acceptance Criteria (March)

- AC-1: Duplicate webhook deliveries do not create duplicate invoice payments or KHQR confirmations.
- AC-2: Invalid signature deliveries are rejected with zero domain effects.
- AC-3: Unmatched payment events are stored as evidence and surfaced as operational notifications.
- AC-4: Proof mismatches never unlock/finalize anything automatically; they produce a signal for review.

