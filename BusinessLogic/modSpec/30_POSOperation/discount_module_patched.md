# Discount Module — Feature Module Spec (Capstone 1)

**Version:** 1.3 (Patched)  
**Status:** Patched to align with `discount_domain.md` (eligibility + scope, no money math) and the Self-Contained vs Cross-Module structure.  
**Module Type:** Feature Module  
**Primary Domain:** POS Operations → Discount  
**Depends on:** Tenant & Branch Context, Authentication & Authorization, Audit Logging  
**Collaborates with (Cross-Module):** Sale/Order (Finalize Sale), Menu (item/category refs), Reporting

---

## 1. Purpose

The Discount module allows tenant administrators to define and manage discount rules that may apply during sales.

Discounts are designed to:
- reduce cashier cognitive load (avoid manual price overrides)
- ensure pricing consistency
- support controlled promotional strategies at the **branch level**

**Boundary (important):**  
This module defines **discount eligibility and scope** only.  
It does **not** calculate line totals, taxes, rounding, payment totals, or mutate financial truth.

During sale finalization, the **Finalize Sale** process:
- asks Discount for **eligible discount rules**
- applies them to line amounts (Sale owns money math)
- snapshots applied discounts for audit/reporting

---

## Why This Exists (User Reality)

Owners want promotions that don’t confuse staff or customers:
- “10% off Coffee today”
- “15% off everything at Branch A”
- “Promo ends at 6pm”

If the system doesn’t define clear rules:
- staff apply discounts inconsistently
- customers feel treated unfairly
- totals differ between devices
- reporting can’t explain discount impact later

Modula keeps discounts predictable by separating responsibilities:
- Discount decides **eligibility + which rules apply**
- Sale decides **how money is calculated**
- Finalize Sale decides **when the result becomes immutable truth**

---

## 2. Scope (Capstone 1)

### Included
- Percentage-based discounts only
- Item-level discounts (targets specific menu items)
- Branch-wide discounts (applies to all items sold in assigned branches)
- Discount scheduling (start / end time)
- Deterministic stacking rule (multiplicative)
- Read-only visibility for Manager and Cashier
- Full audit logging of discount lifecycle events
- Sale snapshot lock-in (applied discounts stored at finalize time)

### Excluded
- Fixed-amount discounts
- Coupon / voucher codes
- Customer-specific discounts
- Sale-type-specific discounts (e.g., dine-in only)
- “Best-of / exclusive” discount selection (future)
- Buy X Get Y mechanics (future)

---

## 3. Core Concepts

### 3.1 DiscountRule
A DiscountRule defines:
- percentage (0–100)
- scope:
  - **Item-level** (targets specific menu items)
  - **Branch-wide** (targets all items sold in assigned branches)
- assigned branches (required)
- optional schedule (active window)
- status (ACTIVE / INACTIVE / ARCHIVED)

**All discounts are evaluated in branch context.**  
If a rule is not assigned to the active branch, it is ignored.

---

### 3.2 Discount Stacking (Locked Behavior)

When multiple discounts are eligible, they stack **multiplicatively**.

If:
- line amount = `L`
- eligible discounts = `d1, d2, ..., dn` (each as a fraction, e.g., 0.10)

Then the Sale module computes:
- discounted_line = `L × (1 − d1) × (1 − d2) × … × (1 − dn)`

Notes:
- prevents total discount exceeding 100%
- deterministic and explainable
- **Discount module does not compute `L` or discounted totals**; it only returns eligible rules

---

### 3.3 Sale Snapshot Lock-In

When a sale is finalized, the Sale/Finalize process snapshots:
- which discount rules were applied (rule IDs)
- the percentages used
- the resulting discount amounts (computed by Sale)

Later changes to discount rules must not affect historical sales.

---

## 4. Use Cases — Self-Contained Processes

### UC-1: View Discount Rules
**Actors:** Admin, Manager, Cashier  
**Flow:** list rules with name, percentage, scope, assigned branches, schedule, status  
**Postconditions:** none

---

### UC-2: Create Discount Rule
**Actors:** Admin  
**Preconditions:**
- at least one branch exists
**Flow:**
1. enter name, percentage, scope
2. assign branches (required)
3. assign items (required if item-level)
4. set schedule (optional)
5. save
**Postconditions:** rule can become eligible for future sales in assigned branches

---

### UC-3: Update Discount Rule
**Actors:** Admin  
**Rules:** changes affect future sales only  
**Flow:** edit properties; validate; save

---

### UC-4: Activate / Deactivate Discount Rule
**Actors:** Admin  
**Flow:** toggle status; save  
**Postconditions:** eligibility updated for future sales

---

### UC-5: Assign / Unassign Menu Items (Item-Level Rules)
**Actors:** Admin  
**Flow:** select rule; add/remove menu items; save

---

### UC-6: Assign / Unassign Branches
**Actors:** Admin  
**Rules:** branch assignment is required; a rule with zero branches must be INACTIVE  
**Flow:** add/remove branches; save

---

## 5. Cross-Module Participation (Contracts)

### CM-1: Get Eligible Discount Rules (Read Contract)
**Consumer:** Finalize Sale process (Sale/Order module)  
**Inputs (minimum):**
- tenant_id
- branch_id
- timestamp
- cart lines: (menu_item_id, category_id optional, quantity)
**Outputs:**
- list of **eligible DiscountRule metadata**:
  - rule_id
  - percentage
  - scope (item-level / branch-wide)
  - target refs (item IDs if item-level)
  - stacking policy identifier (multiplicative)

**Important:** Discount returns **rules/metadata only**, not computed amounts.

---

### CM-2: Application & Snapshot (Owned by Finalize Sale, not Discount)
**Trigger:** Sale finalized  
**Finalize Sale responsibilities:**
- compute line amount `L` (base price + add-ons) × quantity
- apply multiplicative stacking to compute discounted amounts
- snapshot applied discounts + amounts as immutable sale history

---

## 6. Requirements

- R1: Discounts are percentage-based only
- R2: Discounts are scoped by branch (explicit assignment required)
- R3: Item-level discounts require explicit item assignment
- R4: Branch-wide discounts apply to all items sold in assigned branches
- R5: Stacking is multiplicative (deterministic)
- R6: Discount module does not compute totals/tax/rounding
- R7: Only Admin can mutate discount rules
- R8: All lifecycle actions are audit logged
- R9: Historical sales remain immutable via snapshot lock-in

---

## 7. Acceptance Criteria

- AC-1: Discounts apply only within assigned branches
- AC-2: Multiple discounts stack multiplicatively as defined
- AC-3: INACTIVE and ARCHIVED rules are never eligible
- AC-4: Managers and Cashiers cannot edit discounts
- AC-5: Changing rules does not change historical sales
- AC-6: Audit log records all lifecycle events
- AC-7: GetEligibleDiscountRules returns metadata only; Sale computes money

---

## 8. Data Integrity & Deletion Rules

- Rules are not hard-deleted in Capstone 1
- Deactivation/archival preserves auditability and reporting
- Sales retain locked discount snapshots permanently

---

## Audit Events Emitted

- `DISCOUNT_RULE_CREATED`
- `DISCOUNT_RULE_UPDATED`
- `DISCOUNT_RULE_ACTIVATED`
- `DISCOUNT_RULE_DEACTIVATED`
- `DISCOUNT_RULE_ARCHIVED`
