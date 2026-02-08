# Tenant Membership Domain — Who Belongs to the Business

## Domain Name
Tenant Membership

## Domain Type
Identity & Organization Domain

## Domain Group
10_Identity&Authorization

## Status
Draft (Derived from Anchor Stories A1, A5)

---

## Purpose

The Tenant Membership domain defines **who belongs to a tenant** and what kind of membership relationship exists between the business and its people.

It exists to answer:
- Who is part of this business?
- Who has authority to add/remove people?
- What is the difference between an owner, an admin, and a staff member as membership types?
- How does membership remain stable over time, even when staff leave?

Tenant Membership is the **foundation of “team setup”** without mixing in authentication credentials or day-to-day HR operations.

---

## Why This Exists (Story Traceability)

Derived from:

- **Anchor A1 — Setting Up the Business Team Quickly**  
  A tenant owner needs to bring staff into the business quickly while staying in control.

- **Anchor A5 — Preventing Unauthorized or Excessive Work**  
  The system must prevent unauthorized access; that requires knowing whether someone belongs to the tenant at all.

Tenant Membership provides the **base truth of belonging** that all other domains depend on.

---

## Core Concepts

### Tenant Member

A **Tenant Member** is an identity that belongs to a tenant in some capacity.

This is not “staff profile.”  
This is the membership link:
> “This person is part of this business.”

A tenant member may later have:
- staff profile and branch assignment (operational view),
- attendance records (reality),
- shift planning (expectations),
but membership is the root.

---

### Membership Type (Business Relationship)

A membership type expresses the relationship between a person and the tenant.

Example membership types:
- **Owner** — ultimate authority for the tenant
- **Admin** — trusted operator for setup and management
- **Staff** — operational worker

Important:
- Membership type expresses business meaning.
- Permission decisions remain owned by Access Control.
- A membership type is a stable classification that helps structure policies later.

For March delivery:
- Access Control evaluates permission roles such as `ADMIN`, `MANAGER`, `CASHIER`.
- `OWNER` membership implies an `ADMIN`-level permission role (owners are never less powerful than admins).
- Staff operational role labels (e.g., manager/cashier) live in Staff Profile & Assignment.

---

### Membership Status

Membership also has a lifecycle:
- ACTIVE
- DISABLED
- ARCHIVED (removed from active roster; history preserved)

Status defines whether a person is currently considered part of the tenant.

Historical records remain traceable even after removal.

---

### Invitation / Onboarding Method (March Delivery)

For March delivery, staff onboarding is simplified:
- The owner/admin **adds a member** by phone number (owner-provisioned).
- The system links that member to an existing Authentication identity (if one already exists for the phone),
  or **provisions** a new Authentication identity (phone identifier only; unverified; no password set).
- Authentication handles **phone verification + password setup** using OTP/self-service flows.
- No membership acceptance/decline workflow is required (membership is created immediately).

Tenant Membership stores the fact:
- this person was created/added as a tenant member,
- who created them,
- and when.

Authentication handles password setup/reset and account recovery (Staff Management never sets or knows passwords).

---

## Key Attributes (Conceptual)

A Tenant Member typically includes:
- `tenant_id`
- `member_id` (tenant-scoped membership id; the same person may have different `member_id`s in other tenants)
- `auth_account_id` (link to Authentication identity)
- `membership_type` (OWNER, ADMIN, STAFF)
- `membership_status` (ACTIVE, DISABLED, ARCHIVED)
- `created_by_member_id` (who added them)
- `created_at`
- `updated_at`

Optional:
- `removed_at`
- `notes`

---

## Invariants

- Every tenant must have at least one OWNER membership.
- An authentication identity may belong to multiple tenants (multi-tenant SaaS).
  - This enables a person to work for multiple businesses using one login identity.
- Membership must be unique per `(tenant_id, auth_account_id)` (no duplicate memberships for the same person in the same tenant).
- A tenant member’s identity must be unique within tenant constraints (Authentication enforces credential uniqueness).
- Removing membership must not delete historical records (attendance, sales, audit).

Capability-aware invariants:
- Some tenants may be entitled to limited staff capacity.
  - Membership may exceed capacity (e.g., many staff created), while operational concurrency is enforced elsewhere.
  - This supports “pay only for what you use” while still keeping onboarding flexible.

---

## What This Domain Is NOT

Tenant Membership does **not**:
- store staff operational details (branch assignment, display labels),
- plan shifts,
- record attendance,
- enforce permissions on requests,
- enforce staff concurrency,
- implement billing/subscription engine.

It only defines belonging and membership relationships.

---

## Relationship to Other Domains

### Tenant Membership ↔ Authentication
- Authentication manages credentials and login sessions.
- Tenant Membership links a tenant to an authentication identity.

---

### Tenant Membership ↔ Staff Profile & Assignment
- Staff Profile is the operational representation (who they are in day-to-day work).
- Staff Profile usually exists only for members of type STAFF (and sometimes ADMIN/OWNER as operational actors).
- Membership is the prerequisite; profile is the operational view.

---

### Tenant Membership ↔ Access Control
- Access Control uses membership type and status as primary inputs.
- Membership says “belongs”; access control says “may.”

---

### Tenant Membership ↔ Licensing / Capacity
- Membership defines the roster.
- Licensing defines concurrent usage constraints (enforced at work-start or other gated moments).

---

## Reality Considerations

- Businesses often have turnover.
- Some members are seasonal.
- Some admins may also work shifts.
- Membership must preserve history and stay understandable even after years.

The domain keeps belonging explicit so later conflicts (“who had access?”) can be resolved fairly.

---

## Out of Scope

- Enterprise account hierarchies (grouped tenants, franchise org trees)
- Advanced invitation workflows (explicitly removed for March)
- External SSO
- Subscription billing implementation

---

## Summary

Tenant Membership defines **who belongs to the business** and their relationship type (Owner/Admin/Staff), with lifecycle states that preserve history.

It provides the stable foundation that:
- Staff Profile builds on,
- Access Control evaluates,
- Attendance records against,
while staying compatible with multi-tenant SaaS (one identity can belong to multiple tenants) and future subscription entitlements.

---

_End of Tenant Membership Domain_
