# Identity & Access Revamp Workboard (Temporary)

**Goal:** Solidify Authorization by making Tenant + Staff/Branch facts explicit, consistent, and implementable (March MVP).

This file exists to prevent drift/hallucination by:
- locking the scope
- listing concrete deliverables
- tracking decisions + open questions
- defining “done” criteria per step

---

## 0) Locked Decisions (Do Not Re-debate Unless Needed)

1. **Auth is renamed to Authentication**
   - Domain file: `authentication.md` (renamed by you)
   - Modspec file: `authentication_module.md` (patched)

2. **Authorization is a separate concern named Access Control**
   - Files:
     - `accessControl_domain_v2.md`
     - `accessControl_module_v2.md`

3. **Branch access is mandatory**
   - Every operational action must have `{tenant_id, branch_id}` context.
   - Authorization fails closed if branch access cannot be proven.

4. **Edge Case C uses explicit assignment**
   - Admin/Manager do NOT implicitly get all branches.
   - “All branches” requires explicit assignments for each branch.

5. **Authorization is middleware-style**
   - Central decision function `Authorize(actor_id, tenant_id, branch_id, action)` used by API handlers + jobs + sync reconciliation.

---

## 1) Inputs We Will Use (Source Artifacts)

### Existing module specs (to patch)
- `tenant_module.md`
- `staffManagement_module.md`
- `branch_module.md`

### Already patched / baseline references
- `authentication_module.md`
- `accessControl_domain_v2.md`
- `accessControl_module_v2.md`

---

## 2) Work Plan (Deep Steps) — With Outputs + Done Criteria

### Step 1 — Patch Tenant Module (Membership + Roles as Tenant Facts)
**Why:** Access Control needs tenant membership + role truth.

**Output file**
- `tenant_module_patched_v2.md`

**What must exist in the patched module**
- Explicit entity: `TenantMembership`
  - keys: `tenant_id`, `actor_id` (auth_account_id), `role`, `status`
- Lifecycle:
  - grant membership (invite/attach)
  - change role
  - revoke membership (immediate effect)
  - optional: suspend membership
- Read contracts:
  - get membership for actor+tenant
  - list memberships for tenant
- Clear boundary statement:
  - Tenant owns membership truth (or Staff owns, but one must be declared)

**Done criteria**
- Access Control can answer:
  - “is actor a member of tenant?”
  - “what is their role?”
  - “is membership active?”
- No tenant provisioning/billing logic introduced (that is future SaaS layer).

**Open question (must resolve during patch)**
- Does Tenant own membership records, or does StaffManagement own them?
  - Default recommendation: Tenant owns membership/roles; Staff owns staff profile.

---

### Step 2 — Patch Staff Management Module (Explicit Branch Assignments)
**Why:** Access Control needs branch assignment truth (mandatory) and explicit assignment for Admin/Manager.

**Output file**
- `staffManagement_module_patched_v2.md`

**What must exist**
- Explicit concept: `BranchAssignment` (or `StaffBranchAccess`)
  - keys: `tenant_id`, `branch_id`, `actor_id` (or staff_id), `status`
- Operations:
  - assign staff to branch (one or many)
  - revoke staff from branch (immediate effect)
  - list staff assignments by branch
  - list branches assigned to staff
- Edge case behavior codified:
  - Revocation mid-session takes effect on next request (deny)

**Done criteria**
- Access Control can answer:
  - “is actor assigned to branch?”
  - “what branches can actor operate?”
- Assignment is explicit for all roles (including Admin/Manager).

**Open question**
- Do we key assignments by `auth_account_id` or `staff_id`?
  - Implementation can store by `staff_id` but must be resolvable from `auth_account_id`.
  - We will document the mapping contract.

---

### Step 3 — Patch Branch Module (Branch as Org Object, Not Access Rules)
**Why:** Keep boundaries clean: Branch module describes the branch; access is elsewhere.

**Output file**
- `branch_module_patched_v2.md`

**What must exist**
- Branch lifecycle:
  - create/update/deactivate branch
- Branch status:
  - ACTIVE/INACTIVE
- Branch metadata:
  - name, address, timezone/currency (if relevant)
- Explicit non-ownership:
  - Branch does not own staff access rules

**Done criteria**
- Branch module contains no authorization logic beyond branch status checks.
- Other modules refer to `branch_id` confidently.

---

### Step 4 — Cross-Module Consistency Notes (Light patches)
**Why:** POS operation modules must consistently state required context and middleware.

**Output changes (small patches)**
- Add a short section to:
  - Sale, CashSession, Inventory, Menu, Receipt
  describing:
  - required `{tenant_id, branch_id}`
  - Access Control middleware action keys used

**Done criteria**
- Reader understands that operational processes always run under tenant+branch.

---

## 3) Deliverable Tracker (Status)

| Step | Artifact | Status |
|---|---|---|
| 1 | tenant_module_patched_v2.md | TODO |
| 2 | staffManagement_module_patched_v2.md | TODO |
| 3 | branch_module_patched_v2.md | TODO |
| 4 | small consistency patches (POS ops modules) | TODO |

---

## 4) Action Catalog — Initial Mapping (For Authorization)

This is a working list to keep consistent across modules:

### POS Operation
- sale.create
- sale.finalize
- sale.void.request
- sale.void.approve
- cashSession.open
- cashSession.close
- receipt.print

### Inventory/Menu
- inventory.receive
- inventory.adjust
- inventory.forceCorrect
- menu.manage

### Reporting
- reports.view

---

## 5) “No Drift” Rules During Patching

1. Do not invent new modules unless required by scope.
2. Do not add billing/subscription logic yet (paper-managed at launch).
3. Keep branch access mandatory and explicit assignment as previously decided.
4. Prefer minimal, implementable MVP rules over “ideal future”.
5. Any new assumption must be written into this file first under “Locked Decisions” or “Open Questions”.

---

## 6) Next Inputs Needed From You

To execute Step 1–3, please provide (upload or re-upload):
- `tenant_module.md`
- `staffManagement_module.md`
- `branch_module.md`

(Then we patch them in the order listed.)
