# Menu Module — Feature Module Spec (Capstone 1)

**Version:** 1.2  
**Status:** Revised & Approved  
**Module Type:** Feature Module  
**Depends on:** Auth & Authorization, Tenant & Branch Context, Policy, Inventory (recipe mapping), Sync  

---

## 1. Purpose

The Menu Module enables tenants to manage **menu items**, **modifier groups**, and **menu categories**, and to control **item visibility per branch**.  
It supports full CRUD operations with clear rules for categorization, branch assignment, and recipe linkage to inventory.

The module is designed to be flexible during onboarding (minimal required fields) while remaining strict enough to ensure consistency for sales, inventory deduction, and reporting.

---

## 2. Scope & Boundaries

### Includes
- Full CRUD for **Menu Items**
- Full CRUD for **Menu Categories**
- Full CRUD for **Modifier Groups & Options**
- Assigning menu items to one or more branches
- Removing menu items from branches
- Recipe mapping to stock items (via Inventory module)
- Handling “Uncategorized” items
- Enforcing tenant menu item soft/hard limits

### Excludes
- Price rules by time/date
- Combo meals
- Menu scheduling
- Per-branch price overrides (Capstone 2)
- Promotions logic (handled by Discount module)

---

## 3. Core Concepts

### 3.1 Menu Item
A sellable item (e.g. Iced Latte, Hot Americano).

### 3.2 Menu Category
An optional grouping for menu items (e.g. Coffee, Milk Tea).

### 3.3 Uncategorized
- If a menu item has **no category assigned**, it is treated as **“Uncategorized”**
- “Uncategorized”:
  - Is **system-derived**
  - Is **not** a persisted category
  - Cannot be edited, deleted, or renamed
  - Does **not** count toward category quota

### 3.4 Branch Assignment
- A menu item may be assigned to:
  - All branches
  - Selected branches
  - No branches (hidden item)
- Only assigned items are visible in POS for that branch

---

## 4. Use Cases

---

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

**Actor:** Admin, Manager  

#### Preconditions
- Modifier soft limit not exceeded

#### Main Flow
1. Create modifier group (e.g. Sugar Level)
2. Add options (Normal, Less, Extra)
3. Optionally attach price delta
4. Assign group to menu item

#### Postconditions
- Modifiers selectable during sale

---

### UC-11 — Link Menu Item to Inventory (Recipe Mapping)

**Actor:** Admin  

#### Preconditions
- Inventory items exist
- Menu item exists

#### Main Flow
1. Admin opens Recipe Mapping
2. Selects stock item(s)
3. Defines quantity per sale
4. Saves recipe

#### Postconditions
- Sale deductions triggered via Inventory module

---

## 5. Functional Requirements

- FR-1: Menu item CRUD must respect soft/hard limits
- FR-2: Category assignment optional
- FR-3: Uncategorized items must be supported
- FR-4: Branch assignment controls POS visibility
- FR-5: Archived items not usable in POS
- FR-6: Recipe mapping optional but required for stock deduction
- FR-7: Modifier price deltas affect sale total

---

## 6. Acceptance Criteria

- AC-1: Items can be created without category
- AC-2: Uncategorized items are clearly shown
- AC-3: Branch assignment directly affects POS visibility
- AC-4: Archived items disappear from POS
- AC-5: Recipe-based stock deduction works correctly
- AC-6: Soft/hard limits enforced consistently

---

## 7. Deletion & Recovery Rules

| Action | Behavior |
|------|---------|
| Archive | Hides item, frees soft limit |
| Restore | Re-activates item if limit allows |
| Hard Delete | System-only, no history allowed |
| Category delete | Items move to Uncategorized |

---

## 8. Out of Scope (Capstone 1)

- Menu scheduling
- Price rules by time
- Combo products
- Branch-specific pricing
- Promotion logic

---

# End of Document

## Audit Events Emitted

The following events MUST be written to the Audit Log when triggered (with tenant_id, branch_id where applicable, actor_id, and relevant entity IDs):

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

