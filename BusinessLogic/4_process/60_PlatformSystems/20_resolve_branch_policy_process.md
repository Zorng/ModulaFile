# 20 — Resolve Branch Policy (Branch-Scoped Read; Tenant or Branch Entry)

## Purpose

This process defines how modules consistently resolve the branch-scoped Policy record:
- Tax & Currency (VAT, FX rate, KHR rounding)
- Sale workflow toggle (pay-later enablement)

It exists so Sale/Receipt/Reporting do not “guess” policy values or re-implement their own sources of truth.

This is a read process. It does not mutate policy or sales.

---

## Domains Involved (Read-Only)

- Policy (policy values)
- Access Control (who may read)
- Tenant/Branch (tenant context, branch context or explicit target branch; frozen status)

---

## When This Process Runs

Triggered whenever the system needs policy inputs, for example:
- sale pre-checkout pricing display
- sale finalization
- admin viewing branch policy settings

This process supports two invocation modes:
- **branch-layer operational read**: active branch context already exists
- **tenant-layer management read**: target `branch_id` is supplied explicitly

---

## Inputs

- `tenant_id`
- `branch_id` (active branch context or explicit target branch)

---

## Orchestration Steps

### Step 1 — Validate context + authorize read

- Require both `tenant_id` and `branch_id`.
- Ensure the actor is allowed to read policy for the branch context or explicit target branch.

Notes:
- Read access is required for checkout display and should be allowed for roles that can sell in the branch.
- Frozen branches remain readable (history and administration), even though operational writes are blocked.

---

### Step 2 — Resolve policy record (defaults if missing)

- Fetch the policy record for `(tenant_id, branch_id)`.
- If missing, return default values (fail-safe) and emit a discoverable system/audit signal.

---

### Step 3 — Return canonical policy object

Return the canonical policy values (and metadata such as `updated_at`) so clients can cache safely and refresh when needed.

---

## Degradation Rules (March)

- If offline: allow returning last-known cached values, but staleness must be explicit.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/policy_edge_case_sweep.md`
