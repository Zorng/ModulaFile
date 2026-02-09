# 40 — Audit Event Recording (Atomic + Idempotent)

## Purpose

This process defines how Modula records audit evidence for business-critical actions.

It exists to ensure:
- no silent state changes (accountability)
- deterministic behavior under retries and offline sync replay
- consistent audit structure across modules

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/saas_governance/reviewing_audit_logs_for_accountability.md`

---

## Domains Involved

- Audit Logging (append-only evidence store)
- Access Control (reason codes when denial is authorization-related)
- Offline Sync (replay and retry patterns)
- Calling business domain/module (Sale, CashSession, Inventory, HR, Policy, etc.)

---

## When This Process Runs

Triggered whenever:
- a state-changing operation succeeds, or
- a meaningful action attempt is rejected/failed and must remain auditable.

---

## Inputs (Minimum)

- `event_type` (stable string; defined by the calling module ModSpec)
- `actor_id` (or `SYSTEM`)
- `tenant_id`
- `branch_id` (required for branch-scoped actions)
- `occurred_at` (client or server time)
- `outcome` (`SUCCESS | REJECTED | FAILED`)
- `reason_code` (optional; required for REJECTED/FAILED when possible)
- `subject` / `entity_refs` (IDs only)
- `idempotency_key` (required for offline-safe operations)
- `metadata` (small, non-sensitive context)

---

## Orchestration Steps

### Step 1 — Determine event type + context

- Select the event type defined by the calling module’s ModSpec.
- Ensure `tenant_id` is present.
- If the audited action is branch-scoped, ensure `branch_id` is present.

If context is missing for a branch-scoped action, the action should be rejected and the rejection should be auditable.

---

### Step 2 — Determine outcome + reason code

- If the action is denied by Access Control, record the Access Control `reason_code`.
- If denied by validation, use `VALIDATION_FAILED`.
- If denied due to missing prerequisites (example: missing open cash session), use `DEPENDENCY_MISSING`.
- If denied by a domain business rule, use `BUSINESS_RULE_BLOCKED`.

---

### Step 3 — Enforce idempotency key

Audit event recording must be idempotent.

Minimum approaches for March:
- Use the client operation ID provided by Offline Sync, or
- Use a deterministic key based on `(event_type, entity_id, operation_id)` when the operation has a stable ID.

---

### Step 4 — Persist atomically for state changes

For state-changing operations:
- write the audit event in the same transaction as the state change
- if the audit insert fails, the state change must fail (no “unaudited write”)

For observational events (example: report viewed):
- best-effort insert is acceptable
- failures must not block the read surface

---

### Step 5 — Return stable acknowledgment

Return success to the caller even if the same event was already recorded (idempotent no-op).

---

## Failure / Degradation Rules (March)

- If audit recording fails during a state-changing operation, fail the operation.
- If audit recording fails during observational logging, proceed but do not retry aggressively in the critical path.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/audit_edge_case_sweep.md`

