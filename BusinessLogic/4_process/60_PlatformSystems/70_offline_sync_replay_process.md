# 70 — Offline Sync Replay (Reconnect)

## Purpose

This process defines how queued offline operations are replayed when connectivity returns.

It exists to:
- apply operations exactly once
- preserve backend integrity under retries
- surface failures clearly

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/selling_and_checkout/operating_during_internet_outage.md`

---

## Domains Involved

- Offline Sync (queue + replay)
- Access Control (authorization recheck)
- Sale / CashSession / Attendance (operation targets)
- Audit (business modules emit audit events on apply)

---

## When This Process Runs

Triggered when connectivity is restored or periodically while online and the queue is non-empty.

---

## Inputs

- `tenant_id`
- queued OfflineOperations (ordered by enqueue time)

---

## Orchestration Steps

### Step 1 — Prepare replay context

- Confirm the current tenant context matches the queued operations.
- If tenant context differs, pause replay until the correct tenant is active.

---

### Step 2 — Push queued operations as a deterministic batch

Replay uses a push-sync batch interface (conceptual):
`SyncPushBatch(tenant_id, device_id, operations[]) -> results[]`

Rules:
- preserve FIFO order per device (the queue is authoritative ordering from the client perspective),
- backend is authoritative on acceptance/rejection and must still validate:
  - authorization (Access Control),
  - tenant/branch status,
  - prerequisites (cash session open, etc.),
- backend applies idempotently using `client_op_id` (exactly-once),
- same `client_op_id` with a different payload must be rejected deterministically
  (example reason code: `CLIENT_OP_ID_REUSE_WITH_DIFFERENT_PAYLOAD`).

---

### Step 3 — Update local status

- On success: mark operation `SYNCED`.
- On permanent rejection: mark `FAILED` with a clear reason.
- On transient failure: leave `PENDING` for retry.

---

### Step 4 — Handle dependency failures

If an earlier operation fails and later operations depend on it:
- do not apply dependent operations silently
- mark them failed with `DEPENDENCY_MISSING` (or equivalent)
- surface a clear UI signal

Dependencies may be:
- implicit (FIFO ordering), and/or
- explicit (`depends_on[]`) within a push batch (recommended when different operation types are interleaved).

---

## Failure / Degradation Rules (March)

- If branch is frozen, queued writes must be rejected deterministically.
- Failed operations are not auto-retried indefinitely; they require user/admin awareness.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/offline_sync_edge_case_sweep.md`
