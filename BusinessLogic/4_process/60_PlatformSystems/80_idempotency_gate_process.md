# 80 — Idempotency Gate (Critical Writes)

## Purpose

This process defines how Modula guarantees **exactly-once** application for critical write operations.

It exists to absorb retries (online or offline) without duplicating business effects.

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/correcting_mistakes/avoiding_duplicate_actions_due_to_retries.md`

---

## Domains Involved

- Idempotency (guard rail)
- Access Control (authorization)
- Calling module (Sale, CashSession, Attendance, Inventory)

---

## When This Process Runs

Triggered before any critical write is applied (finalize sale, cash movement, attendance start/end, inventory adjustment).

---

## Inputs (Minimum)

- `idempotency_key` (client_op_id / request_id)
- `action_key` (stable action identifier)
- `tenant_id`
- `branch_id` (required for branch-scoped actions)
- `payload_hash` (for conflict detection)
- `actor_id`

---

## Orchestration Steps

### Step 1 — Validate context

- Require `tenant_id`.
- If action is branch-scoped, require `branch_id`.
- If missing context, reject early.

---

### Step 2 — Authorization

- Run Access Control for the action.
- If denied, return denial (no idempotency record is applied).

---

### Step 3 — Idempotency gate

- Check for existing idempotency record using `(tenant_id, action_key, idempotency_key, branch_id)`.
- If none:
  - create a record with status `IN_PROGRESS` (atomic transaction boundary begins).
- If found:
  - If `payload_hash` matches, return stored result (duplicate).
  - If `payload_hash` differs, reject as conflict.

---

### Step 4 — Apply business write atomically

- Perform the domain write within the same transaction as the idempotency record.
- On success, update idempotency record to `APPLIED` and store `result_ref`.
- On failure, rollback both state change and idempotency record.

---

## Failure / Degradation Rules (March)

- If idempotency storage is unavailable, fail the operation (do not risk duplicates).
- Conflicts must surface clearly for investigation.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/idempotency_edge_case_sweep.md`

