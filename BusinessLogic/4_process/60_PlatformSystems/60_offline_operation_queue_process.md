# 60 — Offline Operation Queueing

## Purpose

This process defines how Modula records offline operations when connectivity is unavailable.

It exists to:
- preserve user actions without blocking the UI
- guarantee durable replay later
- keep the system honest about offline state

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/selling_and_checkout/operating_during_internet_outage.md`

---

## Domains Involved

- Offline Sync (queue + replay semantics)
- Authentication (actor identity)
- Access Control (cached facts for UX)
- Sale / CashSession / Attendance (action sources)

---

## When This Process Runs

Triggered when a write operation is attempted and the device is offline or the server cannot be reached.

---

## Inputs (Minimum)

- `actor_id`
- `tenant_id`
- `branch_id` (required for branch-scoped actions)
- `device_id`
- `operation_type`
- `payload`
- `client_op_id` (idempotency key)
- `payload_hash` (or equivalent; used to detect client_op_id reuse with different payload)
- `depends_on[]` (optional; explicit dependency edges for deterministic batch replay)
- `occurred_at` (client time)

---

## Orchestration Steps

### Step 1 — Detect offline state

- If connectivity is unavailable, do not attempt immediate server writes.
- Continue the operation via offline queue.

---

### Step 2 — Validate minimal prerequisites locally

Use cached facts to prevent obviously invalid actions:
- Required context present (`tenant_id`, `branch_id` for branch-scoped ops)
- Local session state (example: cash session open) if cached

If prerequisites are missing, fail locally with a clear reason.

---

### Step 3 — Enqueue the operation

Create an OfflineOperation with:
- `client_op_id` (unique)
- `tenant_id`, `branch_id`
- `operation_type` + payload
- `occurred_at`
- `status = PENDING`

Persist durably (survives app restart).

---

### Step 4 — Apply local UX effects (best-effort)

For user continuity:
- Create local read models (example: offline sale receipt)
- Mark actions as “pending sync”
- Surface offline state and pending count in UI

Local effects must not be mistaken for authoritative backend truth.

---

## Failure / Degradation Rules (March)

- If local cache is missing critical data (menu/policy), block offline finalize with a clear message.
- Do not reorder or delete queued operations manually.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/offline_sync_edge_case_sweep.md`
