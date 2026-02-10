# Subscription & Entitlements Domain Model — Modula POS

## Domain Name
Subscription & Entitlements (Billing Guard Rails)

## Domain Type
Platform / Supporting Domain (Commercial Enforcement + Capability Snapshot)

## Domain Group
60_PlatformSystems

## Status
Draft (Baseline for March pilot: USD-only, KHQR self-serve; branch-scoped entitlements; tenant billing anchor)

---

## Purpose

This domain defines the **commercial enforcement truth** for Modula:
- how a tenant’s subscription lifecycle is represented (active vs overdue vs frozen),
- how “what is unlocked” is represented (entitlements),
- and the guard rails that keep pricing decisions enforceable without contaminating operational domains.

It exists so that other domains can reliably answer:
- “Is this tenant allowed to operate right now?”
- “Is this module enabled for this branch?”
- “Is this action writable or read-only under the current plan?”
- “Are operator seats available for starting work?”

This domain is intentionally **capability-first**:
it controls *whether* capabilities are available, not *how* operational workflows are executed.

---

## Domain Boundary & Ownership

### What This Domain Owns

- Tenant billing anchor (`DL-BE-SUB-01`).
- Subscription billing state model: `ACTIVE` / `PAST_DUE` / `FROZEN` (`DL-BE-SUB-03`).
- Invoice concept (USD-only for March) and line-item intent (not pricing tables).
- Entitlement catalog keys and entitlement snapshot semantics:
  - scope: `TENANT` vs `BRANCH`
  - enforcement: `ENABLED` / `READ_ONLY` / `DISABLED_VISIBLE`
- Branch activation gating (“no branch exists until paid”) and first-branch activation draft (`DL-BE-BRANCH-01`, `DL-BE-ACT-01`).
- Operator seat limits per branch (commercial limit), consumed at `START_WORK` (`DL-BE-SLOT-01`).

### What This Domain Does NOT Own

- Payment rail protocols and SDK details (Bakong integration mechanics).
- Identity, login sessions, and authentication (Authentication domain/module).
- Authorization policy (Access Control decides who may do what).
- Operational workflows and state machines (Sale, CashSession, Inventory, Workforce).
- UI/UX layout (Branch page, Plan screens, etc.).
- Anti-abuse technical caps for entity creation (documented as Fair-Use; not a pricing rule).

This domain provides **states and invariants**; processes orchestrate timing.

---

## Core Concepts (Ubiquitous Language)

### Billing Anchor (Tenant-Level)
The tenant has a single renewal day (“billing anchor”), set when the tenant activates the **first paid branch**.

### Subscription State (Tenant-Level)
Represents the commercial operating status of the tenant:
- `ACTIVE`: operating normally.
- `PAST_DUE`: invoice unpaid at renewal; 24h grace where operations continue with warnings.
- `FROZEN`: after grace window; operational writes are blocked; read-only remains available.

### Invoice (Tenant-Level, USD-Only for March)
An invoice is the commercial request for payment, with:
- a stable invoice identity,
- USD-denominated totals,
- line-item intent (branch capacity, module enablement, seats),
- and a payment confirmation state.

### Entitlement (Capability Snapshot)
An entitlement answers “is capability X available under plan Y?” for a given scope.

Fields (conceptual):
- `entitlement_key` (stable string key)
- `scope`: `TENANT` or `BRANCH`
- `enforcement`: `ENABLED` | `READ_ONLY` | `DISABLED_VISIBLE`

### Branch Activation Draft (Zero-Branch Entry Point)
When a tenant has zero branches, Modula may collect minimal branch details as a draft
and only provision the real branch after payment is verified.

### Operator Seats (Branch-Level Commercial Limit)
Defines the maximum number of concurrent operators for a branch.
Seats are consumed at `START_WORK` and released at `END_WORK`.

---

## Invariants (Non-Negotiable)

