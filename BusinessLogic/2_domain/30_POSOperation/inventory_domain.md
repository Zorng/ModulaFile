# Inventory Domain Model — Modula POS

## Domain Name
Inventory

## Domain Type
Core Business Domain (POS Operations)

## Status
Defined (Capstone baseline)

## Purpose
The Inventory domain models **stock truth** for a branch through an audit-friendly ledger of stock movements, supporting restocking, adjustments, and stock visibility for day-to-day café operations.

This domain prioritizes:
- **Auditability** (append-only history)
- **Operational simplicity** (works without warehouse-grade allocation)
- **Offline safety** (idempotent, replay-friendly commands)

---

## 1. Domain Boundary & Ownership

### Boundary
Inventory begins when a stock item is defined for a tenant/branch context and continues through stock movements (restock, adjustment, deductions, reversals). Inventory ends at providing current on-hand projections and audit history.

### Owns (Write Model)
- Stock item definitions (what can be stocked)
- Restock batches (inputs to stock increases)
- Inventory journal (append-only stock movement ledger)
- Domain invariants over stock movements (idempotency, immutability rules)
- Stock visibility projections (current on-hand, low stock, expiring soon)

### Explicitly NOT Responsible For
- Deciding **when** to deduct stock for a sale (orchestrated by processes)
- Menu recipe logic or product configuration (recipe vs direct-stock mapping)
- Cash accountability and payment outcomes
- Procurement workflows (purchase orders, suppliers) — future scope
- Enforcing warehouse-grade allocation (FIFO/FEFO picking) — out of scope for Capstone baseline

---

## 2. Core Concepts (Ubiquitous Language)

| Term | Meaning |
|---|---|
| StockItem | A thing that can be counted (e.g., milk, beans, bottled water) |
| Unit | Measurement unit (pcs, ml, g, etc.) |
| RestockBatch | A restock input with quantity and optional metadata (expiry, note) |
| InventoryJournal | Append-only ledger of movements that produce stock truth |
| Movement | A single stock change entry (IN/OUT) recorded in the journal |
| Adjustment | A corrective movement (waste, count correction, damage) |
| On-hand Stock | Derived quantity currently available for a branch |
| Expiry Date | Optional batch metadata; if missing, item is treated as non-expiring for reporting |
| BranchStock Projection | Materialized/derived view of on-hand per item per branch |

---

## 3. Entities, Value Objects, Aggregates

### Aggregate Candidates
Inventory is modeled as **ledger-first**:
- **InventoryJournal** is the source of truth (append-only)
- **BranchStock Projection** is derived from the journal (may be materialized for performance)

### Entities / Records
- **StockItem**
  - stock_item_id
  - tenant_id
  - branch_scope (if per-branch catalog) or tenant-wide + branch availability
  - name / sku (optional)
  - base_unit (e.g., pcs, ml, g)
  - low_stock_threshold (optional)
  - is_active

- **RestockBatch**
  - restock_batch_id
  - stock_item_id
  - branch_id
  - quantity_in_base_unit
  - received_at
  - expiry_date (optional)
  - note (optional)
  - created_by
  - status: RECORDED (baseline; no complex receiving workflow)

- **InventoryMovement (Journal Entry)**
  - movement_id
  - branch_id
  - stock_item_id
  - direction: IN | OUT
  - quantity_in_base_unit (positive number)
  - reason_code: RESTOCK | SALE_DEDUCTION | VOID_REVERSAL | ADJUSTMENT | TRANSFER (future) | OTHER
  - source_type (e.g., RESTOCK_BATCH, ORDER, ADJUSTMENT)
  - source_id (e.g., restock_batch_id, order_id, adjustment_id)
  - idempotency_key (required for externally-triggered movements)
  - occurred_at
  - actor_id (who initiated)
  - note (optional)

### Value Objects
- **Quantity** (amount + unit; converted to base_unit for storage)
- **Expiry** (date or null)
- **ReasonCode** (enumeration)

---

## 4. Lifecycle & Rules

