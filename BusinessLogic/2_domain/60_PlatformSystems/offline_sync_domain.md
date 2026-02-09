# Offline Sync Domain Model — Modula POS

## Domain Name
Offline Sync (Connectivity Resilience)

## Domain Type
Platform / Supporting Domain (Infrastructure)

## Domain Group
60_PlatformSystems

## Status
Draft (Baseline for Capstone 1: offline queue + idempotent replay)

---

## Purpose

The Offline Sync domain ensures Modula remains operational during unstable or unavailable connectivity **without compromising backend integrity**.

It provides:
- local durability for offline operations
- idempotent replay when connectivity returns
- clear separation between cached “local truth” and backend “authoritative truth”

Offline Sync is not a business workflow. It is a resilience layer.

---

## Domain Boundary & Ownership

### What This Domain Owns

- Offline operation queue model (durable, ordered per device)
- Idempotency rules for offline replay
- Local cache requirements (menu, policy, tenant/branch context)
- Sync state semantics (pending/synced/failed)
- Replay sequencing rules (device FIFO; server authoritative)

### What This Domain Does NOT Own

- Business truth and state machines (Sale, CashSession, Inventory, HR)
- Authorization policy (Access Control decides; Offline Sync enforces via cached facts and server recheck)
- Audit logging schema (Audit owns evidence records)
- Conflict resolution UI (future)
- Background analytics sync (future)

Offline Sync transports operations; it does not decide outcomes.

---

## Core Concepts (Ubiquitous Language)

### OfflineOperation (Entity)

A durable queued operation created when the device cannot reach the server.

Fields (conceptual):
- `client_op_id` (globally unique idempotency key)
- `type` (operation kind, defined by the calling module)
- `payload` (minimal inputs required for replay)
- `tenant_id`
- `branch_id` (required when the action is branch-scoped)
- `actor_id`
- `occurred_at` (client time; not authoritative)
- `status` (`PENDING | SYNCING | SYNCED | FAILED`)
- `failure_reason` (optional)

### Local Cache

A tenant/branch-scoped cache of read data required for offline operation:
- menu items + modifiers
- policy snapshot (VAT/FX/rounding)
- actor identity + current tenant/branch context

Local cache enables offline UX, but is not authoritative.

### Idempotency Key

`client_op_id` is the stable dedupe anchor.
Replaying the same `client_op_id` must not create duplicates on the backend.

### Sync State

Offline operations move through:
- `PENDING` → queued
- `SYNCING` → in replay
- `SYNCED` → accepted and applied
- `FAILED` → rejected (no automatic retries)

---

## Invariants (Non-Negotiable)

- **INV-OS-1 (Tenant isolation):** Every offline operation must include `tenant_id`.
- **INV-OS-2 (Branch explicitness):** Branch-scoped actions must include `branch_id`.
- **INV-OS-3 (Idempotent replay):** `client_op_id` guarantees exactly-once application.
- **INV-OS-4 (Server authority):** Backend decides acceptance and ordering; client timestamps are for audit only.
- **INV-OS-5 (No bypass):** Offline replay must still pass Access Control and branch/tenant status checks.
- **INV-OS-6 (Durable queue):** Pending operations survive app restarts.

---

## Relationship to Other Domains

- **Sale / CashSession / Attendance / Inventory**: produce offline operations; Offline Sync replays them.
- **Access Control**: validates permissions and scope at replay time; offline uses cached facts for UX only.
- **Policy**: provides cached pricing inputs for offline sale computation; authoritative policy may differ.
- **Audit**: business modules record audit events when operations are applied.
- **Operational Notification**: may be emitted after operations are applied (best-effort).

---

## Out of Scope (For This Domain Doc)

- Conflict resolution UI
- Multi-device conflict arbitration
- Offline menu/policy editing
- Cross-tenant batching

---

## Summary

Offline Sync is a platform domain that preserves **operational continuity** while keeping backend truth authoritative and auditable.

