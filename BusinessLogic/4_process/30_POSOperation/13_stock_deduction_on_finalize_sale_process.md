# Stock Deduction on Sale Finalization — Cross-Module Process

## Process Name
Finalize Sale → Inventory Deduction

## Process Type
Cross-Module Business Process

## Status
Defined (Baseline for Capstone 1)

---

## 1. Purpose

This process defines **how and when inventory stock is deducted** as a result of a sale.

It exists to:
- keep **Menu**, **Inventory**, and **Sale/Order** responsibilities cleanly separated
- ensure **correct, idempotent, auditable** inventory deductions
- avoid hidden coupling such as “has recipe → deduct stock”

This process is the **only place** where sale-based inventory deduction is orchestrated.

---

## 2. Trigger

**Trigger:** Sale is finalized  
(Execution of the *Finalize Sale* business flow)

The process may be implemented:
- synchronously within the finalize transaction, or
- asynchronously via an event/outbox

The trigger mechanism does **not** change the business rules below.

---

## 3. Participating Modules & Responsibilities

| Module | Responsibility |
|------|----------------|
| Sale / Order | Confirms sale finalization and provides order lines |
| Menu | Evaluates composition for each order line (base + modifiers) |
| Inventory | Records stock movements in ledger and updates projections |
| Process Layer | Orchestrates calls and enforces sequencing |

---

## 4. Preconditions

- Sale exists and is eligible for finalization
- Sale has not already been finalized (idempotency)
- Branch is active (not frozen)
- Required Menu and Inventory data is accessible

---

## 5. Core Rule (Authoritative)

> **Inventory deduction occurs if and only if the evaluated composition of an order line contains at least one TRACKED component.**

- “Has recipe” is **not** a valid condition
- NOT_TRACKED components are ignored for deduction
- This rule applies uniformly to:
  - recipe-based items
  - direct-stock items (e.g., bottled water)
  - items with modifiers

---

## 6. Process Steps (Happy Path)

For each order line in the finalized sale:

1. **Evaluate Composition (Menu)**
   - Input: menu_item_id + selected_modifier_option_ids
   - Output: aggregated component list with quantities and tracking_mode

2. **Filter TRACKED Components**
   - Exclude NOT_TRACKED components (operational inventory)

3. **Generate Deduction Lines**
   - One deduction line per TRACKED component:
     - stock_item_id
     - quantity_in_base_unit
     - source = SALE_DEDUCTION
     - source_id = sale_id / order_id

4. **Apply Inventory Deduction (Inventory)**
   - Append SALE_DEDUCTION ledger movements
   - Enforce idempotency per sale_id / order_id
   - Update Branch Stock projection atomically

5. **Continue Finalize Sale Flow**
   - Cash movement, receipt issuance, etc. (out of scope here)

---

## 7. Idempotency & Retry Rules

This process must be safe under:
- network retries
- offline sync replay
- duplicate finalize requests

Rules:
- Deduction is idempotent per `(branch_id, sale_id)`
- Retrying the same finalized sale must not duplicate ledger entries
- Inventory enforces idempotency at the ledger level

---

## 8. Failure Modes & Handling

### A. Menu Composition Evaluation Fails
- Sale finalization fails
- No inventory mutation occurs

### B. Inventory Deduction Fails
- Finalize Sale fails or is rolled back
- No partial deductions allowed

### C. Duplicate Finalize Request
- Process detects prior completion
- No-op for inventory deduction

### D. Negative Stock Result
- Behavior governed by Inventory policy:
  - allow with warning, or
  - reject finalize (future configuration)

---

## 9. Postconditions

After successful completion:
- Sale is finalized
- Inventory ledger reflects SALE_DEDUCTION entries
- Branch Stock projection is updated
- Audit events are recorded

---

## 10. Audit & Observability

The following events must be recorded:
- SALE_FINALIZED
- INVENTORY_DEDUCTION_APPLIED

Audit records must include:
- tenant_id
- branch_id
- sale_id
- actor_id (who finalized the sale)

---

## 11. Out of Scope

- Void / reversal logic (handled by a separate process)
- Inventory adjustments unrelated to sales
- Discount and pricing calculations
- Reporting aggregation

---

## Related
- Entry point orchestration:
  - `10_finalize_sale_orch.md`
- Void/reversal counterpart:
  - `20_void_sale_orch.md`

---

## 12. Design Notes (Why This Exists)

This process exists to reflect **real operational behavior**:

- Some inventory is transactional (deducted per sale)
- Some inventory is operational (reconciled by counting)
- Menu describes composition
- Inventory records truth
- This process decides *when* deduction happens

Separating this logic avoids:
- hidden product rules
- accidental coupling
- future industry-specific breakage
