# Reporting Module

**Version:** 1.4  
**Status:** Draft (March scope: Sales management + Attendance insights; cash closeout surfaced as operational artifacts)  
**Module Type:** Feature Module  
**Depends on:**
- Authentication (Core)
- Tenant & Branch Context (Core)
- Access Control (Core)
- Sale/Order (Feature; source of finalized sales truth)
- Attendance + Work Review (HR; interpreted attendance insight)
- Cash Session (Feature; operational closeout artifacts X/Z)
- Inventory (Feature; restock batch metadata for spend visibility)
- Audit Logging (Core)

---

## 1. Purpose

The Reporting module provides **read-only management insight** by aggregating and presenting business facts owned by other domains.

For March, the primary outcomes are:
- **Sales performance reporting** (owners/admins/managers)
- **Attendance insight reporting** (owners/admins/managers) using Work Review summaries
- **Restock spending visibility** (owners/admins/managers) using Inventory restock batch metadata

Cash session X/Z views are treated as **operational closeout artifacts** owned by Cash Session.
Reporting may surface them for navigation/review, but must not redefine their meaning or mutate their history.

Authoritative references:
- Reporting trust rules: `BusinessLogic/2_domain/50_Reporting/reporting_domain.md`
- Reporting edge cases: `BusinessLogic/3_contract/10_edgecases/reporting_edge_case_sweep.md`
- Management reporting UX behavior: `BusinessLogic/3_contract/20_ux_specs/management_reporting_ux_spec.md`
- Sales reporting process: `BusinessLogic/4_process/50_Reporting/10_sales_reporting_process.md`
- Restock spend reporting process: `BusinessLogic/4_process/50_Reporting/20_restock_spend_reporting_process.md`
- Attendance reporting process: `BusinessLogic/4_process/10_WorkForce/40_attendance_report.md`
- Cash Session X/Z artifacts (owner-of-truth): `BusinessLogic/5_modSpec/40_POSOperation/cashSession_module_patched_v2.md`

---

## 2. Scope (March Baseline)

Included:
- Sales performance reporting:
  - time window filters (day/week/month/custom)
  - branch scope filters (single branch or `ALL_BRANCHES` tenant-wide when allowed; enforced by Access Control)
  - summary + drill-down lists
  - provisional exposure visibility (`VOID_PENDING`)
- Attendance insight reporting:
  - time window filters (day/week/month/custom)
  - branch scope filters (single branch or `ALL_BRANCHES` tenant-wide when allowed)
  - summary + drill-down (per staff)
  - fair degradation when shift plan data is missing
- Restock spend reporting:
  - time window filters (month/custom range)
  - branch scope filters (single branch or `ALL_BRANCHES` tenant-wide when allowed)
  - totals + missing-cost visibility + drill-down
- Frozen branches remain reportable (read-only history)
- Report access is audit logged (observational)

Excluded (explicit):
- Export automation (email, scheduled PDF)
- Real-time dashboards and streaming analytics
- Forecasting and anomaly detection
- Inventory valuation / COGS (requires explicit costing method design)
- Inventory usage/waste analytics beyond existing Inventory views (deferred story)
- advanced multi-branch comparison dashboards (beyond simple tenant-wide totals)

---

## 3. Actors & Access Control

All reporting is **read-only**.

### 3.1 Management Reporting (Sales + Attendance + Restock Spend)

Requires `reports.view`.

- **Owner / Admin**
  - May view per-branch, and may view `ALL_BRANCHES` (tenant-wide) when Access Control allows (requires explicit access to all branches)
- **Manager**
  - May view within single-branch scope only (no `ALL_BRANCHES`)
- **Cashier**
  - No management reporting access by default

### 3.2 Cash Session Closeout Review (X/Z Operational Artifacts)

X/Z are owned by Cash Session.
If the UI surfaces them under "Reports", scope must follow Cash Session rules:
- **Cashier**: view only sessions they opened (March baseline)
- **Manager / Admin / Owner**: view sessions in branch scope

---

## 4. Core Concepts

### 4.1 Final vs Provisional

Reports must distinguish between:
- **Finalized facts** (stable outcomes)
- **Provisional facts** (pending approval/completion)

