# Subscription & Entitlements Module — Platform Module (Billing Guard Rails)

**Version:** 1.0  
**Status:** Draft (March baseline: USD-only invoices; KHQR pay-now; branch-scoped entitlements; single tenant billing anchor)  
**Module Type:** Platform / Supporting Module  
**Depends on:** Access Control (authorization + entitlement enforcement), Tenant & Branch (OrgAccount), Audit Logging (evidence), Idempotency (replay safety), Offline Sync (cache)  
**Related:** Policy (defaults), Reporting (invoice views), Operational Notification (optional awareness)

---

## 1. Purpose

This module makes Modula's "pay for what you use" enforceable by providing:
- tenant subscription state (`ACTIVE` / `PAST_DUE` / `FROZEN`),
- explicit entitlements (branch-scoped modules, optional add-ons, operator seats),
- and upgrade/downgrade orchestration inputs (invoice issuance, downgrade pending).

This module is not a payment SDK. It defines enforcement state and evidence so other modules can behave consistently.

Authoritative domain:
- `BusinessLogic/2_domain/60_PlatformSystems/subscription_entitlements_domain.md`

Authoritative edge cases:
- `BusinessLogic/3_contract/10_edgecases/subscription_billing_edge_case_sweep.md`

Authoritative processes:
- `BusinessLogic/4_process/60_PlatformSystems/90_first_branch_activation_orchestration.md`
- `BusinessLogic/4_process/60_PlatformSystems/91_enable_branch_module_or_seats_proration_process.md`
- `BusinessLogic/4_process/60_PlatformSystems/92_disable_branch_module_downgrade_pending_process.md`
- `BusinessLogic/4_process/60_PlatformSystems/93_subscription_renewal_grace_and_freeze_orchestration.md`
- `BusinessLogic/4_process/60_PlatformSystems/94_additional_branch_activation_orchestration.md`
- `BusinessLogic/4_process/60_PlatformSystems/95_payment_collection_and_confirmation_orchestration.md`

---

## 2. Scope (March Baseline)

Included:
- first-branch activation (zero-branch entry) using activation draft + invoice + payment confirmation
- additional branch activation (upgrade mid-cycle with proration intent)
- enable branch modules/add-ons/seats (upgrade; immediate after payment)
- disable branch modules/add-ons (end-of-cycle; Workforce uses `DOWNGRADE_PENDING`)
- renewal invoices at billing anchor; `PAST_DUE` grace; `FROZEN` enforcement; recovery to `ACTIVE`
- invoice history viewing + PDF download (read-only)
- branch archive/delete does not create reusable free branch activation entitlement by default

Excluded (explicit):
- pricing tables and marketing plans (business artifacts live in `secrets/*`)
- refunds, chargebacks, and complex settlement workflows
- multi-currency billing (USD-only invoices for March)
- automated dunning beyond `PAST_DUE` warnings

---

## 3. Canonical Read Interfaces (Facts)

- `GetSubscriptionState(tenant_id) -> ACTIVE | PAST_DUE | FROZEN`
- `GetEntitlementSnapshot(tenant_id, branch_id?) -> { entitlement_key -> enforcement }`
- `GetOperatorSeatLimit(tenant_id, branch_id) -> { included_seats, total_seats }`
- `ListInvoices(tenant_id) -> invoice summaries`
- `GetInvoice(invoice_id) -> invoice detail`

These reads feed:
- Access Control (deny writes when frozen; deny writes when entitlement read-only/disabled)
- UX (display plan state + warnings without inventing rules)

---

## 3.1 Denial Code Direction (Billing)

Use payment/subscription-oriented denial reason codes for branch monetization and upgrades:
- `BRANCH_ACTIVATION_PAYMENT_REQUIRED`
- `SUBSCRIPTION_UPGRADE_REQUIRED`

Avoid slot-capacity denial semantics (for example `BRANCH_SLOT_LIMIT_REACHED`) in branch billing flows.

---

## 4. Entitlement Keys (Baseline)

Branch-scoped modules:
- `core.pos` (implicitly enabled when a branch exists/paid)
- `module.inventory`
- `module.workforce`

Branch-scoped add-ons:
- `addon.workforce.gps_verification`

Branch-scoped capacity:
- `capacity.operator_seats`

---

## 5. Audit Events (Locked For March)

These event types are the canonical names used by Subscription/Entitlements processes.
They must be recorded using the Audit Logging module and remain stable once introduced.

### Invoice
- `SUBSCRIPTION_INVOICE_ISSUED`
- `SUBSCRIPTION_INVOICE_PAID`

### Branch Activation
- `BRANCH_ACTIVATION_INITIATED` (draft created; no branch yet)
- `BRANCH_ACTIVATED` (real branch provisioned after payment)
- `BILLING_ANCHOR_SET` (only once; first paid branch)

### Branch Plan Changes (Modules/Add-ons/Seats)
- `BRANCH_PLAN_UPGRADE_APPLIED` (after payment; entitlements/seats changed)
- `BRANCH_PLAN_DOWNGRADE_REQUESTED` (end-of-cycle scheduled)
- `BRANCH_PLAN_DOWNGRADE_COMPLETED` (effective after safe boundary / effective date)

### Subscription State Transitions
- `SUBSCRIPTION_PAST_DUE_ENTERED`
- `SUBSCRIPTION_FROZEN_ENTERED`
- `SUBSCRIPTION_ACTIVE_RESTORED`

### Required Metadata (Minimum)

All events must include:
- `tenant_id`
- `actor_id` (`SYSTEM` allowed)
- `outcome` (`SUCCESS | REJECTED | FAILED`)

Context rules:
- If the action is branch-scoped, include `branch_id`.
- Invoice-related events include `invoice_id` in `entity_refs`.
- Plan change events include changed keys in metadata (entitlements/seats).

Reason code alignment:
- If Access Control denies an attempted plan change, record the Access Control `reason_code`.

---

## 6. Out of Scope (Future)

- detailed invoice tax treatment (VAT/GST)
- couponing for subscription discounts
- third-party billing provider integrations
