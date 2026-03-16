# Branch Module (Core)

**Version:** 1.1  
**Status:** Revised (Branch provisioning is system-driven)  
**Module Type:** Core Module  
**Depends on:** Authentication (identity), Access Control (authorization decisions), Tenant (workspace boundary), Audit (logging)  
**Related Modules:** Subscription/Billing (Phase 2), Sale & Orders, Inventory, Cash Session, Attendance, Reporting, Receipt/eReceipt, Policy, Audit Logging

---

## 1. Purpose

The Branch module manages the **lifecycle state and profile** of branches under a tenant, including:
- Maintaining branch profile (name, address, contact)
- Controlling branch operational status (ACTIVE/FROZEN)
- Providing branch identity as a consistent reference for branch-scoped modules
- Defining the branch-layer operational workspace entered from tenant layer

**Branch provisioning is NOT user-driven.** Branch records are created by the system when:
- A tenant completes first-branch paid activation (`branch activation`; branch provisioned after payment confirmation), and/or
- A tenant purchases an `additional branch subscription` (additional branch activation), and/or
- Developers/system operators provision branches manually (Capstone I temporary approach)

### Workspace model note

In the Modula workspace model:
- branch entry begins from **tenant layer**
- entering a branch activates **branch layer**
- branch layer is used for operational workflows
- branch-scoped management actions do not require branch entry; they may be initiated from tenant layer with explicit target branch

---

## 2. Core Concepts

### 2.1 System-Provisioned Branches
Branches are created by a **provisioning process**, not by Admin UI.
- Capstone I: provisioning may be performed by developers (manual command/seed/admin tooling)
- Capstone II: provisioning is performed automatically by Subscription/Billing
- Billing model lock: a branch is a billable workspace unit; archive/delete does not grant reusable free branch activation by default.

### 2.2 Branch Status
- `ACTIVE`: operations allowed
- `FROZEN`: operations blocked (read-only allowed)

### 2.3 Branch Guard Rule (Cross-cutting)
Any operation that creates or mutates branch-scoped operational records requires:
- `branch.status == ACTIVE`

---

## 3. Use Cases

### UC-1 Provision Branch (System)
**Actors:** System (Provisioning workflow), Developers (Capstone I provisioning only)  
**Preconditions:**
- Tenant exists
- Provisioning trigger occurs:
  - First-branch activation payment confirmed, or
  - Additional-branch activation payment confirmed, or
  - Developer executes provisioning command (Capstone I)

**Main Flow:**
1. Provisioning workflow requests branch creation for a tenant.
2. System creates a new branch record with `status = ACTIVE`.
3. System returns the new branch identifier.
4. (Optional) System attaches default branch settings (e.g., default receipt footer empty).

**Postconditions:**
- Branch exists and is available for assignment (staff, inventory, sales contexts).
- Audit log entry recorded (if audit is enabled).

**Acceptance Criteria:**
- No end-user UI creates a branch record directly (branch creation is system provisioning).
- Branch creation is only accessible via system provisioning path.
- Provisioned branch is `ACTIVE` by default.

---

### UC-2 Update Branch Profile
**Actors:** Admin  
**Preconditions:**
- Actor authenticated as Admin
- Branch exists (provisioned)
- Tenant context active
- Target branch selected explicitly in tenant layer management flow

**Main Flow:**
1. Admin selects an existing target branch from tenant layer.
2. Admin updates branch profile fields (name/address/contact).
3. System validates and persists.

**Postconditions:**
- Branch profile updated.
- Audit log recorded.

**Acceptance Criteria:**
- Admin can update profile fields for any branch in the tenant.
- Manager/Cashier cannot update branch profile.

---

### UC-3 Freeze Branch
**Actors:** Admin  
**Preconditions:**
- Actor authenticated as Admin
- Branch exists and is `ACTIVE`
- Tenant context active
- Target branch selected explicitly in tenant layer management flow

**Main Flow:**
1. Admin selects target branch → “Freeze branch”.
2. System sets branch to `FROZEN`.

**Postconditions:**
- Branch Guard Rule blocks operational writes for that branch.

**Acceptance Criteria:**
- When frozen, the system blocks:
  - Creating cart / adding items to cart (if you enforce guard at cart stage)
  - Finalizing sales/orders
  - Opening cash sessions / writing cash movements
  - Inventory restock/adjust/journal writes
  - Attendance check-in/out
- Read-only access remains (reports/history).

---

### UC-4 Unfreeze Branch
**Actors:** Admin  
**Preconditions:**
- Actor authenticated as Admin
- Branch exists and is `FROZEN`
- Tenant context active
- Target branch selected explicitly in tenant layer management flow

**Main Flow:**
1. Admin selects target branch → “Unfreeze branch”.
2. System sets branch to `ACTIVE`.

**Postconditions:**
- Operational actions resume.

---

### UC-5 List Branches
**Actors:** Admin, Manager, Cashier  
**Preconditions:**
- Actor authenticated

**Main Flow:**
1. System returns accessible branches:
   - Branch entry list is derived from **explicit branch assignments** (for all roles).
   - Filter to branches where:
     - BranchAssignment is ACTIVE for the actor, and
     - Branch status is ACTIVE (not FROZEN)

**Acceptance Criteria:**
- Branch visibility respects explicit assignment (no implicit “Admin can access all branches”).
- To grant “all branches”, create assignments for every branch.
- This list is used from tenant layer to enter branch-layer operations.

---

## 4. Requirements

### Functional
- Branch profile update (Admin)
- Freeze/unfreeze (Admin)
- Branch list (assignment-scoped)
- System provisioning hook (non-UI)

### Non-functional
- Enforcement must be server-side
- Audit trails for update/freeze/unfreeze/provision

---
