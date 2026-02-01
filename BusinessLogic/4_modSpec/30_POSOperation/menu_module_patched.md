# Menu Module — Feature Module Spec (Capstone 1)

**Version:** 1.3 (Patched)  
**Status:** Patched to align with Menu Domain (composition + TRACKED components)  
**Module Type:** Feature Module  

**Depends on:** Auth & Authorization, Tenant & Branch Context, Policy (limits only), Sync  
**Collaborates with (Cross-Module):** Sale/Order (Finalize Sale process), Inventory (ledger deductions), Discount (pricing only)

---

## 1. Purpose

The Menu Module defines **what can be sold** and **how items are composed**, including:
- Menu items and categories
- Modifier groups and options
- Branch visibility
- **Composition metadata** used to derive inventory deductions

Menu does **not** deduct stock.

Stock deduction is executed by the **Finalize Sale** cross-module process. During finalization, the process evaluates each order line’s **composition** (menu item + selected modifiers). If the evaluated composition contains any **TRACKED** components, deduction lines are generated and passed to Inventory. If no TRACKED components exist, no inventory deduction occurs.

This design avoids hidden rules such as “has recipe = deduct” and supports both food recipes and retail-style items.

---

## 2. Scope & Boundaries

### Includes
- Full CRUD for **Menu Items**
- Full CRUD for **Menu Categories**
- Full CRUD for **Modifier Groups & Options**
- Assigning menu items to one or more branches
- Removing menu items from branches
- Defining **composition** (base components + modifier deltas)
- Explicit **TRACKED / NOT_TRACKED** component configuration
- Handling “Uncategorized” items
- Enforcing tenant menu item soft/hard limits

### Excludes
- Time-based pricing rules
- Combo/bundled meals
- Menu scheduling
- Per-branch price overrides (Capstone 2)
- Promotion logic (handled by Discount module)
- Inventory mutation logic

---

## 3. Core Concepts

### 3.1 Menu Item
A sellable item (e.g., Iced Latte, Croissant, Bottled Water).

A Menu Item may be:
- **Recipe-based** (multiple components)
- **Direct-stock** (single component, e.g., bottled water)

Both are modeled using the same composition mechanism.

### 3.2 Menu Category
An optional grouping for menu items (e.g., Coffee, Bakery).

### 3.3 Uncategorized
- If a menu item has **no category assigned**, it is treated as **“Uncategorized”**
- “Uncategorized” is:
  - system-derived
  - not persisted
  - not editable
  - not counted toward category quota

### 3.4 Branch Assignment
- Menu items may be assigned to:
  - All branches
  - Selected branches
  - No branches (hidden item)
- Only assigned items are visible in POS for that branch

### 3.5 Composition Model (Important)

Menu defines **composition**, not deduction.

Composition =  
**Base Components** + **Modifier Deltas** − **Omitted Components**

Each component has:
- `stock_item_id`
- `quantity_in_base_unit`
- `tracking_mode`: **TRACKED** | **NOT_TRACKED**

Examples:
- Latte: beans (TRACKED), milk (TRACKED)
- Cup: may be TRACKED or NOT_TRACKED depending on store setup
- Ice: typically NOT_TRACKED
- Bottled water: one TRACKED component (itself)

---

## 4. Use Cases
### UC-1 — Create Menu Item

**Actor:** Admin, Manager  

#### Preconditions
- Tenant exists and is active
- User has Admin or Manager role
- Menu item soft limit not exceeded
- At least one branch exists for the tenant

#### Main Flow
1. User opens “Create Menu Item”
2. User enters required fields:
   - Name
   - Price
3. User optionally:
   - Assigns a category
   - Assigns modifier groups
   - Assigns branches
   - Uploads image
4. System validates quota and input
5. Menu item is created
6. If no category assigned → item is treated as **Uncategorized**
7. Item becomes visible only in assigned branches

#### Postconditions
- Menu item exists in ACTIVE state
- Item appears in menu list
- Item is usable in POS (if assigned to branch)

#### Alternate Flows
- If soft limit exceeded → creation blocked with message
- If no branch assigned → item created but hidden from POS

---

### UC-2 — View Menu Items

**Actor:** Admin, Manager  

#### Preconditions
- User authenticated
- Branch context available

#### Main Flow
1. User opens Menu list
2. System displays:
   - Categorized items
   - “Uncategorized” section (if applicable)
3. User may filter by:
   - Category
   - Branch
   - Active/Archived

#### Postconditions
- No state change

---

### UC-3 — Update Menu Item

**Actor:** Admin, Manager  

#### Preconditions
- Menu item exists
- User has edit permission

#### Main Flow
1. User opens menu item detail
2. User edits allowed fields:
   - Name
   - Price
   - Category
   - Modifiers
   - Branch assignment
   - Image
3. System validates changes
4. Updates are saved

#### Postconditions
- Updated item reflected in:
  - POS
  - Reports
  - Inventory recipe mapping (if applicable)

#### Notes
- Removing category → item moves to **Uncategorized**
- Removing all branches → item hidden from POS

---

### UC-4 — Archive Menu Item (Soft Delete)

**Actor:** Admin  

