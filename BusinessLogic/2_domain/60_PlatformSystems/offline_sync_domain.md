# Offline Sync Domain Model — Modula POS

## Domain Name
Offline Sync (Connectivity Resilience)

## Domain Type
Platform / Supporting Domain (Infrastructure)

## Domain Group
60_PlatformSystems

## Status
Draft (Baseline for Capstone 1: offline-first sync = push/replay + pull/hydration)

---

## Purpose

The Offline Sync domain ensures Modula remains operational during unstable or unavailable connectivity **without compromising backend integrity**.

It provides:
- local durability for offline operations
- idempotent replay when connectivity returns
- clear separation between cached “local truth” and backend “authoritative truth”
- an authoritative **pull/hydration** interface so offline-first clients converge without per-module polling

Offline Sync is not a business workflow. It is a resilience layer.

---

## Domain Boundary & Ownership

### What This Domain Owns

- Offline operation queue model (durable, ordered per device)
- Idempotency rules for offline replay
- Local cache requirements (menu, policy, tenant/branch context)
- Sync state semantics (pending/synced/failed)
- Replay sequencing rules (device FIFO; server authoritative)
- **Authoritative change feed** semantics (pull/hydration):
  - cursor-based incremental reads
  - tombstone semantics (hard removals)
  - module scope filtering (to avoid pulling unnecessary domains)
  - device checkpoints (resume without rehydrating)
  - branch stream fan-out for tenant-scoped writes that affect all branches

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
- `device_id` (stable device identifier for per-device ordering)
- `type` (operation kind, defined by the calling module)
- `payload` (minimal inputs required for replay)
- `payload_hash` (or equivalent; detects `client_op_id` reuse with different payload)
- `depends_on[]` (optional; explicit dependency edges for deterministic batch processing)
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

### Branch Stream (Unit of Sync)

For March baseline, **branch streams are the unit of sync**.

A client hydrates data for a specific `(tenant_id, branch_id)` context by pulling from that branch stream.

Implications:
- A device does not need to aggregate multiple branch feeds to operate a single branch.
- Tenant-scoped writes that affect all branches (example: menu catalog changes) are **fanned out** into each active branch stream.

### Change Feed (Authoritative Hydration)

The backend exposes an ordered, cursor-based change feed per branch stream.

A feed entry represents a server-authoritative mutation that offline clients must converge to.

Conceptual fields:
- `seq` (monotonic sequence within the branch stream)
- `entity_kind` (module-defined)
- `entity_id`
- `op_kind` = UPSERT | TOMBSTONE
- `payload` (for UPSERT) or minimal tombstone (for TOMBSTONE)
- `occurred_at` (server time)
- `actor_id` (optional; for traceability)

### Cursor (Resume Token)

A cursor represents “I have applied changes up to sequence N for this stream”.

Cursor is:
- opaque to clients (treat as token),
- stable across short reconnects,
- invalidatable by the server (retention/reset).

### Module Scopes (Selective Hydration)

Pull sync supports `module_scopes` so clients can request only the domains they need.

Examples:
- `menu.*`, `policy.*`, `discount.*`
- `sale.*` (if local order list needs hydration)

Scope safety rule:
- The cursor must embed a `module_scope_hash` (or equivalent) so the server can detect scope drift.

### Device Checkpoint (Client Progress)

Device checkpoint is the client’s recorded hydration progress (cursor + metadata).

It exists so:
- devices can resume sync without rehydrating,
- server can optimize change feed delivery (optional).

Checkpoint is not authoritative truth. The feed is authoritative.

### Two-Lane Write Model (Offline-First Reality)

Offline-first systems must support both:
1) **Online direct writes**: feature endpoints guarded by idempotency keys
2) **Offline replay writes**: push a batch of offline operations for deterministic replay

Both lanes must converge into the same server truth and the same change feed.

---

## Invariants (Non-Negotiable)

- **INV-OS-1 (Tenant isolation):** Every offline operation must include `tenant_id`.
- **INV-OS-2 (Branch explicitness):** Branch-scoped actions must include `branch_id`.
- **INV-OS-3 (Idempotent replay):** `client_op_id` guarantees exactly-once application.
- **INV-OS-4 (Server authority):** Backend decides acceptance and ordering; client timestamps are for audit only.
- **INV-OS-5 (No bypass):** Offline replay must still pass Access Control and branch/tenant status checks.
- **INV-OS-6 (Durable queue):** Pending operations survive app restarts.
- **INV-OS-7 (Hydration convergence):** A branch stream change feed must be sufficient for a client to converge its local cache without per-module polling.
- **INV-OS-8 (Feed atomicity):** For any accepted write that changes business truth, the system must not publish “half a change”.
  - A change feed append must be committed atomically with the business write (and its audit/outbox records if present).
- **INV-OS-9 (Scope safety):** A cursor must not be reused across different module scope sets without explicit reset/rehydration.

---

## Relationship to Other Domains

- **Sale / CashSession / Attendance / Inventory**: produce offline operations; Offline Sync replays them.
- **Access Control**: validates permissions and scope at replay time; offline uses cached facts for UX only.
- **Policy**: provides cached pricing inputs for offline sale computation; authoritative policy may differ.
- **Audit**: business modules record audit events when operations are applied.
- **Operational Notification**: may be emitted after operations are applied (best-effort).
- **Job Scheduler**: may run reconciliation/compaction tasks (future; not required for March hydration correctness).

---

## Out of Scope (For This Domain Doc)

- Conflict resolution UI
- Multi-device conflict arbitration
- Offline menu/policy editing
- Cross-tenant batching

---

## Summary

Offline Sync is a platform domain that preserves **operational continuity** while keeping backend truth authoritative and auditable.

For March baseline, “offline-first” includes both:
- durable offline queue + idempotent replay (push),
- and authoritative read-model hydration via change feed (pull).
