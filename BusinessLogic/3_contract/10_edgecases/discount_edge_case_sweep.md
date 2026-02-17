# Edge Case Contract — Discount (Eligibility, Scope, Snapshot Integrity)

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: Discount rule lifecycle, eligibility evaluation, branch scoping, finalize/open-ticket snapshot integrity, and reporting stability
- **Primary Audience**: Backend, Frontend, QA
- **Owner(s)**: Discount, Sale/Order, Finalize Sale Orchestration, Reporting, Menu, Access Control
- **Last Updated**: 2026-02-15
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred
- **Related Domains**:
  - `BusinessLogic/2_domain/40_POSOperation/discount_domain.md`
  - `BusinessLogic/2_domain/40_POSOperation/order_domain_patched.md`
- **Related Processes**:
  - `BusinessLogic/4_process/30_POSOperation/06_place_order_open_ticket_process.md`
  - `BusinessLogic/4_process/30_POSOperation/08_checkout_open_ticket_process.md`
  - `BusinessLogic/4_process/30_POSOperation/10_finalize_sale_orch.md`
  - `BusinessLogic/4_process/50_Reporting/10_sales_reporting_process.md`
- **Related ModSpecs**:
  - `BusinessLogic/5_modSpec/40_POSOperation/discount_module_patched.md`
  - `BusinessLogic/5_modSpec/40_POSOperation/sale_module_patched.md`

---

## Purpose

This contract defines the discount edge-case behavior that must remain consistent across:
- Discount rule management,
- Sale finalization and open-ticket checkout,
- historical reporting.

It prevents pricing drift and "same cart, different result" behavior.

---

## Definitions / Legend

- **Eligible rule**: a rule that passes all scope/time/status checks for the current branch/cart context.
- **Preview**: a client-side or pre-finalize estimate.
- **Snapshot**: immutable finalized record used for receipt/reporting.
- **Open Ticket**: pay-later order with stored batch money snapshots.

---

## Edge Case Catalog

### EC-DIS-01 — Rule Exists but Not Assigned to Active Branch
- **Scenario**: A discount rule is active in tenant scope but not assigned to the branch where the sale is happening.
- **Trigger**: Finalize or place-order eligibility evaluation.
- **Expected Behavior**:
  - Rule is ignored.
  - No cross-branch leakage of discounts.
- **Owner**: Discount
- **March**: Yes

### EC-DIS-02 — Rule Is INACTIVE/ARCHIVED
- **Scenario**: A rule was active before, then deactivated or archived.
- **Trigger**: Eligibility evaluation for new sales.
- **Expected Behavior**:
  - Rule is never returned as eligible for new sales.
  - Historical finalized sales keep previous snapshots unchanged.
- **Owner**: Discount + Sale/Order
- **March**: Yes

### EC-DIS-03 — Discount Value Out of Valid Range
- **Scenario**: Admin attempts to create/update a percentage rule outside valid range.
- **Trigger**: Rule mutation.
- **Expected Behavior**:
  - Reject mutation.
  - Do not persist invalid rule state.
- **Owner**: Discount
- **March**: Yes

### EC-DIS-04 — Schedule Boundary Ambiguity
- **Scenario**: Rule starts/ends exactly at boundary time.
- **Trigger**: Eligibility at schedule edge.
- **Expected Behavior**:
  - Time window semantics are deterministic and shared by backend/frontend.
  - March lock: treat schedule as `[start_at, end_at)` in `Asia/Phnom_Penh` unless tenant timezone is introduced.
- **Owner**: Discount + Frontend + QA
- **March**: Yes

### EC-DIS-05 — Referenced Item/Category No Longer Active
- **Scenario**: Item-level/category-targeted rule references menu entities that were archived/removed.
- **Trigger**: Eligibility evaluation.
- **Expected Behavior**:
  - Rule is not eligible for missing/inactive targets.
  - Rule remains editable/auditable by admins (not silently deleted).
