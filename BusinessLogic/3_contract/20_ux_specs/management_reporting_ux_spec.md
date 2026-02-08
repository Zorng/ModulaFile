# UX Spec — Management Reporting (Sales + Attendance + Inventory)

## Metadata
- **Contract Type**: UX Spec
- **Scope**: Owner/Admin/Manager reporting flows for Sales performance + Attendance insights + Restock spend; inventory operational views remain in Inventory
- **Primary Audience**: Frontend, Backend, QA
- **Owner(s)**: Reporting, Sale/Order, Work Review, Attendance, Inventory, Access Control, Tenant/Branch
- **Last Updated**: 2026-02-08
- **Delivery Level**:
  - **March**: baseline behavior
  - **Later**: explicitly deferred
- **Related Contracts**:
  - `BusinessLogic/3_contract/10_edgecases/reporting_edge_case_sweep.md`
  - `BusinessLogic/3_contract/10_edgecases/pos_operation_edge_case_sweep_patched.md`
- **Related Processes**:
  - `BusinessLogic/4_process/10_WorkForce/40_attendance_report.md`
  - `BusinessLogic/4_process/50_Reporting/10_sales_reporting_process.md`
  - `BusinessLogic/4_process/50_Reporting/20_restock_spend_reporting_process.md`

---

## Purpose

This document defines the UX behavior that management reporting must support, without prescribing layouts.

It ensures:
- scope is explicit (time window + branch),
- provisional states remain visible,
- and management reports degrade safely when data is stale/offline.

---

## Core UX Principles

1) **Start with summary, allow drill-down**
   - Managers need answers first, then details only when something looks unusual.

2) **Scope must be obvious**
   - Every view clearly communicates the selected time window and branch scope.

3) **Provisional and exception states are not hidden**
   - `VOID_PENDING` and other provisional states must be visible and labeled.

4) **Honesty beats completeness**
   - When data is stale/offline, the UI must say so explicitly rather than implying completeness.

---

## 1. Entry and Access Rules

### Who Can Access Management Reporting

- Management reporting requires `reports.view` permission (role policy baseline: Manager/Admin).
- If a user lacks permission:
  - hide the entry point, or
  - show a deny state ("You don't have access to reports.").

### Scope Resolution Rules

- Branch picker options include only branches the actor can report on (Access Control enforced).
- If the actor can report on all branches, include an "All branches (tenant-wide)" option.
- If the user has access to only one branch, auto-select it.
- If multiple options are available, require explicit selection (do not default silently).
- Frozen branches remain selectable for reporting (read-only).

---

## 2. Sales Performance Reporting (Owner/Admin/Manager)

### Filters (Baseline)

- Time window: day / week / month / custom range
- Branch scope: selected branch (manager) or selected branch / ALL_BRANCHES (admin/owner when allowed)

### Output (Baseline)

The UI must support:
- a sales summary (totals and key breakdowns)
- visibility of exceptions (voids/refunds/provisional exposure)
- drill-down to the underlying sale list when needed

### Provisional Visibility Rules

- `VOID_PENDING` sales:
  - excluded from confirmed totals
  - visible as exposure (count/value)
  - present in drill-down with a provisional label

### Historical Stability Rules

- Reports must not change because policy/config changed later.
- If the UI shows current policy hints, it must be display-only and not affect totals.

---

## 3. Attendance Insights (Owner/Admin/Manager)

Attendance reporting is built from the Work Lifecycle pipeline.

### Filters (Baseline)

- Time window: day / week / month / custom range
- Branch scope: selected branch (manager) or selected branch / ALL_BRANCHES (admin/owner when allowed)
- Optional staff filter (drill-down)

### Output (Baseline)

The UI must support:
- per-staff summaries (planned vs actual when available)
- exceptions/pattern indicators (late/absent/early leave/overtime)
- drill-down to a staff member's detailed history

### Degradation Rule: No Shift Plan Data

If shifts are not planned for the reviewed period:
- the UI must not present "absence" as a fact
- show actual attendance history and label that planning data is missing

---

## 4. Inventory Visibility (March MVP)

Inventory "reporting" is shipped through the Inventory module's read surfaces:
- on-hand stock (branch scope)
- inventory journal/history
- restock batch history

In addition, management reporting includes a simple **restock spending** view:
- total restock spend for the selected time window and branch scope (single branch or ALL_BRANCHES when allowed)
- missing cost is shown as UNKNOWN (not treated as zero)

UX note:
- Restock spending must be labeled as "spend" / "purchases" (not "profit" or "COGS") in March.

Advanced inventory analytics (usage rate, waste trends) and inventory valuation/COGS are explicitly deferred.

---

## 5. Offline / Staleness UX (March Baseline)

When offline or when the client believes results may be stale:
- the UI may block management reporting and request connectivity, OR
- show last known results with a clear "stale" indicator.

The UI must not imply that the report is complete when it cannot be.

---

## Explicit Non-Goals (March)

- scheduled exports (email/PDF automation)
- forecasting and anomaly detection
- inventory valuation (COGS, FIFO/FEFO costing)
- advanced multi-branch comparison dashboards (beyond simple tenant-wide totals)

---

## Summary

Management reporting UX is defined by:
- explicit scope,
- drill-down from summary,
- visibility of provisional states,
- and honest degradation when data is stale.

This keeps owner/admin reporting useful without turning it into a second source of truth.

_End of UX Spec_
