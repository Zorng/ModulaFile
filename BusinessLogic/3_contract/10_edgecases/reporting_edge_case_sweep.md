# Edge Case Contract — Reporting (Management Insight)

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: Sales + Attendance + Restock spending management reporting; reporting trust rules (final vs provisional, scope, frozen branch, offline staleness)
- **Primary Audience**: Frontend, Backend, QA
- **Owner(s)**: Reporting, Sale/Order, Cash Session, Inventory, Attendance, Work Review, Access Control, Tenant/Branch
- **Last Updated**: 2026-02-08
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred
- **Related Domains**:
  - `BusinessLogic/2_domain/50_Reporting/reporting_domain.md`
  - `BusinessLogic/2_domain/40_POSOperation/order_domain_patched.md`
  - `BusinessLogic/2_domain/40_POSOperation/cashSession_domain.md`
  - `BusinessLogic/2_domain/40_POSOperation/inventory_domain.md`
  - `BusinessLogic/2_domain/30_HR/work_review_domain.md`
- **Related ModSpecs**:
  - `BusinessLogic/5_modSpec/40_POSOperation/sale_module_patched.md`
  - `BusinessLogic/5_modSpec/40_POSOperation/cashSession_module_patched_v2.md`
  - `BusinessLogic/5_modSpec/40_POSOperation/inventory_module_patched.md`
  - `BusinessLogic/5_modSpec/30_HR/attendance_module.md`
  - `BusinessLogic/5_modSpec/50_Reporting/report_module.md`
- **Related Contracts**:
  - `BusinessLogic/3_contract/10_edgecases/pos_operation_edge_case_sweep_patched.md`
- **Related Processes**:
  - `BusinessLogic/4_process/10_WorkForce/40_attendance_report.md`
  - `BusinessLogic/4_process/50_Reporting/10_sales_reporting_process.md`
  - `BusinessLogic/4_process/50_Reporting/20_restock_spend_reporting_process.md`

---

## Purpose

This contract locks the **minimum consistency rules** that make management reporting trustworthy:
- numbers do not shift silently after the fact,
- provisional/exception states remain visible,
- scope is explicit,
- and degraded states are handled safely (especially offline / staleness).

It prevents "report drift" where multiple modules compute the same totals differently.

---

## Definitions / Legend

- **Finalized**: a committed business outcome that should be stable for reporting (example: a finalized sale total).
- **Provisional**: a state pending approval or completion (example: `VOID_PENDING`).
- **Operational artifact**: a session-bound reconciliation view (example: X/Z in Cash Session).
- **Management report**: owner/admin oversight summaries across time windows (sales performance, attendance insights, restock spending).

**March timezone assumption:** Cambodia time (`Asia/Phnom_Penh`) is used for all day/week boundaries unless/until branch timezone is introduced explicitly.

---

## Edge Case Catalog

### EC-REP-01 — Policy/Configuration Changes Must Not Rewrite Past Reports
- **Scenario**: VAT rate, FX rate, rounding rules, or discount rules are changed after sales occurred.
- **Trigger**: User views a report for a historical time window.
- **Expected Behavior**:
  - Report totals are aggregated from **stored finalized values** captured at the time of finalize (no recompute using current policy).
  - If current policy is shown in the UI, it must be display-only and must not alter historical totals.
- **Owner**: Sale/Order (store snapshots) + Reporting (aggregate from snapshots)
- **March**: Yes

### EC-REP-02 — VOID_PENDING Must Be Visible (Never Silently Hidden)
- **Scenario**: A cashier requests a void and it is awaiting approval.
- **Trigger**: User views sales totals for a window containing `VOID_PENDING` sales.
- **Expected Behavior**:
  - `VOID_PENDING` sales appear as **provisional** in drill-down lists.
  - `VOID_PENDING` is excluded from **confirmed revenue totals**, but its exposure (count/value) is visible as a separate line/flag.
- **Owner**: Reporting (presentation rules) + Void process (state correctness)
- **March**: Yes

### EC-REP-03 — Voids/Refunds After Session Close Are Not Backfilled (March Baseline)
- **Scenario**: Business attempts to void/refund after the related cash session is CLOSED.
- **Trigger**: A request exists or a user expects reports to "fix" prior session close.
- **Expected Behavior**:
  - March baseline follows POS contract: void execution is blocked when the cash session is CLOSED.
  - Reporting must not invent retroactive adjustments to closed session artifacts.
  - If the UI needs to explain the situation, it should direct to the deferred "refund after close" workflow.
