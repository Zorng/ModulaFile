# Tenant Module — Core Module (Capstone 1)

**Version:** 1.1  
**Status:** Updated (Branch creation clarified; aligned with system-driven provisioning)  
**Module Type:** Core Module  
**Depends on:** Authentication & Authorization (Core), Audit Logging (Core)  
**Related Modules:** Branch (Core), Policy & Configuration (Core), Staff Management, Menu, Inventory, Sale, Receipt, Reporting  

---

## 1. Purpose

The Tenant module represents the **business workspace** within Modula.

A **tenant** is a distinct business environment where:
- Data is isolated from other tenants
- Roles, staff membership, branches, policies, menu, inventory, sales, attendance, cash sessions, and reports operate under the tenant boundary
- Capabilities (e.g., multi-branch) are enabled by the tenant’s subscription/build (Billing in Capstone 2; simulated in Capstone 1)

Tenant is the **root context** that all feature modules operate within.

---

## 2. Scope & Boundaries

### In scope (Capstone 1)
- Tenant representation (business identity):
  - Business name
  - Business logo (optional upload)
  - Contact info (tenant-level “business contact”, optional)
- Tenant metadata retrieval for other modules (fast read)
- Tenant lifecycle (minimal for Capstone 1):
  - System provisioning (creation)
  - Admin profile update
  - Status field reserved for future billing freeze (not enforced in Capstone 1)

### Explicitly out of scope / not owned by this module
- **Branch creation / additional branch provisioning**
  - Branches are **system-provisioned** based on the subscription/build.
  - Branch CRUD and branch freeze live in the **Branch module**.
- Authentication, sessions, tokens, password/OTP flows → Auth module
- Staff invites/membership workflows → Staff Management module
- Subscription, billing, automated freeze/unfreeze → Billing module (Capstone 2+)
- Policy configuration UI and persistence → Policy module
- Sales, menu, inventory, cash, attendance, receipts, reporting logic → Feature modules

### Key boundary rule
A tenant **cannot be directly created by an end user via a “Create Tenant” UI**.  
Tenant provisioning is performed by the **System** (or by developers/admin scripts during Capstone 1).

---

## 3. Use Cases (with actor + permissions)

### UC-1: System provisions a tenant from a subscription/build (and creates the initial branch)

**Actor:** System (internal provisioning flow; developer-run command in Capstone 1)  
**Permissions:** System-level

**Preconditions:**
- A build/subscription is completed (Capstone 2), OR
- Developer/admin executes provisioning command (Capstone 1 workaround)

**Main Flow:**
1. System creates a tenant with:
   - `business_name` (from onboarding/build input)
   - `logo_url` empty (optional)
   - `status = ACTIVE`
2. System provisions the **initial branch** for the tenant (delegated to Branch module).
3. System assigns the creator as **Admin** in Auth/membership context.
4. Audit event recorded: `TENANT_CREATED`.

**Postconditions:**
- Tenant exists and is ACTIVE.
- Tenant has at least **one branch**.
- Creator has Admin access within the tenant.

**Exceptions:**
- If any step fails, provisioning must not leave a partially created tenant without a usable branch/admin membership.

---

### UC-2: Admin updates tenant profile (business name, logo, contact info)

**Actor:** Admin  
**Permissions:** Admin-only

**Preconditions:**
- Actor is authenticated and scoped to the tenant.

**Main Flow:**
1. Admin opens **Business Profile**.
2. Admin updates allowed fields:
   - business name
   - logo (upload)
   - tenant-level contact info (optional)
3. System validates fields and persists updates.
4. Audit event recorded: `TENANT_PROFILE_UPDATED` and/or `TENANT_LOGO_UPDATED`.

**Postconditions:**
- Updated tenant profile is reflected in tenant-scoped UI where applicable (e.g., admin portal header, reports).

**Exceptions:**
- Non-admin attempt → forbidden
- Invalid image/size/type → validation error

---

### UC-3: Retrieve tenant metadata for cross-module display

**Actor:** Any authenticated module / frontend client  
**Permissions:** Authenticated within tenant context

**Preconditions:**
- Auth context provides `tenant_id`.

**Main Flow:**
1. Request tenant metadata by tenant_id.
2. Return lightweight metadata used by other modules:
   - business name
   - logo url
   - tenant status (reserved)
   - created time (optional)

**Postconditions:**
- Modules can display consistent tenant identity and apply tenant-scoped access validation.

---

### UC-4: Tenant status check (reserved for future billing enforcement)

**Actor:** System / other modules  
**Permissions:** System-level or internal service calls

**Notes (future):**
- Billing may mark tenant as `FROZEN`.
- Modules will consult status to restrict operations (Capstone 2+).

---

## 4. Functional Requirements (FR)

### Tenant representation
- **FR-1:** Store tenant business name (required), logo URL (optional), tenant-level contact info (optional), timestamps, status.
- **FR-2:** Business name must be non-empty.

### Provisioning rule
- **FR-3:** Tenant is provisioned by the system (or developer provisioning script for Capstone 1).
- **FR-4:** Tenant must always have at least **one branch** provisioned by the system.

### Updates & permissions
- **FR-5:** Only Admin may update tenant profile fields.
- **FR-6:** Tenant module must not implement membership/roles; those remain in Auth.

### Isolation
- **FR-7:** All tenant operations must be tenant-scoped; cross-tenant access must be rejected.

### Metadata access
- **FR-8:** Expose a lightweight method/API to fetch tenant metadata efficiently (cacheable).

---

## 5. Acceptance Criteria (AC)

- **AC-1:** Tenant cannot be created via end-user UI. It is system-provisioned.
- **AC-2:** Provisioning creates a tenant with at least one branch and one admin membership.
- **AC-3:** Admin can update tenant business name and logo; changes reflect across relevant UI.
- **AC-4:** Non-admin cannot update tenant profile.
- **AC-5:** Tenant data is isolated; requests cannot access another tenant’s data.
- **AC-6:** Other modules can reliably retrieve tenant metadata.

---

## 6. Out of Scope (Capstone 1)

- Billing engine integration / auto provisioning from payments
- Tenant freeze/unfreeze enforcement
- Tenant deletion and ownership transfer
- Cross-tenant corporate grouping / multi-business dashboard
- Advanced branding (themes/colors per tenant)

---

## 7. Notes for Developers

- Treat Tenant as a **stable root boundary**.
- Keep Tenant minimal: identity + metadata + status placeholder.
- Branch lifecycle is not handled here. Provisioning may *call* Branch module, but Branch remains the owner of branch data and branch lifecycle rules.

---

## Audit Events Emitted

- `TENANT_CREATED`
- `TENANT_PROFILE_UPDATED`
- `TENANT_LOGO_UPDATED`

(Branch-related audit events are owned by the Branch module.)