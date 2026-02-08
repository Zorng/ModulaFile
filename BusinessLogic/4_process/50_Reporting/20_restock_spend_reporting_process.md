# 20 — Restock Spend Reporting (Management Insight)

## Purpose

This process produces a **restock spending summary** so owners/admins can understand how much money is being spent to supply the business.

It is an **on-demand read process**.
It does not mutate inventory or cash movements.

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/managing_stock/reviewing_restock_spending.md`

---

## Domains Involved (Read-Only)

- Reporting domain (trust rules and scope semantics)
- Inventory (restock batch metadata, including optional purchase cost)
- Tenant/Branch (scope + frozen status; timezone context)
- Access Control (permission + scope enforcement)
- Audit Logging (observational report access logs)

---

## When This Process Runs

Triggered when:
- an owner/admin reviews monthly spending
- a manager reviews branch restock spending

This is not a scheduled job in March.

---

## Reporting Scope (Input)

The user selects:
- time window (month or custom range)
- branch scope:
  - a single branch (`branch_id`), OR
  - `ALL_BRANCHES` (tenant-wide; owners/admins only when allowed by Access Control)

March timezone assumption:
- boundaries are evaluated in Cambodia time (`Asia/Phnom_Penh`) until a per-branch timezone is introduced explicitly.

---

## Orchestration Steps

### Step 1 — Validate access and scope

- Ensure the actor has `reports.view`.
- Enforce branch scope via Access Control:
  - managers are limited to a single branch at a time
  - `ALL_BRANCHES` (tenant-wide) is allowed only when the actor has explicit access to **all branches** in the tenant
- Allow reporting for frozen branches (read-only).

---

### Step 2 — Fetch restock batches for the scope

Retrieve restock batch records within the selected scope, including:
- received_at
- stock item reference
- quantity
- purchase cost (if recorded)

Important:
- Purchase cost is optional metadata and may be missing.
- If `ALL_BRANCHES` is selected, fetch across all branches included in scope.

---

### Step 3 — Aggregate spending (deterministic)

Compute:
- total restock spend from batches where purchase cost is known
- count of batches with unknown cost (do not treat as zero)

If needed for the UI, also compute:
- monthly breakdown within the range (sum per month)

---

### Step 4 — Provide drill-down for investigation

Allow drill-down to:
- list of restock batches included in the summary
- filtered list of "missing cost" batches so the owner/admin can correct metadata

---

### Step 5 — Audit the report access (observational)

Emit an observational audit event:
- `REPORT_VIEWED` with report type and scope metadata

---

## Failure / Degradation Rules (March)

- If offline or data is stale:
  - restock spending reports may be blocked, OR
  - last-known results may be shown with an explicit stale indicator.

Reporting must not imply completeness when it cannot be guaranteed.
