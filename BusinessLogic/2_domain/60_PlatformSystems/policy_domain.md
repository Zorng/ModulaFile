# Policy Domain Model — Modula POS

## Domain Name
Policy (Tax & Currency Configuration)

## Domain Type
Supporting Domain (Configuration)

## Domain Group
60_PlatformSystems

## Status
Draft (Patched: branch-scoped; Capstone 1 includes Tax & Currency only)

---

## Purpose

The Policy domain provides a single, explicit source of truth for **configurable sale computation inputs**:
- VAT enablement + VAT rate
- FX rate (KHR per USD)
- KHR cash rounding rules

It exists to keep money behavior:
- consistent across modules
- explainable to operators
- stable for historical records (via snapshots stored at finalization time)

Policy is configuration, not workflow.

---

## Domain Boundary & Ownership

### What This Domain Owns

- The **policy catalog** (a fixed set of keys shipped by Modula).
- The **branch policy record** for each `(tenant_id, branch_id)`.
- Validation rules and invariants for policy values.
- The rule that policy updates are **atomic** and **forward-applying** (do not rewrite past truth).

### What This Domain Does NOT Own

- Sale truth (totals, lifecycle, receipts) — owned by Sale/Order.
- Cash session controls and approval workflows — owned by POSOperation + Access Control.
- Inventory deduction enablement — owned by Menu composition + Inventory.
- Attendance/shift enforcement rules — owned by HR + processes and contract flags/signals.
- Authorization (who may update) — owned by Access Control.
- Audit log storage — owned by Audit.

Policy provides values; it does not decide who can change them.

---

## Core Concepts (Ubiquitous Language)

### Branch Policy
A configuration record resolved by:
- `tenant_id`
- `branch_id`

It contains a fixed set of fields (keys). It is updated via partial updates.

### Policy Key (Fixed Catalog)
Policy keys are stable identifiers that other modules depend on:
- `saleVatEnabled`
- `saleVatRatePercent`
- `saleFxRateKhrPerUsd`
- `saleKhrRoundingEnabled`
- `saleKhrRoundingMode`
- `saleKhrRoundingGranularity`

Keys are **not creatable** at runtime.

### Policy Snapshot (Downstream Requirement)
Policy values used to compute a finalized sale must be **snapshotted by the Sale domain** at finalize time.

This ensures:
- past receipts remain consistent
- reporting aggregates from stored finalized values

Policy itself does not own those snapshots; it only defines the source values.

---

## Invariants (Non-Negotiable)

- **INV-POL-1 (Branch-scoped):** Policy resolution requires `(tenant_id, branch_id)`.
- **INV-POL-2 (Fixed catalog):** Policy keys are fixed; no tenant-defined new keys in Capstone 1.
- **INV-POL-3 (Default existence):** For every active branch, a policy record exists (defaults apply until updated).
- **INV-POL-4 (Atomic update):** A policy update either fully applies or does not apply at all.
- **INV-POL-5 (Validation):**
  - VAT rate must be in `[0, 100]`.
  - FX rate must be `> 0`.
  - KHR rounding mode must be one of `NEAREST | UP | DOWN`.
  - KHR rounding granularity must be one of `100 | 1000`.
- **INV-POL-6 (Forward-applying):** A policy update must not mutate previously finalized sales/receipts.

---

## Relationship to Other Domains

- **Sale/Order** consumes policy values at finalize time and stores a snapshot for historical correctness.
- **Receipt** renders from the finalized sale snapshot; it must never re-resolve live policy.
- **Reporting** aggregates from stored finalized values; it may display current policy as read-only context, but must not recompute history.
- **Access Control** enforces who can read/update policy values.
- **Audit** records policy changes for accountability.

---

## Out of Scope (For This Domain Doc)

- Attendance, cash session, and inventory configuration toggles (intentionally removed from Policy in Capstone 1).
- Service charges, multi-VAT rules, and time-based pricing policies.
- Policy inheritance/copy workflows across branches.
- Policy versioning/rollback UX.

---

## Summary

Policy is a small, branch-scoped configuration domain that ensures VAT/FX/KHR rounding behavior is:
- consistent in sale computation,
- auditable in changes,
- and historically safe through downstream snapshots.

