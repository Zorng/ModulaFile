# Access Control Domain Model — Modula POS

## Domain Name
Access Control (Authorization)

## Domain Type
Platform / Supporting Domain (Authorization)

## Domain Group
10_Identity&Authorization

## Status
Draft (Branch access is mandatory — academic promise)

---

## Purpose

The Access Control domain answers one question consistently:

- **Is an authenticated actor allowed to perform an action in a specific tenant — and when required, in a specific branch?**

This domain is the **authorization layer** of Modula.
It must remain:
- explainable (easy to reason about)
- enforceable (no “silent bypass”)
- compatible with offline-first operation
- compatible with future SaaS entitlements

**Authentication (who you are) is not solved here.**  
Authentication belongs to the *Authentication* domain/module.

---

## Why This Exists (User Reality)

In real cafés, “having an account” is not enough.

People work in different places:
- a cashier may work only at Branch A
- a manager may work at Branch A and Branch B
- an owner might view everything but not operate POS in every location

If the system cannot enforce branch access:
- staff can operate in the wrong location
- cash sessions, sales, and inventory get mixed across branches
- audit trails become unreliable
- the business loses trust in the system

So Modula treats **branch access as mandatory**:
- every operational action must be tied to a tenant and a branch
- the system must be able to prove the actor is allowed in that branch
- if the system cannot prove it, it must deny

---

## 1. Domain Boundary & Ownership

### Owns
- Authorization decision model (ALLOW / DENY + reason)
- Permission model (actions and what roles can do)
- Branch access enforcement rules
- Tenant/branch operational context requirements
- Integration contract for future entitlement checks

### Explicitly NOT Responsible For
- login, OTP, password reset, session issuance (Authentication)
- staff onboarding/termination workflows (Staff Management)
- tenant/branch provisioning (Tenant/Branch modules)
- business approval workflows (e.g., void approval lifecycle)
- billing, subscription, payment processing (future)

Access Control **consumes facts** (membership, role, branch assignment, tenant status, entitlements) and produces decisions.

---

## 2. Ubiquitous Language

| Term | Meaning |
|---|---|
| Actor | The authenticated user identity (`auth_account_id`) |
| Tenant | The business workspace boundary |
| Branch | A physical operating location within a tenant |
| Action (Permission Key) | A stable operation key (e.g., `sale.finalize`, `inventory.adjust`) |
| Action Scope | Whether an action is `TENANT`-scoped or `BRANCH`-scoped |
| Role Key | Tenant-scoped authorization role identifier (e.g., `ADMIN`, `MANAGER`, `CASHIER`, …) |
| Membership | Actor ↔ Tenant relationship (status + `role_key` + optional ownership flag) |
| Branch Assignment | Actor ↔ Branch relationship (ACTIVE/REVOKED) |
| Decision | ALLOW or DENY with a reason code |
| Context | The chosen tenant, and branch when required by the action |
| Entitlements | SaaS capability snapshot (future) |

---

## 3. Core Concepts

### 3.1 AuthorizationDecision (Value Object)
Output of the domain:
- `result`: ALLOW | DENY
- `reason_code`: string (only when DENY)
- `policy_version`: optional (to trace changes)

Example deny reasons:
- TENANT_NOT_ACTIVE
- NO_MEMBERSHIP
- MEMBERSHIP_DISABLED
- NO_BRANCH_ACCESS
- BRANCH_ACCESS_REVOKED
- BRANCH_FROZEN
- BRANCH_CONTEXT_REQUIRED
- ACTION_NOT_PERMITTED
- ENTITLEMENT_BLOCKED (future)

---

### 3.2 Action (Value Object)
A stable string identifier for operations.

Examples:
- `sale.create`
- `sale.finalize`
- `sale.void.request`
- `sale.void.approve`
- `cashSession.open`
- `cashSession.close`
- `cashSession.x.view`
- `cashSession.z.view`
- `inventory.receive`
- `inventory.adjust`
- `menu.manage`
- `reports.view`
- `audit.view`

Design rule: actions are stable public contract keys, not method names.

### 3.2.1 Action Scope (TENANT vs BRANCH)

Not every action is tied to a specific branch.

Access Control classifies actions into two scopes:

- **TENANT-scoped actions**: operate on tenant-owned resources.
  - require `tenant_id`
  - do **not** require `branch_id`
  - do **not** require branch assignment
  - examples: `tenant.updateProfile`, tenant membership administration, `audit.view`, tenant-wide policy configuration (future)

- **BRANCH-scoped actions**: operate on branch-scoped operational records.
  - require `tenant_id` + `branch_id`
  - require explicit branch assignment (fail closed)
  - examples: `sale.finalize`, `cashSession.open`, `inventory.adjust`, `reports.view`

Design rule:
- If an action is BRANCH-scoped and `branch_id` is missing → deny with `BRANCH_CONTEXT_REQUIRED`.
- Multi-branch queries (example: reporting `ALL_BRANCHES`) are implemented as repeated BRANCH-scoped authorization checks across each included branch; there is no implicit bypass.

---

### 3.3 RolePolicy (Policy Model)
Defines which roles are permitted for which actions.

MVP uses RBAC:
- CASHIER, MANAGER, ADMIN (extendable via additional role keys)

RolePolicy may be:
- code-defined mapping (MVP)
- later configurable via a Policy module (optional)

