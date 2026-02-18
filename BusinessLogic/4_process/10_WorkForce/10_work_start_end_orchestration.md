# Work Start Orchestration

## Purpose

This document describes the orchestration that happens when a staff member **starts work** at a branch.

This is the Identity & HR equivalent of **Finalize Sale Orchestration** in POS operations.

Starting work is not a single-domain action.  
It coordinates multiple domains to ensure the business remains safe, consistent, and accountable.

This process is commonly known as **Check-in**.

---

# Why This Process Exists

In real cafés, starting work is simple:

A staff member arrives → starts working.

But for the system, many conditions must be true before work can begin safely:

- The person must be authenticated.
- They must belong to the tenant.
- They must be allowed to work at the branch.
- The branch must be operational.
- The branch must have available staff capacity.
- Work should align with the planned shift when applicable.

This orchestration ensures that **only valid work begins**.

---

# High-Level Outcome

If successful:
- A new Attendance session is created.
- The staff member becomes **actively working** at the branch.
- Branch staff capacity is consumed.

If rejected:
- Work does not start.
- The staff member receives a clear reason.

---

# Participating Domains

| Domain                       | Responsibility                                    |
| ---------------------------- | ------------------------------------------------- |
| Authentication               | Confirms who the user is                          |
| Tenant Membership            | Confirms the user belongs to the tenant           |
| Staff Profile & Assignment   | Confirms branch assignment eligibility            |
| Branch                       | Confirms branch is ACTIVE                         |
| Access Control               | Authorizes the action                             |
| Shift                        | Provides planned work expectation (optional gate) |
| Attendance                   | Creates the work session                          |
| Capability / Policy (future) | Enables optional checks (GPS, shift enforcement)  |

---

# Trigger

The process begins when a staff member attempts to:

**“Start work at a branch”**

Typically via the POS or staff app.

---

# Step-by-Step Flow

## Step 0 — Idempotency Gate (Process Layer)

Apply the platform idempotency gate:
- `action_key = attendance.startWork`
- `idempotency_key = client_op_id` (request-level key)

If a matching attendance session already exists for this key:
- return the existing session (no duplicate check-in)

Reference:
- `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`

---

## Step 1 — Authentication Gate

The system verifies the user is authenticated.

Outcome:
- If not authenticated → stop immediately.

---

## Step 2 — Tenant Membership Gate

Verify the user belongs to the selected tenant.

Checks:
- User is a member of the tenant.
- Membership is ACTIVE (not INVITED and not REVOKED).

Failure examples:
- User removed from tenant.
- Membership revoked.

---

## Step 3 — Branch Selection & Assignment Gate

Verify the staff member can work at the selected branch.

Checks:
- Branch exists.
- Staff is assigned to this branch.

Failure examples:
- Staff tries to work at the wrong branch.
- Branch not found.

---

## Step 4 — Branch Status Gate

Verify the branch is operational.

Check:
- `branch.status == ACTIVE`

Failure examples:
- Branch frozen.
- Branch suspended.

This prevents operational writes at frozen branches.

---

## Step 5 — Authorization Gate

Access Control verifies permission to start work.

Examples:
- Role allows attendance actions.
- Not blocked by policy.

This keeps authorization centralized and consistent.

---

## Step 6 — Capacity Gate (Operator Seats)

Ensure branch operator seat capacity is not exceeded.

Check:
- Current active attendance sessions at branch
- Must be below branch operator seat limit

Failure example:
- Maximum active staff already reached.

This enforces the **concurrent operator seat model**.

Notes:
- Seats are a branch-scoped commercial limit (Subscription & Entitlements), consumed at `START_WORK`.
- This gate applies to both staff and admins when operating the branch.

Implementation-neutral read requirement:
- Resolve seat limit from Subscription & Entitlements (conceptually: `GetOperatorSeatLimit(tenant_id, branch_id)`).

---

## Step 7 — Shift Alignment Check (Soft Gate)

If shift planning exists, compare with planned shift.

This is **not a hard blocker**.

Purpose:
- Detect early/late/out-of-schedule starts.
- Record context for later review.

Important:
Work can still start even if shift does not match.

This supports real-world flexibility.

---

## Step 8 — Location Confirmation (Capability-Gated)

If attendance location confirmation is enabled (`attendance.location_verification_mode` is `checkin_only` or `checkin_and_checkout`):

Best-effort capture:
- Capture staff device location (if available).
- Compare against branch workplace location + allowed radius (if configured).
- Record the result as evidence: MATCH / MISMATCH / UNKNOWN.

Important (March rule):
- Location confirmation is **evidence-first**. It does not block check-in.
- If permission is denied, signal is weak, or branch workplace location is not configured → record UNKNOWN and continue.

If capability disabled → skip this step entirely.

---

## Step 9 — Create Attendance Session

Attendance domain creates a new **active attendance record**:

Fields include:
- staff_id
- tenant_id
- branch_id
- check_in_time
- location context (if enabled)
- shift context (if available)

Staff is now considered **actively working**.

---

## Step 10 — System Becomes Ready for Work

After successful check-in:
- Staff may open POS
- Staff may access branch-scoped features
- Staff capacity is consumed

The workday has officially begun.

---

# Failure Handling

If any gate fails:
- No attendance record is created.
- Staff receives a clear reason.
- System remains unchanged.

Failures must be:
- immediate
- explicit
- understandable

---

# Key Guarantees

This orchestration guarantees:

- Only authorized staff can start work.
- Branch capacity is respected.
- Attendance records begin at a clear moment.
- Real-world flexibility is preserved.
- Optional features are capability-gated.

---

# Related Processes

This process pairs with:

- `Work End Orchestration`
- `Shift vs Attendance Evaluation`

Together they define the lifecycle of work in Modula.
