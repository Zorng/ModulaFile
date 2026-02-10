# 90 — First Branch Activation Orchestration (Subscription Start)

## Purpose

This process defines how a first-time tenant (with **zero branches**) activates their first paid branch and starts their billing month.

It exists to keep three truths consistent:
- **No unpaid branches exist** (branch is provisioned only after payment is confirmed).
- Tenant billing has **one anchor date**, set by first paid activation.
- Access becomes available immediately after payment verification (KHQR self-serve).

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/saas_governance/activating_first_branch_and_starting_subscription.md`

---

## Domains Involved

- Subscription & Entitlements (billing state, invoice, activation draft)
- Tenant (workspace boundary)
- Branch (system-provisioned branch)
- Audit (billing and provisioning evidence)
- Policy (defaults for new branch; forward-only config)

---

## When This Process Runs

Triggered when an owner/admin attempts to activate the first branch for a tenant that currently has zero branches.

---

## Inputs (Minimum)

- `tenant_id`
- activation draft fields (minimal branch identity, e.g., `branch_display_name`)
- `actor_id` (owner/admin)

Billing constraints (March baseline):
- invoice currency is USD-only
- payment collection primary rail is KHQR `Pay Now` with backend verification

---

## Orchestration Steps

### Step 1 — Validate eligibility

- Confirm `tenant_id` exists and is in a state that can be activated.
- Confirm tenant currently has **zero branches**.
- Confirm actor is allowed to activate branch capacity (Access Control).

---

### Step 2 — Create activation draft + issue invoice (USD)

- Create a `BranchActivationDraft` (pre-branch).
- Issue an invoice for "first branch activation" (USD).
- Record audit events (canonical names):
  - `BRANCH_ACTIVATION_INITIATED`
  - `SUBSCRIPTION_INVOICE_ISSUED`

Notes:
- This step must be idempotent so repeated attempts do not create multiple invoices for the same activation intent.

---

### Step 3 — Present `Pay Now (KHQR)` and wait for confirmation

- System displays a KHQR payment request for the invoice amount to Modula’s receiver.
- Backend verifies payment (do not trust client confirmation).
- Payment collection and confirmation details:
  - `BusinessLogic/4_process/60_PlatformSystems/95_payment_collection_and_confirmation_orchestration.md`
- While awaiting confirmation:
  - show a "waiting for payment" state
  - allow retry/poll without duplicating activation

Degradation:
- If KHQR verification cannot complete (network/service issues), activation remains pending; no branch is created.

---

### Step 4 — On payment confirmed: provision the real branch + set billing anchor

When payment is confirmed:
- Provision the real branch (system-only `ProvisionBranch` intent).
- Set the tenant billing anchor if not already set (this is the first paid activation).
- Initialize required branch defaults (example: tax/currency policy defaults).
- Mark the invoice `PAID`.
- Record audit events (canonical names):
  - `SUBSCRIPTION_INVOICE_PAID`
  - `BRANCH_ACTIVATED`
  - `BILLING_ANCHOR_SET`

---

### Step 5 — Return branch access

- The branch becomes selectable for operational work.
- The system returns the newly created `branch_id` for subsequent context selection.

---

## Failure / Degradation Rules (March)

- Never create a branch if payment is not confirmed.
- If provisioning fails after payment confirmation, the system must remain in a recoverable "activating" state (no duplicate charges, no duplicate branches).

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/subscription_billing_edge_case_sweep.md`
- Audit event catalog: `BusinessLogic/5_modSpec/60_PlatformSystems/subscriptionEntitlements_module.md`
