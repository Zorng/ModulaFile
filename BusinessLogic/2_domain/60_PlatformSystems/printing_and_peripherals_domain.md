# Printing & Peripherals Domain Model — Modula POS

## Domain Name
Printing & Peripherals (Operational Effects)

## Domain Type
Platform / Supporting Domain (Hardware Effects; Not Business Truth)

## Domain Group
60_PlatformSystems

## Status
Draft (March pilot scope: receipt printer + kitchen sticker printer)

---

## Purpose

This domain defines how Modula models **hardware outputs** as *operational effects*:
- receipt printing (front counter)
- kitchen ticket/sticker printing (back of house)

Printing helps operations, but it must **never** be treated as business truth.
If printing fails, the sale still must remain finalized and auditable.

This domain exists so modules can answer consistently:
- "What does it mean to print a receipt?"
- "What is a kitchen sticker print, and what data does it use?"
- "How do we avoid duplicates under retries without blocking operations?"

---

## Boundary & Ownership

### What This Domain Owns

- Vocabulary for printing roles and print intents.
- The invariants that separate:
  - **business truth** (sale/order/receipt snapshots)
  - from **operational effects** (printing)
- The minimal "printer assignment" concept needed for pilot:
  - receipt printer vs kitchen sticker printer.

### What This Domain Does NOT Own

- Sale truth (totals, finalization) — owned by Sale/Finalize processes.
- Receipt snapshot content — owned by Receipt (derived from finalized sale snapshot).
- Kitchen order truth — owned by Order/Fulfillment (derived from finalized sale snapshot).
- Printer drivers, pairing, OS-level configuration (implementation concern).
- Terminal identity model (explicitly deferred).

---

## Core Concepts (Ubiquitous Language)

### Printer Role
A stable role that answers "what is this printer used for?".

March baseline roles:
- `RECEIPT_PRINTER`
- `KITCHEN_STICKER_PRINTER`

### Print Document Kind
What type of document is being printed.

- `RECEIPT` (derived from immutable receipt snapshot)
- `KITCHEN_STICKER` (derived from immutable order snapshot)

### Print Intent (Operational Effect)
A request to print a specific document kind for a specific business anchor.

Key attributes (conceptual):
- `tenant_id`
- `branch_id`
- `print_kind` (`RECEIPT` | `KITCHEN_STICKER`)
- `source_anchor`:
  - receipt prints are anchored to `(branch_id, sale_id)` via `receipt_id`
  - kitchen prints are anchored to `(branch_id, sale_id)` via `order_id`
- `requested_by` (`actor_id` or `SYSTEM`)
- `purpose`:
  - `AUTO_AFTER_FINALIZE` (best-effort)
  - `MANUAL_REPRINT` (explicit user intent)

Important:
- Print intent is **not** a financial or inventory effect.
- It does not change sale status, cash ledger, inventory ledger, or order status.

### Printer Assignment (Pilot-Level)

For the pilot, the system needs a way to route print intents to the correct printer role.

This can be:
- device-local configuration (recommended for pilot),
- or branch-level configuration (future).

The domain does not enforce how assignment is stored; it only requires that the mapping exists.

---

## Invariants (Non-Negotiable)

- **INV-PRN-1 (Printing is not truth):** printing must not mutate business truth.
- **INV-PRN-2 (No rollback on print failure):** a finalized sale must never be rolled back due to printer failures.
- **INV-PRN-3 (Snapshot-based payloads):**
  - receipt prints must use the immutable receipt snapshot,
  - kitchen prints must use the immutable order snapshot.
- **INV-PRN-4 (Retry-safe auto printing):**
  - auto-after-finalize printing must be safe under retries (best-effort idempotency).
- **INV-PRN-5 (Manual reprint is explicit):** manual reprints are allowed and do not require idempotent suppression.

---

## Relationship to Other Domains

- **Sale / Finalize Sale**
  - triggers print intents as best-effort effects after truth commit.
- **Receipt**
  - owns receipt snapshots and rendering data used for receipt printing.
- **Order / Fulfillment**
  - owns order snapshot used for kitchen printing.
- **Offline Sync + Idempotency**
  - offline retries must not cause uncontrolled duplicate auto-prints.
- **Audit**
  - print requests/failures may be recorded as observational evidence (not business state).

---

## Process Reference (Enforcement / Dispatch)

Printing is orchestrated as an operational effect:
- `BusinessLogic/4_process/60_PlatformSystems/55_printing_effects_dispatch_process.md`

