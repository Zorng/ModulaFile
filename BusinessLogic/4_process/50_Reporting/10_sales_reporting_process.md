# 10 — Sales Reporting (Management Insight)

## Purpose

This process generates **management sales reports** for owners/admins/managers by aggregating finalized sale facts.

It exists to answer questions like:
- "How did we perform this day/week?"
- "What payment types were used?"
- "Were there unusual voids or heavy discount usage?"

This is an **on-demand read process**.
It does not mutate sales, cash sessions, inventory, or attendance.

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/selling_and_checkout/reviewing_sales_performance.md`

---

## Domains Involved (Read-Only)

- Reporting domain (aggregation rules, final vs provisional presentation)
- Sale/Order (finalized sale records and status)
- Tenant/Branch (scope + frozen status; timezone context)
- Access Control (permission + scope enforcement)
- Audit Logging (observational report access logs)

Notes:
- Cash Session X/Z are **operational artifacts** owned by Cash Session.
  This process may reference them only as supporting drill-down, not as reporting truth.

---

## When This Process Runs

Triggered when:
- an owner/admin opens a sales dashboard
- a manager reviews branch performance

This process is not a scheduled job in March.

---

## Reporting Scope (Input)

The user selects:
- time window (day/week/month/custom range)
- branch scope:
  - a single branch (`branch_id`), OR
  - `ALL_BRANCHES` (tenant-wide; owners/admins only when allowed by Access Control)

March timezone assumption:
- boundaries are evaluated in Cambodia time (`Asia/Phnom_Penh`) until a per-branch timezone is introduced explicitly.

---

## Orchestration Steps

### Step 1 — Validate access and scope

- Ensure the actor has `reports.view`.
- Enforce branch scope:
  - managers are limited to a single branch at a time
  - `ALL_BRANCHES` (tenant-wide) is allowed only when the actor has explicit access to **all branches** in the tenant
- Allow reporting for frozen branches (read-only).

---

### Step 2 — Fetch sale facts for the scope

Retrieve sale records within the selected scope, including:
- sale identity + timestamps (sale_id, finalized_at, branch_id)
- sale status (`FINALIZED`, `VOID_PENDING`, `VOIDED`)
- finalized totals snapshot (stored at finalize time; no recompute), including at minimum:
  - grand totals (USD + KHR)
  - VAT amount (if enabled)
  - discount total (sum of applied discounts; from stored discount snapshot)
- sale type (dine-in / takeaway / delivery)
- line snapshot (for item/category aggregations), including at minimum:
  - menu_item_id + item display name snapshot
  - optional category snapshot (category_id + category display name) or explicit Uncategorized marker
  - quantity
  - line monetary snapshot (optional; enables revenue-per-item/category)
- tender/payment snapshot:
  - payment method (cash / QR; extend later)
  - cash tender currency (USD vs KHR) when payment method is cash (optional but recommended)

Important:
- Fetch **stored finalized values** (do not recompute totals using current policy).
- If `ALL_BRANCHES` is selected, fetch across all branches included in scope.

---

### Step 3 — Classify finalized vs provisional

Partition the result set:
- confirmed sales (FINALIZED)
- provisional exposure (VOID_PENDING)
- voided sales (VOIDED)

Rules:
- `VOID_PENDING` must remain visible as provisional exposure.
- `VOIDED` is excluded from confirmed totals.

---

### Step 4 — Aggregate report totals (deterministic)

Compute:
- confirmed sales totals (from FINALIZED only):
  - transaction count
  - total grand amount (USD + KHR)
  - VAT total (if enabled)
  - discount total (from stored discount snapshots)
  - average ticket size (optional derived metric)
  - total items sold (sum of finalized line quantities)
- tender/payment breakdowns (from FINALIZED only):
  - totals by payment method (cash vs QR)
  - totals by cash tender currency (USD vs KHR) (optional)
- order type comparison (from FINALIZED only):
  - by sale type (dine-in / takeaway / delivery): counts + revenue + item quantity
- top items by quantity (from FINALIZED only):
  - top N menu items by quantity (include item snapshot name; optionally include revenue)
- category comparison (from FINALIZED only):
  - totals by item category (quantity + optionally revenue), including Uncategorized
- exception visibility (shown separately; excluded from confirmed totals):
  - `VOID_PENDING` exposure: count + amount
  - `VOIDED`: count + amount

This step must be deterministic:
same scope + same underlying facts → same output.

---

### Step 5 — Provide drill-down views (when requested)

The UI may request:
- sale list for the window (with status labels)
- filtered lists (example: only VOID_PENDING)

Drill-down remains read-only.

---

### Step 6 — Audit the report access (observational)

Emit an observational audit event:
- `REPORT_VIEWED` with report type and scope metadata

This is not a state change; it is a traceability signal.

---

## Failure / Degradation Rules (March)

- If offline or data is stale:
  - management reporting may be blocked, OR
  - last-known results may be shown with an explicit stale indicator.

Reporting must not imply completeness when it cannot be guaranteed.

---

## Relationship to Other Processes

- This process is independent of finalize/void orchestration.
- It consumes the results of those processes through stable read models.
- It must respect March POS constraints from:
  - `BusinessLogic/3_contract/10_edgecases/pos_operation_edge_case_sweep_patched.md`
