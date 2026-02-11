# 05 — Tenant Provisioning Orchestration (Create Business)

## Purpose

This process defines the canonical behavior for **creating a new tenant** (business workspace) in Modula.

It exists to keep three truths consistent:
- Tenant creation is **user-triggered** for March ("Create Business"), but still a **controlled provisioning flow** (not arbitrary CRUD).
- Provisioning creates a tenant plus the initial `OWNER` membership for the creator.
- A tenant may exist with **zero branches** until the first paid branch is activated (branch is provisioned after payment confirmation).

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/saas_governance/creating_a_business_workspace.md`

---

## Domains / Modules Involved

- Authentication (actor identity + session)
- Tenant (workspace boundary)
- Tenant Membership (ownership + role facts)
- Audit (evidence of provisioning)
- Platform guardrails:
  - Idempotency gate (prevent duplicate tenants on retries/double-tap)
  - Fair-use limits (anti-abuse)

---

## When This Process Runs

Triggered when an authenticated user chooses `Create Business`:
- from the tenant selection screen, or
- from the "no memberships" empty state after login.

This process does not require an existing `tenant_id` context.

---

## Inputs (Minimum)

- `auth_account_id` (actor identity)
- `business_name`
- optional: lightweight contact info (deferred if not needed for March)

---

## Orchestration Steps

### Step 1 — Validate actor + apply platform guardrails

- Confirm the actor is authenticated (`auth_account_id`).
- Apply fair-use / rate limits for tenant provisioning.
- Accept an idempotency key (recommended) and pass through the idempotency gate.

Notes:
- Idempotency should guarantee that retries return the same `tenant_id` without creating duplicates.

---

### Step 2 — Provision tenant (ACTIVE)

- Create a new `Tenant(status = ACTIVE)`.

---

### Step 3 — Create initial OWNER membership

- Create a `TenantMembership` for the creator:
  - `membership_kind = OWNER`
  - `role_key = ADMIN` (March baseline)
  - `membership_status = ACTIVE`

Governance rule:
- A tenant must always have at least one ACTIVE owner.

---

### Step 4 — Record audit evidence

- Record audit events:
  - `TENANT_CREATED`
  - `TENANT_OWNER_GRANTED` (or equivalent membership-grant event)

---

### Step 5 — Return tenant context (no branch yet)

- Return `tenant_id`.
- The tenant may have **zero branches** at this moment.
- Next step (not part of this process): first branch activation (payment -> branch provisioned) using:
  - `BusinessLogic/4_process/60_PlatformSystems/90_first_branch_activation_orchestration.md`

---

## Failure / Degradation Rules (March)

- If the request is retried (double tap / network retry), idempotency must prevent creating multiple tenants unintentionally.
- If tenant provisioning succeeds but later steps fail (e.g., audit logging), the system must still return a usable `tenant_id` and remain recoverable.

