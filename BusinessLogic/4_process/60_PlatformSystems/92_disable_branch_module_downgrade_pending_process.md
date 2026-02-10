# 92 — Disable Module/Add-on (End-of-Cycle + `DOWNGRADE_PENDING`)

## Purpose

This process defines how an owner/admin requests a downgrade (disable a module/add-on) without breaking operations.

Key rules:
- Downgrades are end-of-cycle (no mid-cycle refunds).
- Some downgrades (Workforce) must be safety-gated using `DOWNGRADE_PENDING`.

---

## Domains Involved

- Subscription & Entitlements (downgrade scheduling + entitlement enforcement)
- Workforce (safe boundary facts: active attendance/cash sessions)
- Inventory (disable semantics: read-only history; selling unaffected)
- Audit (evidence of change request + completion)

---

## When This Process Runs

Triggered when an owner/admin requests to disable a branch module/add-on.

---

## Inputs (Minimum)

- `tenant_id`
- `branch_id`
- `actor_id`
- downgrade target:
  - disable `module.inventory` / `module.workforce`
  - disable add-on (e.g., `addon.workforce.gps_verification`)

---

## Orchestration Steps

### Step 1 — Validate authorization + current plan

- Confirm actor can change plan for the branch.
- Confirm the target is currently enabled.

---

### Step 2 — Schedule end-of-cycle disablement

- Record a downgrade request with effective date = end of current billing period (billing anchor).
- If the downgrade affects Workforce:
  - immediately mark branch as `DOWNGRADE_PENDING` for Workforce.
  - apply wind-down rules while pending:
    - block new staff check-ins
    - block opening new staff cash sessions
    - allow existing sessions to end/close

---

### Step 3 — Enforce at safe boundary (Workforce) / effective date (others)

At or after the effective date:
- Inventory downgrade:
  - set inventory entitlements to read-only/disabled-visible for the branch.
- Workforce downgrade:
  - only complete when safe boundary holds:
    - no active staff attendance
    - no open staff cash sessions
  - then set workforce entitlements OFF and auto-suspend team operation for that branch.

Record audit events:
- `BRANCH_PLAN_DOWNGRADE_REQUESTED`
- `BRANCH_PLAN_DOWNGRADE_COMPLETED` (when it becomes effective)

---

## Failure / Degradation Rules (March)

- Downgrade must not brick the counter mid-operation.
- Workforce downgrades must remain in a visible "pending" state until completion.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/subscription_billing_edge_case_sweep.md`
- Audit event catalog: `BusinessLogic/5_modSpec/60_PlatformSystems/subscriptionEntitlements_module.md`