`VOID_PENDING` is provisional and must never be silently hidden.

### 4.2 Policy vs Reporting Truth (No Retroactive Recompute)

Reports must aggregate totals from **stored finalized values** captured at the time of sale finalization.

Policies may be used only for display hints (example: label that KHR rounding was enabled),
but must not change historical totals.

### 4.3 Scope and Time Boundaries

Every report is queried with explicit scope:
- tenant
- branch scope (single branch or `ALL_BRANCHES`)
- time window

March timezone assumption:
- day/week boundaries use Cambodia time (`Asia/Phnom_Penh`) until branch timezone is introduced explicitly.

### 4.4 Frozen Branch Reporting

Frozen branches remain reportable (read-only history) and must be labeled as frozen.

---

## 5. Use Cases

### UC-1: View Sales Performance (Summary)

**Actors:** Owner, Admin, Manager (`reports.view`)

**Main Flow:**
1. Actor opens Reports → Sales.
2. Actor selects a scope (time window + branch or `ALL_BRANCHES`).
3. System returns:
   - confirmed sales totals (FINALIZED only)
   - key breakdowns (tender/payment, discounts, VAT if applicable)
   - exception visibility (VOIDED + VOID_PENDING), shown separately

**Acceptance Criteria:**
- Uses stored finalized values (no recompute).
- Shows `VOID_PENDING` exposure explicitly.

**Minimum Output (March Baseline):**
- **Scope echo**: tenant + branch scope + time window + timezone rule
- **Confirmed (FINALIZED only)**:
  - transaction count
  - total grand amount (USD + KHR totals, aggregated from stored sale snapshots)
  - VAT total (if enabled; aggregated from stored sale snapshots)
  - discount total (sum of discounts applied; aggregated from stored sale discount snapshots)
  - average ticket size (optional derived metric)
  - total items sold (sum of finalized line item quantities)
- **Tender / payment breakdown (FINALIZED only)**:
  - by payment method (cash / QR; extend later)
  - for cash: breakdown by tender currency (USD vs KHR) (optional)
- **Order type comparison (FINALIZED only)**:
  - by sale type (dine-in / takeaway / delivery): counts + revenue + item quantity
- **Top items (FINALIZED only)**:
  - top N sold menu items by quantity (include item snapshot display name; show quantity and optionally revenue)
- **Category comparison (FINALIZED only)**:
  - totals by menu category (quantity + optionally revenue)
  - include `Uncategorized` where category is missing
- **Exceptions / visibility**:
  - `VOID_PENDING` exposure: count + amount (shown separately; excluded from confirmed totals)
  - `VOIDED`: count + amount (shown separately; excluded from confirmed totals)

Notes:
- Amounts must come from **stored finalized snapshots** (including FX/VAT/rounding snapshots at the time of finalize). Reporting must not “recompute” using current policy.
- Item, order-type, and category aggregations must be derived from the **stored sale line snapshot** captured at finalize (do not join to current Menu in a way that rewrites historical reports when items/categories are edited later).
- “Confirmed totals” are about sales performance, not cash drawer reconciliation. Cash drawer accountability remains owned by Cash Session (X/Z + cash movements).

---

### UC-2: Drill Down Into Sales (Lists + Exceptions)

**Actors:** Owner, Admin, Manager (`reports.view`)

**Main Flow:**
1. Actor drills down from the sales summary.
2. System shows:
   - sale list within scope (status labeled)
   - filtered exception lists (example: VOID_PENDING)
3. Actor opens sale detail/eReceipt (read-only).

---

### UC-3: View Attendance Insights (Work Review Summary)

**Actors:** Owner, Admin, Manager (`reports.view`)

**Main Flow:**
1. Actor opens Reports → Attendance.
2. Actor selects scope (time window + branch or `ALL_BRANCHES` + optional staff).
3. System returns Work Review summaries:
   - planned vs actual (when shift planning exists)
   - explainable indicators (late/absent/early leave/overtime)
   - pattern hints (non-judgmental)

**Acceptance Criteria:**
- If shift planning data is missing, reporting degrades fairly (does not fabricate "absence").