---

### 3.4 Membership (External Fact)
Access Control consumes membership facts:
- membership exists?
- membership status ACTIVE? (or not DISABLED/ARCHIVED)
 - `role_key` in tenant?
 - optional: membership kind / ownership flag for governance-only invariants

Owned by Tenant Membership domain.

---

### 3.5 Branch Assignment (External Fact)
Access Control consumes branch assignment facts:
- is actor assigned to this branch?
- assignment status ACTIVE? (or not REVOKED)

Owned by Staff Management (or a staff/organization domain), but consumed here.

Branch access is **mandatory** for BRANCH-scoped operational actions.

---

### 3.6 Tenant Status (External Fact)
Access Control consumes tenant status:
- ACTIVE / FROZEN (and future states)

Owned by Tenant domain/module.

---

### 3.7 Branch Status (External Fact)
Access Control consumes branch status:
- ACTIVE / FROZEN

Owned by Branch domain/module.

---

### 3.8 Entitlements Snapshot (External Fact — Future)
A computed snapshot of capabilities for the tenant.
Access Control must be compatible with adding:
- feature gating
- branch limits
- staff limits
without rewriting all modules

---

## 4. Invariants

- INV-AC0: Every authorization request must include a `tenant_id` context. `branch_id` is required only for BRANCH-scoped actions.
- INV-AC1: Every BRANCH-scoped operational request must include a `tenant_id` and `branch_id` context.
- INV-AC2: A user must have an ACTIVE membership to a tenant to operate within it.
- INV-AC3: A user must have ACTIVE branch assignment to operate within that branch (BRANCH-scoped actions only).
- INV-AC4: If branch access cannot be verified, deny (fail closed).
- INV-AC5: Authorization decisions must be deterministic based on current facts.
- INV-AC6: RolePolicy must be stable and versionable (for auditability).
- INV-AC7: Tenant frozen status must block operational actions (unless explicitly allow-listed).
- INV-AC8: Branch frozen status must block operational actions (unless explicitly allow-listed).

---

## 5. Commands (Write Intents)

Access Control is typically **decision-only** (mostly reads), but it can own:
- `DefineRolePolicy` (if policy is configurable)
- `SetActionCatalog` (optional)

Most write actions (assigning staff to branches, granting membership) live in other modules.

---

## 6. Domain Events

Access Control may emit:
- AUTHORIZATION_DENIED (with reason) for audit/monitoring
- ROLE_POLICY_UPDATED (if configurable)

But it is acceptable for MVP to log these into the Audit module instead of emitting events.

---

## 7. Read Model

Access Control performs read queries against:
- Membership store (roles + membership status)
- Branch assignment store
- Tenant status store
- (future) Entitlements snapshot store/cache

---

## 8. Offline & Caching Strategy

Because Modula is offline-first, authorization must remain usable when connectivity is poor.

Recommended approach:
- Cache membership + roles + branch assignments + tenant status on device
- Cache a lightweight “policy version”
- Enforce locally using the same decision rules
- On reconnect, server re-validates and resolves conflicts

Design rule: “Offline authorization uses cached facts; server is source of truth.”

---


## 9. Edge Cases (Codified)

### Edge Case A — Branch access revoked during an active session
If an actor is logged in and operating in a branch, and their branch assignment is revoked:
- The **next operational request** for that branch must be denied (fail closed).
- Recommended reason code: `BRANCH_ACCESS_REVOKED` (or `NO_BRANCH_ACCESS` if you do not distinguish).
- Client UX: prompt the user to switch branch (to an allowed branch) or log out.

### Edge Case B — Tenant membership exists but no branch assignments exist
If an actor is a valid member of a tenant but has zero branch assignments:
- They may authenticate and select the tenant.
- They **cannot** enter branch-scoped operational mode because no branch context can be selected.
- They may still perform TENANT-scoped actions if permitted by role policy (example: update tenant profile).
- Client UX: show “No branch assigned. Contact admin/manager.”

### Edge Case C — Admin/Manager needs access to all branches (explicit assignment)
Modula uses **explicit branch assignments** to grant branch access (academic promise + audit clarity):
- Admin/Manager does **not** automatically get access to all branches by role alone.
- To grant “all branches,” create assignments for every branch.
- Authorization remains branch-scoped and provable for every operation.

### Edge Case D — Tenant-wide Report Scope Must Not Bypass Branch Assignment
If a reporting surface offers tenant-wide aggregation (`ALL_BRANCHES`):
- The system must authorize `reports.view` across **every branch included** in the tenant-wide scope.
- If the actor is missing access to any branch, tenant-wide aggregation must be denied (or not offered) and the actor must choose a specific allowed branch.

This preserves the “explicit branch access” promise while still supporting tenant-wide summaries when the actor truly has access to all branches.


## 10. Out of Scope

- how users log in (Authentication)
- staff lifecycle processes (Staff Management)
- approval workflows and state machines (Process docs)
- payment/billing actions (Subscription service)

---

## 11. Future Evolution

- add entitlements snapshot checks (SaaS)
- add branch-level capability gating (e.g., branch count limits)
- add temporary access grants (cover shifts)
- add richer policies (ABAC) if needed, while keeping action contract stable
 - add data-defined roles (custom tenant roles) without changing action keys