- **Owner**: Void orchestration + Cash Session + Reporting (messaging consistency)
- **March**: Yes (block + explain)
- **Later**: introduce explicit refund workflow (separate design)

### EC-REP-04 — Offline / Sync Lag (Stale Reporting Must Be Explicit)
- **Scenario**: Sales or attendance events were recorded offline and have not synchronized yet.
- **Trigger**: Owner/admin opens a management report while offline or shortly after reconnect.
- **Expected Behavior**:
  - Reporting is allowed to degrade for March:
    - Either block management reports while offline, OR
    - show the last known server/cached results with a clear **stale** indicator.
  - Reporting must never pretend the data is complete when it is not.
- **Owner**: Reporting UX + Offline Sync (data availability) + QA (testable states)
- **March**: Yes (explicit degradation)

### EC-REP-05 — Frozen Branch Remains Reportable (Read-Only History)
- **Scenario**: A branch is frozen after operations (billing freeze / temporary suspension).
- **Trigger**: User views historical reports for that branch.
- **Expected Behavior**:
  - Reports remain readable for frozen branches.
  - Branch is clearly labeled as frozen in report scope.
  - No operational actions are enabled from reporting surfaces.
- **Owner**: Branch domain + Reporting
- **March**: Yes

### EC-REP-10 — Restock Spend: Missing Cost Is UNKNOWN (Not Zero)
- **Scenario**: Restock records exist but some batches do not have purchase cost recorded.
- **Trigger**: Owner/admin views restock spending for a time window.
- **Expected Behavior**:
  - Spending totals must **not** treat missing cost as zero.
  - Report must show:
    - total spend from batches with known cost
    - count (and optionally list) of batches with unknown cost
  - Drill-down must allow the owner to find and correct missing cost data.
- **Owner**: Inventory (restock metadata) + Reporting (aggregation/presentation)
- **March**: Yes

### EC-REP-06 — Archived/Disabled Staff Must Not Break Attendance Reports
- **Scenario**: A staff member is disabled or archived after leaving the business.
- **Trigger**: Manager views a time window that includes that staff member's history.
- **Expected Behavior**:
  - Historical attendance and work review summaries remain visible.
  - Staff identity is preserved for reporting (at minimum: display name and stable ID).
  - Archiving affects active rosters, not historical reporting.
- **Owner**: Staff Profile & Assignment + Work Review + Reporting
- **March**: Yes

### EC-REP-07 — No Shift Plan Data (Attendance Insight Degrades Fairly)
- **Scenario**: The tenant did not configure shift planning for the reviewed period.
- **Trigger**: Manager opens attendance/time-respecting report.
- **Expected Behavior**:
  - Reporting must not fabricate "absent" judgments without planned shifts.
  - Present actual attendance history and clearly label that planning data is missing.
  - If Work Review generates classifications, they must remain explainable (example: UNSCHEDULED_WORK rather than ABSENT).
- **Owner**: Work Review domain + Attendance reporting process + Reporting UX
- **March**: Yes

### EC-REP-08 — Role/Scope Restrictions Must Hold Under Filtering
- **Scenario**: A manager attempts to view tenant-wide data or another branch they are not assigned to.
- **Trigger**: User changes report scope filters.
- **Expected Behavior**:
  - Access Control must enforce scope consistently (deny with a clear reason).
  - UI should not expose branches/staff that the actor cannot report on.
  - Tenant-wide scope (`ALL_BRANCHES`) is only allowed when the actor has explicit access to **all branches** in the tenant.
- **Owner**: Access Control + Branch assignment + Reporting UX
- **March**: Yes

### EC-REP-09 — Day Boundaries Must Be Consistent (Timezone Rule)
- **Scenario**: A sale occurs near midnight, or a shift crosses midnight.
- **Trigger**: User views a "daily" report.
- **Expected Behavior**:
  - The same timezone rule is used across sales and attendance reporting.
  - Day/week boundaries are consistent and documented (March assumption: `Asia/Phnom_Penh`).
- **Owner**: Reporting + OrgAccount (timezone source, later)
- **March**: Yes (assumption-based)

---

## Summary

For March, reporting must be:
- explicit about scope,
- honest about provisional and stale states,
- and stable against retroactive recomputation.

This contract defines the edge cases that must be handled consistently so management reports can be trusted.

_End of Reporting edge case contract_
