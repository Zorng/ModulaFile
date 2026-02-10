# 93 — Subscription Renewal, Grace Window, and Freeze

## Purpose

This process defines the tenant-level subscription lifecycle at renewal time:
- issue renewal invoice at billing anchor,
- apply `PAST_DUE` grace for 24 hours if unpaid,
- and apply `FROZEN` enforcement after grace.

This protects operational reality while preserving a clean enforcement boundary.

---

## Domains Involved

- Subscription & Entitlements (billing anchor, state transitions)
- Access Control (deny writes when frozen; consistent error reason)
- Audit (evidence of state transitions and payments)

---

## When This Process Runs

Triggered automatically at:
- the tenant billing anchor (renewal),
- and the end of the 24h grace window if still unpaid.

---

## Inputs (Minimum)

- `tenant_id`
- current billing period end (billing anchor)

Constraints:
- invoices are USD-only (March baseline)

---

## Orchestration Steps

### Step 1 — Renewal invoice issuance

At the billing anchor:
- Issue a renewal invoice (USD) for the tenant's current paid capacities:
  - active branches
  - enabled modules/add-ons per branch
  - operator seats per branch
- Record audit event: `SUBSCRIPTION_INVOICE_ISSUED` (renewal)

---

### Step 2 — If unpaid at anchor: enter `PAST_DUE` (24h grace)

If the renewal invoice is not paid at the anchor time:
- Set subscription state to `PAST_DUE`.
- Allow operations to continue, but require prominent warnings and a clear pay-now path.
- Record audit event: `SUBSCRIPTION_PAST_DUE_ENTERED`.

---

### Step 3 — If still unpaid after 24h: enter `FROZEN`

If unpaid 24 hours after entering `PAST_DUE`:
- Set subscription state to `FROZEN`.
- Enforce "no operational writes":
  - sales cannot be finalized
  - no new cash sessions
  - no start work
  - no inventory writes
- Allow read-only access (history, reports, invoices).
- Record audit event: `SUBSCRIPTION_FROZEN_ENTERED`.

---

### Step 4 — Payment recovery: restore `ACTIVE`

When the overdue renewal invoice is paid:
- Set subscription state to `ACTIVE`.
- Restore normal operation.
- Record audit event: `SUBSCRIPTION_ACTIVE_RESTORED`.
- Payment collection and confirmation details:
  - `BusinessLogic/4_process/60_PlatformSystems/95_payment_collection_and_confirmation_orchestration.md`

---

## Failure / Degradation Rules (March)

- State transitions must be deterministic and auditable.
- Freeze must be enforced server-side; UI-only enforcement is not acceptable.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/subscription_billing_edge_case_sweep.md`
- Audit event catalog: `BusinessLogic/5_modSpec/60_PlatformSystems/subscriptionEntitlements_module.md`