- **Owner**: Discount + Menu
- **March**: Yes

### EC-DIS-06 — Same Rule Applies Through Multiple Paths
- **Scenario**: A cart line qualifies for multiple rules (for example item-level + branch-wide + multiple item-level rules).
- **Trigger**: Sale evaluation.
- **Expected Behavior**:
  - Rules are combined via locked multiplicative policy.
  - Result is deterministic across retries and devices.
- **Owner**: Discount (policy) + Sale (money math)
- **March**: Yes

### EC-DIS-07 — Client Preview Differs from Finalize Result
- **Scenario**: UI preview shows one value but backend finalize computes another because rules changed or stale data was cached.
- **Trigger**: Finalize request.
- **Expected Behavior**:
  - Backend finalize result is authoritative.
  - UI must show finalized totals from server snapshot.
  - No client-side override of finalized truth.
- **Owner**: Sale/Finalize + Frontend
- **March**: Yes

### EC-DIS-08 — Discount Rule Changed After Open Ticket Placed
- **Scenario**: Pay-later ticket was placed, then discount rules changed before checkout.
- **Trigger**: `Checkout Open Ticket`.
- **Expected Behavior**:
  - Checkout uses ticket stored batch money snapshots (no surprise recompute).
  - Discount edits affect only future place-order/finalize paths, not existing open-ticket payable totals.
- **Owner**: Place Order + Checkout Open Ticket + Sale/Order
- **March**: Yes

### EC-DIS-09 — Duplicate Finalize / Retry Must Not Duplicate Discount Effects
- **Scenario**: Finalize retried due to double tap/network/offline replay.
- **Trigger**: Duplicate finalize for same `(branch_id, sale_id)`.
- **Expected Behavior**:
  - Return existing finalized snapshot (no-op).
  - No duplicate discount application records.
- **Owner**: Finalize Sale + Idempotency
- **March**: Yes

### EC-DIS-10 — Rule Edit After Sale Finalization
- **Scenario**: Admin edits or archives a discount rule after sales were finalized.
- **Trigger**: Reporting or receipt rendering for historical window.
- **Expected Behavior**:
  - Historical reports and receipts use stored sale snapshots.
  - No recomputation from current discount rules.
- **Owner**: Sale/Order + Reporting + Receipt
- **March**: Yes

### EC-DIS-11 — Item-Level Multi-Item Rule Has No Common Branch
- **Scenario**: Admin selects multiple items that do not share any valid branch intersection.
- **Trigger**: Create/update item-level discount rule.
- **Expected Behavior**:
  - Backend preflight `ResolveAvailableBranchesForItems(item_ids)` returns empty set.
  - Rule save is rejected with explicit message ("selected items do not share common branches").
- **Owner**: Discount + Menu
- **March**: Yes

### EC-DIS-12 — Item Set Change Shrinks Available Branches
- **Scenario**: Admin edits item targets; available branch intersection becomes smaller than previously selected branches.
- **Trigger**: Update item-level discount rule.
- **Expected Behavior**:
  - Backend re-resolves available branches.
  - Save is rejected unless selected branches are adjusted to valid subset.
  - No partial save with invalid branches.
- **Owner**: Discount
- **March**: Yes

### EC-DIS-13 — Later Menu/Branch Mapping Drift
- **Scenario**: After rule was saved, menu/branch mappings change and some targeted items are no longer valid in some selected branches.
- **Trigger**: Eligibility evaluation for new sales.
- **Expected Behavior**:
  - Existing rule record is not silently rewritten.
  - Runtime eligibility excludes invalid targets/branches.
  - Admin receives warning signal in management UI to review/update rule.
- **Owner**: Discount + Menu + Frontend
- **March**: Yes

---

## Summary

For March, discount behavior must be:
- branch-safe,
- schedule-deterministic,
- snapshot-immutable,
- and idempotent under retry.

This contract is the QA reference for those guarantees.

_End of Discount edge case contract_
