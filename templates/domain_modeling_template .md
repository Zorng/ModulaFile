# Domain Modeling Template

# Domain Modeling Template — `<Domain Name>`

## 1. Domain Overview

**Domain name:**

**Business purpose:**

**Why this domain exists:**

Describe in plain business language what this domain is responsible for and why it must exist as a distinct domain.

---

## 2. Domain Boundary & Ownership

### 2.1 What This Domain Owns

List responsibilities that belong **exclusively** to this domain.

- 
- 
- 

### 2.2 What This Domain Does NOT Own

Explicit exclusions that must be enforced by design.

- 
- 
- 

---

## 3. Core Domain Concepts

### 3.1 Entities

Entities have identity and lifecycle.

| Name | Identity | Lifecycle Description |
| --- | --- | --- |
|  |  |  |

---

### 3.2 Value Objects

Value objects are immutable and have no identity.

| Name | Immutable | Purpose / Usage |
| --- | --- | --- |
|  | Yes / No |  |

---

### 3.3 Aggregate Roots

Aggregates define consistency boundaries.

| Aggregate Root | Consistency Boundary |
| --- | --- |
|  |  |

---

## 4. Entity Lifecycle & State Machine

For each stateful entity:

### `<Entity Name>` — States

- 
- 
- 

### Allowed Transitions

| From State | To State | Trigger / Command |
| --- | --- | --- |
|  |  |  |

### Forbidden Transitions

Explicitly list transitions that must never occur.

- 

---

## 5. Commands (Intent-Based)

Commands express **intent**, not implementation steps.

| Command Name | Actor | Preconditions | Result |
| --- | --- | --- | --- |
|  |  |  |  |

Rules:

- Commands may fail.
- Commands must enforce business invariants.
- Commands may emit domain events.

---

## 6. Business Invariants (Hard Rules)

These rules **must never be violated**, even under concurrency, retries, or offline replay.

- 
- 
- 

---

## 7. Domain Events

Events represent facts that already happened.

| Event Name | Emitted When | Consumed By |
| --- | --- | --- |
|  |  |  |

---

## 8. Read Model vs Write Model

### Write Model Responsibilities

- Enforces invariants
- Handles commands
- Emits domain events

### Read Model Responsibilities

- Optimized for queries
- No business rules
- Eventually consistent

---

## 9. Concurrency, Idempotency & Offline Behavior

Answer explicitly:

- Can commands be retried safely?
- What is the idempotency key?
- Can this domain operate offline?
- Who is authoritative: client or server?
- How are conflicts resolved?

---

## 10. External Dependencies

Other domains or external services this domain depends on.

- 
- 
- 

---

## 11. Failure Modes & Recovery

Describe how the system behaves when:

- A command fails midway
- A duplicate request arrives
- A dependent service is unavailable
- Offline data is replayed

---

## 12. Out of Scope (Explicit)

Clearly state what this domain **will not** handle.

- 
- 
- 

---

## 13. Notes & Future Evolution

Planned extensions or deferred behavior for future phases.

- 

---

## 14. Derived Artifacts (After Approval)

These artifacts may be created **only after** this domain model is approved.

- Module Specification
- API Contracts
- Database Schema
- Tests