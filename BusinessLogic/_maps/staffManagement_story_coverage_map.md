# Story Coverage Map — Staff Management (People, Branch Access, and Operational Capacity)

**Purpose:**  
Identify human-centered stories for Staff Management behavior and map them to technical use cases, ensuring full coverage without “one story per UC”, while explicitly capturing cross-module stories (authorization, attendance, licensing).

**Scope:**  
- Staff Licensing Domain (Concurrent Staff)  
- Staff Management Module Spec (v3, simplified onboarding)

**Rule:**  
Stories are grouped by **human operational context**, not by CRUD actions or internal mechanics.

---

## 1) Source Artifacts (Authoritative)

- **Domain:** `BusinessLogic/domain/10_IdentityAccess/staff_licensing_domain.md`
- **ModSpec:** `BusinessLogic/modSpec/10_IdentityAccess/staffManagement_module_patched_v3.md`

---

## 2) Operational Contexts (Human Reality)

From reading the domain and modspec, staff setup and access is experienced by humans in **five distinct operational contexts**:

1. **Onboarding Staff Quickly (Owner Provisioned)**
2. **Controlling Where People Can Work (Branch Assignment)**
3. **Stopping Access Immediately (Disable / Revoke)**
4. **Keeping Clean History (Archive / Audit)**
5. **Operating Under Fair Limits (Concurrent Staff Capacity)**

These contexts reflect how cafés actually run: shift-based staffing, frequent changes, and strong need for immediate control.

---

## 3) Story Set (Human Situations)

### Context: Onboarding Staff Quickly (Owner Provisioned)

1. **Creating a Staff Account for Someone New**  
   File: `1_stories/identity_and_access/creating_staff_account.md`  
   Human situation: the owner needs to add a new staff member today, set their initial password once, and let them start work without a complicated invite flow.

2. **Updating Basic Staff Information**  
   File: `1_stories/identity_and_access/updating_staff_information.md`  
   Human situation: names, codes, and labels change; the system should stay accurate without touching credentials.

---

### Context: Controlling Where People Can Work (Branch Assignment)

3. **Assigning a Staff Member to One or More Branches**  
   File: `1_stories/identity_and_access/assigning_staff_to_branches.md`  
   Human situation: staff are only allowed to operate in certain locations; assignments must be explicit and easy to review.

4. **Removing a Staff Member’s Access to a Branch**  
   File: `1_stories/identity_and_access/removing_branch_access.md`  
   Human situation: staffing changes happen; access must be removed cleanly without deleting history.

---

### Context: Stopping Access Immediately (Disable / Revoke)

5. **Disabling a Staff Member Immediately**  
   File: `1_stories/identity_and_access/disabling_staff_immediately.md`  
   Human situation: a staff member should be blocked right now; the owner expects the next action to be denied.

---

### Context: Keeping Clean History (Archive / Audit)

6. **Archiving a Staff Member While Keeping History**  
   File: `1_stories/identity_and_access/archiving_staff.md`  
   Human situation: old staff should disappear from daily lists, but past receipts/reports should still make sense.

7. **Reviewing Staff Changes When Something Goes Wrong**  
   File: `1_stories/identity_and_access/reviewing_staff_audit_trail.md`  
   Human situation: when conflicts occur (“why was this blocked?”), the owner wants a clear trail of staffing and access changes.

---

### Context: Operating Under Fair Limits (Concurrent Staff Capacity)

8. **Being Blocked When Too Many Staff Are Active**  
   File: `1_stories/identity_and_access/concurrent_staff_limit_block.md`  
   Human situation: the café is busy and someone tries to start working, but the system blocks it fairly and clearly.

9. **Freeing Capacity When Someone Finishes Work**  
   File: `1_stories/identity_and_access/freeing_concurrent_capacity.md`  
   Human situation: once someone checks out / ends their shift, the next staff should be able to start without confusion.

---

## 4) Coverage Mapping (Use Case → Story)

### Story 1 — Creating a Staff Account for Someone New
Covers:
- UC-SM1 Create Staff Account (Owner Provisioned)

---

