# accessControl_module.md

## Module: Access Control (Authorization)

**Version:** 1.0  
**Status:** Draft (Branch access is mandatory)  
**Module Type:** Platform / Supporting Module  
**Depends on:** Authentication (actor identity), Tenant (status), Branch (status), Subscription & Entitlements (state + entitlements), Staff/Tenant Membership, Staff Branch Assignment, Policy (optional), Audit (logging), Offline Sync (caching)  
**Related Modules:** Tenant, Branch, Staff Management, Staff Attendance, Sale, Cash Session, Inventory, Menu, Audit, Offline Sync, Policy

---

## 1. Purpose

This module provides a single authorization decision surface for Modula:

- **Given an authenticated actor, tenant context, action, and (when required) branch target/context — ALLOW or DENY.**

It is the gatekeeper that prevents:
- cross-branch operations
- cross-tenant access
- role misuse
- (future) capability misuse

---

## 2. Important Principles

### 2.1 Authentication vs Authorization
- Authentication proves identity and issues sessions (handled elsewhere).
- Access Control decides what actions are allowed in a specific context.

### 2.2 Branch access is mandatory
For BRANCH-scoped actions, regardless of UI/workspace entry mode:
- the request must specify `tenant_id` and `branch_id`
- the actor must be assigned to that branch
- if assignment cannot be proven, deny

This applies to both:
- **branch-layer operational actions**, where active branch workspace supplies `branch_id`
- **tenant-layer management actions with branch-scoped effect**, where the UI/command supplies explicit target `branch_id`

For TENANT-scoped actions (example: `tenant.updateProfile`):
- the request specifies `tenant_id`
- `branch_id` is not required and branch assignment must not be used as a proxy gate

### 2.3 Fail closed
If the system cannot verify required facts, it denies rather than guessing.

### 2.4 Centralize checks
Operational modules must not implement their own ad-hoc permission rules.
They should call this module consistently.

---

## 3. Interfaces

### 3.1 Primary function / endpoint
`Authorize(actor_id, tenant_id, branch_id?, action) -> AuthorizationDecision`

**Inputs**
- `actor_id` (`auth_account_id`)
- `tenant_id`
- `branch_id` (required for BRANCH-scoped actions; omitted for TENANT-scoped actions)
- `action` (string key)

**Output**
- `{ result: ALLOW | DENY, reason_code?: string }`

---

## 4. Decision Rules (MVP)

Decision evaluation order (fast denial first):

0) **Action metadata**
- Determine:
  - action scope (`TENANT` vs `BRANCH`)
  - action effect (`READ` vs `WRITE`)
- Determine invocation mode if relevant:
  - branch-layer operational invocation, or
  - tenant-layer management invocation with explicit target branch
- If action is BRANCH-scoped and `branch_id` is missing → DENY(BRANCH_CONTEXT_REQUIRED)

1) **Subscription / operational status gates**
- Subscription state:
  - If subscription is `FROZEN` and action effect is `WRITE` → DENY(SUBSCRIPTION_FROZEN)
  - If subscription is `PAST_DUE` → allow (UX must warn; not an authorization denial)
- Tenant/branch operational status:
  - If tenant is `FROZEN` and action effect is `WRITE` → DENY(TENANT_NOT_ACTIVE)
  - If branch is `FROZEN` and action effect is `WRITE` → DENY(BRANCH_FROZEN)
- Read allow-list:
  - Some `READ` actions may remain allowed while frozen (history, invoices). This must be explicit in the action catalog.

2) **Membership**
- If no active membership for actor in tenant → DENY(NO_MEMBERSHIP)
- Determine `role_key` from membership

3) **Branch assignment (BRANCH-scoped actions only)**
- If action is BRANCH-scoped:
  - If no active assignment for actor to branch in tenant → DENY(NO_BRANCH_ACCESS)

4) **Role policy**
- If `role_key` does not permit action → DENY(ACTION_NOT_PERMITTED)

