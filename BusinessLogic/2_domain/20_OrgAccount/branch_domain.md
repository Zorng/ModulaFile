# Branch Domain Model — Modula POS

## Domain Name
Branch (Physical Operating Location)

## Domain Type
Core Domain (Organizational + Operational Context)

## Domain Group
20_OrgAccount (Tenant + Branch)

## Status
Draft (Aligned with Identity/HR overhaul; Baseline for Capstone 1; SaaS-ready)

---

## Purpose

The Branch domain defines the **physical operating location** within a Tenant where work and POS operations occur.

A **Branch** is the real-world store/workplace context that anchors:
- where staff work and attendance is recorded,
- where sales and cash sessions occur,
- where inventory is stocked and adjusted,
- what location is printed as the “store header” on receipts,
- and the operational status gate for branch-scoped workflows (e.g., frozen branch = no operational writes).

Branch is **not** “just an address record.”  
It is the operational boundary that many workflows must reference consistently.

---

## Why Branch Exists (Story Context)

Branch concepts show up in daily café life:

- “This cashier works at the Sen Sok branch, not the Toul Kork branch.”
- “Attendance check-in should record evidence that they are physically near the branch.”
- “Receipts should show which branch issued the sale.”
- “If a branch is temporarily suspended, the system should stop accepting operational writes.”

This domain exists to make those expectations explicit and enforceable.

---

## Core Concepts

### 1) System-Provisioned Branches
Branch creation is **provisioning-driven**, not a typical CRUD feature.

A branch record is created by the system when:
- a tenant activates/pays for branch capacity (first branch activation; future billing engine), and/or
- a subscription entitlement enables additional branches, and/or
- developers/admin tooling provision branches manually (Capstone 1 practicality).

**Implication:** End-users can update branch profile/configuration, but do not create branches directly in the MVP.

### 2) Branch Profile (Operational Identity)
Branch profile is the human-friendly identity of the location:
- display name (e.g., “Starray — Sen Sok”)
- physical address (where the store is)
- contact number (branch phone)
- optional notes / directions for staff (future)

### 3) Branch Status (Operational Gate)
Branch has an operational status:
- `ACTIVE`: operational writes allowed
- `FROZEN`: operational writes blocked (read-only still allowed)

This status acts as a **cross-cutting guard** used by branch-scoped modules.

### 4) Workplace Location (for Attendance Confirmation)
If attendance location confirmation is enabled, each branch can define:
- a workplace geo-point (lat/long)
- an allowed distance/radius threshold

This does **not** imply staff tracking.  
It is used only to confirm location at check-in/check-out time.

> Capability note: GPS attendance is optional (“pay for what you use”), but Branch still owns the workplace location configuration when the capability is enabled.

---

## Entities and Value Objects

### Entity: Branch
**Identity:** `branch_id` (stable)  
**Key attributes:**
- `tenant_id` (owner tenant)
- `status` (`ACTIVE` | `FROZEN`)
- `profile` (BranchProfile)
- `workplace_location` (optional BranchWorkplaceLocation)
- timestamps / audit metadata

### Value Object: BranchProfile
- `display_name`
- `address_text`
- `phone_number` (optional)
- other “printed identity” fields as needed

### Value Object: BranchWorkplaceLocation (optional)
- `latitude`
- `longitude`
- `allowed_radius_meters`
- `effective_from` (optional, for forward-only changes)

---

## Invariants (Must Always Be True)

1. **Tenant ownership:** A branch belongs to exactly one tenant (`branch.tenant_id` is immutable after creation).
2. **Status gating:** Branch-scoped operational writes must enforce `branch.status == ACTIVE`.
3. **History safety:** Updating branch profile/config affects **future** receipts/operations; historical records remain unchanged.
4. **Location configuration validity:** If workplace location is set, it must include both geo-point and radius.
5. **Capability-gated usage:** Attendance location confirmation must only use branch workplace location if the capability is enabled for the tenant/branch.

---

## Commands (Write Operations) Owned by This Domain

### 1) UpdateBranchProfile
Update branch display name/address/contact information.

**Intent:** Keep the store identity accurate over time (relocation, rebrand, updated contact).

### 2) FreezeBranch / UnfreezeBranch
Change branch operational status.

**Intent:** Temporarily prevent operational writes during suspension, investigation, or closure.

### 3) ConfigureWorkplaceLocation (Capability-gated)
Set or update workplace geo-point and radius for attendance confirmation.

**Intent:** Provide a reference point for “are you at the workplace?” checks during check-in/out.

### 4) ProvisionBranch (System-only)
Create a branch record via provisioning workflow (system/internal tooling).

---

## Domain Events

- `BranchProvisioned`
- `BranchProfileUpdated`
- `BranchFrozen`
- `BranchUnfrozen`
- `BranchWorkplaceLocationConfigured`

These events are useful for:
- audit logging,
- read model updates,
- and cross-module orchestration (e.g., deny check-in if branch frozen).

---

## Read Model vs Write Model

### Write Model (authoritative)
- Branch entity with status/profile/location configuration

### Read Model (optimized queries)
- “Branches accessible to this user”
- “Branch summary cards for selection UI”
- “Branch operational readiness status”
- “Branch workplace location settings” (for attendance flows)

---

## Relationships and Dependencies

### Branch is referenced by
- **Attendance**: check-in/out records reference `branch_id`; location confirmation uses `BranchWorkplaceLocation`.
- **Shift Planning**: shifts often target a branch (`shift.branch_id`).
- **Staff Assignment**: staff are assigned to branch(es) for work eligibility.
- **Access Control**: authorization decisions often include branch scope (branch is a key input).
- **Sale / Cash Session / Inventory**: operational records are branch-scoped.
- **Receipt**: receipts show branch identity as the seller location (branch header). Tenant identity may appear as company header separately.

### Branch depends on
- **Tenant**: branch cannot exist without a tenant.
- **Authentication + Tenant Membership**: to determine who can view/update what branch fields.
- **Capability/Entitlements** (future): to gate optional features (GPS attendance, multi-branch).

---

## Out of Scope (For This Domain Doc)

- How staff are assigned to branches (belongs to Staff Profile & Assignment / Tenant Membership).
- How shifts are created/recurring rules (belongs to Shift domain).
- Attendance check-in/out workflows (belongs to Attendance + Work Start/End orchestration).
- Receipt layout customization (belongs to Receipt module/spec; Branch only supplies branch identity fields).
- Subscription/billing provisioning engine (future platform system).

---

## Failure Modes and Guardrails

- **Branch frozen but operation attempted:** return a clear “branch frozen” error (server-side enforced).
- **Location confirmation enabled but branch location missing:** allow check-in/out but record UNKNOWN; guide admin/manager to configure branch workplace location for stronger evidence.
- **User tries to edit branch without permission:** Access Control denies (consistent error messaging).
- **Stale client / offline edits:** server remains source of truth; conflicts resolved via standard sync rules (offline sync module).

---

## Notes for Capstone 1 Shipping

For March deliverable:
- Branch creation can be system-seeded or dev-provisioned.
- Admin can update branch profile and configure attendance location (if enabled).
- Freeze/unfreeze can exist as a lightweight admin control (even if rarely used).

The key is that Branch becomes a stable reference for:
attendance, authorization, and POS operations — without requiring full billing/provisioning automation yet.
