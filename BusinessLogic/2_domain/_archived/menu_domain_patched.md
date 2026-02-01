# Menu Domain Model — Modula POS

## Domain Name
Menu

## Domain Type
Core Business Domain (POS Operations)

## Status
Draft (Baseline for Capstone 1)

---

## Purpose

The Menu domain defines what can be sold:
- Menu categories and menu items
- Modifiers (e.g., size, sugar level, toppings)
- Pricing and availability by branch (visibility)

Menu also defines **composition metadata** used by other modules:
- how an item is made (recipe / components)
- how modifiers change the composition (deltas)
- which components are inventory-tracked vs not-tracked

Menu does **not** deduct stock. It only describes what should be deducted **if** stock deduction is enabled by cross-module processes.

---

## Why Composition Exists (User Reality)

In a real café, not everything “disappears” the same way.

Some items are **transactional**: when you sell one, you know exactly what was consumed.
Examples: bottled water, croissant.
If you sell 1 bottle, you lose 1 bottle — precise and reliable.

Some items are **operational**: they are consumed while running the store, and the exact usage per sale is messy.
Examples: cups, lids, straws, napkins, ice.
Cups may be defective, drinks may be remade, customers may request double cups, or take extra napkins — so per-sale tracking can drift quickly.

Modula’s composition rules exist to reflect this reality without forcing fake precision.

### Transactional Inventory
Transactional items are best handled by **automatic deduction** during order finalization because usage is deterministic.

### Operational Inventory
Operational items are often best handled by **periodic counting and adjustment**, because real usage depends on day-to-day operations.

### How Menu Supports Both
Menu defines **composition** (what components are involved) and allows each component to be marked:
- **TRACKED** → becomes an inventory deduction line
- **NOT_TRACKED** → informational only (handled operationally, not auto-deducted)

This is why composition is separate from deduction timing: Menu describes “what”, Inventory records “how”, and the Finalize Order process decides “when”.

---

## 1. Domain Boundary & Ownership

### Owns
- Menu categories
- Menu items
- Modifier groups and modifier options
- Composition definition for items (base recipe + modifier deltas)
- Branch availability / visibility of menu items

### Explicitly NOT Responsible For
- When stock is deducted (Finalize Order process owns orchestration)
- How inventory is recorded (Inventory domain owns ledger + projections)
- Cash movement and payment results
- Promotions calculation (Discount domain owns discount logic)

---

## 2. Ubiquitous Language

| Term | Meaning |
|---|---|
| MenuItem | A sellable product (Latte, Croissant, Bottled Water) |
| MenuCategory | Grouping of menu items (Coffee, Bakery) |
| ModifierGroup | A set of options to configure an item (Size, Sugar Level) |
| ModifierOption | One choice inside a group (Large, No Sugar, Cheese Foam) |
| Selection | The chosen options for an order line |
| Composition | Components required to fulfill an item configuration |
| Component | A unit of consumption (Milk, Beans, Cup_16oz, Lid) |
| ComponentDelta | A change in components caused by a modifier |
| Tracked Component | Generates inventory deduction lines |
| Not-Tracked Component | No inventory deduction (informational only) |
| Direct-Stock Item | Maps directly to a single stock item (e.g., bottled water) |
| Recipe-Based Item | Defined by components (ingredients + packaging) |

---

## 3. Core Concepts & Data Model

### 3.1 MenuItem
Represents a sellable product.
Key fields:
- menu_item_id
- name
- base_price
- category_id (optional)
- is_active
- branch_visibility (which branches can sell it)

### 3.2 ModifierGroup
A group attached to a MenuItem:
- modifier_group_id
- name (Size, Sugar Level, Topping)
- selection rule (single-choice vs multi-choice)
- min/max selections
- is_required

### 3.3 ModifierOption
One option in a group:
- modifier_option_id
- label (Large, No Sugar, Cheese Foam)
- price_delta (optional)
- component_deltas (optional)

### 3.4 Composition Model (Baseline)
MenuItem composition is:

**Base Components** + **Modifier Deltas** − **Omitted Components**