- **INV-SUB-1 (Single anchor):** A tenant has exactly one billing anchor date.
- **INV-SUB-2 (Anchor origin):** Billing anchor is set by first paid branch activation (not by signup).
- **INV-SUB-3 (USD-only invoices for March):** Subscription invoices are denominated in USD only (no FX logic in March).
- **INV-SUB-4 (No unpaid branches):** A branch does not exist unless paid/unlocked via subscription provisioning.
- **INV-SUB-5 (Capability is explicit):** Modules/capabilities are never inferred from “data exists”; they must be represented as entitlements.
- **INV-SUB-6 (Read-only semantics):** When a module is not subscribed, it may still be visible, but writes must be blocked (enforcement is not a UI choice).
- **INV-SUB-7 (Freeze semantics):** When subscription state is `FROZEN`, operational writes must be blocked system-wide; read-only access is allowed.
- **INV-SUB-8 (Seat fairness):** A branch must not allow more concurrent operators than its seat limit.

---

## Entities and Value Objects

### Entity: TenantSubscription
Represents the tenant’s commercial relationship and enforcement state.

Key attributes (conceptual):
- `tenant_id`
- `billing_anchor_day` (or anchor timestamp)
- `subscription_state` (`ACTIVE` | `PAST_DUE` | `FROZEN`)
- `current_period_start_at`, `current_period_end_at`

### Entity: Invoice
Represents a USD payment request for a tenant at a point in time.

Key attributes (conceptual):
- `invoice_id`
- `tenant_id`
- `currency = USD`
- `line_items[]`
- `status` (`ISSUED` | `PAID` | `VOID` | `FAILED`)

### Value Object: EntitlementSnapshot
A resolved, read-optimized view of:
- which entitlements are enabled for a tenant and a branch at “now”
- and which are read-only/disabled-visible

It is consumed by:
- Access Control (deny writes when read-only/disabled)
- Modules (consistent enable/disable behavior)
- UX (upgrade prompts, warnings)

### Entity: BranchActivationDraft
Represents a pre-branch activation record:
- minimal branch details (draft)
- invoice reference
- intended outcome: provision a real branch after payment verification

### Value Object: OperatorSeatLimit
Per-branch seat limit and included seats, consumed by workforce “start work” gating.

---

## Read Contract (Conceptual)

This domain must be consumable as simple read decisions (not ad-hoc queries):

- `GetSubscriptionState(tenant_id) -> ACTIVE | PAST_DUE | FROZEN`
- `GetEntitlementSnapshot(tenant_id, branch_id?) -> EntitlementSnapshot`
  - `branch_id` is optional for tenant-scoped entitlements
- `GetOperatorSeatLimit(tenant_id, branch_id) -> OperatorSeatLimit`

These reads are consumed by:
- Access Control (deny writes when not entitled or frozen),
- Processes (gating transitions like first-branch activation and start-work),
- and UX (displaying plan state without inventing rules).

---

## Relationships to Other Domains

- **Tenant / Branch (OrgAccount):**
  - This domain provisions branches only after payment is verified (system-only).
  - It may drive tenant/branch operational gating via `FROZEN` enforcement.
- **Access Control (Identity&Authorization):**
  - Access Control consumes entitlements to deny writes when not entitled.
  - Access Control consumes tenant/branch operational gating (ACTIVE vs FROZEN).
- **Workforce (HR):**
  - Workforce processes consume seat limits at `START_WORK` to enforce concurrency fairly.
- **Audit (PlatformSystems):**
  - Plan changes, invoice events, and state transitions must emit audit-grade events.

---

## Out of Scope (For This Domain Doc)

- Pricing tables, discounts, and promotions for subscription pricing.
- Exact proration formulas and rounding rules (belongs to process/modspec later).
- Payment processor integration mechanics (Bakong API details).
- Customer support policies and refunds.

---

## Summary

Subscription & Entitlements is a platform domain that makes Modula’s “pay for what you use” enforceable by defining:
- tenant subscription states (`ACTIVE` / `PAST_DUE` / `FROZEN`),
- explicit entitlements with scope and enforcement levels,
- branch activation gating (“no unpaid branches”),
- and operator seat limits consumed at `START_WORK`.
