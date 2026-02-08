# Policy Module — Core Module (Capstone 1)

**Version:** 1.4  
**Status:** Patched (Branch-scoped; configurable policies limited to Tax & Currency)  
**Module Type:** Core Module  
**Depends on:** Authentication & Authorization (Core), Tenant & Branch Context (Core), Audit Logging (Core)  
**Related Modules:** Sale, Receipt, Reporting

---

## 1. Purpose

The Policy module centralizes **configurable pricing behavior** that affects how sales are computed and displayed.

For Capstone 1, Policy is intentionally small:
- **Configurable:** VAT, FX rate, KHR rounding
- **Not configurable (moved to product rules / other modules):** attendance gating, cash session controls, inventory deduction toggles

Policy values are read-only at the point of sale and cannot be overridden by staff.

Authoritative references:
- Story: `BusinessLogic/1_stories/handling_money/configuring_tax_and_currency_settings.md`
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/policy_domain.md`
- Edge cases: `BusinessLogic/3_contract/10_edgecases/policy_edge_case_sweep.md`
- Processes:
  - `BusinessLogic/4_process/60_PlatformSystems/10_update_tax_currency_policy_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/20_resolve_tax_currency_policy_process.md`
- Downstream snapshot requirement: `BusinessLogic/5_modSpec/40_POSOperation/sale_module_patched.md`

---

## 2. Scope & Boundaries

### 2.1 Scope Clarification (Patched)

Policies are **branch-scoped**.
- A tenant may have multiple branches, and each branch has its own policy values.
- Dependent modules resolve policies using `(tenant_id, branch_id)`.

### 2.2 This Module Covers

- VAT enablement + VAT rate
- FX rate (KHR per USD) for display/checkout
- KHR rounding enablement + rounding rules

### 2.3 This Module Does Not Cover

- Cash session requirements and permissions
- Inventory deduction enable/disable switches
- Attendance + shift enforcement rules
- Discounts
- Menu pricing/modifiers
- Tenant/branch profile fields (name/logo/address)

---

## 3. Policy Catalog (Canonical Keys)

Policies are **not creatable**. Modula ships a fixed set of keys with default values.
- Admin can update values (partial update)
- Other roles can read (if permitted by Access Control)

### Tax & Currency (Branch Policies)
| UI Label | Data Key | Type | Values / Options | Description |
|---|---|---|---|---|
| Enable VAT | `saleVatEnabled` | boolean | true/false | If on, VAT is applied at checkout. |
| VAT rate (%) | `saleVatRatePercent` | number | 0–100 | VAT percentage used in sale totals. |
| FX rate (KHR per USD) | `saleFxRateKhrPerUsd` | number | > 0 | Conversion rate used to show totals in both USD and KHR. |
| Enable KHR rounding | `saleKhrRoundingEnabled` | boolean | true/false | If on, KHR tender/totals follow rounding rules. |
| KHR rounding mode | `saleKhrRoundingMode` | enum | `NEAREST` \| `UP` \| `DOWN` | How to round KHR. |
| KHR rounding step | `saleKhrRoundingGranularity` | enum | `"100"` \| `"1000"` | Rounding granularity in riel. |

### Update Behavior
- Partial updates use `UpdateBranchPoliciesInput` (all fields optional).
- Writes must be idempotent and recorded in Audit Log.
- Reads are cached client-side per branch and refreshed on:
  - login
  - branch switch
  - after a policy update

> Note: Backend implementations may still expose a `TenantPolicies` DTO. For Capstone 1, treat it as the policy record resolved by `(tenant_id, branch_id)` even if the storage naming has not been refactored yet.

---

## 4. Removed From Policy (Now Product Rules / Other Modules)

These were previously described as “policies” but are now **not configurable**:

### 4.1 Attendance & Shift Rules (Removed)
Attendance behavior should avoid hard-gating workflows. The system records signals/flags and leaves evaluation to managers/owners.

### 4.2 Cash Session Controls (Removed)
Cash session permissions and approval workflows are governed by:
- Access Control (RBAC + action scopes)
- POSOperation processes (e.g., void/refund approval)

### 4.3 Inventory Behavior Toggles (Removed)
Sale-based inventory deduction is not a policy toggle.
- Whether an item deducts stock is defined by Menu composition (recipe/direct-stock mapping).
- Inventory expiry is optional batch metadata (not a mode toggle).

---

## 5. Use Cases (Actor + Permission)

### UC-1: Admin views tax/currency policies for a branch
**Actor:** Admin  
**Preconditions:** Admin is operating inside a selected `branch_id`  
**Main Flow:**
1. Admin opens Tenant Admin Portal → Settings → Tax & Currency
2. System displays the branch policy values
**Acceptance:**
- Branch selection affects displayed values

---

### UC-2: Admin updates VAT policy (branch-scoped)
**Actor:** Admin  
**Main Flow:**
1. Admin toggles VAT and/or sets VAT percentage
2. System validates inputs and persists policy atomically
3. Audit log entry is written
**Acceptance:**
- Cashier/Manager cannot modify VAT
- VAT applies only to the selected branch

---

### UC-3: Admin updates FX rate and KHR rounding (branch-scoped)
**Actor:** Admin  
**Main Flow:**
1. Admin sets FX rate (KHR per USD)
2. Admin configures rounding enabled/mode/granularity
3. System saves and audits changes
**Acceptance:**
- Sale UI displays USD exact + KHR rounded according to branch policy
- KHR tender follows rounding policy

---

## 6. Functional Requirements

### General
- **FR-1:** Policies must be stored and resolved per `(tenant_id, branch_id)`.
- **FR-2:** Only Admin can modify policy values.
- **FR-3:** Policy changes must be persisted atomically and audited.

### Tax & Currency
- **FR-4:** VAT must be applied automatically based on branch policy; staff cannot override it.
- **FR-5:** FX rate must be used consistently in sale calculations and display.
- **FR-6:** Rounding settings must apply when tender currency is KHR.

### Audit
- **FR-7:** Policy modifications must create an audit entry containing old and new values.

---

## 7. Acceptance Criteria

- **AC-1:** Policy resolution uses `branch_id` consistently across consuming modules.
- **AC-2:** Policy updates affect only the selected branch (no unintended cross-branch bleed).
- **AC-3:** Sale finalize stores a policy snapshot (VAT/FX/Rounding values used) for historical correctness.
- **AC-4:** All changes produce audit logs visible to admins.

---

## 8. Out of Scope (Capstone 1)

- Scheduled policies (time-based rules)
- Service charges, multi-VAT rules
- Policy versioning + rollback UI
- Cross-branch inheritance UI (“copy from HQ”)

---

## 9. Developer Notes

- Do not embed VAT/FX/rounding “policy logic” inside multiple modules with separate sources of truth. Modules must consume policy values.
- Keep sale/receipt/reporting historically correct by snapshotting policy values on finalize.

---

## 10. Audit Events

- `POLICY_UPDATED`
- `POLICY_RESET_TO_DEFAULT`
