# Reporting Domain Model — Modula POS

## Domain Name
Reporting (Management Insight)

## Domain Type
Supporting Domain (Read-Only Aggregation)

## Status
Draft (Baseline: management reporting reads; does not own operational truths)

---

## Purpose

The Reporting domain provides **management insight** by aggregating and presenting
business truth that is owned elsewhere (Sale/Order, Cash Session, Inventory, HR).

It exists to answer questions like:
- "How did we perform this day/week?"
- "What changed and why?"
- "What looks unusual and needs attention?"
- "How much did we spend on restocking this month?"

Reporting is a decision-support layer.
It must never become a second system of record.

---

## Domain Boundary & Ownership

### What This Domain Owns

- The definition of **reporting rules** that preserve trust:
  - finalized vs provisional presentation
  - no silent mutation of history
  - deterministic aggregation rules
- Read-only **report views** and **summaries** (derived artifacts).
- Reporting scope concepts (time window, tenant/branch scope) and consistent aggregation semantics.

### What This Domain Does NOT Own

- Sales truth (final amounts, payment breakdown, status transitions) — owned by Sale/Order.
- Cash drawer accountability and reconciliation artifacts (X/Z) — owned by Cash Session.
- Stock truth and movements — owned by Inventory.
- Attendance truth and interpreted time-respecting metrics — owned by Attendance / Work Review.
- Policies as mutable sources of recalculation for past records.

Reporting may **consume** the above, but must not redefine them.

---

## Core Concepts (Ubiquitous Language)

### Reporting Scope

A report is always queried with an explicit scope:
- `tenant scope` (who owns the business)
- `branch scope` (where operations happened)
- `time window` (day/week/month/custom)

Branch scope may be:
- a single branch (`branch_id`)
- tenant-wide (`ALL_BRANCHES`) meaning aggregate across **all branches** in the tenant

Tenant-wide scope is only allowed when Access Control can prove the actor has access to **all branches**
(via explicit branch assignments per branch). Otherwise, the actor must select a specific allowed branch.

Regardless of scope, the report must remain explainable: the reader should always know what was included.

### Finalized vs Provisional

Reporting must distinguish between:
- **Finalized facts**: stable, committed outcomes (e.g., finalized sales totals; closed cash session Z artifacts)
- **Provisional facts**: outcomes pending approval or completion (e.g., `VOID_PENDING`)

Provisional states must be **visible**, not silently excluded.

### Operational Artifacts vs Management Reports

Some "reports" are operational artifacts, not analytics:
- Cash Session X (in-session snapshot)
- Cash Session Z (close summary)

These artifacts remain owned by the Cash Session domain.
Reporting may link to them or summarize across them, but must not reinterpret their meaning.

---

## Reporting Rules (Non-Negotiable)

1) **Read-only**
   - Reporting must not mutate operational truth.
2) **No retroactive recompute using current policy**
   - Historical totals must be aggregated from stored finalized values, not recalculated using today's policy/settings.
3) **Explainability**
   - A number must be traceable to underlying recorded facts (sales, movements, adjustments, attendance records, review summaries).
4) **Determinism**
   - Given the same underlying facts and the same scope, the report output must be stable.
5) **Auditability**
   - Access to sensitive reports must be attributable (who viewed what, when).

---

## Inputs (Owned Elsewhere)

Reporting consumes read models from:
- **Sale/Order**: finalized sales snapshots, payment breakdown, void/refund states
- **Cash Session**: session metadata, movement ledger summaries, X/Z artifacts
- **Inventory**: on-hand projections, movement history, and restock batch metadata (including optional purchase cost)
- **Work Review**: interpreted planned-vs-actual outcomes and summaries (lateness, absence, overtime)
- **OrgAccount (Tenant/Branch)**: branch identity, branch status (active/frozen), timezone context

Reporting does not require these sources to change ownership.
It only requires stable, permissioned read access.

---

## Relationship to Other Domains

- Reporting depends on **immutable or append-only histories** to preserve trust.
- Reporting is downstream of **operational truth**:
  - if truth changes via explicit workflows (e.g., a void is approved and executed within allowed rules),
    reporting reflects the new state without hiding the transition.
- Reporting must respect **frozen branch** semantics:
  - frozen blocks operational writes,
  - but historical reporting remains available (read-only).

---

## Out of Scope (For This Domain Doc)

- Accounting settlement and tax filing workflows
- Forecasting, anomaly detection, and "AI insights"
- Automated scheduled exports and notifications
- Inventory valuation methods (FIFO/FEFO/avg cost) and full COGS accounting

These may be added later as explicit, separately designed domains/processes.

---

## Summary

Reporting exists to produce **trustworthy management insight** by aggregating facts from other domains,
while preserving:
- ownership boundaries,
- explainability,
- and audit safety.

It is a read-only domain that must never drift into becoming "another source of truth".
