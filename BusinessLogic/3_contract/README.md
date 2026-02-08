# Contracts — Cross-Layer Agreements (Edge Cases + UX Specs)

## Purpose

This directory contains **contracts**: documents that make cross-layer agreements explicit so that:
- Backend, Frontend, and QA share the same expectations
- edge cases are handled consistently (or intentionally degraded)
- UI behavior reflects locked domain/process decisions

Contracts sit between **Domain** and **Process**:

> Stories → Domain → Contract → Process → ModSpec

They are not “nice-to-have” docs. They are how Modula prevents silent assumptions.

---

## What a Contract Is (and Is Not)

### A contract IS
- a stable agreement with explicit expected behavior
- a list of edge cases, UX rules, or boundary constraints
- clear about ownership (which domain/process is responsible)
- explicit about March vs later delivery

### A contract is NOT
- domain truth/invariants (belongs in `BusinessLogic/2_domain/`)
- orchestration sequencing and idempotency rules (belongs in `BusinessLogic/4_process/`)
- implementation details (belongs in `BusinessLogic/5_modSpec/`)

---

## Directory Structure

- `10_edgecases/`
  - catalogs “weird but real” scenarios + expected behavior
  - includes boundary contracts (e.g., Identity/HR ↔ POS safety)
- `20_ux_specs/`
  - UI rules and states required to honor domain/process decisions
- `_archive/`
  - older versions kept for traceability (not authoritative)

---

## Document UX Standard (Consistency Rules)

All contract docs should follow the same UX structure:

1) Start with a `## Metadata` section (bullet list)
- Contract Type
- Scope
- Primary Audience
- Owner(s)
- Last Updated (YYYY-MM-DD)
- Delivery Level (March vs Later)
- Related docs (optional but recommended)

2) Use stable IDs
- Edge cases use IDs like `EC-...` (example: `EC-IDH-01`)
- UX rules may use IDs like `UX-...` if the document grows large

3) Reference canonical paths
- Use `BusinessLogic/...` paths in references.
- Do not use legacy paths like `contracts/...` or `domain/...`.

---

## Templates (Copy Pattern)

### Edge Case Contract Template
```md
# Edge Case Contract — <Scope>

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: <what boundary / context>
- **Primary Audience**: Backend, Frontend, QA
- **Owner(s)**: <domain/process owners>
- **Last Updated**: YYYY-MM-DD
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred

---

## Purpose

## Scope

## Definitions / Legend

## Edge Case Catalog

### EC-XXX-01 — <Title>
- **Scenario**:
- **Trigger**:
- **Expected Behavior**:
- **Owner**:
- **March**: Yes/Partial/No
- **Signal**: (optional)

---

## Summary
```

### UX Spec Template
```md
# UX Spec — <Scope>

## Metadata
- **Contract Type**: UX Spec
- **Scope**: <which screens/flows>
- **Primary Audience**: Frontend, Backend, QA
- **Owner(s)**: <domain/process owners>
- **Last Updated**: YYYY-MM-DD
- **Delivery Level**:
  - **March**: baseline behavior
  - **Later**: explicitly deferred
- **Related Contracts**:
  - `BusinessLogic/3_contract/10_edgecases/<...>.md`

---

## Purpose

## UX Principles

## States and Flows

## Error/Degradation Rules

## Non-Goals

## Summary
```

