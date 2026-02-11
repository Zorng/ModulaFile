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

This directory is the **spine of cross-module behavior**.

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
10_finalize_sale_orch.md   ← start here
11_deviceDraft_process.md
12_finalizeOrder_process.md
13_stock_deduction_on_finalize_sale_process.md
```

This means:
- `10_*` is the **orchestration**
- `11–19_*` are **sub-processes that participate in finalize sale**

---

## High-Level Structure
```
BusinessLogic/4_process
├── README.md   ← you are here
│
├── 10_WorkForce
│   ├── 05_staff_provisioning_orchestration.md
│   ├── 10_work_start_end_orchestration.md
│   ├── 20_work_end_orchestrastion.md
│   ├── 30_shift_vs_attendance_evaluation.md
│   └── 40_attendance_report.md
│
├── 20_IdentityAccess
│   └── 10_identity_activation_recovery_orchestration.md
│
├── 20_OrgAccount
│   ├── 05_tenant_provisioning_orchestration.md
│   └── 10_tenant_membership_administration_process.md
│
├── 30_POSOperation
│   ├── 05_khqr_payment_confirmation_process.md
│   ├── 06_place_order_open_ticket_process.md
│   ├── 07_add_items_to_open_ticket_process.md
│   ├── 08_checkout_open_ticket_process.md
│   ├── 09_cancel_unpaid_ticket_process.md
│   ├── 10_finalize_sale_orch.md
│   ├── 11_deviceDraft_process.md
│   ├── 12_finalizeOrder_process.md
│   ├── 13_stock_deduction_on_finalize_sale_process.md
│
│   ├── 20_void_sale_orch.md
│   ├── 21_voidOrder_process.md
│   ├── 22_void_sale_inventory_reversal_process.md
│   └── 23_void_sale_cash_reversal_process.md
│
├── 50_Reporting
│   ├── 10_sales_reporting_process.md
│   └── 20_restock_spend_reporting_process.md
│
├── 60_PlatformSystems
│   ├── 10_update_branch_policy_process.md
│   ├── 20_resolve_branch_policy_process.md
│   ├── 30_operational_notification_emission_process.md
│   ├── 40_audit_event_recording_process.md
│   ├── 50_audit_log_query_process.md
│   ├── 55_printing_effects_dispatch_process.md
│   ├── 60_offline_operation_queue_process.md
│   ├── 70_offline_sync_replay_process.md
│   └── 80_idempotency_gate_process.md
│   ├── 85_fair_use_limit_gate_process.md
│   ├── 90_first_branch_activation_orchestration.md
│   ├── 91_enable_branch_module_or_seats_proration_process.md
│   ├── 92_disable_branch_module_downgrade_pending_process.md
│   └── 93_subscription_renewal_grace_and_freeze_orchestration.md
│   └── 94_additional_branch_activation_orchestration.md
│   └── 95_payment_collection_and_confirmation_orchestration.md
│   └── 96_webhook_ingestion_and_dispatch_process.md
│   └── 97_job_scheduler_tick_and_execution_process.md
```

This structure is designed to scale without becoming unreadable.

---

## How to Read This Directory (Recommended Order)

### 1️⃣ Read Orchestration First (Always)

Files with orchestration intent (typically `10_*` and named `*_orch.md` or `*_orchestration.md`) are **entry points**.

They answer:
- *What business event starts this flow?*
- *What must happen, in what order?*
- *Which steps are atomic vs best-effort?*
- *Which sub-processes participate?*

➡️ **Always start with the orchestration file.**

Example:
- `30_POSOperation/10_finalize_sale_orch.md`
- `30_POSOperation/20_void_sale_orch.md`

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

- **Domain (`BusinessLogic/2_domain/`)**
  - defines *truth*, invariants, ownership
- **Contract (`BusinessLogic/3_contract/`)**
  - defines *cross-layer agreements* (edge cases, required behavioral states)
- **Process (`BusinessLogic/4_process/`)**
  - defines *coordination, timing, and orchestration*
- **ModSpec (`BusinessLogic/5_modSpec/`)**
  - defines *how behavior is implemented*

Priority order:
`Domain > Contract > Process > ModSpec`

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
