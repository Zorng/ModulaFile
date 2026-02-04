# Consistency Sweep Report — Identity & Authorization + HR Domains

## What you asked
You suspected **Authentication** and **Access Control** domains were “remnants” of the old world and might not align with the new Identity+HR overhaul.  
You also wanted a consistency sweep across the two grouped zip sets.

## Bottom line
- The **HR domains are largely consistent** with the new overhaul (Shift/Attendance/Work Review/Staff Profile).
- The **Identity & Authorization set needed small but important alignment patches**:
  - terminology (`Auth` vs `Authentication`)
  - ownership references (Membership owned by Tenant Membership, not “Tenant/Staff”)
  - tenant provisioning invariant (should support controlled provisioning flow + future self-serve)
  - explicit tenant/branch selection UX contract after login

## Concrete fixes made
### 1) Authentication domain
- Renamed framing from “Auth” to **Authentication**.
- Standardized term to **AuthenticationAccount** (kept meaning identical).
- Clarified that tenant + branch selection happens **after login** and is stored in session scope, but not owned by Authentication.
- Corrected ownership reference: **Membership is owned by Tenant Membership**.

### 2) Tenant domain
- Patched INV-T3: from “system-provisioned only” → **controlled provisioning flow** (admin now, self-serve later).
- Patched INV-T4: from “must always have a branch” → “must have an ACTIVE branch **before POS operations run**.”
- Added explicit relationship note: access to tenant is via **Tenant Membership**.

### 3) Access Control domain
- Aligned ownership statement: membership facts owned by **Tenant Membership + Staff Profile**.
- Clarified status check language (ACTIVE vs DISABLED/REMOVED).
- Aligned actor id naming to `authentication_account_id`.

## Remaining known “future work” (intentionally not addressed)
- Staff licensing / entitlements (capability system) is future work and should not be pulled into these domains yet.
- Notification orchestration is future work.

## Outputs
- authentication_domain_consistency_patched.md
- tenant_domain_consistency_patched.md
- accessControl_domain_consistency_patched.md
