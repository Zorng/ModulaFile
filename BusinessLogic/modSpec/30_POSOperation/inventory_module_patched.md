# Inventory Module — Feature Module Spec (Capstone 1)

**Version:** 1.7 (Patched)  
**Status:** Patched to align with **Inventory Domain (ledger + projections)** and the new **Self-Contained vs Cross-Module** structure.  
**Module Type:** Feature Module  
**Primary Domain:** POS Operations → Inventory  
**Depends on:** Auth & Authorization, Tenant & Branch Context, Policy & Configuration (operational limits only), Sync & Offline, Audit Logging  
**Collaborates with (Cross-Module):** Sale/Order (finalize), Menu (recipe/direct-stock mapping), Cash Session (cash accountability), Job Scheduler (rollups/rebuild jobs)

---

## 1. Purpose

The Inventory module manages **stock items**, **stock categories**, **restock batches**, and **inventory movements** to provide a correct, auditable, and long-lived representation of stock.

Inventory is **not CRUD stock**. It is **ledger-first**:
- **Inventory Journal** (append-only movements) is the source of truth
- **Branch Stock** is a **projection** used for fast reads

This design keeps Inventory correct and usable even after years of operation and large data volumes.

---

## 2. Scope & Boundaries

### Includes (Self-Contained)
- CRUD for Stock Items (soft delete/restore only)
- CRUD for Stock Categories (archive only)
- Restock Batches (create, view, metadata update, archive)
- Append-only Inventory Journal (movements)
- Manual adjustments (waste, count correction, damage)
- Branch stock and aggregated stock views (projection-based)
- Expiry metadata per batch (optional)
- Enforcement of tenant soft/hard limits
- Branch status constraints (reject mutations on frozen branches)

### Excludes (Capstone 1)
- Supplier management, purchase orders
- Stock transfers between branches
- FIFO/FEFO allocation enforcement
- Automated reorder alerts
- Advanced analytics (Capstone 2)

---

## 3. Core Concepts

### 3.1 Stock Item
A countable item (Milk, Coffee Beans, Bottled Water). Each has a **base unit** (ml, g, pc). All journal quantities are stored in base units.

### 3.2 Stock Category
Optional grouping for stock items (Dairy, Packaging).

### 3.3 Uncategorized (System-Derived)
- Items with no category are displayed under **Uncategorized**
- Uncategorized is system-derived (not persisted), not editable, not counted toward category quota

### 3.4 Restock Batch
A record of stock received for a stock item at a branch, with optional metadata (expiry, cost, note). Restock quantity is immutable after creation.

### 3.5 Inventory Journal (Source of Truth)
An immutable, append-only log of all stock movements:
- RESTOCK (IN)
- ADJUSTMENT (IN/OUT)
- SALE_DEDUCTION (OUT) — cross-module
- VOID_REVERSAL (IN) — cross-module

### 3.6 Branch Stock (Projection)
The current on-hand quantity per (branch, stock item). It is a **derived/materialized projection** from the journal for fast reads.

### 3.7 Expiry Tracking (Baseline Behavior)
Expiry is **optional batch metadata**:
- If a batch has `expiry_date`, it participates in expiry views (expiring soon/expired)
- If missing, the batch is treated as **non-expiring** for reporting
- FIFO/FEFO enforcement remains out of scope

---

## 4. Integrity Guarantees (Locked)

These guarantees must hold regardless of other modules’ policies (e.g., cash sessions).

1) **Append-only journal is authoritative**  
No stock change rewrites history. Corrections happen via compensating movements.
2) **Idempotent mutations**  
Restock, external deductions, and external reversals must be safe under retry (offline replay, double submit).
3) **Concurrency-safe projections**  
Concurrent movements must not cause lost updates or duplicated totals in Branch Stock.
4) **Server-authoritative ordering**  
Client timestamps may be recorded for audit, but server ordering is authoritative for correctness.

---

## 5. Policy Dependencies (Branch-Scoped)

Inventory no longer uses global “feature toggles” to define core behavior (e.g., expiry exists as metadata, not a mode). Policies are reserved for **operational constraints**.

### Policies Used (Baseline)
- `inventoryAllowNegativeStock` (boolean, default true)
  - If false, block mutations that would cause on-hand < 0 (optional; may be deferred)
- `inventoryLowStockWarningEnabled` (boolean, optional)
- Tenant limits (soft/hard quotas) for items and categories

