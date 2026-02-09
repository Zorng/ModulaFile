# Idempotency Gate Module — Core Module (Capstone 1)

**Version:** 1.0  
**Status:** Defined (Platform guard rail for critical writes)  
**Module Type:** Core Module  
**Depends on:** Access Control, Tenant & Branch Context  
**Used by:** Sale, Cash Session, Attendance, Inventory, Offline Sync (replay)

---

## 1. Purpose

The Idempotency Gate module ensures **exactly-once application** of critical write operations.

It protects against duplicates caused by:
- retries and timeouts
- double taps
- offline replay

Authoritative references:
- Story: `BusinessLogic/1_stories/correcting_mistakes/avoiding_duplicate_actions_due_to_retries.md`
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/idempotency_domain.md`
- Edge cases: `BusinessLogic/3_contract/10_edgecases/idempotency_edge_case_sweep.md`
- Process: `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`

---

## 2. Scope & Boundaries

### This module covers
- Persistent idempotency record storage
- Conflict detection for reused keys
- Result lookup for duplicates

### This module does NOT cover
- Business state changes (owned by calling modules)
- Authorization policy (Access Control)
- Offline queue mechanics (Offline Sync)

---

## 3. Required Inputs (Critical Writes)

Every critical write must provide:
- `idempotency_key` (client_op_id / request_id)
- `action_key` (stable permission/action identifier)
- `tenant_id`
- `branch_id` (required for branch-scoped actions)
- `payload_hash`
- `actor_id`

---

## 4. Output Semantics

The gate returns one of:
- `APPLY` (no record found; proceed)
- `DUPLICATE` (return stored result)
- `CONFLICT` (same key, different payload)

---

## 5. Storage Contract (Logical)

Each idempotency record stores:
- `idempotency_key`
- `action_key`
- `tenant_id`
- `branch_id` (if applicable)
- `payload_hash`
- `status` (`IN_PROGRESS | APPLIED | FAILED`)
- `result_ref` (pointer to domain result)
- `created_at`, `updated_at`

---

## 6. Acceptance Criteria

- AC-1: Duplicate submissions return the same result (no double effects).
- AC-2: Conflicting submissions are rejected deterministically.
- AC-3: Idempotency records are committed atomically with the business write.
- AC-4: Offline replay uses the same gate as online writes.

---

# End of Document
