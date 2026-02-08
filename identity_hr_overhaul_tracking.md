# Identity + HR Overhaul Tracking

This document tracks the progress of the **Identity, Authorization, HR, and OrgAccount overhaul**.

The goal of this overhaul is to rebuild the people/work/business foundation of Modula using the proper layer order:

Stories → Domains → Contracts → Processes → ModSpec

This tracker is the source of truth for **where we are in the overhaul**.

This overhaul and the KnowledgeBase itself are a direct response to earlier backlash from poor design and coordination.
The KB exists to preserve alignment, prevent hidden assumptions, and keep the product vision consistent as the system grows.

---

# Why This Overhaul Exists

Earlier versions of the system mixed:
- identity and authorization,
- staff and tenant concepts,
- operational workflows and account structures.

As the POS matured, the boundaries became unclear and risky for future SaaS capabilities.

This overhaul ensures the system can support:
- multi-tenant SaaS growth
- modular capabilities (“pay for what you use”)
- staff workflows and attendance
- clean authorization boundaries
- consistent cross-module orchestration

---

# Phase Overview

| Phase     | Description                    | Status      |
| --------- | ------------------------------ | ----------- |
| Phase 1   | Anchor Stories (Human reality) | ✅ Completed |
| Phase 2   | Domain Derivation              | ✅ Completed |
| Phase 2.5 | Edge Case Sweep                | ✅ Completed |
| Phase 2.6 | OrgAccount Alignment           | ✅ Completed |
| Phase 3   | Process Definition             | ⏳ Next      |
| Phase 4   | ModSpec Revamp                 | 🔜 Later     |

---

# Phase 1 — Anchor Stories (Completed)

Six anchor stories were created and later rewritten to align with the narrative style of the repository.

They describe the **human lifecycle of work**:

A1 — Setting up the business team quickly  
A2 — Planning who is expected to work and where  
A3 — Starting work at a branch  
A4 — Ending work and leaving the system clean  
A5 — Preventing unauthorized or excessive work  
A6 — Reviewing attendance and time respecting  

These stories define the **real-world foundation** for Identity + HR.

---

# Phase 2 — Domain Derivation (Completed)

From the stories, the following domains were derived:

Identity & Authorization:
1. Authentication (global identity)
2. Access Control (authorization gate)

HR & Workforce:
3. Tenant Membership
4. Staff Profile & Assignment
5. Shift (planned work)
6. Attendance (actual work)
7. Work Review (interpretation of work history)

These domains establish the conceptual model of **people and work** in Modula.

---

# Phase 2.5 — Edge Case Sweep (Completed)

Several critical edge-case documents were produced and normalized:

Edge case contracts:
- identity_hr_edge_case_sweep.md
- identity_hr_pos_boundary_edge_cases.md

UX specifications:
- identity_tenant_workflow_ux_spec.md

These documents ensure:
- cross-module consistency
- safe multi-tenant behavior
- clear login and tenant selection flows

This phase strengthened the reliability of the domains before moving to processes.

---

# Phase 2.6 — OrgAccount Alignment (Completed)

During the overhaul, a major gap was discovered:

Branch and Tenant concepts were not fully aligned with the new Identity + HR model.

OrgAccount was therefore **added to the overhaul bus**.

New/updated work:
- Stories written for:
  - Updating tenant profile (branding)
  - Updating branch profile
  - Configuring branch location for attendance
- Branch domain reconstructed from:
  - stories
  - attendance requirements
  - authorization scope
  - receipt usage
  - existing branch modspec

Result:
- `branch_domain.md` created under `BusinessLogic/2_domain/`

At this point, the **organizational foundation is aligned** with the new people/work model.

---

# Current Status

We now have a consistent foundation across:

Stories  
Domains  
Edge cases  
OrgAccount alignment  

The system is ready to move to the next layer.

Context reminder:
- Modules under `POSOperation` are stabilized (with potential patches as new discoveries appear).
- We are actively overhauling Identity & Authorization, HR, and OrgAccount to make Modula a real SaaS.
- Next after this is System/Platform.

---

# Phase 3 — Process Definition (Next)

We now begin defining cross-domain orchestration for the work lifecycle.

Planned process documents:

1. Work Start / Check-in orchestration
2. Work End / Check-out orchestration
3. Shift vs Attendance evaluation
4. Authorization + capacity gating during work start

These processes will connect:
Authentication → Membership → Branch → Shift → Attendance → Access Control.

This phase is the equivalent of **Finalize Sale orchestration** for POS.

---

# Phase 4 — ModSpec Revamp (Future)

After processes are defined, the following modules will be rewritten:

Identity & Authorization:
- authentication_module
- accessControl_module

HR:
- staffManagement_module
- staffAttendance_module

OrgAccount:
- tenant_module
- branch_module

This will complete the overhaul.

---

# Guiding Reminder

This overhaul is intentionally slow and layered.

The goal is not speed.  
The goal is a foundation that will not collapse as Modula grows into a SaaS platform.
