# Inventory Module — Feature Module Spec (Capstone 1)

**Version:** 1.6  
**Status:** Patched (Aligns with Sale integrity guarantees; removes optional cash-session wording)  
**Module Type:** Feature Module  
**Depends on:** Auth & Authorization, Tenant & Branch Context, Policy & Configuration, Menu (recipe mapping), Sync & Offline, Audit Logging  

---

## 1. Purpose

The Inventory Module enables tenants to manage **stock items**, **stock categories**, **restock batches**, and **branch-level stock quantities**, while maintaining a complete and immutable history of all stock changes.

The module is designed to:
- Support recipe-based stock deduction from sales
- Provide accurate per-branch and aggregated inventory visibility
- Preserve data integrity through append-only journals
- Allow flexibility during setup while enforcing operational limits

---

## 2. Scope & Boundaries

### Includes
- Full CRUD for **Stock Items**
- Full CRUD for **Stock Categories**
- Full CRUD-like lifecycle for **Restock Batches** (no hard delete)
- Append-only **Inventory Journal**
- Per-branch stock tracking in base units
- Aggregated stock view across branches (Admin)
- Expiry-date tracking per batch
- Enforcement of tenant soft/hard limits

### Excludes
- Supplier management
- Purchase orders
- Stock transfer between branches
- FIFO / FEFO enforcement
- Automated reorder alerts
- Inventory analytics (Capstone 2)

---

## 3. Core Concepts

### 3.1 Stock Item
A physical inventory object such as Milk, Coffee Beans, Cups, or Sugar.  
Each stock item has a **base unit of measurement** (ml, g, pc) used consistently for deduction and reporting.

### 3.2 Stock Category
An optional grouping for stock items (e.g. Dairy, Packaging).

### 3.3 Uncategorized (System-Derived)
- If a stock item has **no category assigned**, it is treated as **“Uncategorized”**
- “Uncategorized”:
  - Is system-derived (not persisted)
  - Cannot be edited, renamed, or deleted
  - Does not count toward category quota

### 3.4 Restock Batch
A record of stock received for a specific stock item at a specific branch, optionally with expiry and cost metadata.

### 3.5 Branch Stock
The current available quantity of a stock item at a branch, expressed in base units and **derived from journal entries**.

### 3.6 Inventory Journal
An immutable, append-only log of all stock changes:
- Restocks
- Sale deductions
- Manual adjustments

### 3.7 Branch Lifecycle (Important Boundary)
- **Branches are system-created** (not created by Admin/Manager directly), as part of tenant onboarding or subscription changes (Capstone 2 billing engine; Capstone 1 developer/admin tooling).  
- The Inventory Module only:
  - Lists and lets users **select from existing branches** in tenant context
  - Enforces **branch status constraints** (e.g., disallow operations on frozen branches)

### 3.8 Integrity Guarantees for Stock Mutations (New, Locked)
Inventory integrity must hold **regardless of whether cash sessions are required for sales**. Cash-session policy affects cash-accountability UX, not inventory correctness.

Locked rules (Capstone 1):
1. **Append-only journal is authoritative**  
   Stock levels are computed from journal entries; no mutation may “rewrite history”.
2. **Idempotent deductions and restocks**  
   Operations that can be retried (offline replay, network retries, double submit) must not create duplicate journal entries.
   - For sale deductions: the system must be able to detect “already deducted for this sale”.
3. **Concurrency-safe deductions**  
   When multiple devices/users finalize sales concurrently, deductions must not produce inconsistent totals (e.g., missing deductions or duplicated deductions).  
   (Note: preventing negative stock may be a business rule; integrity here means consistent, auditable mutation.)
4. **Server-authoritative processing order**  
   Client timestamps may be recorded for audit but must not be used as the primary ordering mechanism for correctness. The backend processes mutations in a consistent server-controlled order.

---

## 4. Policy Dependencies (Branch-Scoped)

> **Important:** Inventory behavior is controlled by **branch-scoped** policy values (Policy & Configuration module).  
> When performing inventory actions, the system must resolve the **effective policy for the selected branch context**.

### Policies Used
- `inventoryAutoSubtractOnSale` (boolean)
  - If `true`, finalized sales trigger recipe-based stock deduction for that branch.
- `inventoryExpiryTrackingEnabled` (boolean)
  - If `true`, restock batches require/enable expiry date tracking and expiry-based views.

---

## 5. Use Cases

### UC-1 — Create Stock Item

**Actor:** Admin  

#### Preconditions
- Tenant exists and is active
- User has Admin role
- Stock item soft limit not exceeded