**Minimum Output (March Baseline):**
- **Scope echo**: tenant + branch scope + time window + timezone rule
- **Per-staff summary list** (within scope):
  - planned shift count (only when shift planning exists for the window)
  - attended count
  - classification counts (from Work Review): ON_TIME / LATE / EARLY_LEAVE / ABSENT / OVERTIME / UNSCHEDULED_WORK / INCOMPLETE_RECORD
  - total scheduled hours (only when shift planning exists)
  - total worked hours
  - total overtime minutes (optional derived metric; sum of Work Review `overtime_minutes`)
  - total late minutes (optional derived metric; sum of Work Review `late_minutes`)
- **Branch summary** (aggregate across staff):
  - totals for the above metrics within scope
- **Fair degradation / data quality**:
  - if no shift planning exists in the selected window:
    - do not show absences or "planned shifts"
    - show actual attendance/worked-time driven metrics and UNSCHEDULED_WORK visibility only
- **Drill-down**:
  - list Work Review entries with: staff identity, branch, expected vs actual times, classification, evidence notes, and optional location results (MATCH/MISMATCH/UNKNOWN where recorded)

---

### UC-4: View Cash Session Closeout (X/Z Artifacts)

**Actors:** Cashier (own sessions), Manager, Admin, Owner

**Main Flow:**
1. Actor opens Reports → Cash Sessions.
2. System lists sessions in scope (date range + optional status).
3. Actor opens a session detail:
   - X snapshot for OPEN sessions
   - Z closeout summary for CLOSED sessions

**Acceptance Criteria:**
- Cashier is limited to sessions they opened.
- Read-only; no mutation of ledger/history.

**Minimum Output (March Baseline):**
- **Session list**:
  - session status (OPEN/CLOSED)
  - opened_at/closed_at
  - opened_by (and closed_by where applicable)
  - branch identity
- **X snapshot (OPEN)**:
  - opening float
  - sum of cash movements so far
  - expected cash-in-drawer at this moment
- **Z summary (CLOSED)**:
  - opening float
  - total cash movements
  - counted cash
  - variance
  - closure metadata (who, when, reason)

Note:
- X/Z are operational reconciliation artifacts owned by Cash Session (see `BusinessLogic/5_modSpec/40_POSOperation/cashSession_module_patched_v2.md`). Reporting surfaces them read-only and must not reinterpret their logic.

---

### UC-5: View Restock Spend (Summary)

**Actors:** Owner, Admin, Manager (`reports.view`)

**Main Flow:**
1. Actor opens Reports → Inventory Spend.
2. Actor selects scope (time window + branch or `ALL_BRANCHES`).
3. System returns:
   - total restock spend from batches with known purchase cost
   - count of restock batches with unknown cost
4. Actor drills down to restock batch history (read-only) and may identify missing cost records.

**Acceptance Criteria:**
- Missing cost is treated as UNKNOWN (not zero).
- Reporting is read-only and audit logged.

**Minimum Output (March Baseline):**
- **Scope echo**: tenant + branch scope + time window + timezone rule
- **Spending summary**:
  - total restock spend from batches with known purchase cost
  - count of batches with unknown cost (not treated as zero)
  - optional time breakdown (sum per month) when the selected window spans multiple months
- **Drill-down**:
  - list restock batches included in the scope, including stock item, quantity, received_at, branch, and purchase cost (known/unknown)

---

## 6. Reporting Rules (Locked for March)

- `VOID_PENDING` sales:
  - visible and labeled as provisional
  - excluded from confirmed totals
  - included in exposure summaries (count/value)
- Historical totals:
  - aggregated from stored finalized values
  - never recomputed from current policy/config
- Offline/staleness:
  - reporting may be blocked while offline, or show last-known results with an explicit stale indicator
- Frozen branches:
  - remain reportable as read-only history
- Restock spend:
  - missing cost is UNKNOWN (not treated as zero)

---

## 7. Non-Functional Requirements

- Deterministic: same scope + same facts → same output
- Auditable: report access is logged (observational)
- Role-restricted: scope enforcement is consistent across filters

---

## Audit Events (Observational)

- `REPORT_VIEWED` (include report type + scope metadata)
- `REPORT_EXPORTED` (future; export is out-of-scope for March)
