# 91 — Enable Module/Add-on/Seats (Immediate + Prorated Upgrade)

## Purpose

This process defines how an owner/admin upgrades a branch mid-cycle:
- enabling a module (Inventory, Workforce),
- enabling an add-on (example: Workforce GPS verification),
- or increasing operator seats.

It keeps a single tenant billing anchor while still allowing fair mid-cycle upgrades.

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/saas_governance/upgrading_branch_capabilities_mid_cycle.md`

---

## Domains Involved

- Subscription & Entitlements (invoice, proration intent, entitlement activation)
- Branch (scope boundary)
- Access Control (who can change plan)
- Audit (evidence of plan changes)

---

## When This Process Runs

Triggered when an owner/admin requests an upgrade for an existing paid branch.

---

## Inputs (Minimum)

- `tenant_id`
- `branch_id`
- `actor_id`
- requested change:
  - enable `module.inventory` / `module.workforce`
  - enable add-on (e.g., `addon.workforce.gps_verification`)
  - increase seat count (operator seats)

Constraints:
- invoice currency is USD-only (March baseline)

---

## Orchestration Steps

### Step 1 — Validate context + authorization

- Confirm tenant and branch exist and are eligible for upgrades.
- Confirm actor is authorized to change branch plan (owner/admin).

---

### Step 2 — Compute upgrade charge intent (proration to billing anchor)

- Determine current billing period end (billing anchor).
- Compute the prorated charge for the change for "now → end of period".

Notes:
- Exact proration math is an implementation detail; the process only requires that:
  - upgrades are fair to the remaining time, and
  - the user is shown a clear charge before confirming.

---

### Step 3 — Issue invoice and collect payment (KHQR `Pay Now`)

- Issue a USD invoice for the upgrade.
- Display KHQR `Pay Now` for self-serve payment to Modula.
- Backend verifies payment.
- Payment collection and confirmation details:
  - `BusinessLogic/4_process/60_PlatformSystems/95_payment_collection_and_confirmation_orchestration.md`

Degradation:
- If payment fails/cancels, no entitlement is activated.

---

### Step 4 — Activate entitlements and record audit

After payment is confirmed:
- Update entitlement snapshot for the branch (enable module/add-on/seats).
- Record audit events (canonical names):
  - `SUBSCRIPTION_INVOICE_PAID`
  - `BRANCH_PLAN_UPGRADE_APPLIED` (include changed keys in metadata)

---

## Failure / Degradation Rules (March)

- Do not activate entitlements before payment is confirmed.
- If the client repeats the request (double tap/retry), ensure idempotency (same invoice/change intent) rather than duplicating charges.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/subscription_billing_edge_case_sweep.md`
- Audit event catalog: `BusinessLogic/5_modSpec/60_PlatformSystems/subscriptionEntitlements_module.md`