### Story 2 — Updating Basic Staff Information
Covers:
- (implicit) Update StaffProfile fields (display_name, staff_code, PIN setup rules)
- “No credential handling” boundary (staff profile edits do not touch passwords)

---

### Story 3 — Assigning a Staff Member to One or More Branches
Covers:
- UC-SM4 Assign Staff to Branch
- BranchAssignment is explicit for all roles (no admin bypass)

---

### Story 4 — Removing a Staff Member’s Access to a Branch
Covers:
- UC-SM5 Revoke Staff from Branch
- Immediate effect semantics (deny on next request)

---

### Story 5 — Disabling a Staff Member Immediately
Covers:
- UC-SM2 Disable Staff (Immediate Block)
- Immediate effect semantics (deny on next request)

---

### Story 6 — Archiving a Staff Member While Keeping History
Covers:
- UC-SM3 Archive Staff
- “Archive is historical and irreversible” rule (if adopted)

---

### Story 7 — Reviewing Staff Changes When Something Goes Wrong
Covers:
- Audit & observability expectations:
  - STAFF_ACCOUNT_CREATED / DISABLED / ARCHIVED
  - BRANCH_ACCESS_GRANTED / REVOKED
  - STAFF_LIMIT_REACHED

---

### Story 8 — Being Blocked When Too Many Staff Are Active
Covers:
- Staff Licensing enforcement points (check-in / operating session / cash session open)
- Failure mode: STAFF_LIMIT_REACHED

---

### Story 9 — Freeing Capacity When Someone Finishes Work
Covers:
- Licensing invariant: capacity is freed when operational activity ends
- Human expectation: “someone ended work, now it should work”

---

## 5) Cross-Module Story Coverage (Do Not Lose These)

These stories are **not owned by Staff Management alone**, but must exist for the product to feel correct.
They should be written under the operational context where users experience them, and referenced by multiple maps.

1. **Starting Work at a Specific Branch**  
   Primary owner: Attendance stories  
   Depends on: Staff Management (branch assignment), Access Control (gate), Staff Licensing (capacity), Authentication (session)

2. **Being Denied Because You Are Not Assigned to This Branch**  
   Primary owner: Access Control stories  
   Depends on: Staff Management (BranchAssignment fact)

3. **Being Denied Immediately After You Were Disabled or Unassigned**  
   Primary owner: Access Control stories  
   Depends on: Staff Management (status/assignment changes) + session behavior

4. **Capacity Block Messaging (What Staff Sees and What Owner Sees)**  
   Primary owner: Staff Licensing stories  
   Depends on: Attendance/Cash Session signals + consistent UX

> Rule: When you write these, keep them human-centered and avoid explaining middleware/authorization internals.

---

## 6) Story File Checklist (Tracking)

| Story | File | Status |
|---|---|---|
| Creating a Staff Account for Someone New | `creating_staff_account.md` | ⬜ |
| Updating Basic Staff Information | `updating_staff_information.md` | ⬜ |
| Assigning a Staff Member to One or More Branches | `assigning_staff_to_branches.md` | ⬜ |
| Removing a Staff Member’s Access to a Branch | `removing_branch_access.md` | ⬜ |
| Disabling a Staff Member Immediately | `disabling_staff_immediately.md` | ⬜ |
| Archiving a Staff Member While Keeping History | `archiving_staff.md` | ⬜ |
| Reviewing Staff Changes When Something Goes Wrong | `reviewing_staff_audit_trail.md` | ⬜ |
| Being Blocked When Too Many Staff Are Active | `concurrent_staff_limit_block.md` | ⬜ |
| Freeing Capacity When Someone Finishes Work | `freeing_concurrent_capacity.md` | ⬜ |

Legend:
- ⬜ Not written
- 🟨 Drafted
- ✅ Final

---

## 7) Guardrails (Important)

- Do **not** write one story per UC.
- Keep stories **non-technical** and human-centered (no “middleware”, no “auth_account_id”, no “idempotency”).
- “Password rules” are expressed as **trust boundaries**, not cryptography.
- Branch assignment is **explicit** for all roles (including admins).
- Concurrent staff is framed as **fairness to shift-based operations**, not “anti-abuse”.

---

# End of Map
