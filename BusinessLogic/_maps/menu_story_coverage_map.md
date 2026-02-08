# Story Coverage Map — Menu (Defining What Can Be Sold)

**Purpose:**  
Identify human-centered stories for Menu behavior and map them to technical use cases, ensuring complete coverage without “one story per UC”.

**Scope:**  
- Menu Domain (`BusinessLogic/2_domain/40_POSOperation/menu_domain_patched_v2.md`)
- Menu Module Spec (`BusinessLogic/5_modSpec/40_POSOperation/menu_module_patched.md`)

**Rule:**  
Stories are grouped by **human operational context**, not by modules or CRUD operations.

---

## 1) Source Artifacts (Authoritative)

- **Domain:** `BusinessLogic/2_domain/40_POSOperation/menu_domain_patched_v2.md`
- **ModSpec:** `BusinessLogic/5_modSpec/40_POSOperation/menu_module_patched.md`

---

## 2) Operational Contexts (Human Reality)

From reading the domain and modspec, Menu behavior is experienced by humans in **four distinct operational contexts**:

1. **Defining What the Store Sells**
2. **Organizing and Presenting the Menu**
3. **Configuring Customization Options**
4. **Connecting Menu Items to Stock Reality**

These contexts are stable across cafés and retail-like stores.

---

## 3) Story Set (Human Situations)

### Context: Defining What the Store Sells

1. **Creating and Managing Menu Items**  
   File: `BusinessLogic/1_stories/handling_menu/creating_and_managing_menu_items.md`  
   Human situation: deciding what products exist, their names, prices, and whether they are active.

---

### Context: Organizing and Presenting the Menu

2. **Organizing Items into Categories**  
   File: `BusinessLogic/1_stories/handling_menu/organizing_menu_categories.md`  
   Human situation: keeping the menu readable and navigable for staff and customers.

3. **Controlling Which Branch Can Sell What**  
   File: `BusinessLogic/1_stories/handling_menu/menu_visibility.md`  
   Human situation: hiding or showing items depending on location or readiness.

---

### Context: Configuring Customization Options

4. **Allowing Customers to Customize Items**  
   File: `BusinessLogic/1_stories/handling_menu/configuring_item_customizations.md`  
   Human situation: defining sizes, toppings, and options that change how an item is ordered.

---

### Context: Connecting Menu Items to Stock Reality

5. **Defining How an Item Is Made**  
   File: `BusinessLogic/1_stories/handling_menu/define_item_compostion.md`  
   Human situation: describing ingredients, packaging, or retail units without worrying about deduction timing.

6. **Deciding What Should Be Automatically Tracked**  
   File: `BusinessLogic/1_stories/handling_menu/deciding_what_inventory_is_tracked.md`  
   Human situation: choosing which components are precise enough to auto-deduct and which are operational.

---

## 4) Coverage Mapping (Use Case → Story)

### Story 1 — Creating and Managing Menu Items
Covers:
- **UC-1:** Create Menu Item
- **UC-3:** Update Menu Item
- **UC-4:** Archive Menu Item (Soft Delete)
- **UC-5:** Restore Menu Item
- **UC-6:** Delete Menu Item (System only)

Notes:
- CRUD grouped intentionally as one human responsibility.

---

### Story 2 — Organizing Items into Categories
Covers:
- **UC-9:** Manage Menu Categories (CRUD)
- Uncategorized behavior (implicit system behavior)

---

### Story 3 — Controlling Which Branch Can Sell What
Covers:
- **UC-7:** Assign Menu Item to Branch(es)
- **UC-8:** Remove Menu Item from Branch

---

### Story 4 — Allowing Customers to Customize Items
Covers:
- **UC-10:** Manage Modifier Groups & Options

Notes:
- Story should emphasize flexibility and clarity, not internal rules.

---

### Story 5 — Defining How an Item Is Made
Covers:
- **UC-11:** Define Menu Item Composition

Notes:
- Focus on describing reality (“what goes into this item”), not deduction mechanics.

---

### Story 6 — Deciding What Inventory Is Automatically Tracked
Covers:
- Tracking mode decisions (TRACKED vs NOT_TRACKED)
- Direct-stock vs recipe-based items
- Modifier component deltas affecting composition

Notes:
- This is a conceptual, policy-like story grounded in real café operations.

---

## 5) Coverage Summary (Quick Check)

| Use Case | Covered By Story |
|---|---|
| UC-1 Create Menu Item | Story 1 |
| UC-2 View Menu Items | Story 2 (implicit visibility) |
| UC-3 Update Menu Item | Story 1 |
| UC-4 Archive Menu Item | Story 1 |
| UC-5 Restore Menu Item | Story 1 |
| UC-6 Hard Delete Menu Item | Story 1 |
| UC-7 Assign Item to Branch | Story 3 |
| UC-8 Remove Item from Branch | Story 3 |
| UC-9 Manage Categories | Story 2 |
| UC-10 Manage Modifiers | Story 4 |
| UC-11 Define Composition | Story 5 & 6 |

---

## 6) Status Tracker (Fill As You Write)

| Story | File | Status | Notes |
|---|---|---|---|
| Creating and Managing Menu Items | `BusinessLogic/1_stories/handling_menu/creating_and_managing_menu_items.md` | ⬜ | |
| Organizing Items into Categories | `BusinessLogic/1_stories/handling_menu/organizing_menu_categories.md` | ⬜ | |
| Controlling Which Branch Can Sell What | `BusinessLogic/1_stories/handling_menu/menu_visibility.md` | ⬜ | |
| Allowing Customers to Customize Items | `BusinessLogic/1_stories/handling_menu/configuring_item_customizations.md` | ⬜ | |
| Defining How an Item Is Made | `BusinessLogic/1_stories/handling_menu/define_item_compostion.md` | ⬜ | |
| Deciding What Inventory Is Automatically Tracked | `BusinessLogic/1_stories/handling_menu/deciding_what_inventory_is_tracked.md` | ⬜ | |

Legend:
- ⬜ Not written
- 🟨 Drafted
- ✅ Final

---

## 7) Guardrails (Important)

- Do **not** write one story per UC.
- CRUD is grouped unless it represents different human tension.
- Stories must remain non-technical and human-centered.
- If new menu features appear later:
  - first fit them into an existing story,
  - only add a new story if it represents a new human situation.

---

# End of Map
