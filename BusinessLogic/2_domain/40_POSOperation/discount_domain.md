# Discount Domain Model — Modula POS

## Domain Name
Discount

## Domain Type
Supporting Business Domain (POS Operations)

## Status
Draft (Baseline for Capstone 1)

---

## Purpose

The Discount domain defines **discount rules** and determines **eligibility** during a sale.
Its goal is to help a business run promotions without turning pricing into chaos.

Discount does **not** compute final totals, taxes, rounding, or cash movement.
It provides **eligible rules** and **how they should apply**, then the Sale/Finalize process applies them to amounts and snapshots the outcome.

---

## Why This Exists (User Reality)

A café or shop owner wants promotions, but they also want:
- staff to apply discounts consistently
- customers to receive predictable results
- reports to be trustworthy
- “what happened in that sale?” to be explainable later

In many POS systems, discounts become confusing because rules overlap:
- “10% off Coffee today”
- “Buy 2 get 1 free (Bakery)"
- “20% off for staff”
- “Member discount”

If the POS doesn’t define clear rules:
- staff argue about which discount applies
- customers feel unfairly treated
- totals differ between devices
- reporting cannot explain discount impact

Modula separates responsibilities so discounts remain **predictable**:
- Discount domain decides **eligibility and rule definition**
- Sale domain decides **how money is calculated**
- Finalize Sale process decides **when discounts are locked in**

---

## 1. Domain Boundary & Ownership

### Owns
- DiscountRule lifecycle (create, activate, deactivate, archive)
- Eligibility rules (time, branch, category, item)
- Application scope definition (what the discount targets)
- Stacking/combination policy (how multiple discounts interact)

### Explicitly NOT Responsible For
- Calculating final totals, taxes, rounding
- Mutating cash session or payment state
- Inventory deductions
- Order finalization or persistence of financial truth

---

## 2. Ubiquitous Language

| Term | Meaning |
|---|---|
| DiscountRule | A configured rule that may reduce price |
| Eligibility | Conditions that must be true for the rule to apply |
| Scope | What the discount targets (item, category, whole sale) |
| Candidate Discount | A rule that might apply before filtering |
| Eligible Discount | A rule that passes all conditions |
| Stacking Policy | Rules for combining multiple discounts |
| Applied Discount | A discount that was actually applied to a sale/line |
| Snapshot | The finalized record of applied discounts (immutable) |

---

## 3. Core Concepts

### 3.1 DiscountRule (Entity)
Represents a single discount configuration.

Common attributes:
- discount_rule_id
- name
- type: PERCENTAGE | FIXED_AMOUNT | (future: BUY_X_GET_Y)
- value (e.g., 10% or $1.00)
- scope: SALE | CATEGORY | ITEM
- target_ref (category_id, item_id set) depending on scope
- branch applicability (all branches or selected branches)
- schedule (always-on or within time window)
- status: ACTIVE | INACTIVE | ARCHIVED
- priority (optional; used only if you need deterministic ordering)

Item-level note (March):
- a single item-level rule may target multiple menu items when they share the same discount properties.
- branch assignment is still rule-level and must be validated against the selected target items.

### 3.2 Eligibility (Value Object)
Defines whether the rule applies.

Baseline eligibility for Capstone 1:
- branch allowed
- time window valid (if configured)
- target exists and is active (item/category still valid)

Future eligibility (out of scope):
- customer segment
- staff-only discounts
- membership tiers
- minimum spend thresholds

### 3.3 Available Branch Resolution (Validation Helper)
For item-level rules with multiple target items, the backend provides a validation helper:
- `ResolveAvailableBranchesForItems(item_ids)`
- returns the branch set where all selected items are valid for discount targeting

This helper is used for rule create/update validation, not for money calculation.

### 3.4 Stacking Policy (Domain Rule)
Defines how multiple discounts interact.

Baseline (Capstone 1):
- Multiple percentage discounts apply **multiplicatively**
  - Example: 10% then 20% means remaining amount × 0.9 × 0.8

Notes:
- Multiplicative stacking is deterministic and prevents “over 100%” edge cases.
- If future policies are added (exclusive / best-of), they must be explicit.

### 3.5 AppliedDiscount Snapshot (Read/Write Boundary)
When a sale is finalized, the system captures:
- which discount rules were applied
- which lines they affected
- the discount amounts (computed by Sale/Finalize process)

This snapshot becomes immutable truth for:
- audit
- reporting
- dispute resolution

Discount domain itself does **not** own the money calculation, but the system must persist the results at finalize time.

---

## 4. Invariants

- INV-D1: A DiscountRule must be well-formed (type/value/scope consistent)
- INV-D2: Percentage discount value must be between 0% and 100% (exclusive of >100)
- INV-D3: A rule in ARCHIVED state cannot be eligible
- INV-D4: Eligibility evaluation must be deterministic (same inputs → same eligible outputs)
- INV-D5: Stacking policy must be explicit; no “hidden default” behavior
- INV-D6: Applied discounts snapshot is immutable after sale finalization
- INV-D7: For item-level multi-item rules, selected `branch_ids` must be a subset of `ResolveAvailableBranchesForItems(item_ids)`.
- INV-D8: Existing rules are not silently rewritten when later menu/branch mappings change; runtime eligibility filters invalid targets.

---

## 5. Commands (Write Intents)

Self-contained commands:
- CreateDiscountRule
- UpdateDiscountRule
- ActivateDiscountRule / DeactivateDiscountRule
- ArchiveDiscountRule
- ResolveAvailableBranchesForItems (validation helper for item-level multi-item rules)

Cross-module consumed query (read-only):
- GetEligibleDiscountRules(context)
  - context includes:
    - tenant_id
    - branch_id
    - timestamp
    - cart lines (item_id, category_id, quantity)
  - returns:
    - eligible rules (rule_id, type, value, scope, target_ref, stacking info)

Discount returns rules/metadata only — not computed money totals.

---

## 6. Domain Events

- DISCOUNT_RULE_CREATED / UPDATED / ACTIVATED / DEACTIVATED / ARCHIVED

(Any “discount applied” events belong to Sale/Finalize processes because that is where monetary truth is computed and snapshotted.)

---

## 7. Read Model

Typical read queries:
- list active discount rules for a branch
- view discount rule detail
- preview eligible discount rules for a cart (optional UI feature)

---

## 8. Cross-Module Integration (Contracts)

### Discount → Finalize Sale Process
Discount provides:
- eligible discount rules for the sale/cart context

Finalize Sale process:
- computes discounted amounts on each line
- applies stacking policy
- snapshots applied discounts (immutable)
- proceeds to inventory deduction and cash recording

Discount must never:
- write to Sale totals directly
- mutate cash or inventory

---

## 9. Out of Scope (Baseline)

- Buy X Get Y mechanics
- Customer segmentation / membership discounts
- Coupon codes
- Exclusive/best-of discount selection
- Minimum spend or conditional thresholds
- Tax-inclusive vs tax-exclusive discount rules

---

## 10. Future Evolution

- Add discount “exclusivity groups” (best-of within group)
- Coupon redemption and usage limits
- Customer membership tiers
- Promotions calendar / scheduled campaigns
- Audit-friendly “discount explainability” UI
