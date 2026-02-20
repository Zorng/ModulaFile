# Offline Sync Module — Core Module (Capstone 1)

**Version:** 1.3  
**Status:** Patched (derived from Story→Domain→Contract→Process)  
**Module Type:** Core Module  
**Depends on:** Authentication, Access Control, Tenant & Branch Context, Policy (cached), Audit Logging  
**Related Modules:** Sale, Cash Session, Attendance, Inventory (via replay)

---

## 1. Purpose

The Offline Sync module keeps Modula usable during unstable connectivity **without weakening backend integrity**.

It provides:
- durable offline queueing
- idempotent replay on reconnect
- clear UI signals about offline/online state
- authoritative read-model hydration (pull) so offline-first clients converge

This is a resilience layer, not a business workflow.

Authoritative references:
- Story: `BusinessLogic/1_stories/selling_and_checkout/operating_during_internet_outage.md`
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/offline_sync_domain.md`
- Edge cases: `BusinessLogic/3_contract/10_edgecases/offline_sync_edge_case_sweep.md`
- Processes:
  - `BusinessLogic/4_process/60_PlatformSystems/60_offline_operation_queue_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/65_offline_sync_pull_hydration_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/70_offline_sync_replay_process.md`

---

## 2. Scope & Boundaries

### This module covers
- Offline queue for write operations
- Durable local storage of queued ops
- Idempotent replay using `client_op_id`
- Local cache of menu + policy + context for offline sale computation
- Pull-based hydration of authoritative server changes into local cache
- UI exposure of offline/online state + pending count

### This module does NOT cover (Capstone 1)
- Offline menu editing
- Offline inventory restocking
- Offline policy editing
- Admin conflict resolution UI
- Cross-device reconciliation

---

## 3. Offline Operations Allowed (March)

Allowed offline operations (queued and replayed):
- `sale.finalize` (cash sales only; requires open cash session)
- `cashSession.open`
- `cashSession.movement` (paid in / paid out / adjustment)
- `cashSession.close`
- `attendance.startWork`
- `attendance.endWork`

Not allowed offline in March:
- menu editing
- inventory restock
- policy updates
- void approvals

If an operation is not supported offline, the UI must block it with a clear message.

---

## 4. Core Integrity Rules

### 4.1 Exactly-Once Replay

- Every offline op includes a globally unique `client_op_id`.
- Backend applies the operation once; duplicates return the existing result.

### 4.2 Server-Authoritative Ordering

- Client timestamps are not authoritative for integrity.
- Backend ordering and transactions decide correctness.

### 4.3 Access Control Revalidation

- Replay must pass Access Control and branch/tenant status checks.
- If branch is frozen, queued writes are rejected deterministically.

### 4.4 No Silent Bypass

Offline UX may continue using cached facts, but replay is the source of truth.
Failures must be surfaced to the user.

### 4.5 Two-Lane Writes (Online + Offline)

Offline-first clients must support two write lanes:
- **Direct online writes**: feature endpoints guarded by idempotency keys.
- **Offline replay writes**: queued operations pushed as an ordered batch for deterministic replay.

Both lanes must converge into the same backend truth and must be observable via pull hydration.

---

## 5. Interfaces (Conceptual)

This module is an umbrella capability with two backend interfaces.

### 5.1 Push Sync (Replay)

`SyncPushBatch(tenant_id, device_id, operations[]) -> results[]`

Where each operation includes:
- `client_op_id` (idempotency anchor)
- `type` (module operation kind)
- `payload`
- `branch_id` (when branch-scoped)
- optional `depends_on[]` (dependency edges within a batch)
- `payload_hash` (or equivalent) to detect client_op_id reuse with different payload

Rules:
- operations are processed in deterministic order (FIFO per device; stable order within batch),
- duplicates are no-op (return existing result),
- same `client_op_id` with a different payload is rejected deterministically,
- dependency violations are reported (dependent ops must not be applied silently).

### 5.2 Pull Sync (Hydration)

`SyncPull(tenant_id, branch_id, device_id, cursor?, module_scopes[]) -> {changes[], next_cursor}`

Where:
- `cursor` is an opaque resume token for the branch stream,
- `module_scopes` filters which domains/entities are included,
- `changes[]` is ordered and includes tombstones for hard removals.

Rules:
- scope drift invalidates cursor use (client must reset/rehydrate),
- tenant isolation must be enforced (no cross-tenant leak),
- branch streams are the unit of sync; tenant-scoped catalog changes that affect all branches are fanned out.

---

## 6. Triggering Pull (Degradation-Safe)

Clients must pull:
- at bootstrap (first load of a branch),
- on app resume / reconnect,
- periodically while online (low frequency),
- and optionally upon receiving a lightweight “sync nudge” signal.

Important:
- a nudge channel is a trigger only; it must not be relied upon to deliver full state.

---

## 7. Functional Requirements

### Queue & Storage
- FR-1: Offline operations are stored durably and survive app restart.
- FR-2: Each operation includes `client_op_id`, `tenant_id`, `branch_id`, `actor_id`, `occurred_at`, payload.
- FR-3: Queue is FIFO per device.

### Local Cache
- FR-4: Menu and policy values are cached per tenant + branch.
- FR-5: If cache is missing critical data, offline finalize is blocked.

### Replay
- FR-6: Replay is idempotent via `client_op_id`.
- FR-7: Replay revalidates access, branch status, and prerequisites.
- FR-8: Failed operations are marked `FAILED` with reason; not silently retried forever.

### UX Signals
- FR-9: UI exposes offline/online state.
- FR-10: UI shows pending sync count and failure reasons.

---

## 8. Use Cases (Selected)

### UC-1 — Offline Finalize Sale
1. Cashier finalizes sale offline.
2. Operation is queued with `client_op_id`.
3. UI shows “pending sync”.
4. Replay applies exactly once on reconnect.

### UC-2 — Offline Cash Session Open/Close
1. Cashier opens or closes a cash session offline.
2. Operations are queued in order.
3. Replay applies in order; rejections are surfaced.

### UC-3 — Offline Start/End Work
1. Staff starts/ends work offline.
2. Attendance operations are queued.
3. Replay applies via Attendance module idempotently.

### UC-4 — Sync Replay on Reconnect
1. Connectivity restored.
2. Queue replays FIFO.
3. Success → `SYNCED`; rejection → `FAILED` with reason.

---

## 9. Acceptance Criteria

- AC-1: Offline sales sync exactly once (no duplicates).
- AC-2: Cash session open/close works offline and syncs in order.
- AC-3: Attendance start/end queues offline and syncs idempotently.
- AC-4: Frozen branch rejects queued writes deterministically with clear reason.
- AC-5: UI always shows offline/online state and pending count.

---

## 10. Audit Events

Offline Sync does not define new business audit events.
Business modules emit audit events when operations are applied.

---

# End of Document