> Note: Sale-based deduction is not a policy toggle in Inventory. Whether an item deducts stock is defined by Menu configuration (recipe/direct-stock mapping).

---

# 6. Use Cases — Self-Contained Processes

## UC-1 — Create Stock Item
**Actor:** Admin  
**Preconditions:** Tenant active, Admin role, quota not exceeded  
**Flow:** Create item with name + base unit; optional category/image  
**Postconditions:** Stock item ACTIVE; available for restocking/mapping

## UC-2 — View Stock Items
**Actor:** Admin, Manager  
**Flow:** List items; filter by category/branch/active state  
**Postconditions:** None

## UC-3 — Update Stock Item
**Actor:** Admin  
**Rules:** Base unit immutable once referenced by any journal history or mapping  
**Flow:** Update allowed fields (name, image, category, active flag)  
**Postconditions:** Updated data visible everywhere

## UC-4 — Archive Stock Item (Soft Delete)
**Actor:** Admin  
**Flow:** Set inactive  
**Postconditions:** Hidden from restocking/mapping; history preserved

## UC-5 — Restore Stock Item
**Actor:** Admin  
**Flow:** Reactivate (quota permitting)  
**Postconditions:** Item usable again; category restored or Uncategorized

## UC-6 — Manage Stock Categories (CRUD + Archive)
**Actor:** Admin  
**Flow:** Create/rename/archive categories  
**Rules:** Archiving a category moves items to Uncategorized; items not deleted

## UC-7 — Create Restock Batch (Restock)
**Actor:** Admin, Manager  
**Preconditions:** Item ACTIVE; Branch ACTIVE (not frozen)  
**Flow:**
- Enter quantity (base unit) and optional expiry date, note, cost, supplier
- System creates RestockBatch and appends RESTOCK journal movement (IN)
- System updates Branch Stock projection
**Postconditions:** On-hand increased; journal auditable; batch visible

## UC-8 — View Restock Batches
**Actor:** Admin, Manager  
**Flow:** List batches by item/branch; sort by received date/expiry (if present)  
**Postconditions:** None

## UC-9 — Update Restock Batch Metadata
**Actor:** Admin  
**Preconditions:** Branch ACTIVE (not frozen)  
**Flow:** Update metadata only (expiry_date, note, supplier, cost)  
**Rules:** No stock quantity change; journal unchanged  
**Postconditions:** Updated metadata visible

## UC-10 — Archive Restock Batch
**Actor:** Admin  
**Flow:** Archive batch from active views  
**Rules:** No stock change; journal unchanged  
**Postconditions:** Batch hidden; audit preserved

## UC-11 — Manual Inventory Adjustment (Ledger-Based)
**Actor:** Admin  
**Preconditions:** Item exists; Branch ACTIVE (not frozen)  
**Supported Adjustment Styles:**
- **Set-to-count (recommended):** Admin enters counted_on_hand; system computes delta vs current Branch Stock and appends a single ADJUSTMENT movement
- **Delta adjustment:** Admin enters +/- quantity directly
**Required Fields:** reason_code (COUNT_CORRECTION, WASTE, DAMAGE, OTHER) and optional note  
**Flow:**
- Validate quantities in base unit
- Append ADJUSTMENT journal movement (IN or OUT)
- Atomically update Branch Stock projection
**Postconditions:** On-hand updated; adjustment permanently recorded; auditable

## UC-12 — View Inventory Journal (Audit)
**Actor:** Admin, Manager  
**Flow:** Paginated view filtered by item/branch/date/type/source  
**Postconditions:** None

## UC-13 — View Branch Stock (Fast Read)
**Actor:** Admin, Manager  
**Flow:** Read from Branch Stock projection for selected branch  
**Postconditions:** None

## UC-14 — View Aggregated Stock Across Branches
**Actor:** Admin  
**Flow:** Aggregate from Branch Stock projections (sum per item across branches)  
**Postconditions:** None

---

# 7. Cross-Module Participation (Contracts)

Inventory participates in cross-module flows but does not own the orchestration. Orchestration is defined in Process docs (e.g., Finalize Order, Void Order).

## CM-1 — Apply Sale-Based Stock Deduction (Triggered by Finalize Order)
**Trigger:** Sale/Order finalization process  
**Inputs:** (branch_id, sale_id/order_id, actor_id, deduction_lines[{stock_item_id, qty_base_unit}])  
**Rules (Locked):**
- **Idempotent per sale_id/order_id:** retry must not create duplicate SALE_DEDUCTION movements
- **Atomic from system perspective:** finalize + deduction + projection update behaves as one operation (transaction or outbox-consistent)
- **Branch frozen check:** reject if branch frozen
**Outputs:** SALE_DEDUCTION journal movements + updated Branch Stock projection

