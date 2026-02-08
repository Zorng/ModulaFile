# Story Coverage Map — Inventory (Managing Stock Truth)

**Purpose:**  
Identify human-centered stories for Inventory behavior and map them to technical use cases, ensuring full coverage without “one story per UC”.

**Scope:**  
- Inventory Domain (ledger-first, projections)
- Inventory Module Spec (Capstone 1, patched)

**Rule:**  
Stories are grouped by **human operational context**, not by CRUD actions or internal mechanics.

---

## 1) Source Artifacts (Authoritative)

- **Domain:** `BusinessLogic/2_domain/40_POSOperation/inventory_domain.md`
- **ModSpec:** `BusinessLogic/5_modSpec/40_POSOperation/inventory_module_patched.md`

---

## 2) Operational Contexts (Human Reality)

From reading the domain and modspec, Inventory is experienced by humans in **five distinct operational contexts**:

1. **Defining What Is Tracked**
2. **Receiving Stock**
3. **Correcting Stock Reality**
4. **Seeing What Is Available**
5. **Handling Stock Changes Caused by Sales**

These contexts reflect how cafés and small retailers actually work.

---

## 3) Story Set (Human Situations)

### Context: Defining What Is Tracked

1. **Creating and Managing Stock Items**  
   File: `BusinessLogic/1_stories/managing_stock/creating&managing_stock_item.md`  
   Human situation: deciding what materials exist, their units, and whether they are active.

2. **Organizing Stock Items into Categories**  
   File: `BusinessLogic/1_stories/managing_stock/creating_managing_stock_category.md`  
   Human situation: grouping stock items to make inventory easier to understand.

---

### Context: Receiving Stock

3. **Receiving New Stock into the Store**  
   File: `BusinessLogic/1_stories/managing_stock/receiving_stock.md`  
   Human situation: recording delivered items and acknowledging that stock has arrived.

4. **Recording Expiry Information When Relevant**  
   File: `BusinessLogic/1_stories/managing_stock/recording_stock_expiry.md`  
   Human situation: adding expiry information when it matters, without enforcing it everywhere.

---

### Context: Correcting Stock Reality

5. **Adjusting Stock When Reality Differs**  
   File: `BusinessLogic/1_stories/managing_stock/adjusting_stock.md`  
   Human situation: correcting mistakes, waste, damage, or count differences.

6. **Force-Correcting Stock After Serious Issues**  
   File: `BusinessLogic/1_stories/managing_stock/force_correcting_stock.md`  
   Human situation: dealing with abnormal situations while preserving audit history.

---

### Context: Seeing What Is Available

7. **Checking Current Stock Levels**  
   File: `BusinessLogic/1_stories/managing_stock/checking_current_stock_levels.md`  
   Human situation: seeing what is available right now to support operations.

8. **Reviewing Stock History and Movements**  
   File: `BusinessLogic/1_stories/managing_stock/reviewing_stock_history.md`  
   Human situation: understanding how stock changed over time.

---

### Context: Handling Stock Changes Caused by Sales

9. **Having Stock Automatically Updated by Sales**  
   File: `BusinessLogic/1_stories/managing_stock/stock_changes_from_sales.md`  
   Human situation: trusting the system to reflect sales without manual intervention.

10. **Restoring Stock When a Sale Is Voided**  
    File: `BusinessLogic/1_stories/managing_stock/restoring_stock_after_void.md`  
    Human situation: seeing stock return when a sale is undone.

---

## 4) Coverage Mapping (Use Case → Story)

### Story 1 — Creating and Managing Stock Items
Covers:
- UC-1 Create Stock Item
- UC-3 Update Stock Item
- UC-4 Archive Stock Item
- UC-5 Restore Stock Item

---

### Story 2 — Organizing Stock Items into Categories
Covers:
- UC-6 Manage Stock Categories
- Uncategorized behavior (system-derived)

---

### Story 3 — Receiving New Stock into the Store
Covers:
- UC-7 Create Restock Batch
- UC-8 View Restock Batches

---

### Story 4 — Recording Expiry Information When Relevant
Covers:
- UC-9 Update Restock Batch Metadata (expiry)
- Expiry reporting behavior

---

### Story 5 — Adjusting Stock When Reality Differs
Covers:
- UC-11 Manual Inventory Adjustment (set-to-count, delta)
- Reason codes (waste, damage, correction)

---

### Story 6 — Force-Correcting Stock After Serious Issues
Covers:
- UC-11 (exceptional use)
- Branch freeze / abnormal correction scenarios

---

### Story 7 — Checking Current Stock Levels
Covers:
- UC-13 View Branch Stock
- UC-14 View Aggregated Stock

---

### Story 8 — Reviewing Stock History and Movements
Covers:
- UC-12 View Inventory Journal

---

### Story 9 — Having Stock Automatically Updated by Sales
Covers:
- CM-1 Apply Sale-Based Stock Deduction
- Projection update expectations

---

### Story 10 — Restoring Stock When a Sale Is Voided
Covers:
- CM-2 Apply Void Reversal

---

## 5) Coverage Summary (Quick Check)

| Inventory Behavior | Covered By Story |
|---|---|
| Define stock items | Story 1 |
| Categorize stock | Story 2 |
| Restock | Story 3 |
| Expiry metadata | Story 4 |
| Manual adjustment | Story 5 |
| Exceptional correction | Story 6 |
| On-hand visibility | Story 7 |
| Audit trail | Story 8 |
| Sale deduction | Story 9 |
| Void reversal | Story 10 |

All domain and modspec behaviors are accounted for.

---

## 6) Status Tracker (Fill As You Write)

| Story | File | Status |
|---|---|---|
| Creating and Managing Stock Items | `BusinessLogic/1_stories/managing_stock/creating&managing_stock_item.md` | ⬜ |
| Organizing Stock Items into Categories | `BusinessLogic/1_stories/managing_stock/creating_managing_stock_category.md` | ⬜ |
| Receiving New Stock into the Store | `BusinessLogic/1_stories/managing_stock/receiving_stock.md` | ⬜ |
| Recording Expiry Information When Relevant | `BusinessLogic/1_stories/managing_stock/recording_stock_expiry.md` | ⬜ |
| Adjusting Stock When Reality Differs | `BusinessLogic/1_stories/managing_stock/adjusting_stock.md` | ⬜ |
| Force-Correcting Stock After Serious Issues | `BusinessLogic/1_stories/managing_stock/force_correcting_stock.md` | ⬜ |
| Checking Current Stock Levels | `BusinessLogic/1_stories/managing_stock/checking_current_stock_levels.md` | ⬜ |
| Reviewing Stock History and Movements | `BusinessLogic/1_stories/managing_stock/reviewing_stock_history.md` | ⬜ |
| Having Stock Automatically Updated by Sales | `BusinessLogic/1_stories/managing_stock/stock_changes_from_sales.md` | ⬜ |
| Restoring Stock When a Sale Is Voided | `BusinessLogic/1_stories/managing_stock/restoring_stock_after_void.md` | ⬜ |

Legend:
- ⬜ Not written
- 🟨 Drafted
- ✅ Final

---

## 7) Guardrails (Important)

- Do **not** write one story per UC.
- Keep stories human-centered; no ledger, projection, or idempotency language.
- Treat “sale-driven stock change” as **trust in the system**, not a manual action.
- Expiry is described as **optional and contextual**, not mandatory.

---

# End of Map
