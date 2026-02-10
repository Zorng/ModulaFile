# 94 — Additional Branch Activation Orchestration (Buy Branch → Provision)

## Purpose

This process defines how a tenant with an existing paid branch activates **another** branch.

It preserves the same core rules as first-branch activation:
- **No unpaid branches exist** (a new branch is provisioned only after payment confirmation).
- The tenant still has **one billing anchor date** (no new renewal date per branch).
- New branches are `core_pos`-enabled by default; optional modules are branch-scoped.

---

## Story Traceability

Derived from:
- `BusinessLogic/1_stories/saas_governance/upgrading_branch_capabilities_mid_cycle.md`
- `BusinessLogic/1_stories/saas_governance/activating_first_branch_and_starting_subscription.md` (activation pattern)

---

## Domains Involved

- Subscription & Entitlements (invoice, proration intent, activation draft)
- Branch (system-provisioned branch)
- Tenant (billing state: ACTIVE/PAST_DUE/FROZEN)
- Audit (evidence of purchase/provisioning)
- Policy (defaults for new branch; forward-only config)

---

## When This Process Runs

Triggered when an owner/admin requests to add a new branch.

---

## Inputs (Minimum)

- `tenant_id`
- activation draft fields (minimal branch identity, e.g., `branch_display_name`)
- `actor_id` (owner/admin)

Constraints (March baseline):
- billing currency is USD-only
- payment collection primary rail is KHQR `Pay Now` with backend verification

---

## Orchestration Steps

### Step 1 — Validate eligibility

- Confirm tenant is allowed to upgrade:
  - if tenant is `FROZEN`, deny until paid.
  - if tenant is `PAST_DUE`, deny upgrades until paid (avoid increasing debt).
- Confirm actor is authorized to add branches (owner/admin via Access Control).

---

### Step 2 — Create activation draft + issue invoice (USD)

- Create a `BranchActivationDraft` for the new branch (pre-branch).
- Compute the upgrade charge intent:
  - if mid-cycle: prorate from now → billing anchor end
  - if at renewal boundary: normal full-period charge
- Issue a USD invoice for "additional branch activation".
- Record audit events (canonical names):
  - `BRANCH_ACTIVATION_INITIATED`
  - `SUBSCRIPTION_INVOICE_ISSUED`

Idempotency:
- Repeated requests must not create multiple invoices for the same activation intent.

---

### Step 3 — Collect payment (KHQR `Pay Now`) and verify

- Display KHQR payment request for the invoice amount to Modula’s receiver.
- Backend verifies payment.
- Payment collection and confirmation details:
  - `BusinessLogic/4_process/60_PlatformSystems/95_payment_collection_and_confirmation_orchestration.md`
- While waiting:
  - show “waiting for payment”
  - allow retry/poll without duplicating activation

---

### Step 4 — On payment confirmed: provision the real branch

When payment is confirmed:
- Provision the real branch (system-only `ProvisionBranch` intent).
- Initialize required branch defaults (example: tax/currency policy defaults).
- Mark the invoice `PAID`.
- Record audit events (canonical names):
  - `SUBSCRIPTION_INVOICE_PAID`
  - `BRANCH_ACTIVATED`

Note:
- The tenant billing anchor does not change; it was set by the first paid activation.

---

### Step 5 — Return branch access + allow plan configuration

- The new branch becomes selectable for operational work.
- Optional modules remain locked until enabled per-branch (inventory/workforce).
- Follow `91_enable_branch_module_or_seats_proration_process.md` for enabling optional modules/add-ons/seats.

---

## Failure / Degradation Rules (March)

- Never create a branch if payment is not confirmed.
- If provisioning fails after payment confirmation, remain in a recoverable "activating" state (no duplicate charges, no duplicate branches).

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/subscription_billing_edge_case_sweep.md`
- Audit event catalog: `BusinessLogic/5_modSpec/60_PlatformSystems/subscriptionEntitlements_module.md`