### StockItem lifecycle
- ACTIVE → INACTIVE
Inactive items cannot be restocked or moved, but remain in history.

### RestockBatch lifecycle
- RECORDED (baseline)
A restock batch is an input record. Stock truth is produced by the journal movement(s) referencing it.

### InventoryJournal lifecycle
- Append-only; never edited or deleted in normal operation.
Corrections happen via new movements (adjustments or reversals).

---

## 5. Commands (Write Intents)

Self-contained domain commands:
- CreateStockItem
- UpdateStockItem (name/threshold/active flag; **not** historical quantities)
- RecordRestockBatch (creates batch + associated IN movement)
- RecordAdjustment (creates OUT/IN movement with reason)
- UpdateRestockBatchMetadata (expiry_date, note) — metadata only

Externally-triggered commands (accepted but not orchestrated here):
- ApplyExternalDeduction (e.g., sale deduction) — idempotent
- ApplyExternalReversal (e.g., void reversal) — idempotent

---

## 6. Invariants (Non-negotiable Rules)

- INV-1 **Append-only journal**: movements are never edited/deleted; corrections are new movements.
- INV-2 **Idempotency**: for externally-triggered movements, `(branch_id, source_type, source_id)` or `idempotency_key` must be unique; duplicates are no-op.
- INV-3 **Quantity stored in base unit**: all movements store quantity in stock item base unit.
- INV-4 **StockItem must be active** to accept new restock or movement commands.
- INV-5 **Metadata edits do not mutate stock**: changing expiry/note on a batch never changes on-hand quantities.
- INV-6 **BranchStock is derived** from the journal (source of truth is journal, not projection).

Optional policy (product decision; if enabled later):
- INV-OPT-NEG **Negative stock handling**: allow vs prevent negative on-hand is a policy. Baseline: allow with warnings and audit, not hard-block.

---

## 7. Domain Events

- STOCK_ITEM_CREATED
- STOCK_ITEM_UPDATED
- RESTOCK_BATCH_RECORDED
- INVENTORY_MOVEMENT_APPENDED
- INVENTORY_ADJUSTMENT_RECORDED
- RESTOCK_BATCH_METADATA_UPDATED

(Events are for audit/projection updates; cross-module orchestration uses process docs.)

---

## 8. Read Model (Queries / Projections)

Read views are derived from the journal:
- Current on-hand per stock item per branch
- Movement history (audit trail)
- Low stock list (threshold-based)
- Expiring soon / expired list (based on batch metadata)

Notes:
- Expiry is **reporting-only** in baseline. No FEFO allocation enforcement.
- If expiry_date is null, the batch is treated as **non-expiring** for reporting.

---

## 9. Offline, Concurrency, Idempotency

- Commands that may be retried (offline replay) must include idempotency keys.
- Journal append operations are safe under retries when uniqueness is enforced.
- Projection updates must be replayable from the journal.

---

## 10. External Inputs & References (Not Orchestration)

Inventory accepts references supplied by application/process orchestrators:
- order_id / sale_id for external deductions and reversals
- actor_id and role context for authorization (if enforced at app layer)
- branch_id context

Inventory does not compute recipes or decide deduction timing.

---

## 11. Failure Modes & Recovery (Domain-local)

- Invariant violation → command rejected (no state change)
- Duplicate externally-triggered movement → no-op (idempotent)
- Projection mismatch → recompute projection from journal

Cross-module partial failures and compensations belong to Process docs.

---

## 12. Out of Scope (Capstone baseline)

- FIFO/FEFO stock allocation enforcement
- Multi-location transfers and warehouse workflows
- Supplier purchase orders and receiving pipeline
- Lot-level traceability beyond optional expiry metadata
- Automated reorder procurement

---

## 13. Future Evolution

Potential future extensions:
- Strict negative stock prevention policies
- FEFO/FIFO allocation and batch consumption rules
- Transfers between branches
- Supplier + purchasing domain integration
- Lot/serial tracking for regulated items (pharmacy)