> Note: Whether a sold item produces deduction_lines comes from Menu configuration (recipe mapping or direct-stock mapping).

## CM-2 — Apply Void Reversal (Triggered by Void Order)
**Trigger:** Void Order process  
**Inputs:** (branch_id, sale_id/order_id, actor_id)  
**Rules:**
- Reverse only what was deducted for that sale_id/order_id (compensating IN movements)
- Idempotent under retry
**Outputs:** VOID_REVERSAL movements + updated projection

---

## 8. Operational & Performance Requirements (Long-Run Scale)

### Read Path
- Current on-hand must be served from **Branch Stock projection**, not by summing the entire journal.
- Journal queries are for audit/history and must be paginated and time-filtered.

### Write Path
- Every stock mutation appends journal movement(s) and updates projection safely under concurrency.
- External mutations (deduction/reversal) require unique idempotency keys.

### Projection Rebuild & Maintenance
The system must support:
- Rebuilding Branch Stock from the journal (admin tool)
- Integrity checks (detect projection drift)
- Optional rollups for reporting (daily/monthly), executed via Job Scheduler

### Recommended Indexing (Implementation Guidance)
- InventoryJournal: (tenant_id, branch_id, stock_item_id, occurred_at)
- InventoryJournal: unique constraint for idempotency on (branch_id, movement_type/source_type, source_id) or (idempotency_key)
- BranchStock: (tenant_id, branch_id, stock_item_id) unique

---

## 9. Functional Requirements

- FR-1: Stock items support CRUD with archive/restore only
- FR-2: Base unit immutable once referenced by journal or mapping
- FR-3: Restock quantity cannot be edited; restock creates journal movement
- FR-4: Inventory journal is append-only; no update/delete
- FR-5: Uncategorized grouping supported as system-derived
- FR-6: Soft/hard limits enforced for items/categories
- FR-7: Expiry date is optional batch metadata (not a mode)
- FR-8: Inventory mutations blocked when branch is frozen
- FR-9: Manual adjustments are ledger movements with reason codes; support set-to-count and delta
- FR-10: External deductions/reversals are idempotent per sale_id/order_id
- FR-11: Branch Stock projection is the required read model for on-hand

---

## 10. Acceptance Criteria

- AC-1: Items can exist without category; shown under Uncategorized
- AC-2: Restock creates a batch + journal entry and updates Branch Stock
- AC-3: Journal history is complete and immutable
- AC-4: Manual adjustment requires reason and produces auditable movement
- AC-5: Branch stock totals match journal-derived totals (projection integrity)
- AC-6: Attempts to mutate stock on frozen branch are rejected with clear error
- AC-7: Expiry views show only batches with expiry dates; missing expiry treated as non-expiring
- AC-8: Retrying the same sale finalization does not duplicate deductions
- AC-9: Two concurrent deductions do not produce lost updates in Branch Stock

---

## 11. Deletion & Recovery Rules

| Entity | Behavior |
|---|---|
| Stock Item | Archive/restore only |
| Category | Archive → items move to Uncategorized |
| Restock Batch | Archive only |
| Journal | No update, no delete |
| Branch Stock | Rebuildable projection (may be recomputed from journal) |

---

## 12. Audit Events Emitted

Must be written to Audit Log (tenant_id, branch_id where applicable, actor_id, entity IDs):

- STOCK_ITEM_CREATED
- STOCK_ITEM_UPDATED
- STOCK_ITEM_ARCHIVED
- STOCK_ITEM_RESTORED
- STOCK_CATEGORY_CREATED
- STOCK_CATEGORY_UPDATED
- RESTOCK_BATCH_CREATED
- RESTOCK_BATCH_METADATA_UPDATED
- RESTOCK_BATCH_ARCHIVED
- INVENTORY_JOURNAL_APPENDED
- INVENTORY_ADJUSTMENT_RECORDED
- INVENTORY_DEDUCTION_APPLIED
- INVENTORY_VOID_REVERSAL_APPLIED

---

## 13. Out of Scope (Capstone 1)

- Supplier & purchasing
- Stock transfers between branches
- FIFO/FEFO enforcement and batch allocation
- Automated reorder logic
- Cost and inventory analytics (Capstone 2)

---

# End of Document
