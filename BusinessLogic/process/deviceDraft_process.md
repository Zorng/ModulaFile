# Device Draft Handling — Cross-Module Process

## Overview
Manage client-side drafts across device and session changes.

## Rules
- One draft per device per branch
- Draft must be resolved before:
  - branch switch
  - logout
  - cash session close

## Responsibility
Application / UX layer (not domain-owned)
