# IdentityAccess + HR + OrgAccount Overhaul Tracking

This document tracks the progress of the **Identity, Authorization, HR, and OrgAccount overhaul**.

The goal of this overhaul is to rebuild the people/work/business foundation of Modula using the proper layer order:

Stories → Domains → Contracts → Processes → ModSpec

This tracker is the source of truth for **where we are in the overhaul**.

This overhaul and the KnowledgeBase itself are a direct response to earlier backlash from poor design and coordination.
The KB exists to preserve alignment, prevent hidden assumptions, and keep the product vision consistent as the system grows.

Note: IdentityAccess, HR, and OrgAccount are batched into one overhaul because they depend on each other heavily and must remain consistent across layers.

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
| Phase 2.5 | Contracts (Edge Cases + UX)    | ✅ Completed |
| Phase 2.6 | OrgAccount Alignment           | ✅ Completed |
| Phase 3   | Process Definition             | ✅ Completed |
| Phase 4   | ModSpec Revamp                 | ✅ Completed (March baseline) |

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

# Phase 2.5 — Contracts (Edge Cases + UX) (Completed)

Several critical edge-case documents were produced and normalized:

Edge case contracts:
- `BusinessLogic/3_contract/10_edgecases/identity_hr_edge_case_sweep_patched.md`
- `BusinessLogic/3_contract/10_edgecases/identity_hr_pos_boundary_edge_cases_patched.md`

UX specifications:
- `BusinessLogic/3_contract/20_ux_specs/identity_tenant_workflow_ux_spec.md`

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
- `branch_domain.md` created under `BusinessLogic/2_domain/20_OrgAccount/branch_domain.md`

At this point, the **organizational foundation is aligned** with the new people/work model.

---

# Phase 3 — Process Definition (Completed)

Canonical, cross-domain orchestration for March MVP:

- Identity activation + recovery + context selection:
  - `BusinessLogic/4_process/20_IdentityAccess/10_identity_activation_recovery_orchestration.md`
- Tenant membership administration (grant / role change / revoke):
  - `BusinessLogic/4_process/20_OrgAccount/10_tenant_membership_administration_process.md`
- Staff provisioning (owner/admin provisioned onboarding):
  - `BusinessLogic/4_process/10_WorkForce/05_staff_provisioning_orchestration.md`
- Work lifecycle processes:
  - `BusinessLogic/4_process/10_WorkForce/10_work_start_end_orchestration.md`
  - `BusinessLogic/4_process/10_WorkForce/20_work_end_orchestrastion.md`
  - `BusinessLogic/4_process/10_WorkForce/30_shift_vs_attendance_evaluation.md`
  - `BusinessLogic/4_process/10_WorkForce/40_attendance_report.md`

---

# Phase 4 — ModSpec Revamp (Completed for March Baseline)

Aligned modspecs for March MVP foundations:

IdentityAccess:
- `BusinessLogic/5_modSpec/10_IdentityAccess/authentication_module.md`
- `BusinessLogic/5_modSpec/10_IdentityAccess/accessControl_module.md`

OrgAccount:
- `BusinessLogic/5_modSpec/20_OrgAccount/tenant_module.md`
- `BusinessLogic/5_modSpec/20_OrgAccount/branch_module.md`

HR:
- `BusinessLogic/5_modSpec/30_HR/staffManagement_module.md`
- `BusinessLogic/5_modSpec/30_HR/attendance_module.md`

Note:
- Shift and Work Review are defined at the domain/process level, but do not need dedicated modspecs until they become implementation targets.

---

# Locked Decisions Snapshot (March)

- One identity may belong to multiple tenants (multi-tenant SaaS).
- Staff onboarding is owner/admin provisioned, while credentials remain user-owned (OTP/self-service).
- Branch access is explicit via assignments; no implicit “admins can access all branches”.
- Attendance location verification is evidence-first (record MATCH/MISMATCH/UNKNOWN; do not block solely due to GPS).
- RBAC must be extendable: new roles are added as `role_key` + policy mapping (permission-first), without rewriting HR.
- “Owner” is governance and must not be overloaded into the same mechanism as operational authorization roles.

---

# Current Status (2026-02-08)

We now have consistent, implementable design across:
- Stories
- Domains
- Contracts
- Processes
- ModSpecs (for March baseline)

Context reminder:
- Modules under `POSOperation` are stabilized (but may be patched as new discoveries appear).
- IdentityAccess + HR + OrgAccount foundation is considered **closed for March baseline**; remaining work should be hygiene and implementation, not inventing new rules.

---

# Remaining Overhauls (Not Started)

- Reporting
- PlatformSystem

---

# Guiding Reminder

This overhaul is intentionally slow and layered.

The goal is not speed.  
The goal is a foundation that will not collapse as Modula grows into a SaaS platform.
