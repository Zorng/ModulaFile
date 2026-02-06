# Org Account Story Coverage Map (Tenant + Branch)

## Purpose
This map ensures we **do not forget** the everyday operational stories around:
- updating tenant info (business profile)
- updating branch info (address/location/contact)
- configuring branch location (for GPS attendance checks)

It complements the Identity+HR overhaul by covering the **Org Account maintenance** layer.

---

## Inputs
- Tenant modspec: `BusinessLogic/5_modSpec/tenant_module_patched_v2.md` (found)
- Branch modspec: `BusinessLogic/5_modSpec/20_OrgAccount/branch_module.md` (found)

> Note: UC numbering in existing modspecs may not cover all maintenance needs. This map prioritizes **real operational goals** for March.

---

## Folder Placement (recommended)
- Tenant + branch profile stories: `BusinessLogic/1_stories/handling_branding/`
- Branch workplace location (GPS) stories: `BusinessLogic/1_stories/handling_staff&attendance/`
- Later: derive/patch domains in:
  - Tenant: `BusinessLogic/2_domain/OrgAccount/tenant_domain_consistency_patched.md`
  - Branch: `BusinessLogic/2_domain/OrgAccount/branch_domain.md`
- Ensure modspecs in `BusinessLogic/5_modSpec/20_OrgAccount/` (and the patched tenant spec) align

---

## Story Set A — Tenant (Business Profile)

### A1 — Updating the Business Profile (Tenant Settings)
**Goal**: The owner updates the tenant’s business-facing information (name, receipt header info, contact info, branding).  
**Primary Actor**: Tenant Owner  
**Involves**: Tenant module  
**Notes**:
- Must not break historical records (receipts/audits use snapshots).
- Does **not** include billing/plan changes (future SaaS capability).

**Likely modspec touchpoints**: UC(s) present: N/A (not detected)  
**Outputs**:
- Tenant profile updated
- Changes visible in future receipts and UI

---

## Story Set B — Branch (Branch Profile)

### B1 — Updating a Branch Profile
**Goal**: Manager/owner edits branch name, address, phone, notes.  
**Primary Actor**: Owner, Manager  
**Involves**: Branch module  
**Notes**:
- Branch identity is stable (branch_id); name/address changes are allowed.
- Keep audit log for changes.

**Likely modspec touchpoints**: UC(s) present: [1, 2, 3, 4, 5]  

---

### B2 — Activating/Deactivating a Branch (Operational-safe)
**Goal**: Owner can deactivate a branch (e.g., temporary closure).  
**Primary Actor**: Owner  
**Involves**: Branch module + POS ops guards  
**March Rules**:
- Block deactivation if there is:
  - active cash session
  - active attendance
  - ongoing sale session (optional)

**This story ties to contracts**:
- Add to `BusinessLogic/3_contract/10_edgecases/` once finalized.

---

## Story Set C — Branch Location (GPS Attendance Support)

### C1 — Configuring Branch Location for Attendance Confirmation
**Goal**: Owner configures the branch’s “workplace location” used to compare against staff check-in GPS.  
**Primary Actor**: Owner  
**Involves**: Branch module + Attendance domain  
**Notes**:
- Only meaningful when GPS attendance capability is enabled.
- Fields typically include:
  - latitude/longitude
  - allowed radius (meters)
  - location label (optional)

**Related docs**:
- Attendance domain (capability-gated)
- UX spec for check-in location confirmation

---

### C2 — Updating Branch Location Without Breaking Existing Attendance
**Goal**: Owner updates the branch location; past attendances remain correct.  
**Rule**:
- Attendance records should store **captured location + evaluated result** at check-in time.
- Updating branch location affects only future check-ins.

---

## Cross-cutting Edge Cases to Add (small)
These should become entries in the edge-case contracts:

1) **Editing tenant/branch info must not mutate historical outputs**
   - receipts/audits should remain verifiable (use snapshots)

2) **Deactivating a branch with active responsibilities**
   - block until cash session closed and staff ended work

3) **Location confirmation capability off**
   - hide/disable branch GPS settings and ignore GPS checks

---

## Next Actions (recommended order)
1) Write stories A1, B1, C1 (March essentials)
2) Patch branch modspec to explicitly include branch profile + GPS config
3) Patch tenant modspec to include business profile update story alignment
4) Add the 3 cross-cutting edge cases to the edge-case contract docs

---

_End of Org Account story coverage map_
