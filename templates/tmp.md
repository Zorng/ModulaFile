# Edge Case Contract Template

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: (single context / cross-context boundary)
- **Primary Audience**: (backend, frontend, QA)
- **Owner(s)**: (domains/processes responsible)
- **Last Updated**: YYYY-MM-DD
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred

---

## Purpose
One paragraph explaining why this edge-case contract exists and what problem it prevents.

---

## Scope
### In-scope
- (bullets)

### Out-of-scope
- (bullets)

---

## Definitions / Legend
- **Owner**: where the decision belongs (domain/process)
- **March**: required for March delivery
- **Later**: intentionally deferred
- **Signal**: audit/log/notification expected (if any)

---

## Boundary Principles (Optional)
Use this section when the document covers a **boundary** between contexts or modules.

1. ...
2. ...

---

## Edge Case Catalog

> **Format rule:** each edge case must include **ID, scenario, trigger, expected behavior, owner, March/Later**.

### EC-XX — Title
- **Scenario**: ...
- **Trigger**: ...
- **Expected Behavior**:
  - ...
- **Signal** (optional):
  - Audit flag / log / UI message / notification
- **Owner**: ...
- **March**: Yes/Partial/No
- **Related** (optional):
  - Domains: ...
  - Processes: ...
  - ModSpecs: ...

---

## Summary
- 2–5 bullets summarizing the highest risk areas and what is locked for March.

_End of template_
