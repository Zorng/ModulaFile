# Business Processes — How to Read This Directory

## Purpose

This directory defines **cross-module business processes** in Modula.

Processes describe **how multiple domains work together over time**:
- what triggers a workflow
- which modules participate
- the order of steps
- what must be atomic vs best-effort
- how failures, retries, and reversals are handled

If a behavior spans more than one domain, it **must** be described here.

This directory is the **spine of POS operations**.

---

## How This Directory Is Organized

Processes are grouped to reflect **how a reader should think**, not just how files are stored.

```
process/
├── README.md              ← you are here
│
├── 30_POSOperation
│   ├── 00_orchestration
│   │   └── finalize_sale_orchestration_process.md
│   │
│   ├── 10_sale_lifecycle
│   │   ├── deviceDraft_process.md
│   │   ├── finalizeOrder_process.md
│   │   ├── voidOrder_process.md
│   │   └── void_sale_inventory_reversal_process.md
│   │
│   ├── 20_inventory_effects
│   │   └── stock_deduction_on_finalize_sale_process.md
│   │
│   ├── 30_receipt_and_hardware_effects
│   │   └── (future) print_receipt_process.md
│   │
│   └── _archived
│
└── _archived
```

This structure is intentional.

---

## Reading Order (Very Important)

### 1️⃣ Start with Orchestration (Big Picture)

📁 `00_orchestration/`

These documents define **the top-level workflow**.

They answer:
- *What is the main business event?*
- *What sequence of things must happen?*
- *Which steps change business truth vs trigger effects?*

➡️ **Always start here**

Example:
- `finalize_sale_orchestration_process.md`

If you skip orchestration, you will misunderstand the system.

---

### 2️⃣ Read Sale Lifecycle Processes

📁 `10_sale_lifecycle/`

These describe **state transitions of a sale**:
- draft
- finalized
- voided

They answer:
- *When can this action happen?*
- *What does “finalized” or “voided” really mean?*
- *What states are allowed?*

Examples:
- `deviceDraft_process.md`
- `finalizeOrder_process.md`
- `voidOrder_process.md`

These are usually referenced by orchestration docs.

---

### 3️⃣ Read Effect Sub-Processes (Deep Dive)

📁 `20_inventory_effects/`  
📁 `30_receipt_and_hardware_effects/`

These describe **what happens because of a finalized sale**, but are not the sale itself.

They answer:
- *How is inventory affected?*
- *How are reversals handled?*
- *Which steps are idempotent?*

Examples:
- `stock_deduction_on_finalize_sale_process.md`
- `void_sale_inventory_reversal_process.md`

These processes are **never entry points**.
They are always triggered by higher-level orchestration.

---

## Core Principles (Non-Negotiable)

- Processes orchestrate, domains do not
- Processes do not store long-term state
- Processes reference domains and modspecs, not duplicate them
- All cross-module timing rules live here
- Idempotency and retry behavior must be explicit

If you see cross-module logic **not documented here**, that is a design smell.

---

## Relationship to Other Directories

- **Domain (`BusinessLogic/domain`)**
  - defines *truth*, invariants, ownership
- **Process (`BusinessLogic/process`)**
  - defines *coordination and timing*
- **ModSpec (`BusinessLogic/modSpec`)**
  - defines *how to implement the behavior*

Priority order: