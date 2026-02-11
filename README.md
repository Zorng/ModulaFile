# Modula POS - Knowledge Base

This repository is Modula's **design memory**: product intent, business rules, and system behavior captured as documents.

If a behavior exists in code, it should be **traceable to this knowledge base** (primarily `BusinessLogic/`).

## Where To Start

1. `BusinessLogic/README.MD` - authoritative business behavior (read this first)
2. `Architecture_Rationale_and_Capacity_Assumptions.md` - architecture rationale + capacity assumptions
3. `playbooks/modula_business_logic_modeling_playbook.md` - how we model and document behavior
4. `templates/` - writing templates (domain/process)

## Repository Layout

- `BusinessLogic/` - the layered spec: `Stories -> Domain -> Contract -> Process -> ModSpec`
- `playbooks/` - guidance on *how to think* and how to model business behavior
- `templates/` - templates for writing consistent domain/process docs
- `_ad-hoc_artifact/` - raw notes/workboards moved from root; **not authoritative**; to be distilled into `BusinessLogic/` or archived
- `AgentPlanDoc/` - AI/agent tracking docs (often gitignored); not a source of truth
- `secrets/` - private artifacts (tokens, competitor research, etc.); gitignored
- `WORKFLOW.md` - local collaboration workflow (gitignored)

## Non-Negotiables

- Keep the layer order: `Stories -> Domain -> Contract -> Process -> ModSpec`
- Resolve conflicts as: `Domain > Contract > Process > ModSpec`
- Prefer consistency over novelty; avoid ad-hoc rules that bypass the KB

## Product Context (Short)

Modula is a **cloud POS** for Cambodian SMEs (starting with cafes/F&B). It is designed to be:
- **SaaS-first** (multi-tenant)
- **Offline-friendly** (retry-safe workflows)
- **Modular** (capabilities can be gated; "pay only for what you use")
- **Operationally honest** (evidence, not surveillance)
