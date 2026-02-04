# Identity & HR Overhaul — Tracking Document

**Project:** Modula POS  
**Area:** Identity & Access + HR  
**Goal:** Rebuild Identity & HR with clean layering and story-first methodology  
**Status:** In Progress (Phase 1)

---

## 1. Why This Document Exists

This document exists to:
- keep the overhaul **trackable and non-chaotic**
- prevent design drift or circular debates
- make progress visible over time
- allow pausing and resuming work confidently

This is a **process artifact**, not a design artifact.

---

## 2. Overhaul Principles (Locked)

These principles are agreed and must not be re-debated during this overhaul.

1. **Stories are the source of truth**
   - Stories describe real human situations.
   - They do not mirror modules or implementation details.

2. **Artifacts are derived in order**
   ```
   Stories → Domain → Process → ModSpec
   ```

3. **No forced 1:1 mapping**
   - Stories ≠ Modules
   - Modules may serve multiple stories.
   - Stories may span multiple modules.

4. **Plan vs Reality separation**
   - Shift = planned work
   - Attendance = actual work
   - Neither replaces the other.

5. **Access Control is a gate, not business logic**
   - Centralized
   - Deterministic
   - Cross-cutting by design

6. **Shipping by March is a constraint**
   - Overhaul must remain minimal and intentional.
   - No payroll, no optimization, no enterprise HR.

---

## 3. Overhaul Scope

### In Scope
- Authentication (phone + password)
- Tenant membership & roles
- Branch-scoped access
- Staff profile & lifecycle
- Shift (planned schedule)
- Attendance (check-in / check-out)
- Foundations for staff performance & behavior reports

### Out of Scope
- Billing engine / subscription payments
- Marketing notifications
- Payroll and labor law enforcement
- Enterprise SSO
- Advanced scheduling optimization

---

## 4. Current Phase

### 🟨 Phase 1 — Anchor Stories (Current)

**Objective:**  
Describe core human situations that define Identity & HR behavior.

**Rules:**
- Essay-style stories
- No implementation detail
- No technical vocabulary
- Focus on intent, expectation, and reality

---

## 5. Anchor Stories (Planned)

| ID | Anchor Story Title | Status |
|---|---|---|
| A1 | Setting Up the Business Team Quickly | ⬜ |
| A2 | Planning Who Is Expected to Work and Where | ⬜ |
| A3 | Starting Work at a Branch | ⬜ |
| A4 | Ending Work and Leaving the System Clean | ⬜ |
| A5 | Preventing Unauthorized or Excessive Work | ⬜ |
| A6 | Reviewing Attendance and Time Respecting | ⬜ |

Legend:
- ⬜ Not started
- 🟨 Drafted
- ✅ Final

---

## 6. Upcoming Phases (Locked Sequence)

### Phase 2 — Domain Derivation
From anchor stories, derive domain documents:
- Authentication
- Tenant Membership
- Staff Profile & Assignment
- Shift (Planned Work)
- Attendance (Actual Work)
- Access Control
- Staff Licensing (Concurrency)

---

### Phase 3 — Process Definition
Define cross-module flows:
- Start Work orchestration
- End Work orchestration
- Shift vs Attendance evaluation
- Authorization & capacity gates

---

### Phase 4 — ModSpec Writing
Produce implementation-facing specs:
- staffManagement_module
- shift_module
- attendance_module
- accessControl_module (minor alignment)
- authentication_module (minor alignment)

---

## 7. Explicit Pauses & Checkpoints

The overhaul **must pause** at these checkpoints:
- After all anchor stories are finalized
- After domains are derived
- Before writing any modspec

No phase skipping is allowed.

---

## 8. Known Risks & Guardrails

### Risks
- Reintroducing module-first thinking
- Writing stories that explain system behavior
- Over-expanding HR scope
- Mixing scheduling with payroll logic

### Guardrails
- Always ask: “Is this plan or reality?”
- Always ask: “Is this human expectation or system mechanism?”
- If unclear, stop and write a story instead of a spec.

---

## 9. Definition of Success

This overhaul is successful if:
- Stories read naturally to non-technical readers
- Domains feel obvious when derived
- Processes explain real-world flows cleanly
- ModSpecs are smaller and less shaky than before
- No major rework is needed before March delivery

---

## 10. Notes & Decisions Log

(Use this section to log key decisions as they happen.)

- [ ] Shift included to support future performance reporting
- [ ] Stories are no longer mirrored to modules
- [ ] Branch treated as supporting noun, not protagonist
- [ ] Attendance is actual, Shift is planned

---

_End of Tracking Document_
