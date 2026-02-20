# 65 — Offline Sync Pull (Hydration)

## Purpose

This process defines how a client **hydrates and incrementally updates** its local cache from the backend’s
authoritative change feed.

It exists to ensure offline-first clients converge without bespoke per-module polling.

---

## Domains Involved

- Offline Sync (hydration semantics)
- Access Control (scope + membership revalidation)
- Policy / Menu / Discount / other modules (change producers)
- Audit (traceability; some changes reference actors)

---

## When This Process Runs

Clients should run pull sync:
- at branch bootstrap (first time opening a branch on a device),
- on reconnect / app resume,
- periodically while online (low frequency),
- and optionally when receiving a lightweight “sync nudge” signal.

Important:
- nudges are best-effort triggers only; they must not be required for convergence.

---

## Inputs (Conceptual)

- `tenant_id`
- `branch_id`
- `device_id`
- `cursor` (optional; resume token)
- `module_scopes[]` (requested filter set)

---

## Outputs (Conceptual)

- `changes[]` (ordered)
- `next_cursor`
- optional `checkpoint` metadata (for device resume optimization)

Each change includes:
- `seq` (monotonic within the branch stream)
- `op_kind` = UPSERT | TOMBSTONE
- `entity_kind`, `entity_id`
- `payload` for UPSERT (module-defined)

---

## Orchestration Steps

### Step 1 — Resolve and validate context

Backend must validate:
- tenant exists and is active,
- branch exists and is active (or readable if frozen; reads are still allowed),
- actor has an ACTIVE membership in the tenant and is authorized to read the requested scopes.

### Step 2 — Validate cursor and module scope safety

If `cursor` is provided:
- server validates the cursor is still valid (retention window, no reset),
- server validates cursor scope matches the requested `module_scopes` set
  (cursor embeds a `module_scope_hash` or equivalent).

If invalid:
- return a deterministic error indicating “reset required” (see edge cases).

### Step 3 — Produce an ordered change batch

Server reads from the branch stream feed starting after `cursor` and returns:
- ordered changes (ascending seq),
- `next_cursor` that represents the last included seq and the scope hash.

Notes:
- Branch streams are the unit of sync.
- Tenant-scoped catalog writes that affect all branches must be **fanned out** into each active branch stream.

### Step 4 — Client applies changes locally (deterministic)

Client must apply changes in order:
- For UPSERT: write/replace the local record for `(entity_kind, entity_id)`.
- For TOMBSTONE: remove the local record (or mark as deleted) so it no longer appears in active views.

Tombstone requirement:
- Tombstones must be applied even if the entity is not present locally (no-op is allowed).

### Step 5 — Update checkpoint

After successfully applying a batch, client updates its local checkpoint:
- store `next_cursor`,
- store `module_scopes` used,
- store last applied server `seq` and timestamps as needed.

Checkpoint is client-side progress only; server truth remains authoritative.

---

## Degradation / Resets

If the cursor is invalid (retention reset, scope mismatch, corrupted token):
- client must perform a **rehydration reset** for that branch context:
  - clear or rebuild affected local projections,
  - bootstrap via pull with no cursor (or with a server-issued bootstrap cursor),
  - then resume incremental pulls.

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/offline_sync_edge_case_sweep.md`
- `BusinessLogic/2_domain/60_PlatformSystems/offline_sync_domain.md`