5) **Entitlements (billing guard rail)**
- Resolve entitlement requirements for the action (from catalog metadata).
- If entitlement enforcement is `DISABLED_VISIBLE` → DENY(ENTITLEMENT_BLOCKED)
- If entitlement enforcement is `READ_ONLY` and action effect is `WRITE` → DENY(ENTITLEMENT_READ_ONLY)

If all pass → ALLOW

---

## 5. Data Sources (Consumed Facts)

This module does not own these; it reads them.

### 5.1 Membership store
Needed fields:
- `actor_id`
- `tenant_id`
- `membership_kind` (OWNER | MEMBER)
- `role_key` (e.g., `ADMIN`, `MANAGER`, `CASHIER`, ...)
- `membership_status` (INVITED/ACTIVE/REVOKED)

### 5.2 Branch assignment store
Branch access is granted **only** via explicit assignments (including for Admin/Manager).

Needed fields:
- `actor_id`
- `tenant_id`
- `branch_id`
- `assignment_status` (ACTIVE/REVOKED)

### 5.3 Tenant status
Needed fields:
- `tenant_id`
- `status` (ACTIVE/FROZEN)

### 5.3.1 Subscription state
Needed fields:
- `tenant_id`
- `subscription_state` (`ACTIVE` | `PAST_DUE` | `FROZEN`)

### 5.3.2 Entitlements snapshot
Needed fields (conceptual):
- `tenant_id`
- `branch_id` (for branch-scoped entitlements)
- `entitlement_key -> enforcement` mapping (`ENABLED` | `READ_ONLY` | `DISABLED_VISIBLE`)

### 5.4 Role policy
MVP: code-defined mapping.

Later: policy module + tenant-specific overrides.

---

## 6. Action Catalog (Recommended)

Use stable string keys. Examples:

### Tenant (Org / Workspace)
- `tenant.updateProfile`
- `tenant.changeStatus` (reserved)

### Subscription / Billing (Tenant governance)
- `subscription.view`
- `subscription.invoice.download`
- `subscription.payNow`
- `subscription.activateFirstBranch` (zero-branch flow)
- `subscription.addBranch`
- `subscription.changeBranchPlan` (branch exists; enables/disables modules, seats)

Action metadata guidance (so tenants can recover while `FROZEN`):
- `subscription.view`, `subscription.invoice.download`, `subscription.payNow`:
  - scope: `TENANT`
  - effect: `READ`
- `subscription.activateFirstBranch`, `subscription.addBranch`, `subscription.changeBranchPlan`:
  - scope: `TENANT`
  - effect: `WRITE`

### POS Operations
- `sale.create`
- `sale.finalize`
- `sale.void.request`
- `sale.void.approve`
- `cashSession.open`
- `cashSession.close`
- `cashSession.paidIn`
- `cashSession.paidOut`
- `cashSession.adjust`
- `receipt.print`
- `order.print.kitchen` (kitchen ticket/sticker print; operational effect)

Cash session movement action metadata guidance:
- `cashSession.paidIn`, `cashSession.paidOut`, `cashSession.adjust`:
  - scope: `BRANCH`
  - effect: `WRITE`
  - target: the currently OPEN cash session for the branch
  - design rule: authorization is branch-scoped, not session-opener-scoped

Printing action metadata guidance:
- `receipt.print`, `order.print.kitchen`:
  - scope: `BRANCH`
  - effect: `READ` (printing is observational; must remain usable for history)

### Branch-Targeted Management (Invoked from Tenant Layer; Still BRANCH-scoped)
- `branch.updateProfile`
- `branch.freeze`
- `branch.unfreeze`
- `policy.view`
- `policy.update`
- `inventory.view`
- `inventory.receive`
- `inventory.adjust`
- `menu.manage`
- `discount.manage`
- `staff.branch.assign`
- `staff.branch.revoke`