Composition is evaluated at order finalization time.

#### Component (Value Object)
- stock_item_id (or component_key)
- quantity_in_base_unit
- tracking_mode: TRACKED | NOT_TRACKED

Components can represent:
- Ingredients (milk, beans, syrup)
- Packaging (cup, lid, straw)
- Retail goods (bottled water as direct-stock)

---

## 4. Composition Rules

### Rule C-1: Recipe is configuration-aware
A MenuItem’s recipe is the total composition after applying selected modifiers.

### Rule C-2: Modifiers contribute deltas
ModifierOptions may add or subtract components.
Examples:
- Large cup: +100ml milk, +1 Cup_16oz
- BYO cup: −1 Cup_16oz, −1 Lid (optional), −1 Straw (optional)

### Rule C-3: Packaging is just another component
Cups, lids, straws are modeled as components, not special cases.

### Rule C-4: Tracking is explicit
Every component is either:
- TRACKED (deduct from inventory)
- NOT_TRACKED (ignored by inventory deduction)

This allows modeling ice/water/small consumables without forcing inventory tracking.

### Rule C-5: Direct-stock mapping is supported
Some items deduct a single stock item directly:
- Bottled water
- Packaged snacks

This can be expressed as:
- a one-component recipe (TRACKED), or
- a direct mapping field (implementation choice)

---

## 5. Commands (Write Intents)

Self-contained:
- CreateMenuCategory / UpdateMenuCategory / ArchiveMenuCategory
- CreateMenuItem / UpdateMenuItem / ArchiveMenuItem
- DefineMenuItemComposition (base components)
- CreateModifierGroup / UpdateModifierGroup / ArchiveModifierGroup
- AddModifierOption / UpdateModifierOption / ArchiveModifierOption
- AttachModifierGroupToMenuItem (selection rules)
- SetBranchVisibilityForMenuItem

Cross-module consumed query (read-only):
- GetCompositionForOrderLine(menu_item_id, selected_modifier_option_ids)
  - returns aggregated components with tracking_mode

---

## 6. Invariants

- INV-M1: MenuItem must be active to be sold
- INV-M2: ModifierGroup selection rules must be valid (min ≤ max; required implies min ≥ 1)
- INV-M3: Composition quantities must be non-negative after aggregation
- INV-M4: Negative deltas are allowed only for “omit” behaviors (e.g., BYO cup)
- INV-M5: Tracking mode is explicit per component
- INV-M6: Branch visibility must remain within tenant scope

---

## 7. Domain Events

- MENU_ITEM_CREATED / UPDATED / ARCHIVED
- MENU_CATEGORY_CREATED / UPDATED / ARCHIVED
- MODIFIER_GROUP_CREATED / UPDATED / ARCHIVED
- MODIFIER_OPTION_CREATED / UPDATED / ARCHIVED
- MENU_ITEM_COMPOSITION_DEFINED
- MENU_ITEM_BRANCH_VISIBILITY_CHANGED

---

## 8. Read Model

- List menu items for a branch (visible + active)
- Get menu item detail (modifier groups/options)
- Evaluate composition for a selected configuration

Composition evaluation must be deterministic and side-effect free.

---

## 9. Cross-Module Integration (Contracts)

Menu provides composition metadata to cross-module processes.

### Menu → Finalize Order Process
Input:
- menu_item_id
- selected_modifier_option_ids

Output:
- aggregated component list (with tracking_mode)

Process layer behavior:
- excludes NOT_TRACKED components from inventory deduction lines
- passes deduction_lines to Inventory (idempotent)

Menu does not trigger deductions and does not write to Inventory.

---

## 10. Out of Scope (Baseline)

- Recipe version snapshots for historical reconstruction (future)
- Advanced costing and margin analysis
- Time-based menus / seasonal rules (future)
- Kitchen routing / prep stations (future)

---

## 11. Future Evolution

- Recipe snapshotting tied to sales to ensure historical correctness
- Industry variants:
  - Retail: emphasis on direct-stock items
  - Pharmacy: regulated items and lot/expiry enforcement
- More complex modifier pricing and bundling rules
