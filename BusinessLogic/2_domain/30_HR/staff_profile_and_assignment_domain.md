# Staff Profile & Assignment Domain — People in the Business

## Domain Name
Staff Profile & Assignment

## Domain Type
Identity & HR Foundation Domain

## Status
Draft (Derived from Anchor Stories A1, A3, and A5)

---

## Purpose

The Staff Profile & Assignment domain represents **people who work for a tenant** and the stable facts the business needs to operate safely.

It captures:
- who a staff member is (as a person within the tenant),
- what role context they operate under (business meaning, not permission logic),
- where they are allowed to work (branch assignment / eligibility),
- whether they are currently active or not.

This domain exists so the business can prepare the team ahead of time (A1) and so starting work at a branch can be controlled and unambiguous (A3, A5).

---

## Why This Exists (Story Traceability)

Derived from:

- **Anchor A1 — Setting Up the Business Team Quickly**  
  The business must be able to create staff accounts quickly and stay in control.

- **Anchor A3 — Starting Work at a Branch**  
  Work starts at a branch, so staff must be tied to branch context meaningfully.

- **Anchor A5 — Preventing Unauthorized or Excessive Work**  
  The system must prevent clearly invalid working conditions such as wrong branch or unauthorized work.

This domain provides the **stable “who/where” facts** that other domains rely on.

---

## Core Concepts

### Staff Member

A **Staff Member** is a person recognized by the tenant as part of the working team.

A staff member is not just an authentication identity.  
They represent a business relationship and operational responsibility.

---

### Staff Profile

A **Staff Profile** contains business-level information about the staff member, such as:
- display name,
- contact identifiers (e.g., phone number),
- internal labels (e.g., staff code),
- and status.

Profiles support operations and clarity in the workplace.

---

### Branch Assignment (Workplace Eligibility)

A **Branch Assignment** defines where a staff member is eligible to work.

It answers:
- “Which branches is this staff member allowed to work at?”
- “Is this person restricted to one branch?”

This domain stores the **assignment facts**.  
Whether an action is permitted at request-time is decided by Access Control.

---

### Role Context (Business Meaning)

A staff member may have a role label such as:
- owner
- manager
- cashier
- barista

In this domain, role is treated as **business meaning** and a stable label, not an enforcement engine.

The permission logic that maps roles to allowed actions belongs to Access Control.

---

## Key Attributes (Conceptual)

A Staff Member / Profile typically includes:
- `staff_id`
- `tenant_id`
- `display_name`
- `phone_number` (or reference to Authentication identity)
- `status` (ACTIVE, INACTIVE, SUSPENDED, ARCHIVED)
- `created_at`
- `updated_at`

A Branch Assignment typically includes:
- `tenant_id`
- `staff_id`
- `branch_id`
- `assignment_status` (ACTIVE / INACTIVE)
- `assigned_at`
- `unassigned_at` (nullable)

Role context may include:
- `role_key` (e.g., cashier, manager)
- `role_assigned_at`

---

## Invariants

- Every staff member belongs to exactly one tenant.
- A staff member may be assigned to one or more branches.
- A staff member’s branch assignments may change over time; history should be preservable.
- A staff member’s identity (authentication credential) must be unique within the system constraints (handled by Authentication).
- Deactivating or archiving staff does not delete history (attendance and sales remain traceable).
- Role labels are not authorization decisions.

Capability-aware invariants:
- Some tenants may only be entitled to a single branch.
  - In that case, branch assignment is still valid but effectively constrained to the only branch available.
- Multi-branch assignment is a capability-gated behavior (enforcement belongs outside this domain).

---

## Staff Lifecycle (Business Lifecycle)

A staff member can move through states such as:

- **ACTIVE** — can work and can be scheduled
- **INACTIVE** — temporarily not working, but kept in records
- **SUSPENDED** — explicitly blocked due to business decision
- **ARCHIVED** — no longer part of the team, retained for history

The exact transitions are process concerns, but the concept of lifecycle is owned here.

---

## What This Domain Is NOT

This domain does **not**:
- authenticate users (login, password reset),
- decide permissions for each request,
- manage shift planning,
- record actual attendance,
- enforce staff capacity limits,
- implement billing or subscriptions.

It only stores stable facts about people and their workplace eligibility.

---

## Relationship to Other Domains

### Staff Profile ↔ Authentication
- Authentication proves identity and manages credentials.
- Staff Profile represents membership and business identity within the tenant.
- A staff profile links to an authentication identity via `auth_account_id` (phone number is an identifier, not a stable key).

---

### Staff Profile ↔ Tenant Membership
- Tenant Membership defines who belongs to the tenant and their ownership relationships.
- Staff Profile is the concrete representation of staff members under that tenant.
- In multi-tenant SaaS, the same authentication identity may have multiple StaffProfiles (one per tenant membership).

(Depending on design, Tenant Membership may be merged or remain distinct. For March delivery, Staff Profile is the operational view.)

---

### Staff Profile ↔ Shift
- Shifts are planned expectations for staff.
- Staff profile must exist for planning to reference a person.

---

### Staff Profile ↔ Attendance
- Attendance references staff profile for actual work.
- Attendance remains valid even if staff is later archived.

---

### Staff Profile ↔ Access Control
- Access Control decides what actions are allowed.
- Access Control consumes branch assignment and role context as inputs.

---

## Reality Considerations

- Staff turnover is normal.
- Branch assignments change.
- Some staff are full-time with stable branch placement; others float across branches.
- The system must preserve history even when staff leave.

The business needs control, but the system must remain usable under real operational messiness.

---

## Out of Scope

- Invitation workflows (explicitly removed for March delivery)
- Payroll and HR compensation
- Labor law compliance
- Scheduling optimization
- Advanced RBAC configuration UI

---

## Summary

Staff Profile & Assignment captures **who the staff are** and **where they are eligible to work**, with a lifecycle that preserves history.

It provides the stable inputs that other domains rely on:
- Shift (planning),
- Attendance (reality),
- Access Control (permission),
while remaining compatible with modular service capabilities such as single-branch vs multi-branch tenants.

---

_End of Staff Profile & Assignment Domain_
