# Edge Case Contract — Discount (Eligibility, Scope, Snapshot Integrity)

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: Discount rule lifecycle, eligibility evaluation, branch scoping, finalize/open-ticket snapshot integrity, and reporting stability
- **Primary Audience**: Backend, Frontend, QA
- **Owner(s)**: Discount, Sale/Order, Finalize Sale Orchestration, Reporting, Menu, Access Control
- **Last Updated**: 2026-02-18
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
- **Scenario**: A discount rule exists but belongs to a different branch than where the sale is happening (branch-owned rules).
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

### EC-DIS-11 — Item-Level Multi-Item Rule Includes Invalid Items For Branch
- **Scenario**: Admin selects one or more items that are not eligible targets in the rule’s branch.
- **Trigger**: Create/update item-level discount rule (branch-owned).
- **Expected Behavior**:
  - Backend preflight `ResolveEligibleItemsForBranch(branch_id, item_ids)` returns invalid items (or empty eligible set).
  - Rule save is rejected with explicit message ("some selected items are not eligible in this branch").
- **Owner**: Discount + Menu
- **March**: Yes

### EC-DIS-12 — Item Set Update Introduces Invalid Items For Branch
- **Scenario**: Admin edits item targets and introduces items that are not eligible targets in the rule’s branch.
- **Trigger**: Update item-level discount rule (branch-owned).
- **Expected Behavior**:
  - Backend re-validates item eligibility via `ResolveEligibleItemsForBranch(branch_id, item_ids)`.
  - Save is rejected until the invalid items are removed.
  - No partial save with invalid targets.
- **Owner**: Discount
- **March**: Yes

### EC-DIS-13 — Later Menu/Branch Mapping Drift
- **Scenario**: After rule was saved, menu/branch mappings change and some targeted items are no longer valid in the rule’s branch.
- **Trigger**: Eligibility evaluation for new sales.
- **Expected Behavior**:
  - Existing rule record is not silently rewritten.
  - Runtime eligibility excludes invalid targets.
  - Admin receives warning signal in management UI to review/update rule.
- **Owner**: Discount + Menu + Frontend
- **March**: Yes

### EC-DIS-14 — Overlapping Rules at Configuration Time (Warn-Only)
- **Scenario**: Admin configures a new discount rule that overlaps with existing rules in the same branch and schedule window (example: item-level rules overlapping on an item).
- **Trigger**: Create/update discount rule.
- **Expected Behavior**:
  - Overlap is evaluated within the same `branch_id` against existing `ACTIVE` rules (ignore `INACTIVE` and `ARCHIVED`).
  - Overlap requires both:
    - schedule window overlap (`[start_at, end_at)` semantics), and
    - target overlap (item intersection for item-level rules; branch-wide overlaps all items in the branch).
  - System warns and requires explicit confirmation before saving:
    - recommended warning code: `DISCOUNT_RULE_OVERLAP_WARNING`
    - include conflicting rule ids so UI can show what overlaps
  - Save is allowed after confirmation; runtime stacking remains multiplicative and deterministic.
- **Owner**: Discount + Frontend
- **March**: Yes

### EC-DIS-15 — Attempt to Edit a Currently-Eligible Rule
- **Scenario**: Admin attempts to edit a discount rule that is currently eligible (ACTIVE and within its schedule window).
- **Trigger**: Update discount rule.
- **Expected Behavior**:
  - Reject mutation with a clear message ("deactivate first" or "rule currently active").
  - Recommended denial code: `DISCOUNT_RULE_UPDATE_REQUIRES_EFFECTIVE_INACTIVE`.
  - Admin can deactivate, edit, then activate.
- **Owner**: Discount + Frontend
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
