# Idempotency Domain Model — Modula POS

## Domain Name
Idempotency (Operation De-duplication)

## Domain Type
Platform / Supporting Domain (Integrity Guard)

## Domain Group
60_PlatformSystems

## Status
Draft (Baseline for Capstone 1: critical writes are idempotent)

---

## Purpose

The Idempotency domain ensures that **retries do not create duplicate business effects**.

It exists to protect the system from:
- network timeouts
- double taps
- offline replay
- retry storms

Idempotency is a **guard rail** for integrity, not a business feature.

---

## Domain Boundary & Ownership

### What This Domain Owns

- Idempotency key semantics
- De-duplication rules and invariants
- Storage of idempotency records (status + result reference)
- Conflict detection when the same key is reused with different intent

### What This Domain Does NOT Own

- Business truth and state machines (Sale, CashSession, Inventory, HR)
- Authorization decisions (Access Control)
- Offline queue and replay mechanics (Offline Sync)
- Audit schema (Audit Logging)

Idempotency records *what has already been applied*; it does not decide *what should happen*.

---

## Core Concepts (Ubiquitous Language)

### Idempotency Key

A unique client-generated token that represents **one user intent**.

Examples:
- `client_op_id`
- `request_id`

### Idempotency Record

A persisted record that binds:
- `idempotency_key`
- `action_key` (operation type)
- `tenant_id` (required)
- `branch_id` (required for branch-scoped actions)
- optional `actor_id`
- `payload_hash` (for conflict detection)
- `status` (`IN_PROGRESS | APPLIED | FAILED`)
- `result_ref` (pointer to the applied outcome)

### Idempotent Write

A write that:
- applies exactly once for the same `idempotency_key`
- returns the existing result for duplicates

---

## Invariants (Non-Negotiable)

- **INV-ID-1 (Exactly-once):** Replaying the same idempotency key must not create duplicate effects.
- **INV-ID-2 (Scope isolation):** `tenant_id` is mandatory; branch-scoped actions must include `branch_id`.
- **INV-ID-3 (Conflict detection):** Reusing the same key with a different payload is a conflict and must be rejected.
- **INV-ID-4 (Atomicity):** Idempotency record and business state change must be committed atomically.
- **INV-ID-5 (Deterministic result):** Duplicate requests return the same outcome and reference.

---

## Relationship to Other Domains

- **Sale / CashSession / Attendance / Inventory**: invoke idempotency gate before applying writes.
- **Offline Sync**: uses idempotency keys for replay safety.
- **Access Control**: still authorizes each request; idempotency does not bypass permission checks.
- **Audit**: records outcomes; idempotency helps prevent duplicate audit entries.

---

## Out of Scope (For This Domain Doc)

- UI states and error messaging
- Retention policy for idempotency records (may be a future platform policy)
- Cross-service distributed idempotency (future)

---

## Summary

Idempotency is a platform integrity guard that ensures **retries are safe** and **business truth is not duplicated**.

