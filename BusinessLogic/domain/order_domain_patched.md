# Order Domain Model — Modula POS

## Domain Name
Order

## Domain Type
Core Business Domain

## Status
Defined (Capstone baseline)

## Applies To
Food & Beverage (single-register-first)

---

## 1. Domain Purpose & Responsibility

### Purpose
The Order domain models the lifecycle of a customer purchase from item selection through payment finalization, fulfillment, and potential void authorization, ensuring financial correctness, operational clarity, and auditability.

### Responsibilities
- Capturing customer purchase intent
- Enforcing checkout invariants (e.g. cash session required)
- Producing immutable financial outcomes at checkout
- Managing fulfillment state transitions
- Authorizing void requests and approvals
- Emitting domain events required for external reversal processes

### Explicitly NOT Responsible For
- Inventory stock rules or deduction logic
- Cash accountability totals
- Cross-domain orchestration
- Reporting aggregation
- Staff authorization logic

---

## 2. Domain Boundary & Ownership

### Boundary
The Order domain begins when a purchase intent is created and ends when the order reaches a terminal financial state.

### Owns
- Order identity and lifecycle
- Order line composition
- Pricing, discount, and policy snapshots
- Financial state (PAID, VOIDED)
- Fulfillment state transitions

### External Inputs & References
- cash_session_id (required at finalize time)
- pricing / discount / policy snapshots
- actor role information (authorization context)

---

## 3. Core Concepts (Ubiquitous Language)

| Term | Meaning |
|------|--------|
| Order | Aggregate root representing a paid or voidable purchase |
| Draft | Client-only cart state before finalization |
| Order Line | Item + modifiers + quantity |
| Finalize | Commit order as paid-in-full |
| Void | Approved reversal of a finalized order |
| Fulfillment | Operational preparation/delivery state |

---

## 4. Aggregate & Entities

### Aggregate Root
Order

### Order Entity
- order_id
- tenant_id
- branch_id
- cash_session_id
- created_by_user_id
- sale_type

### Child Concepts
- OrderLine (value object)
- PricingSnapshot (value object)
- DiscountSnapshot (value object)
- PolicySnapshot (value object)

External effects (not owned):
- Payment records
- Cash movements
- Inventory journal entries

---

## 5. Lifecycle & State Model

### 5.1 Financial State
PAID → VOID_PENDING → VOIDED

### 5.2 Fulfillment State
IN_PREP → READY → DELIVERED
(CANCELLED only via VOID)

---

## 6. Commands

- FinalizeOrder
- UpdateFulfillmentStatus
- RequestVoid
- ApproveVoid
- RejectVoid

---

## 7. Invariants

- ORD-INV-1: Cannot finalize without OPEN cash session
- ORD-INV-2: Finalized pricing is immutable
- ORD-INV-3: Payment is immutable; void uses reversal
- ORD-INV-4: Fulfillment never alters financial totals
- ORD-INV-5: Only authorized roles can approve void
- ORD-INV-6: Finalize is idempotent

---

## 8. Domain Events

- ORDER_FINALIZED
- ORDER_FULFILLMENT_UPDATED
- VOID_REQUESTED
- VOID_APPROVED
- VOID_REJECTED
- ORDER_VOIDED

---

## 9. Read vs Write Model

### Write Model
- Order aggregate enforcing invariants

### Read Model
- Order list and detail views
- Reporting via event consumption

---

## 10. Offline & Idempotency

- Drafts are client-only
- FinalizeOrder supports retry via idempotency key
- Server ordering is authoritative

---

## 11. Failure Modes (Domain-Local)

- Invariant violation → reject command
- Duplicate finalize → no-op (idempotent)
- Void rejection → return to PAID

---

## 12. Out of Scope

- Partial payments
- Split bills
- Refunds after close
- Multi-register concurrency

---

## 13. Future Evolution

- Partial/split payments
- Multi-register support
- Server-side shift evaluation
