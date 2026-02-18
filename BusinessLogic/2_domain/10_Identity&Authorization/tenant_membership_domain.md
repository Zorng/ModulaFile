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
- What is the difference between **ownership (governance)** and **operational roles** (admin/manager/cashier)?
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

### Membership Kind (Governance Relationship)

A membership kind expresses the **governance relationship** between a person and the tenant.

For March delivery:
- `OWNER` — ultimate authority for the tenant (governance invariant)
- `MEMBER` — everyone else

Important:
- Membership kind is governance meaning, not an operational permission bundle.
- Permission decisions remain owned by Access Control (via role policy + action keys).
- Ownership transfer is out of scope for March, but the model must support it later.

---

### Tenant Role Key (Authorization Role)

A tenant role key expresses the **tenant-scoped authorization role** granted to a member.

The role key is a stable fact owned by Tenant Membership and **consumed by Access Control**.
Access Control owns the permission matrix (which actions each role may perform).

For March delivery, the built-in role keys are:
- `ADMIN` — trusted operator for setup and management
- `MANAGER` — operational supervisor role
- `CASHIER` — operational worker role

Design rule:
- Role keys must be **extendable** (e.g., adding `INVENTORY_CLERK`) without rewriting HR domains/processes.
- Role keys are not job titles; job titles belong to Staff Profile & Assignment for display/scheduling context only.

---

### Membership Status

Membership has a lifecycle that supports explicit invitations:
- INVITED (pending acceptance)
- ACTIVE
- REVOKED (removed from active roster; history preserved)

Status defines whether a person is currently considered part of the tenant.

Historical records remain traceable even after removal.
If membership is granted again later, the actor may view their own historical records under current role policy and tenant scope.

---

### Invitation / Onboarding Method (March Delivery)

For March delivery, onboarding uses **explicit invitations**:
- The owner/admin **invites a member** by phone number.
- An invite creates a TenantMembership in `INVITED` state (no operational access yet).
- The invited person **accepts or rejects** the invite from their account.
- Acceptance moves membership to `ACTIVE` and enables downstream HR provisioning.

Authentication handles **phone verification + password setup** using OTP/self-service flows.
Admins/owners never set or know passwords.

Tenant Membership stores the facts:
- this person was invited,
- who invited them,
- and when they accepted (or rejected).

---

## Key Attributes (Conceptual)

A Tenant Member typically includes:
- `tenant_id`
- `member_id` (tenant-scoped membership id; the same person may have different `member_id`s in other tenants)
- `auth_account_id` (link to Authentication identity)
- `membership_kind` (`OWNER` or `MEMBER`)
- `role_key` (`ADMIN`, `MANAGER`, `CASHIER`, ...)
- `membership_status` (INVITED, ACTIVE, REVOKED)
- `invited_by_member_id` (who invited them)
- `created_at`
- `updated_at`

Optional:
- `invited_at`
- `accepted_at`
- `rejected_at`
- `removed_at`
- `notes`

---

## Invariants

- Every tenant must have at least one ACTIVE `OWNER` membership.
- An authentication identity may belong to multiple tenants (multi-tenant SaaS).
  - This enables a person to work for multiple businesses using one login identity.
- Membership must be unique per `(tenant_id, auth_account_id)` (no duplicate memberships for the same person in the same tenant).
- A tenant member’s identity must be unique within tenant constraints (Authentication enforces credential uniqueness).
- Removing membership must not delete historical records (attendance, sales, audit).
- INVITED memberships grant **no** operational access until accepted.
  
Authorization-aware invariants:
- Each ACTIVE membership must have a `role_key`.
- An `OWNER` must never be less powerful than an `ADMIN` in Access Control policy (implementation can satisfy this by ensuring owners are granted `ADMIN`-equivalent permissions).

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
- Staff Profile exists for members who operate day-to-day in the tenant (including owners/admins if they work shifts).
- Membership is the prerequisite; profile is the operational view.

---

### Tenant Membership ↔ Access Control
- Access Control uses membership status and `role_key` as primary inputs (and may use `membership_kind` for governance-only invariants).
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
- Advanced invitation workflows (bulk invites, reminders, auto-expiry) out of scope for March
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
