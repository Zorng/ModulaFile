# accessControl_module.md

## Module: Access Control (Authorization)

**Version:** 1.0  
**Status:** Draft (Branch access is mandatory)  
**Module Type:** Platform / Supporting Module  
**Depends on:** Authentication (actor identity), Tenant (status), Staff/Tenant Membership, Staff Branch Assignment, Policy (optional), Audit (logging), Offline Sync (caching)  
**Related Modules:** Tenant, Branch, Staff Management, Staff Attendance, Sale, Cash Session, Inventory, Menu, Audit, Offline Sync, Policy

---

## 1. Purpose

This module provides a single authorization decision surface for Modula:

- **Given an authenticated actor, tenant context, branch context, and action — ALLOW or DENY.**

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
For operational actions:
- the request must specify `tenant_id` and `branch_id`
- the actor must be assigned to that branch
- if assignment cannot be proven, deny

### 2.3 Fail closed
If the system cannot verify required facts, it denies rather than guessing.

### 2.4 Centralize checks
Operational modules must not implement their own ad-hoc permission rules.
They should call this module consistently.

---

## 3. Interfaces

### 3.1 Primary function / endpoint
`Authorize(actor_id, tenant_id, branch_id, action) -> AuthorizationDecision`

**Inputs**
- `actor_id` (`auth_account_id`)
- `tenant_id`
- `branch_id` (required for operational actions)
- `action` (string key)

**Output**
- `{ result: ALLOW | DENY, reason_code?: string }`

---

## 4. Decision Rules (MVP)

Decision evaluation order (fast denial first):

1) **Tenant status**
- If tenant is not ACTIVE → DENY(TENANT_NOT_ACTIVE)
- Exception allow-list (optional): read-only actions, support actions

2) **Membership**
- If no active membership for actor in tenant → DENY(NO_MEMBERSHIP)
- Determine role from membership

3) **Branch assignment**
- If no active assignment for actor to branch in tenant → DENY(NO_BRANCH_ACCESS)

4) **Role policy**
- If role does not permit action → DENY(ACTION_NOT_PERMITTED)

5) **Entitlements (future hook)**
- If entitlements block feature → DENY(ENTITLEMENT_BLOCKED)

If all pass → ALLOW

---

## 5. Data Sources (Consumed Facts)

This module does not own these; it reads them.

### 5.1 Membership store
Needed fields:
- `actor_id`
- `tenant_id`
- `role`
- `status` (ACTIVE/REVOKED)

### 5.2 Branch assignment store
Branch access is granted **only** via explicit assignments (including for Admin/Manager).

Needed fields:
- `actor_id`
- `tenant_id`
- `branch_id`
- `status` (ACTIVE/REVOKED)

### 5.3 Tenant status
Needed fields:
- `tenant_id`
- `status` (ACTIVE/SUSPENDED)

### 5.4 Role policy
MVP: code-defined mapping.

Later: policy module + tenant-specific overrides.

---

## 6. Action Catalog (Recommended)

Use stable string keys. Examples:

### POS Operations
- `sale.create`
- `sale.finalize`
- `sale.void.request`
- `sale.void.approve`
- `cashSession.open`
- `cashSession.close`
- `receipt.print`

### Inventory & Menu
- `inventory.receive`
- `inventory.adjust`
- `inventory.forceCorrect`
- `menu.manage`

### Reporting
- `reports.view`

---

## 7. Role Policy (MVP Suggested)

Example baseline mapping (adjust as needed):

- CASHIER:
  - sale.create, sale.finalize, receipt.print
  - cashSession.open, cashSession.close
- MANAGER:
  - everything CASHIER can do
  - inventory.adjust
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
- memberships + roles
- branch assignments
- tenant status
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
- Branch selection list is empty → actor cannot operate POS actions.
- Client UX: show “No branch assigned. Contact admin/manager.”

### Edge Case C — Admin/Manager access to all branches via explicit assignment
- There is **no implicit** “ADMIN can access all branches” rule.
- To grant access to all branches, create **explicit assignments** for each branch.
- This preserves branch-scoped auditability and keeps enforcement simple.


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
