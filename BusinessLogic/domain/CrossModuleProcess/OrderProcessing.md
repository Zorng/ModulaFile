# Order Processing

---

# **Order Domain Model — Modula POS**

**Domain Name:** Order

**Domain Type:** Core Business Domain

**Status:** Defined (Capstone 2 baseline)

**Applies to:** Food & Beverage (single-register-first)

**Last Updated:** 2026-01

---

## **1. Domain Purpose & Responsibility**

### **Purpose**

The Order domain models the lifecycle of a customer purchase from item selection through payment finalization, fulfillment, and potential voiding, ensuring **financial correctness**, **operational clarity**, and **auditability**.

### **Responsibilities**

The Order domain is responsible for:

- Capturing customer purchase intent
- Enforcing checkout rules (cash session requirement)
- Producing immutable financial outcomes at checkout
- Managing fulfillment state transitions
- Coordinating reversal workflows (voids) through other domains

### **Explicitly NOT Responsible For**

- Discount rule creation (Discount domain)
- Inventory stock rules (Inventory domain)
- Cash accountability totals (Cash Session domain)
- Staff authorization (Auth / Staff Management)
- Reporting aggregation (Reporting domain)

---

## **2. Domain Boundary & Ownership**

### **Boundary**

The Order domain begins when a user starts building a purchase (draft) and ends when the order reaches a terminal financial state.

### **Owns**

- Order identity and lifecycle
- Order line composition
- Pricing and discount snapshots
- Financial state (paid / voided)
- Fulfillment state (prep → delivered)

### **Depends On**

- **Cash Session** (must be OPEN to finalize)
- **Policy** (VAT, FX, rounding – branch-scoped)
- **Discount** (effective discounts for branch)
- **Inventory** (journal effects, not stock rules)
- **Audit Log** (recording actions)

---

## **3. Core Concepts (Ubiquitous Language)**

| **Term** | **Meaning** |
| --- | --- |
| Order | Aggregate root representing a finalized or voidable purchase |
| Draft | Local-only cart state before finalization |
| Order Line | Item + modifiers + quantity within an order |
| Finalize | Commit order as paid-in-full |
| Payment | Immutable record of what the customer paid |
| Void | Approved reversal of a finalized order |
| Fulfillment | Operational progress of preparing/delivering order |
| Cash Session | Accountability window required for checkout |

---

## **4. Aggregate & Entities**

### **Aggregate Root:**

### **Order**

### **Order Entity**

- order_id
- tenant_id
- branch_id
- cash_session_id
- created_by_user_id
- sale_type (dine-in / takeaway / delivery)

### **Child Concepts**

- **OrderLine** (value object)
- **PricingSnapshot** (value object)
- **DiscountSnapshot** (value object)
- **PolicySnapshot** (value object)

> Payment, CashMovement, InventoryJournal are
> 
> 
> **external effects**
> 

---

## **5. Lifecycle & State Model**

### **5.1 Draft Lifecycle (Client-only)**

| **State** | **Notes** |
| --- | --- |
| NO_DRAFT | Default |
| DRAFT_ACTIVE | Exists in local storage only |

Rules:

- One draft per device per branch
- Draft persists during an OPEN cash session
- Draft must be resolved on:
    - branch switch
    - logout
    - cash session close

Resolution options:

- Finalize
- Discard

---

### **5.2 Financial State (Server)**

| **State** | **Meaning** |
| --- | --- |
| PAID | Finalized, paid-in-full |
| VOID_PENDING | Void requested, awaiting approval |
| VOIDED | Void approved and executed |

---

### **5.3 Fulfillment State (Server)**

| **State** | **Meaning** |
| --- | --- |
| IN_PREP | Default after finalize |
| READY | Prepared |
| DELIVERED | Completed |
| CANCELLED | Terminal (via void only) |

---

## **6. Commands (Write Intents)**

### **Draft Commands (Client)**

- CreateDraft
- AddItemToDraft
- UpdateDraftItem
- RemoveDraftItem
- DiscardDraft

### **Order Commands (Server)**

- FinalizeOrder
- UpdateFulfillmentStatus
- RequestVoid
- ApproveVoid
- RejectVoid

---

## **7. Invariants (Hard Rules)**

- **ORD-INV-1:** An order cannot be finalized without an OPEN cash session
- **ORD-INV-2:** Finalized pricing is immutable
- **ORD-INV-3:** Payment is immutable; void uses reversal records
- **ORD-INV-4:** Fulfillment changes never alter financial totals
- **ORD-INV-5:** Only Admin/Manager can approve void
- **ORD-INV-6:** Idempotent finalize (retry-safe)
- **ORD-INV-7:** One OPEN cash session per branch (baseline)

---

## **8. Domain Events**

- ORDER_FINALIZED
- ORDER_FULFILLMENT_UPDATED
- VOID_REQUESTED
- VOID_APPROVED
- VOID_REJECTED
- ORDER_VOIDED

---

## **9. Read vs Write Model**

### **Write Model**

- Order aggregate + commands
- Strong invariants enforced transactionally

### **Read Model**

- Orders list (by branch)
- Order detail + eReceipt view
- Reporting consumes events, not aggregates

---

## **10. Offline, Concurrency & Idempotency**

- Drafts are client-only
- FinalizeOrder supports offline queueing
- Each finalize carries idempotency_key
- Server authoritative ordering
- Concurrent finalize handled transactionally

---

## **11. Failure Modes & Recovery**

| **Scenario** | **Handling** |
| --- | --- |
| Network drop during finalize | Retry via idempotency |
| Cash session closed mid-draft | Prompt user |
| Void rejected | Order returns to PAID |
| Inventory conflict | Allowed to go negative (policy decision) |

---

## **12. Out of Scope (Explicit)**

- Partial payments
- Split bills
- Multi-register concurrency
- Refund after day close
- QR void/refund

---

## **13. Notes & Future Evolution**

- Payment modes: paid-in-full → partial → split
- Multi-register support (multiple sessions)
- Server-side shift evaluation
- Stronger inventory reservation models

---