# Tenant Domain Model — Modula POS

## Domain Name
Tenant (Business Workspace)

## Domain Type
Core Domain (Organizational Boundary)

## Domain Group
20_OrgAccount (Tenant + Branch)

## Status
Draft (Baseline for Capstone 1; SaaS-ready)

---

## Purpose

The Tenant domain defines the **business workspace boundary** in Modula.

A **Tenant** represents a real-world business entity (e.g., a café or shop) under which:
- data is isolated,
- staff operate,
- branches exist,
- operational rules apply,
- and future subscription capabilities will be enforced.

The Tenant is the **root organizational context** for all POS operations.

---

## Why This Exists (User Reality)

In the real world, a POS system is not used by individuals in isolation.
It is used **by a business**.

That business may:
- have multiple staff,
- operate multiple branches,
- grow or shrink over time,
- eventually pause or stop using the system.

Without a clear Tenant boundary:
- data leaks across businesses,
- roles lose meaning,
- authorization becomes inconsistent,
- subscription enforcement becomes impossible later.

The Tenant domain exists to give Modula a **stable business identity** that all other domains can rely on.

---

## 1. Domain Boundary & Ownership

### Owns
- Tenant identity (business-level workspace)
- Tenant metadata (name, logo, contact info)
- Tenant lifecycle state (ACTIVE / future FROZEN)
- Tenant as the **root isolation boundary**

### Explicitly NOT Responsible For
- Authentication or sessions (Authentication domain)
- Staff onboarding, roles, or branch assignment workflows (Staff Management)
- Branch lifecycle details (Branch domain)
- Authorization decisions (Access Control domain)
- Billing, subscription, or payment enforcement (future SaaS layer)
- POS operations (sales, inventory, cash, attendance, etc.)

The Tenant domain defines **what a business is**, not **who works there** or **what they can do**.

---

## 2. Ubiquitous Language

| Term | Meaning |
|---|---|
| Tenant | A business workspace boundary |
| Business Profile | Human-facing metadata of a tenant (name, logo, contact info) |
| Tenant Status | Operational state of the tenant (ACTIVE, future FROZEN) |
| Provisioning | System-driven creation of a tenant |
| Isolation Boundary | Guarantee that data does not cross tenants |
| Root Context | The top-level context all operations run under |

---

## 3. Core Concepts

### 3.1 Tenant (Aggregate Root)
Represents a single business workspace.

Typical attributes:
- `tenant_id` (stable internal identifier)
- `business_name`
- `logo_url` (optional)
- `contact_info` (optional)
- `status` (ACTIVE; FROZEN reserved)
- `created_at`, `updated_at`

**Invariant:** A tenant must always exist in a valid state before any operational data can be created.

---

### 3.2 Tenant Status (Value Object)
Represents whether a tenant is operational.

Baseline:
- `ACTIVE`

Reserved for future:
- `FROZEN` (e.g., subscription enforcement)

**Design rule:** Status exists early so enforcement can be added later without refactoring.

---

## 4. Invariants

- INV-T1: All operational data must belong to exactly one tenant.
- INV-T2: Cross-tenant data access is forbidden.
- INV-T3: A tenant is created through a **controlled provisioning flow** (admin provisioning today; self-serve signup later). It is not arbitrary UI CRUD.
- INV-T4: POS operations require at least one **ACTIVE branch**. A tenant may temporarily have **zero branches** until the first paid branch is activated/provisioned.
- INV-T5: Tenant identity is stable; renaming a business does not change tenant identity.
- INV-T6: Tenant status is a **fact**, not a permission decision (enforced elsewhere).

---

## 5. Commands (Write Intents)

The Tenant domain supports these commands:

- `ProvisionTenant`
  - provisioning command (admin provisioning today; self-serve signup later)
  - creates tenant in ACTIVE state
- `UpdateTenantProfile`
  - updates business name, logo, contact info
  - admin-only (authorization enforced elsewhere)
- `ChangeTenantStatus` (reserved for future billing)

---

## 6. Domain Events

Tenant emits the following events:

- `TENANT_CREATED`
- `TENANT_PROFILE_UPDATED`
- `TENANT_LOGO_UPDATED`
- `TENANT_STATUS_CHANGED` (future)

These events are used for:
- audit logging,
- cross-module awareness,
- future SaaS automation.

---

## 7. Read Model

Typical read queries:
- get tenant by `tenant_id`
- fetch lightweight tenant metadata (for UI headers, reports, receipts)
- read tenant status for authorization checks (future)

**Design note:** Tenant reads should be lightweight and cacheable.

---

## 8. Relationship to Other Domains

### Tenant ↔ Branch
- A tenant owns one or more branches.
- Branch lifecycle is managed by the Branch domain.
- Tenant does not guarantee a branch exists immediately after provisioning; the first branch may be provisioned only after subscription activation/payment.

### Tenant ↔ Authentication
- Tenant does not manage identities.
- Authentication provides actor identity; Tenant provides workspace context.

### Tenant ↔ Staff Management
- Staff Management handles who belongs to the tenant.
- Tenant is the container that staff belong to.

### Tenant ↔ Access Control
- Access Control consumes tenant status and tenant identity.
- Tenant does not make authorization decisions.

### Tenant ↔ Subscription / Entitlements (Future)
- Tenant will be linked to a subscriber account.
- Subscription will influence tenant status and entitlements, not tenant identity.

---

## 9. Out of Scope (Baseline)

- Billing and subscription enforcement
- Tenant deletion or ownership transfer
- Cross-tenant corporate grouping
- Advanced branding or theming
- Tenant-level feature toggles

---

## 10. Future Evolution

- Enforce tenant status in Access Control
- Link tenant to subscriber account
- Support tenant suspension/reactivation
- Support multi-tenant dashboards (operator view)
- Support tenant transfer between subscribers
