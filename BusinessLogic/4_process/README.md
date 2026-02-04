# Business Processes — How to Read This Directory

## Purpose

This directory defines **cross-module business processes** in Modula.

Processes describe **how multiple domains work together over time**:
- what triggers a workflow
- which modules participate
- the order in which things must happen
- what changes business truth vs what are operational side-effects
- how failures, retries, and reversals are handled

If a behavior spans more than one domain, it **must** be described here.

This directory is the **spine of POS operations**.

---

## Key Idea: Processes Are Read in Sequence, Not in Isolation

Business processes are **stories**, not standalone specs.

Because of this, processes are organized using **numeric prefixes** that encode:
- reading order
- parent orchestration
- sub-process participation

> **Lower number = read earlier**

This is intentional and non-negotiable.

---

## Naming Convention (Very Important)

### Prefix Pattern

`NN_<process_name>.md`

Where:
- `NN` defines **position in the process chain**
- files with the same leading digit belong to the **same storyline**

### Example
```
10_finalize_sale_orchestration.md   ← start here
11_deviceDraft_process.md
12_finalize_order_process.md
13_stock_deduction_on_finalize_sale.md
14_cash_movement_on_finalize_sale.md
15_receipt_creation_process.md
```

This means:
- `10_*` is the **orchestration**
- `11–19_*` are **sub-processes that participate in finalize sale**

---

## High-Level Structure
```
process/
├── README.md   ← you are here
│
├── 30_POSOperation
│   ├── 10_finalize_sale_orchestration.md
│   ├── 11_deviceDraft_process.md
│   ├── 12_finalize_order_process.md
│   ├── 13_stock_deduction_on_finalize_sale.md
│   ├── 14_cash_movement_on_finalize_sale.md
│   ├── 15_receipt_creation_process.md
│   ├── 16_hardware_effects_process.md
│
│   ├── 20_void_sale_orchestration.md
│   ├── 21_void_order_process.md
│   ├── 22_void_sale_inventory_reversal.md
│   ├── 23_void_sale_cash_reversal.md
│
│   ├── 30_cash_session_lifecycle.md
│   ├── 31_open_cash_session.md
│   ├── 32_close_cash_session.md
│   ├── 33_x_report_process.md
│   ├── 34_z_report_process.md
│
│   └── _archived
│
└── _archived
```

This structure is designed to scale without becoming unreadable.

---

## How to Read This Directory (Recommended Order)

### 1️⃣ Read Orchestration First (Always)

Files ending with `_orchestration.md` are **entry points**.

They answer:
- *What business event starts this flow?*
- *What must happen, in what order?*
- *Which steps are atomic vs best-effort?*
- *Which sub-processes participate?*

➡️ **Always start with the orchestration file.**

Example:
- `10_finalize_sale_orchestration.md`
- `20_void_sale_orchestration.md`

If you skip orchestration, you will misunderstand the system.

---

### 2️⃣ Read Sub-Processes in Numeric Order

After reading the orchestration, read the sub-processes that follow it numerically.

Example:
```
10_finalize_sale_orchestration.md
11_deviceDraft_process.md
12_finalize_order_process.md
13_stock_deduction_on_finalize_sale.md
```

Each sub-process:
- focuses on **one responsibility**
- does not repeat the entire flow
- is never an entry point on its own

---

### 3️⃣ Treat Higher Prefixes as Separate Stories

Different prefix ranges represent **different workflows**.

Examples:
- `10_*` → Finalize Sale workflow
- `20_*` → Void Sale workflow
- `30_*` → Cash Session lifecycle

These workflows may reference each other, but they remain conceptually separate.

---

## What Processes Must (and Must Not) Do

### Processes MUST:
- orchestrate cross-module behavior
- define sequencing and triggers
- specify idempotency rules
- describe failure and retry behavior
- reference domain and modspec documents

### Processes MUST NOT:
- own long-term state
- redefine domain invariants
- contain UI or API details
- duplicate modspec logic

If logic spans domains and is not documented here, it is a **design smell**.

---

## Relationship to Other BusinessLogic Directories

- **Domain (`BusinessLogic/domain`)**
  - defines *truth*, invariants, ownership
- **Process (`BusinessLogic/process`)**
  - defines *coordination, timing, and orchestration*
- **ModSpec (`BusinessLogic/modSpec`)**
  - defines *how behavior is implemented*

Priority order:
`Domain > Process > ModSpec`

If a ModSpec contradicts a Process, the Process wins.  
If a Process contradicts a Domain, the Domain wins.

---

## Archived Processes

The `_archived` folders preserve:
- superseded workflows
- earlier thinking
- experimental designs

They exist for traceability and academic reflection.

They are **not authoritative**.

---

## Final Rule

> **If someone has to “piece together” a workflow by reading multiple modspecs, the orchestration process is missing or unclear.**

Processes exist so behavior is:
- explicit
- readable
- defensible
- teachable

This directory is how Modula prevents hidden business logic.