#### Main Flow
1. Admin opens “Create Stock Item”
2. Admin enters:
   - Name
   - Base unit of measurement
3. Admin optionally:
   - Assigns category
   - Uploads image
4. System validates quota and input
5. Stock item is created

#### Postconditions
- Stock item exists in ACTIVE state
- If no category assigned → item is **Uncategorized**
- Item available for restocking

---

### UC-2 — View Stock Items

**Actor:** Admin, Manager  

#### Preconditions
- User authenticated

#### Main Flow
1. User opens Inventory list
2. System displays:
   - Categorized items
   - “Uncategorized” section (if applicable)
3. User may filter by:
   - Category
   - Branch
   - Active / Archived

#### Postconditions
- No state change

---

### UC-3 — Update Stock Item

**Actor:** Admin  

#### Preconditions
- Stock item exists

#### Main Flow
1. Admin edits allowed fields:
   - Name
   - Image
   - Category
   - Active state
2. System validates:
   - Base unit cannot be changed if item has journal history or recipe mapping
3. Changes are saved

#### Postconditions
- Updated information reflected across inventory and recipe views

---

### UC-4 — Archive Stock Item (Soft Delete)

**Actor:** Admin  

#### Preconditions
- Stock item exists

#### Main Flow
1. Admin archives stock item
2. System sets `is_active = false`

#### Postconditions
- Item hidden from restocking and recipe mapping
- Historical journal preserved
- Soft limit freed
- Item appears in Archived list

---

### UC-5 — Restore Stock Item

**Actor:** Admin  

#### Preconditions
- Item is archived
- Stock item soft limit not exceeded

#### Main Flow
1. Admin restores stock item
2. System reactivates item

#### Postconditions
- Item usable again for restocking and recipes
- Category restored or item placed under **Uncategorized** if category no longer exists

---

### UC-6 — Manage Stock Categories (CRUD)

**Actor:** Admin  

#### Preconditions
- Category soft limit not exceeded (on create)

#### Main Flow
- Create, rename, archive categories

#### Postconditions
- Archived categories:
  - Do not delete stock items
  - Cause affected items to move to **Uncategorized**

---

### UC-7 — Create Restock Batch

**Actor:** Admin, Manager  

#### Preconditions
- Stock item exists and is active
- **Branch exists and is ACTIVE (not frozen)**
- If `inventoryExpiryTrackingEnabled = true` for the selected branch:
  - expiry date input is enabled (and may be required by UX rules)

#### Main Flow
1. User selects stock item and branch
2. User enters:
   - Quantity (base unit)
   - Optional expiry date (or required when expiry tracking is enabled, per UX decision)
   - Optional supplier, cost, notes
3. System:
   - Creates restock batch
   - Appends journal entry (RESTOCK)
   - Updates branch stock quantity

#### Postconditions
- Branch stock increased
- Batch visible in batch list
- Journal updated

---

### UC-8 — View Restock Batches

**Actor:** Admin, Manager  

#### Preconditions
- Stock item exists

#### Main Flow
- View batches sorted by expiry or date

#### Postconditions
- No state change

---

### UC-9 — Update Restock Batch Metadata

**Actor:** Admin  

#### Preconditions
- Restock batch exists
- **Branch for the batch is ACTIVE (not frozen)**

#### Main Flow
- Update non-quantity fields:
  - Expiry date (enabled only if `inventoryExpiryTrackingEnabled = true` for the batch branch)
  - Notes
  - Supplier
  - Cost

#### Postconditions
- Batch metadata updated
- No change to stock quantity

---

### UC-10 — Archive Restock Batch

**Actor:** Admin  

#### Preconditions
- Batch exists

#### Main Flow
- Archive restock batch

#### Postconditions
- Batch hidden from active views
- Journal and stock quantities unchanged

---

### UC-11 — Manual Inventory Adjustment

**Actor:** Admin  

#### Preconditions
- Stock item exists
- **Branch exists and is ACTIVE (not frozen)**

#### Main Flow
1. Admin enters adjustment quantity (+/-)
2. Admin provides reason
3. System appends journal entry (ADJUSTMENT)
4. Branch stock recalculated

#### Postconditions
- Stock updated
- Adjustment permanently recorded

---

### UC-12 — Sale-Based Stock Deduction (Integrity-Locked)

**Actor:** System (triggered by Sale module upon finalization)

#### Preconditions
- Sale finalized
- Branch context is known for the sale
- **Branch is ACTIVE (not frozen)**
- Effective policy for the sale branch:
  - `inventoryAutoSubtractOnSale = true`
- Menu item has recipe mapping

