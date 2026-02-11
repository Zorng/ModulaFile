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
The Order domain models the lifecycle of a customer purchase from item selection through:
- order placement (operational commitment),
- fulfillment preparation/delivery,
- payment finalization (immediate pay or pay-later),
- and potential void authorization,
ensuring operational clarity and auditability.

### Responsibilities
- Capturing customer purchase intent in a persisted, branch-scoped record (when the workflow requires it)
- Managing fulfillment state transitions
- Supporting both operating styles:
  - pay-first: place + pay at the counter
  - pay-later: place order first, settle the bill later (table service)
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
| Open Ticket | An Order in `UNPAID` financial state (placed but not yet settled) |
| Order Line | Item + modifiers + quantity |
| Fulfillment Batch | A batch of items placed together (initial place-order or add-items); printed as a kitchen ticket/sticker |
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
- created_by_user_id (who placed the first batch)
- last_modified_by_user_id (who added items / updated fulfillment)
- sale_type
- financial_state (`UNPAID | PAID | VOID_PENDING | VOIDED | CANCELLED`)
- settled_by_user_id (nullable; who finalized payment)
- settled_at (nullable)
- cash_session_id (nullable until settled; captured at checkout/finalize time)

### Child Concepts
- FulfillmentBatch (entity):
  - batch_id
  - placed_at
  - placed_by_user_id
  - batch_fulfillment_state (`IN_PREP | READY | DELIVERED | CANCELLED`)
  - order_line_snapshots (value objects)
  - money_snapshot (value object; locked at batch creation)
- OrderLineSnapshot (value object)
- MoneySnapshot (value object)
- DiscountSnapshot (value object)
- PolicySnapshot (value object)

External effects (not owned):
- Payment records
- Cash movements
- Inventory journal entries

---

## 5. Lifecycle & State Model

### 5.1 Financial State
UNPAID → PAID → VOID_PENDING → VOIDED

For March pay-later, `CANCELLED` is allowed only for `UNPAID` tickets (operational cancellation before payment).

### 5.2 Fulfillment State
Fulfillment is modeled per-batch for March:
- each FulfillmentBatch progresses `IN_PREP → READY → DELIVERED`
- the overall Order can derive an aggregate view for list screens (implementation detail)

---

## 6. Commands

- PlaceOrder (create `UNPAID` order + first batch)
- AddItems (append a new batch)
- CheckoutOrder (settle payment; produces immutable sale snapshot externally)
- UpdateFulfillmentBatchStatus
- RequestVoid
- ApproveVoid
- RejectVoid

---

## 7. Invariants

- ORD-INV-1: Drafts are client-only; they do not mutate the backend.
- ORD-INV-2: Checkout (settlement) requires an OPEN cash session (Capstone I product rule).
- ORD-INV-3: Once a batch is placed, its money snapshot is immutable.
- ORD-INV-4: Adding items creates a new batch; it does not rewrite prior batches.
- ORD-INV-5: Fulfillment never alters money snapshots or settled sale totals.
- ORD-INV-6: Only authorized roles can approve void.
- ORD-INV-7: Checkout is idempotent.

---

## 8. Domain Events

- ORDER_PLACED
- ORDER_ITEMS_ADDED
- ORDER_FINALIZED
- ORDER_BATCH_FULFILLMENT_UPDATED
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
- PlaceOrder/AddItems require online connectivity for March pay-later (no silent offline queue).
- Checkout supports retry via idempotency key
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
