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

This is a resilience layer, not a business workflow.

Authoritative references:
- Story: `BusinessLogic/1_stories/selling_and_checkout/operating_during_internet_outage.md`
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/offline_sync_domain.md`
- Edge cases: `BusinessLogic/3_contract/10_edgecases/offline_sync_edge_case_sweep.md`
- Processes:
  - `BusinessLogic/4_process/60_PlatformSystems/60_offline_operation_queue_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/70_offline_sync_replay_process.md`

---

## 2. Scope & Boundaries

### This module covers
- Offline queue for write operations
- Durable local storage of queued ops
- Idempotent replay using `client_op_id`
- Local cache of menu + policy + context for offline sale computation
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

---

## 5. Functional Requirements

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

## 6. Use Cases (Selected)

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

## 7. Acceptance Criteria

- AC-1: Offline sales sync exactly once (no duplicates).
- AC-2: Cash session open/close works offline and syncs in order.
- AC-3: Attendance start/end queues offline and syncs idempotently.
- AC-4: Frozen branch rejects queued writes deterministically with clear reason.
- AC-5: UI always shows offline/online state and pending count.

---

## 8. Audit Events

Offline Sync does not define new business audit events.
Business modules emit audit events when operations are applied.

---

# End of Document