#### Preconditions
- Menu item exists
- Item is not currently used in an active sale session

#### Main Flow
1. Admin selects “Archive”
2. System sets `is_active = false`
3. Item removed from POS
4. Historical sales remain unchanged

#### Postconditions
- Soft limit freed
- Item appears in Archived list

---

### UC-5 — Restore Menu Item

**Actor:** Admin  

#### Preconditions
- Item is archived
- Menu item soft limit not exceeded

#### Main Flow
1. Admin selects archived item
2. Admin restores item
3. System reactivates item

#### Postconditions
- Item becomes active
- Item appears under:
  - Previous category, or
  - Uncategorized (if category no longer exists)

---

### UC-6 — Delete Menu Item (Hard Delete)

**Actor:** System only (not exposed to tenant UI)

#### Preconditions
- Item has no historical sales
- Item has no recipe mapping
- Item is archived

#### Main Flow
- System permanently deletes item (maintenance operation)

---

### UC-7 — Assign Menu Item to Branch(es)

**Actor:** Admin, Manager  

#### Preconditions
- Menu item exists
- At least one branch exists

#### Main Flow
1. User opens item → Branch Assignment
2. User selects one or more branches
3. System saves assignments

#### Postconditions
- Item visible in selected branches only

---

### UC-8 — Remove Menu Item from Branch

**Actor:** Admin, Manager  

#### Preconditions
- Item assigned to at least one branch

#### Main Flow
1. User removes branch assignment
2. System updates visibility

#### Postconditions
- Item no longer appears in POS for that branch

---

### UC-9 — Manage Menu Categories (CRUD)

**Actor:** Admin  

#### Preconditions
- Category soft limit not exceeded (on create)

#### Main Flow
- Create, rename, reorder, archive categories

#### Postconditions
- Archived categories:
  - Do not delete menu items
  - Affected items move to **Uncategorized**

---

### UC-10 — Manage Modifier Groups & Options

**Notes (Patched):**
- Modifier options may define:
  - price deltas
  - **component deltas** (positive or negative)
- Component deltas participate in composition evaluation

Examples:
- Large size → +100ml milk
- Extra topping → +1 topping component
- BYO cup → −1 cup component (if cups are tracked)

### UC-11 — Define Menu Item Composition (Replaces “Recipe Mapping”)

**Actor:** Admin  

#### Preconditions
- Inventory stock items exist (for TRACKED components)

#### Main Flow
1. Admin opens Menu Item → Composition
2. Defines base components (ingredients, packaging, or direct-stock)
3. Sets quantity and tracking_mode for each component
4. Saves composition

#### Postconditions
- Composition metadata stored
- No inventory mutation occurs at this stage

---

## 5. Cross-Module Participation (Explicit)

### CM-1 — Provide Composition for Finalize Sale

**Trigger:** Finalize Sale process  
**Input:** menu_item_id, selected_modifier_option_ids  
**Output:** aggregated component list with tracking_mode  

**Rules:**
- Components with `NOT_TRACKED` are excluded from deduction lines
- Components with `TRACKED` generate deduction lines
- Menu does not decide timing or perform deduction

---

## 6. Functional Requirements (Patched)

- FR-1: Menu item CRUD must respect soft/hard limits
- FR-2: Category assignment optional
- FR-3: Uncategorized items must be supported
- FR-4: Branch assignment controls POS visibility
- FR-5: Archived items not usable in POS
- FR-6: Stock deduction is based on **TRACKED components**, not on “has recipe”
- FR-7: Modifier price deltas affect sale total
- FR-8: Modifier component deltas affect composition evaluation

---

## 7. Acceptance Criteria (Patched)

- AC-1: Items can be created without category
- AC-2: Uncategorized items are clearly shown
- AC-3: Branch assignment directly affects POS visibility
- AC-4: Archived items disappear from POS
- AC-5: Finalized sales generate inventory deductions **only** for TRACKED components
- AC-6: Items with no TRACKED components do not trigger inventory deduction
- AC-7: Modifier component deltas correctly affect deduction quantities

---

## 8. Deletion & Recovery Rules

| Action | Behavior |
|------|---------|
| Archive | Hides item, frees soft limit |
| Restore | Re-activates item if limit allows |
| Hard Delete | System-only, no history allowed |
| Category delete | Items move to Uncategorized |

---

## 9. Out of Scope (Capstone 1)

- Recipe snapshotting for historical reconstruction
- Menu scheduling
- Time-based pricing rules
- Combo products
- Branch-specific pricing
- Promotion logic

---

# End of Document

## Audit Events Emitted

The following events MUST be written to the Audit Log (tenant_id, branch_id where applicable, actor_id, and relevant entity IDs):

- `MENU_ITEM_CREATED`
- `MENU_ITEM_UPDATED`
- `MENU_ITEM_ARCHIVED`
- `MENU_ITEM_RESTORED`
- `MODIFIER_GROUP_CREATED`
- `MODIFIER_GROUP_UPDATED`
- `MENU_CATEGORY_CREATED`
- `MENU_CATEGORY_UPDATED`
- `MENU_ITEM_BRANCH_ASSIGNED`
- `MENU_ITEM_BRANCH_UNASSIGNED`
