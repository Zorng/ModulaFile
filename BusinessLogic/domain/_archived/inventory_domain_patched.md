# Inventory Domain Model — Modula POS

## Domain Name
Inventory

## Domain Type
Core Business Domain (POS Operations)

## Status
Defined (Capstone baseline)

---

## Purpose

The Inventory domain models **stock truth** for a branch using an **append-only ledger** of stock movements.

Unlike simple CRUD inventory systems, Inventory in Modula is designed to:
- Remain **correct over time**
- Remain **auditable**
- Remain **performant** even as data grows over years and across many tenants

This domain intentionally separates:
- **Truth** (what actually happened)
- **Speed** (what the UI reads)

---

## 1. Domain Boundary & Ownership

### Boundary
Inventory begins when a stock item is defined and continues through all stock-changing events (restock, sale deduction, adjustment, reversal).

### Owns
- Stock item definitions
- Restock batches (inputs to stock)
- Inventory journal (append-only ledger)
- Rules for stock movement correctness
- Stock projections for fast reads

### Explicitly NOT Responsible For
- Deciding *when* stock should be deducted for a sale
- Menu recipes or product configuration
- Cash accountability or payments
- Procurement and supplier workflows
- Warehouse-grade batch allocation (FIFO/FEFO)

---

## 2. Core Concepts (Ubiquitous Language)

| Term | Meaning |
|---|---|
| StockItem | A countable item (milk, beans, bottled water) |
| RestockBatch | A delivery input with quantity and optional expiry |
| InventoryJournal | Append-only ledger of all stock movements |
| InventoryMovement | One stock change event recorded in the journal |
| Adjustment | A corrective movement (waste, count correction) |
| BranchStock | A fast-read projection of current on-hand stock |

---

## 3. Educational Note: Ledger vs CRUD Inventory

### What CRUD Inventory Looks Like (Not Used)

- Single row per item with `on_hand_quantity`
- Each update overwrites the previous value
- History is lost or unreliable

Problems:
- No audit trail
- Hard to explain discrepancies
- Unsafe with retries and offline sync

### Ledger-Based Inventory (Used by Modula)

- Every stock change is appended as a movement
- No historical data is overwritten
- Current on-hand stock is derived

Example:
- IN 10L (restock)
- OUT 3L (sale)
- OUT 2L (waste)

On-hand = 5L

This is the same model used by accounting systems and mature POS platforms.

---

## 4. Restock Batches vs Stock Truth

A **RestockBatch** records where stock came from.
It does not define how much stock remains.

Stock truth is derived from **InventoryMovements**, not batch quantities.

---

## 5. Stock Adjustments (Correction, Not Mutation)

Adjustments correct reality without rewriting history.

Rules:
- Adjustments append new movements
- Restock batches are never edited to fix quantities
- Reasons are mandatory (WASTE, COUNT_CORRECTION, DAMAGE)

---

## 6. Ledger + Projection Strategy (Performance & Scale)

### Why Projections Exist

Over time, the inventory journal grows large.
Summing the journal for every UI request is too slow.

### Solution

- InventoryJournal = source of truth
- BranchStock = fast-read projection

Rules:
- UI reads on-hand stock from BranchStock
- Journal is used for audit and rebuild only
- Projections may be rebuilt at any time

---

## 7. Projection Rebuild & Background Jobs

Because projections are derived, they can drift due to bugs or failures.

The system must support:
- Rebuilding projections from the journal
- Integrity checks between journal and projection
- Periodic rollups for reporting (optional)

---

## 8. Commands (Write Intents)

- CreateStockItem
- UpdateStockItem
- RecordRestockBatch
- RecordAdjustment
- UpdateRestockBatchMetadata
- ApplyExternalDeduction
- ApplyExternalReversal

---

## 9. Invariants

- Append-only journal
- Idempotent external movements
- Projections are derived, not authoritative

---

## 10. Failure Modes

- Duplicate movement → no-op
- Projection drift → rebuild

---

## 11. Out of Scope

- FIFO/FEFO enforcement
- Batch-level remaining quantities
- Warehouse allocation

---

## 12. Future Evolution

- Batch allocation and FEFO
- Transfers and procurement
- Regulatory-grade traceability
