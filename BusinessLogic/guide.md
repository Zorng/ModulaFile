# ModSpec Usage Guide — For Backend Dev (AI) & Frontend Dev (AI)

## Purpose

This guide defines how Modula developers (human or AI) must use **Module Specifications (ModSpecs)** to implement features consistently, safely, and without inventing business logic.

A ModSpec is the implementation-facing contract for a module:
- what the module owns
- what it guarantees
- what it exposes (commands/events/queries)
- what it refuses
- how it participates in cross-module processes

ModSpecs are backed by:
- Domain models (truth and invariants)
- Process docs (cross-module orchestration)

If a ModSpec conflicts with a Domain doc, the **Domain doc wins**.
If cross-module behavior is described inside a Domain doc, it should move to a **Process doc**.

---

## 1) What developers must NOT do

Developers (AI included) must NOT:
- invent business rules not written in domain/process/modspec docs
- implement cross-module orchestration inside a single domain/module “because it’s convenient”
- treat ledger-based domains (Inventory, Cash Session) as CRUD tables
- store critical state in memory (must be DB-truth + idempotent)
- bypass module boundaries by writing directly to another module’s tables

If something is unclear, create a **TODO / Assumption** section and ask for clarification (or flag it as “requires decision”).

---

## 2) Document hierarchy (source of truth)

Use this priority order:

1. **Process Docs (Cross-Module)**  
   Defines orchestration: order of operations across modules.
2. **Domain Docs (Self-Contained Truth)**  
   Defines invariants, state transitions, ownership, and what cannot happen.
3. **ModSpecs (Implementation Contract)**  
   Defines endpoints/commands/events, data models, read/write responsibilities, acceptance criteria.
4. **UI/UX Docs** (if any)  
   Defines user flows, screens, interaction patterns.

Rule:
- Domain tells you *what must be true*.
- Process tells you *how modules cooperate*.
- ModSpec tells you *how to implement it*.

---

## 3) How to implement a feature using ModSpecs (Backend AI)

### Step A — Identify the module of change
Read the ModSpec:
- module purpose
- scope & excludes
- owned entities
- invariants

If the feature crosses modules, DO NOT implement it inside one module.
Instead:
- implement the module-local commands in each module
- implement orchestration in the **Process layer** (application service)

### Step B — Choose the correct path: Self-contained vs Cross-module
- Self-contained use case → implement directly within module boundaries
- Cross-module use case → implement module contracts + orchestration

### Step C — Implement write model rules
For each command:
- enforce invariants
- ensure idempotency if request can retry (offline, double-submit)
- use DB constraints for “only one OPEN per branch” style rules
- append-only rules must not be violated (ledger domains)

### Step D — Implement read models from the ModSpec
- Use projections when specified (e.g., Inventory BranchStock)
- Paginate audit/history views
- Avoid N+1 queries
- Use indexes implied by the spec

### Step E — Emit audit events
If the ModSpec lists audit events, ensure they are written as specified.

### Step F — Validate acceptance criteria
Each ModSpec contains acceptance criteria. Confirm implementation meets them, and add tests.

---

## 4) How to implement a feature using ModSpecs (Frontend AI)

Frontend must treat ModSpecs as:
- what data exists
- what states exist
- what actions are allowed
- what errors to expect
- what is “read model” vs “write model”

### Step A — Identify the module UI you are building
Read:
- Use cases (what user can do)
- Preconditions (what must exist before UI enables actions)
- Postconditions (what changes after success)
- Errors / failure modes (what to show)

### Step B — Respect invariants and states in UI
UI must not allow invalid actions:
- disable finalize if cash session must be open
- prevent stock mutation if branch frozen
- warn/confirm for destructive operations (void, adjustment)

### Step C — Use correct read model endpoints
If a ModSpec says:
- on-hand comes from projection (BranchStock)
Then UI must call that endpoint, not compute totals client-side.

### Step D — UX expectations for ledger domains
Ledger-based flows behave differently:
- you don’t “edit” historical movements
- you append corrections (adjustments/reversals)
UI must reflect this:
- show adjustment history
- show “reversal created” not “deleted”
- show idempotent retry safety (disable double-submit, use client request id)

### Step E — Offline considerations
If Sync/Offline is enabled:
- UI must generate idempotency keys for critical operations
- UI must handle “eventual confirmation” states if required by Sync spec

---

## 5) Cross-module features: the correct pattern

Cross-module behavior must be implemented as:

- Module A provides a command contract
- Module B provides a command contract
- A Process (application service) orchestrates:
  1) validate
  2) call module commands
  3) emit process-level event (if needed)

Example: Finalize Order (cross-module)
- Order: FinalizeOrder (PAID)
- Inventory: ApplyExternalDeduction (idempotent)
- Cash Session: RecordCashMovement (idempotent)
- Process layer: FinalizeOrderService coordinates them

Frontend must call one “Finalize” API that triggers the process.
Backend must not expose multi-step “call this then that” UI coupling.

---

## 6) Handling unclear parts (required practice)

When you hit unclear behavior:
- do not guess silently
- create a short note:

### Assumption format
- **Assumption:** …
- **Reason:** spec unclear / conflict
- **Impact:** what changes if wrong
- **Decision needed by:** product owner / tech lead

This prevents hallucinated logic from becoming production behavior.

---

## 7) What to test (minimum checklist)

### Backend (must)
- invariants enforced
- idempotency works under retries
- concurrency safe updates (no lost updates)
- audit events written
- projections correct or rebuildable

### Frontend (must)
- state gating matches preconditions
- errors handled
- no illegal actions exposed
- double-submit prevented (or safe)

---

## 8) Naming and file placement (consistency)

Use consistent naming:
- `*_domain.md` → domain truth
- `*_process.md` → cross-module orchestration
- `*_module.md` / `*_modspec.md` → implementation spec

Suggested structure:
- `KnowledgeBase/BusinessLogic/domain/`
- `KnowledgeBase/BusinessLogic/process/`
- `KnowledgeBase/BusinessLogic/modSpec/`

Platform modules (sync, jobs, notifications) may live under:
- `KnowledgeBase/Platform/`

---

## 9) Final rule (most important)

> If it changes money, stock, or audit state, it must be:
> - idempotent
> - auditable
> - consistent under retries and concurrency

ModSpecs exist to make these guarantees implementable.