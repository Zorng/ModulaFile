# tenant_module.md

## Module: Tenant (Business Workspace)

**Version:** 2.0  
**Status:** Patched to support Access Control + future SaaS (without implementing billing yet)  
**Module Type:** Core / Organizational Boundary  
**Depends on:** Authentication (identity), Access Control (authorization decisions), Audit (logging), Policy (optional), Offline Sync (optional)  
**Related Modules:** Branch, Staff Management, Staff Attendance, Sale, Cash Session, Inventory, Menu, Reporting, Access Control

---

## 1. Purpose

The Tenant module defines the **business workspace boundary** in Modula.

A **Tenant** represents a real business using the POS. It is the root context for:
- data isolation (no cross-tenant access),
- organizational configuration,
- future subscription ownership (later),
- and authorization context (Access Control consumes tenant facts).

This module is intentionally **not** a billing system and does not implement payments.

---

## 2. Key Boundaries (Avoid “Auth = Everything”)

### 2.1 Authentication vs Tenant
- **Authentication** proves identity and issues sessions.
- **Tenant** defines the business workspace and its lifecycle.

### 2.2 Tenant vs Staff Management (Membership Workflows)
- **Tenant owns the truth of membership + role as tenant-scoped facts** (needed for authorization).
- **Staff Management owns the workflows and staff profile** (inviting, onboarding, disabling staff, etc.).
- Access Control reads:
  - tenant status (from Tenant)
  - tenant membership + role (from Tenant)
  - branch assignment (from Staff Management)

This separation prevents business-critical authorization facts from being scattered.

### 2.3 Tenant vs Access Control
- Tenant does **not** decide permissions.
- Tenant provides facts (tenant status, membership roles) that Access Control uses to decide.

---

## 3. Core Data Model

### 3.1 Tenant (Aggregate Root)
Represents one business workspace.

Minimum fields:
- `tenant_id`
- `business_name`
- `logo_url?`
- `contact_info?` (phone/email/address optional)
- `status` = ACTIVE | (reserved) FROZEN
- `created_at`, `updated_at`

### 3.2 TenantMembership (Tenant-scoped Fact)
Represents a user’s relationship to a tenant and their role.

Minimum fields:
- `tenant_id`
- `auth_account_id` (actor id from Authentication)
- `role` = ADMIN | MANAGER | CASHIER (extend later)
- `status` = ACTIVE | REVOKED | (optional) SUSPENDED
- `created_at`, `updated_at`

**Design rule:** Membership is keyed by `auth_account_id` to keep authorization deterministic.

---

## 4. Invariants

- INV-T1: All operational data belongs to exactly one tenant.
- INV-T2: Cross-tenant data access is forbidden.
- INV-T3: Tenant status is a fact; Access Control enforces behavior based on it.
- INV-T4: Membership must be tenant-scoped (not global).
- INV-T5: A revoked/suspended membership must be reflected immediately in authorization decisions.
- INV-T6: A tenant should have at least one branch provisioned (enforced via provisioning workflow).
- INV-T7: A tenant must have at least one ADMIN membership (MVP rule).

---

## 5. Self-Contained Processes (Owned by Tenant)

### UC-T1 — Provision Tenant (System Provisioning)
**Goal:** Create a tenant workspace in the system.

**Trigger:** internal provisioning flow (manual/paper onboarding at launch is fine).  
**Inputs:** business name, initial owner identity (auth_account_id), optional metadata.  
**Steps:**
- create Tenant (status ACTIVE)
- create initial ADMIN membership for owner
- (recommended) create first branch via Branch module provisioning
- log to Audit

**Outputs:**
- tenant_id
- initial membership created

**Notes:** Tenant creation is system-level. End users do not “sign up and auto-create tenants” for March MVP.

---

### UC-T2 — Update Tenant Profile
**Goal:** Update business-facing metadata.

**Allowed fields:** business name, logo, contact info.  
**Authorization:** Access Control action `tenant.updateProfile` (typically ADMIN).  
**Audit:** record previous + new values (or diffs).

