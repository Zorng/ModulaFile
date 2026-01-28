## Module: Discount Management (Feature Module)

**Version:** 1.2  
**Status:** Revised (Branch-scoped discounts clarified; stacking preserved; sale lock-in enforced)  
**Module Type:** Feature Module  
**Depends on:**  
- Tenant & Branch Context (Core)  
- Authentication & Authorization (Core)  
- Audit Logging (Core)  

**Related Modules:**  
- Sale & Order Management  
- Menu Management  
- Policy & Configuration  
- Reporting  

---

## 1. Purpose

The Discount Management module allows tenant administrators to define and manage discount rules that are applied automatically during sales.

Discounts are designed to:
- Reduce cashier cognitive load (no manual price override)
- Ensure pricing consistency
- Support controlled promotional strategies at the **branch level**

This module defines **discount eligibility and scope** only.  
It does **not** calculate totals, finalize sales, or mutate financial records.

---

## 2. Scope (Capstone I)

### Included
- Percentage-based discounts only
- Item-level discounts (applied to selected menu items)
- **Branch-scoped discounts (discounts apply within selected branches only)**
- Discount scheduling (start / end time)
- Discount stacking (item-level + branch-level)
- Read-only visibility for Manager and Cashier
- Full audit logging of discount lifecycle events

### Excluded
- Fixed-amount discounts
- Coupon / voucher codes
- Customer-specific discounts
- Sale-type-specific discounts (e.g., dine-in only)

---

## 3. Core Concepts

### 3.1 Discount Rule

A Discount Rule defines:
- Discount percentage (0–100)
- Discount scope:
  - Item-level (targets specific menu items)
  - Branch-level (targets one or more branches)
- Schedule (active window)
- Status (enabled / disabled)

**All discounts are scoped by branch context.**  
A discount only applies when the sale occurs in a branch explicitly assigned to the rule.

---

### 3.2 Branch Scope (Clarified)

Discounts do **not** apply globally across the tenant by default.

Rules must explicitly define:
- Which branch or branches they apply to

This ensures:
- Branch-specific promotions
- Different pricing strategies per location
- No accidental cross-branch discount leakage

If a discount is not assigned to the active branch, it is ignored.

---

### 3.3 Discount Stacking (Locked Behavior)

When multiple discounts are applicable, they stack **multiplicatively**.

If:
- Line amount = `L`
- Applicable discounts = `d1, d2, ..., dn`

Then: discounted_line = `L × (1 − d1) × (1 − d2) × … × (1 − dn)`

Notes:
- Prevents total discount exceeding 100%
- Ensures predictable outcomes
- Discount module **does not compute totals**

The **Sale module defines the line amount**.

In Capstone I: `L = (base price + add-ons) × quantity`

---

### 3.4 Sale Snapshot Lock-In

When a sale is finalized:
- Applied discounts
- Discount percentages
- Discounted amounts

are **snapshotted and stored** by the Sale module.

Later changes to discount rules must **not** affect historical sales.

---

## 4. Use Cases

### UC-1: View Discount Rules
**Actors:** Admin, Manager, Cashier

**Preconditions**
- User authenticated
- Tenant context active

**Main Flow**
1. User opens Discount Management screen.
2. System lists discount rules with:
   - Name
   - Percentage
   - Scope (Item / Branch)
   - Assigned branches
   - Schedule
   - Status

**Postconditions**
- None (read-only)

---

### UC-2: Create Discount Rule
**Actors:** Admin

**Preconditions**
- Admin authenticated
- At least one branch exists

**Main Flow**
1. Admin selects “Create Discount”.
2. Admin defines:
   - Name
   - Percentage
   - Scope (item-level or branch-level)
   - Target branches (required)
   - Target items (if item-level)
   - Schedule
3. System validates inputs.
4. Discount is saved.

**Postconditions**
- Discount becomes eligible for future sales in assigned branches.

---

### UC-3: Update Discount Rule
**Actors:** Admin

**Main Flow**
1. Admin edits discount properties.
2. System validates changes.
3. Rule is updated.

**Postconditions**
- Changes apply to future sales only.

---

### UC-4: Enable / Disable Discount Rule
**Actors:** Admin

**Main Flow**
1. Admin toggles discount status.
2. System updates eligibility.

---

### UC-5: Assign / Unassign Menu Items
**Actors:** Admin

**Main Flow**
1. Admin selects discount.
2. Admin adds or removes menu items.
3. System saves changes.

---

### UC-6: Assign / Unassign Branches
**Actors:** Admin

**Main Flow**
1. Admin selects discount.
2. Admin assigns or removes branches.
3. System saves changes.

---

### UC-7: Resolve Applicable Discounts
**Actors:** System (Sale Module)

**Preconditions**
- Active tenant and branch context
- Sale cart exists

**Main Flow**
1. Sale module queries Discount module.
2. Discount module returns:
   - Applicable item-level discounts
   - Applicable branch-level discounts
3. Sale module calculates totals and locks snapshot.

---

## 5. Requirements

- R1: Discounts are percentage-based only.
- R2: Discounts are scoped by branch.
- R3: Item-level discounts require explicit item assignment.
- R4: Branch-level discounts apply to all items sold in assigned branches.
- R5: Stacking is multiplicative.
- R6: Discounts apply to full line amount before tax.
- R7: Only Admin can mutate discounts.
- R8: All actions are audit logged.
- R9: Historical sales remain immutable.

---

## 6. Acceptance Criteria

- AC1: Discounts apply only within assigned branches.
- AC2: Multiple discounts stack correctly.
- AC3: Disabled discounts never apply.
- AC4: Managers and Cashiers cannot edit discounts.
- AC5: Sale totals do not change retroactively.
- AC6: Audit log records all lifecycle events.

---

## 7. Data Integrity & Deletion Rules

- Discounts are soft-disabled, not hard-deleted.
- Archived discounts remain visible for audit and reporting.
- Sales retain locked discount snapshots permanently.

---

## Audit Events Emitted

- `DISCOUNT_RULE_CREATED`
- `DISCOUNT_RULE_UPDATED`
- `DISCOUNT_RULE_DISABLED`
- `DISCOUNT_RULE_RESTORED`