Action metadata guidance:
- These actions are still `BRANCH`-scoped because they affect or read a branch-scoped resource.
- The difference is **invocation mode**, not authorization scope:
  - branch-layer actions get `branch_id` from active branch workspace
  - tenant-layer management actions must supply explicit target `branch_id`

### Inventory & Menu
- `inventory.view`
- `inventory.receive`
- `inventory.adjust`
- `inventory.forceCorrect` (reserved; admin-only correction tool if ever needed)
- `menu.manage`

### Reporting
- `reports.view`

---

## 7. Role Policy (MVP Suggested)

Example baseline mapping (adjust as needed):

- CASHIER:
  - sale.create, sale.finalize, receipt.print
  - order.print.kitchen
  - cashSession.open, cashSession.close
  - cashSession.paidIn, cashSession.paidOut
- MANAGER:
  - everything CASHIER can do
  - inventory.view, inventory.receive
  - cashSession.adjust
  - sale.void.approve
  - reports.view
- ADMIN:
  - everything

Keep this in code for March MVP. Move to policy later only if required.

---

## 8. Offline Strategy

Because Modula is offline-first:

### 8.1 Cached facts
The POS client should cache:
- memberships + `role_key`
- branch assignments
- tenant status
- subscription state (best-effort)
- entitlements snapshot (best-effort)
- policy version

### 8.2 Offline decision
When offline:
- authorize using cached facts
- log authorization decisions locally for audit
- resync decisions/operations when online

### 8.3 Server re-validation
On reconnect:
- server re-validates critical operations
- conflict resolution is handled by business workflows (not by authorization)

---


## 9. Edge Cases (Codified)

### Edge Case A — Branch access revoked mid-session
- If a logged-in actor’s branch assignment is revoked, the **next** authorization check for any operational action in that branch must return **DENY**.
- Use reason code `BRANCH_ACCESS_REVOKED` (or `NO_BRANCH_ACCESS` if you do not distinguish).
- Client UX: force branch re-select or logout.

### Edge Case B — Tenant membership but no branch assignments
- Actor can authenticate and may select the tenant.
- Branch selection list is empty → actor cannot operate BRANCH-scoped actions.
- BRANCH-scoped tenant-layer management actions are also denied because no branch assignment can be proven.
- TENANT-scoped actions may still be allowed (example: `tenant.updateProfile`) if permitted by role policy.
- Client UX: show “No branch assigned. Contact admin/manager.”

### Edge Case C — Admin/Manager access to all branches via explicit assignment
- There is **no implicit** “ADMIN can access all branches” rule.
- To grant access to all branches, create **explicit assignments** for each branch.
- This preserves branch-scoped auditability and keeps enforcement simple.

### Edge Case D — Tenant-wide Reporting Scope (`ALL_BRANCHES`)
For tenant-wide reporting aggregation (`ALL_BRANCHES`):
- Resolve `ALL_BRANCHES` to the branch list in the tenant.
- Authorization must be evaluated as a set of BRANCH-scoped checks:
  - allow only if `Authorize(actor, tenant, branch_id, reports.view)` is ALLOW for **every** branch included.
- If the actor is missing access to any branch, tenant-wide scope must be denied (or not offered) and the actor must select a specific allowed branch.


## 10. Audit & Observability

Log:
- DENY decisions (reason codes)
- sensitive ALLOW decisions (optional)

This supports:
- academic defense (traceability)
- security reviews
- operational debugging (“why was I blocked?”)

---

## 11. Out of Scope

- staff branch assignment CRUD (owned by Staff Management)
- tenant provisioning (owned by Tenant/Billing provisioning)
- approval workflows (void request/approve lifecycles)
- billing and payment processing

---

## 12. Future Evolution

- integrate entitlements snapshot checks without changing call sites
- introduce temporary branch access (shift coverage)
- support tenant-specific permission policies
- support ABAC rules if required, while keeping action catalog stable
