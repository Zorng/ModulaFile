# Webhook Gateway Domain Model — Modula POS

## Domain Name
Webhook Gateway (External Event Ingestion)

## Domain Type
Platform / Supporting Domain (External Signals -> Internal Events)

## Domain Group
60_PlatformSystems

## Status
Draft (March pilot: Bakong KHQR payment confirmations)

---

## Purpose

This domain defines how Modula ingests **external webhook deliveries** (at-least-once, untrusted by default)
and turns them into **internal normalized events** that downstream domains can consume safely.

It exists to prevent drift like:
- "webhook = truth" (wrong),
- ad-hoc per-module webhook handlers,
- duplicate webhook deliveries causing duplicate business effects.

In March pilot scope, this domain focuses on **payment confirmation webhooks**:
- sale KHQR payment confirmations (Bakong)
- subscription invoice pay-now confirmations (Bakong / KHQR)

---

## Domain Boundary & Ownership

### What This Domain Owns

- A canonical "webhook delivery" ingestion pattern:
  - authenticity verification (provider secret/signature)
  - payload parsing and schema validation
  - normalization into an internal event shape
  - dedup/idempotency behavior under retries
- Routing/dispatch to the correct internal handler surface (payment confirmation handlers).
- Storage of received webhook deliveries/events for audit/debug (observational, not business truth).

### What This Domain Does NOT Own

- Payment rules (amount/currency/receiver binding) — owned by `Payment` domain.
- Sale finalization truth — owned by Sale/Finalize processes.
- Subscription state transitions and entitlement activation — owned by Subscription/Entitlements processes.
- UI/UX ("waiting for payment" screens, realtime updates) — contracts/processes define required states only.
- Background reconciliation jobs (polling when webhook is missed) — tracked separately as `PS-JOB-01`.

This domain is a **gateway**, not a workflow.

---

## Core Concepts (Ubiquitous Language)

### Webhook Provider
The external system that sends webhook deliveries.

March baseline provider:
- `BAKONG`

### Webhook Delivery (Raw)
One HTTP delivery attempt from a provider to Modula.

Key attributes (conceptual):
- `provider` (e.g., `BAKONG`)
- `received_at`
- request headers (for signature verification)
- raw payload body
- `delivery_fingerprint` (hash of canonicalized payload + critical headers)

Delivery is **not** guaranteed to be unique (providers retry).

### Normalized Webhook Event (Internal)
An internal canonical representation after verification + parsing.

Key attributes (conceptual):
- `provider`
- `event_kind` (e.g., `PAYMENT_CONFIRMED`)
- `provider_event_id` (if present)
- correlation key(s) (e.g., `md5`, `payment_reference_code`)
- proof fields returned by the provider (amount/currency/receiver/hash/time), if present
- `occurred_at` (provider time, if present)

### Dispatch Target
The internal handler that owns business interpretation, for example:
- "Sale KHQR attempt confirmation"
- "Subscription invoice payment confirmation"

The gateway routes events to the correct handler; it does not decide business outcomes.

---

## Invariants (Non-Negotiable)

- **INV-WH-1 (Verify authenticity):** a webhook delivery must not be processed unless provider authenticity is verified.
- **INV-WH-2 (At-least-once assumed):** webhook deliveries may repeat and arrive out of order.
- **INV-WH-3 (Idempotent handling):** processing the same normalized event multiple times must not create duplicate business effects.
- **INV-WH-4 (Tenant isolation):** an event must resolve to the correct `tenant_id` (and `branch_id` when branch-scoped) before any write effects occur.
- **INV-WH-5 (Webhook is not truth):** webhook deliveries are *signals*; business truth is still decided by domain validation (Payment/Sale/Subscription).

---

## Relationship to Other Domains

- **Payment (POSOperation)**
  - owns proof validation (amount/currency/receiver binding) and confirmation rules.
- **Sale (POSOperation)**
  - consumes payment confirmation results to allow manual finalize.
- **Subscription & Entitlements**
  - consumes invoice payment confirmation results to mark invoices paid and unlock/provision deterministically.
- **Idempotency**
  - provides durable dedup semantics for webhook deliveries and downstream confirmation writes.
- **Audit / Operational Notification**
  - records observational evidence and flags anomalies (unknown payments, mismatches, superseded paid attempts).

---

## Out of Scope (For This Domain Doc)

- Exact provider webhook schemas (Bakong payload specifics).
- Transport security details (WAF, IP allowlists) beyond the invariant "verify authenticity".
- Realtime client update mechanisms (polling vs push).

---

## Summary

Webhook Gateway is a platform domain that makes external webhook deliveries safe by enforcing:
authenticity verification, normalization, idempotent dedupe, and deterministic dispatch to internal handlers.