---

### UC-T3 — Change Tenant Status (Reserved)
**Goal:** Allow future subscription enforcement (freeze/unfreeze).

**Status states:**
- ACTIVE
- FROZEN (reserved for SaaS enforcement)

**Authorization:** `tenant.changeStatus` (ADMIN / system operator)  
**Effect:** Access Control denies operational actions when tenant is not ACTIVE.

---

## 6. Tenant Membership Processes (Facts owned here; workflows may be driven by Staff Management)

> This section defines membership facts and their lifecycle. Staff Management may provide the UI/workflow to trigger these operations.

### UC-T4 — Grant Membership (Add a User to Tenant)
**Goal:** Add an actor to tenant with a role.

**Inputs:** tenant_id, auth_account_id, role  
**Rules:**
- cannot grant membership if tenant is FROZEN (optional MVP)
- role must be in allowed set
- if membership already exists and ACTIVE → idempotent no-op or return conflict

**Outputs:** membership record ACTIVE

**Audit:** MEMBER_ADDED

---

### UC-T5 — Change Member Role
**Goal:** Promote/demote role.

**Rules:**
- cannot change role of last remaining ADMIN (INV-T7)
- changes take effect immediately for authorization

**Audit:** MEMBER_ROLE_CHANGED

---

### UC-T6 — Revoke Membership
**Goal:** Remove actor from tenant.

**Rules:**
- cannot revoke last ADMIN
- revocation takes effect immediately (Access Control must deny next request)

**Audit:** MEMBER_REVOKED

---

### UC-T7 — Suspend/Reactivate Membership (Optional)
**Goal:** Temporarily disable a member without deleting history.

**Why:** Useful for HR discipline or temporary leave.

**Rule:** SUSPENDED behaves like REVOKED for authorization.

---

## 7. Read APIs (What other modules need)

### Tenant queries
- Get tenant by id (for headers, receipts, reports)
- List tenants for actor (for tenant selection at login)

### Membership queries (critical for Access Control)
- Get membership(role, status) by (tenant_id, auth_account_id)
- List memberships for tenant (admin UI)
- List tenants for actor (where membership ACTIVE)

---

## 8. Events & Audit Hooks

Recommended events (or audit logs):
- TENANT_CREATED
- TENANT_PROFILE_UPDATED
- TENANT_STATUS_CHANGED
- TENANT_MEMBER_GRANTED
- TENANT_MEMBER_ROLE_CHANGED
- TENANT_MEMBER_REVOKED
- TENANT_MEMBER_SUSPENDED / REACTIVATED

These support traceability for academic defense and debugging.

---

## 9. Failure Modes

- TENANT_NOT_FOUND
- TENANT_NOT_ACTIVE (when status is FROZEN; enforcement by Access Control)
- MEMBER_NOT_FOUND
- MEMBER_NOT_ACTIVE
- ROLE_INVALID
- CANNOT_REMOVE_LAST_ADMIN
- DUPLICATE_MEMBERSHIP

---

## 10. Data & Performance Notes

- Membership lookups must be fast (hot path for authorization).
  - Index recommended: `(tenant_id, auth_account_id)` unique.
- Tenant read model should be cacheable.
- Membership history should be auditable (do not hard-delete; use status changes).

---

## 11. Out of Scope (March MVP)

- Subscription billing engine / payments
- Automated tenant creation from public website
- Entitlements/plan limits enforcement (will plug in later)
- Cross-tenant enterprise accounts
- Ownership transfer flows

---

## 12. Integration Notes (How Authorization Uses This)

Access Control middleware calls:
- Tenant: `tenant.status`
- TenantMembership: membership existence + role + status

Access Control separately checks:
- Branch assignment (from Staff Management)

**Branch access is mandatory and explicit assignment is required** (even for ADMIN/MANAGER).

---

## Appendix: Minimal Action Keys (for Access Control)

- `tenant.updateProfile`
- `tenant.manageMembers` (grant/revoke/changeRole)
- `tenant.changeStatus` (reserved)
