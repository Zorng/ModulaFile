# 10 — Update Branch Policy (Branch-Scoped)

## Purpose

This process updates the branch-scoped Policy values used by Sale:
- Tax & Currency (VAT, FX rate, KHR rounding)
- Sale workflow toggle (pay-later enablement)

It exists to keep sale computation inputs explicit, validated, and auditable.

This process mutates policy configuration only. It does not mutate sales.

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/handling_money/configuring_tax_and_currency_settings.md`
- `BusinessLogic/1_stories/selling_and_checkout/placing_order_and_paying_later.md` (pay-later motivation; toggle is still configuration)

---

## Domains Involved

- Policy (configuration truth for VAT/FX/rounding)
- Access Control (authorization to update)
- Tenant/Branch (branch context + freeze status)
- Audit Logging (accountability for changes)

---

## When This Process Runs

Triggered when an Admin updates tax/currency settings for a selected branch.

This is not a scheduled job for March.

---

## Inputs

- `tenant_id`
- `branch_id`
- `policy_patch` (partial update; only changed fields provided)

---

## Orchestration Steps

### Step 1 — Validate branch context + authorization

- Ensure `tenant_id` and `branch_id` are present.
- Ensure the actor is authorized to update branch policies for the selected branch.
- If the branch is frozen, deny the update (read may still be allowed).

---

### Step 2 — Validate patch values (fail closed)

Validate updated fields against Policy invariants:
- VAT rate in `[0, 100]`
- FX rate `> 0`
- rounding mode in `NEAREST | UP | DOWN`
- rounding granularity in `100 | 1000`
- `saleAllowPayLater` must be a boolean

Reject invalid updates with a clear validation error.

---

### Step 3 — Apply partial update atomically

- Load current branch policy record (or defaults if missing).
- Apply the patch as a single atomic write.
- Persist the updated policy values for `(tenant_id, branch_id)`.

No partial writes are allowed.

---

### Step 4 — Emit audit record

Write an audit event capturing:
- actor identity
- branch context
- old values and new values (for changed fields)

---

### Step 5 — Enable downstream refresh

Return the updated policy object (or at minimum updated metadata such as `updated_at`) so clients can:
- update their displayed settings immediately
- refresh cached policy used in checkout calculations

---

## Failure / Degradation Rules (March)

- If offline: block updates (do not queue policy edits silently).
- If concurrent updates happen: last-write-wins is acceptable for March, but each update must be auditable.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/policy_edge_case_sweep.md`
