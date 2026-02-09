# 50 — View Audit Logs (Owner / Admin)

## Purpose

This process defines how privileged users view audit evidence safely and consistently.

It exists to support investigation and accountability without turning audit logs into analytics.

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/saas_governance/reviewing_audit_logs_for_accountability.md`

---

## Domains Involved

- Audit Logging (read-only evidence store)
- Access Control (authorization for `audit.view`)
- Tenant / Branch context (filters and display context)

---

## When This Process Runs

Triggered when an Owner/Admin requests the audit log screen or performs a search/filter query.

---

## Inputs

- `tenant_id` (required)
- optional filters:
  - `branch_id`
  - `date_range`
  - `event_type` / `module`
  - `actor_id`
  - `outcome`
  - `entity_id` (subject/entity reference)
- pagination:
  - `cursor` or `(limit, offset)` (cursor preferred)

---

## Orchestration Steps

### Step 1 — Validate authorization

- Ensure `tenant_id` is present.
- Authorize `audit.view` for the actor in the tenant context.
- If denied, return no data (fail closed).

---

### Step 2 — Apply filters safely

- Filters narrow the query; they must never expand scope across tenants.
- Branch filter is optional for March (audit viewing is tenant-governance).

---

### Step 3 — Query with pagination

- Always paginate results.
- Order primarily by `recorded_at` descending (server time), with `occurred_at` displayed for context.

---

### Step 4 — Present read-only detail

Audit log view is read-only. Opening a detail entry must not mutate business truth.

---

## Failure / Degradation Rules (March)

- If offline: show last-known cached results only if clearly marked as stale, or block audit viewing (team decision).
- Export is out of scope for March.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/audit_edge_case_sweep.md`

