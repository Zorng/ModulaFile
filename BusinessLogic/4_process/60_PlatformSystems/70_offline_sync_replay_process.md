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

### Step 2 — Replay in order

Process queued operations in FIFO order:
1. Send operation with `client_op_id` to the backend.
2. Backend validates:
   - authorization
   - tenant/branch status
   - prerequisites (cash session open, etc.)
3. Backend applies idempotently (exactly-once).

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

---

## Failure / Degradation Rules (March)

- If branch is frozen, queued writes must be rejected deterministically.
- Failed operations are not auto-retried indefinitely; they require user/admin awareness.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/offline_sync_edge_case_sweep.md`

