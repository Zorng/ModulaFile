# Staff Licensing Domain — Modula POS

## Domain Name
Staff Licensing (Operational Capacity Control)

## Domain Type
Supporting Domain (Commercial / Operational Policy)

## Status
Draft (Baseline for Capstone 1; Explicit Design Choice)

---

## Purpose

The Staff Licensing domain defines **how Modula limits staff usage** in a way that is:
- fair to real-world café operations,
- enforceable by the system,
- compatible with authorization and attendance,
- and defensible both academically and commercially.

This domain exists to **remove ambiguity** around staff limits and to prevent repeated debates or accidental redesigns.

---

## Why This Exists (User Reality)

In real cafés and small retail stores:
- staff work in shifts,
- many staff are part-time,
- staff turnover is common,
- only a few people operate the POS at the same time.

If a POS system limits **total staff records**, operators are forced to:
- constantly enable/disable accounts,
- delete historical staff,
- work around the system instead of with it.

At the same time, the system must still:
- enforce fair usage limits,
- prevent abuse,
- and scale commercially.

This domain defines *which reality Modula chooses to support*.

---

## 1. Common Staff Licensing Models in Mature POS Systems

Mature POS systems generally adopt **one of three staff licensing models**.

### 1.1 Named Staff (Per-User / Seat-Based Licensing)

**Definition**
- Limits the total number of active staff accounts.
- Every staff record consumes a license, whether or not they are working.

**Characteristics**
- Simple to understand.
- Easy to enforce.
- Poor fit for shift-based or high-turnover environments.

**Used by**
- Enterprise POS systems
- HR-heavy or corporate environments

---

### 1.2 Concurrent Staff (Simultaneous Operator Licensing)

**Definition**
- Limits how many staff can operate the system **at the same time**.
- Staff records may exceed the limit, but only a fixed number can be active concurrently.

**Characteristics**
- Aligns with shift-based work.
- Reduces administrative friction.
- Requires clear definition of “active”.

**Used by**
- Modern SMB-focused POS systems
- Retail and F&B tools optimized for daily operations

---

### 1.3 Unlimited Staff, Limited by Device or Branch

**Definition**
- No staff limits.
- Pricing is based on terminals, devices, or branches.

**Used by**
- Hardware-centric POS vendors
- Terminal-based pricing models

---

## 2. Modula’s Chosen Model: Concurrent Staff

**Design Decision (Explicit)**

Modula adopts the **Concurrent Staff** licensing model.

This is an intentional choice based on:
- real café operations,
- fairness to operators,
- reduced administrative overhead,
- compatibility with offline-first design.

This is **not a loophole** and **not an accident**.

---

## 3. Definition of “Active Staff” in Modula

A staff member counts as **active** if and only if:
- authenticated,
- operating within a selected `{tenant_id, branch_id}`,
- and in an operational state such as:
  - checked in (attendance),
  - holding an active POS session,
  - or holding an open cash session.

The following do **not** count toward the limit:
- off-shift staff,
- staff without an active session,
- disabled or archived staff,
- staff not checked into any branch.

---

## 4. Enforcement Philosophy

### Where Enforcement Happens
The concurrent staff limit is enforced when:
- staff attempts to check in,
- staff starts an operational POS session,
- staff opens a cash session.

If the limit is reached:
- the action is denied,
- a clear reason is returned (e.g., `STAFF_LIMIT_REACHED`),
- no silent fallback occurs.

### What Is NOT Enforced
- creating staff records
- editing staff profiles
- viewing staff lists
- historical data access

Licensing controls **operation**, not data modeling.

---

## 5. Relationship to Other Domains

- **Staff Management**: owns staff records and lifecycle.
- **Attendance**: primary signal for operational activity.
- **Authentication**: establishes identity/session.
- **Access Control**: enforces authorization; licensing enforces capacity.

---

## 6. Invariants

- INV-SL1: Limits apply only to concurrent operation, not staff records.
- INV-SL2: Exceeding the limit blocks new operational sessions.
- INV-SL3: Revocation or logout immediately frees capacity.
- INV-SL4: Enforcement must be deterministic and auditable.
- INV-SL5: Limits apply equally to all roles (ADMIN included).

---

## 7. Out of Scope

- Subscription pricing plans
- Billing and invoicing
- Automatic plan upgrades

---

## 8. Future Evolution

- Tie concurrent limits to entitlements snapshot
- Per-branch concurrency limits
- Peak concurrency analytics
- Policy-based overrides
