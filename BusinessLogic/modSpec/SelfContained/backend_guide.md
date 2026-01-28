# Backend Development Guide — Mod Spec Reading Order

This document defines the **mandatory reading order** for Modula module specifications.  
All AI coding agents and backend developers **must follow this order** before implementing or modifying code.

The order reflects **dependency direction, data ownership, and business criticality**.

Skipping or reordering modules may result in:
- Broken authorization boundaries
- Incorrect policy enforcement
- Financial inconsistencies
- Data integrity issues in multi-tenant scenarios

---

## Orientation (Read First)

### 0. Project Overview & Architecture Context
Purpose:
- Understand Capstone 1 scope
- Distinguish Core Modules vs Feature Modules
- Understand offline-first and modular assumptions

---

## Tier 1 — Core Modules (Foundation)

These modules define **identity, scope, and behavior rules**.  
All feature modules depend on them.

### 1. Authentication & Authorization (`auth_module.md`)
- Accounts, users, roles (Admin / Manager / Cashier)
- Multi-tenant account membership
- Permission boundaries
- Credential lifecycle (login, reset, verification)

### 2. Tenant & Branch Context (`tenant_module.md`)
- Tenant auto-creation from subscriptions
- Tenant lifecycle
- Branch hierarchy and scoping
- Cross-branch visibility rules

### 3. Policy & Configuration (`policy_module.md`)
- System behavior toggles
- Policy keys and defaults
- Enforcement responsibility (backend only)
- Read-only settings model (no policy creation)

### 4. Sync & Offline Support (`sync_offline_module.md`)
- Offline-first assumptions
- Idempotent writes
- Conflict resolution rules
- Deferred execution model

### 5. Audit Logging (`audit_module.md`)
- Mandatory audit events
- Cross-module observability
- Immutable event history
- Compliance and traceability

---

## Tier 2 — Transactional Feature Modules (Money & State)

These modules mutate **financial or irreversible state**.

### 6. Cash Session (`cashSession_module.md`)
- Cash session lifecycle
- Register / device ownership
- Paid-in, paid-out, refunds
- Policy-driven controls

### 7. Sale (`sale_module.md`)
- Draft vs finalized sale
- Cart and checkout flow
- Order lifecycle (in-prep, ready, delivered)
- Void request and approval workflow
- Policy enforcement hooks

### 8. Discount (`discount_module.md`)
- Percentage-based discounts
- Stacking rules
- Lock-in at sale finalization
- Read-only for cashier/manager

---

## Tier 3 — Operational Feature Modules

These modules support daily operations.

### 9. Inventory (`inventory_module.md`)
- Stock items and categories
- Batch-based tracking and expiry
- Journal-based mutation
- Sale-driven subtraction

### 10. Menu (`menu_module.md`)
- Menu items and categories
- Modifier groups and add-ons
- Branch assignment
- Recipe mapping to inventory

### 11. Staff Attendance (`staffAttendance_module.md`)
- Check-in / check-out
- Shift enforcement via policy
- Approval flows
- Read-only history for admin

### 12. Staff Management (`staffManagement_module.md`)
- Staff lifecycle (create, assign, deactivate)
- Role assignment
- Branch linkage
- Seat-based constraints

---

## Tier 4 — Presentation & Aggregation

These modules do **not** own data. They consume results.

### 13. Receipt / eReceipt (`receipt_module.md`)
- Read-only consumer of Sale + Tenant + Branch data
- Formatting and presentation
- Footer customization only

### 14. Reporting (`report_module.md`)
- Aggregated sales and inventory views
- Treatment of VOID_PENDING sales
- Admin-only visibility (Capstone 1)

---

## Non-Negotiable Rules for Coding Agents

- Do not implement feature logic before reading its dependencies
- Do not duplicate policy logic inside feature modules
- Do not bypass audit logging for state-changing actions
- Do not assume online-only behavior
- Do not invent new permissions outside the Auth module
- Do not mutate finalized financial records except via approved workflows

---

## Mental Model

Core modules **define the rules**.  
Feature modules **execute within those rules**.  
Presentation modules **observe results only**.

Build once. Enforce centrally. Mutate carefully.