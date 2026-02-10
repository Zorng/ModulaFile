# 85 — Fair-Use Limit Gate (Safety Quotas + Rate Limits)

## Purpose

This process defines a **platform gate** applied to write operations to prevent:
- runaway entity creation,
- bot abuse,
- and request floods that degrade service.

It is explicitly **not** commercial enforcement.
Billing and entitlements are handled elsewhere.

---

## Domains Involved

- Fair-Use Limits (safety quotas + rate limits)
- Idempotency (duplicate suppression)
- Access Control (authorization + entitlement enforcement)
- Audit/Telemetry (platform observability)

References:
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/fair_use_limits_domain.md`
- Edge cases: `BusinessLogic/3_contract/10_edgecases/fair_use_limits_edge_case_sweep.md`
- Idempotency gate: `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`

---

## When This Gate Runs

This gate runs for **all write operations** that:
- create persistent entities, or
- apply persistent mutations that are susceptible to replay/flood.

Examples:
- create staff profile
- create menu item
- create stock item
- restock / adjustment

Non-goals:
- it does not attempt to rate-limit read queries (handled separately if needed).

---

## Required Inputs (Conceptual)

- `tenant_id`
- `branch_id?` (if branch-scoped write)
- `actor_id` (`auth_account_id`)
- `action_key` (stable action string)
- `client_op_id` (idempotency key for offline/retry safety)
- `entity_kind` (e.g., `menu_item`, `stock_item`, `staff_profile`)
- `intended_delta` (usually `+1` for creates; may be 0 for updates)

---

## Canonical Evaluation Order (Locked)

To avoid breaking offline replay and idempotency, the gate order is:

1. **Idempotency gate**
   - If the operation is already applied for this `client_op_id`, return the prior success result.
   - Do not consume quota or rate-limit budget for duplicates.

2. **Access Control + Entitlements**
   - If authorization or entitlements deny the write (e.g., module is `READ_ONLY`), stop.
   - Fair-use is not a substitute for entitlements.

3. **Rate limit check**
   - If the actor is exceeding allowed frequency for `action_key`, deny with `FAIRUSE_RATE_LIMITED`.

4. **Hard quota check (count-based)**
   - If applying `intended_delta` would exceed the hard quota, deny with `FAIRUSE_HARD_LIMIT_EXCEEDED`.

5. **Soft quota warning (optional)**
   - If near soft threshold, emit `FAIRUSE_SOFT_LIMIT_WARNING` for UX/telemetry (never blocks by itself).

6. **Proceed with the write**
   - The actual write happens after this gate allows.

---

## Outputs (Decision)

- ALLOW
- DENY with reason code:
  - `FAIRUSE_RATE_LIMITED`
  - `FAIRUSE_HARD_LIMIT_EXCEEDED`

Soft warning output:
- `FAIRUSE_SOFT_LIMIT_WARNING` (observational only)

---

## Observability (March Baseline)

When the gate denies:
- record a platform-level audit/telemetry event with:
  - tenant_id, actor_id, action_key, entity_kind, reason_code
- do not present it as "upgrade your plan"

---

## Notes (Implementation Guidance, Non-Authoritative)

- Rate limit windows and quota numbers are configuration.
- March baseline defaults are documented in:
  - `BusinessLogic/2_domain/60_PlatformSystems/fair_use_limits_domain.md`
- Hard quota enforcement should be concurrency-safe (avoid overshoot under races).
- "Create" operations should be idempotent whenever possible to reduce accidental double-creation.