#### Main Flow
1. Sale module finalizes sale (server-side).
2. Inventory deduction is triggered for the sale’s branch as part of the finalize operation.
3. System:
   - Calculates required quantities (base units) from recipe mapping × quantity sold
   - Appends journal entries (SALE_DEDUCTION), referencing sale ID
4. Branch stock is updated from journal state.

#### Integrity Rules (Locked)
- **Atomicity:** sale finalization + journal append(s) + derived stock update must behave atomically from the system perspective (no partial completion visible).
- **Idempotency:** if the same sale finalize is retried (offline replay / network retry), the system must not append duplicate SALE_DEDUCTION entries.  
  Minimum requirement: detect “deduction already applied for sale_id”.
- **Concurrency safety:** multiple finalizations occurring concurrently must not cause missing or duplicated deductions.

#### Postconditions
- Stock decreased correctly for the sale branch
- Journal references sale ID
- Deduction is auditable and replayable

---

### UC-13 — View Inventory Journal

**Actor:** Admin, Manager  

#### Preconditions
- Inventory data exists

#### Main Flow
- View journal entries filtered by:
  - Item
  - Branch
  - Date
  - Type

#### Postconditions
- No state change

---

### UC-14 — View Branch Stock

**Actor:** Admin, Manager  

#### Preconditions
- Stock items exist

#### Main Flow
- View available quantity per branch (base units)

#### Postconditions
- No state change

---

### UC-15 — View Aggregated Stock Across Branches

**Actor:** Admin  

#### Preconditions
- Tenant has at least 1 branch

#### Main Flow
1. Admin opens “All Branches Inventory” (aggregate view)
2. System computes, per stock item:
   - sum of branch stocks across all branches
3. Admin may drill down into a specific branch view

#### Postconditions
- No state change

---

## 6. Functional Requirements

- FR-1: Stock items support full CRUD with soft delete
- FR-2: Base unit immutable once referenced
- FR-3: Restock quantity cannot be edited
- FR-4: Inventory journals are append-only
- FR-5: Category assignment optional
- FR-6: Uncategorized grouping supported
- FR-7: Soft/hard limits enforced
- FR-8: Expiry date stored at batch level (when enabled by policy)
- FR-9: Sale deductions are branch-context and policy-controlled
- FR-10: Inventory mutations (restock/adjust/deduction) are blocked when the target branch is frozen
- FR-11 (New): Sale-based deductions must be idempotent per sale_id (no duplicate journal entries on retries)
- FR-12 (New): Concurrent finalize operations must not produce inconsistent journal state (missing/duplicated deductions)

---

## 7. Acceptance Criteria

- AC-1: Items can exist without category
- AC-2: Uncategorized items displayed correctly
- AC-3: Restocking updates branch stock
- AC-4: Sale deductions reflect accurately when `inventoryAutoSubtractOnSale = true`
- AC-5: Journal history is complete and immutable
- AC-6: Branch stock totals match the sum of restocks minus deductions in journals
- AC-7: Expiry tracking UI/behavior is enabled only when `inventoryExpiryTrackingEnabled = true` for that branch
- AC-8: Attempting to restock/adjust/deduct stock against a frozen branch is rejected with a clear error
- AC-9 (New): Retrying the same finalize sale does not create duplicate SALE_DEDUCTION entries
- AC-10 (New): Two users can finalize sales concurrently without producing missing or duplicated deductions in the journal

---

## 8. Deletion & Recovery Rules

| Entity | Behavior |
|------|---------|
| Stock Item | Archive/restore only |
| Category | Archive → items uncategorized |
| Restock Batch | Archive only |
| Journal | No update, no delete |

---

## 9. Out of Scope (Capstone 1)

- Stock transfers
- FIFO/FEFO enforcement
- Supplier & purchasing
- Automated reorder logic
- Cost analytics
- Inventory analytics (Capstone 2)

---

## Audit Events Emitted

The following events MUST be written to the Audit Log when triggered (with tenant_id, branch_id where applicable, actor_id, and relevant entity IDs):

- `STOCK_ITEM_CREATED`
- `STOCK_ITEM_UPDATED`
- `STOCK_ITEM_ARCHIVED`
- `STOCK_ITEM_RESTORED`
- `STOCK_CATEGORY_CREATED`
- `STOCK_CATEGORY_UPDATED`
- `RESTOCK_BATCH_CREATED`
- `INVENTORY_JOURNAL_APPENDED`
- `INVENTORY_DEDUCTION_APPLIED` (New; emitted when UC-12 successfully appends deductions for a sale)

---

# End of Document
