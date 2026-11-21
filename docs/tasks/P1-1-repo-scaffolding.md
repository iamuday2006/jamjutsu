---
id: P1-1
title: Repository scaffolding (backend and frontend skeletons)
type: chore
status: ready
risk: low
phase: 1
depends_on: []
rules: []
adrs: [0001, 0005, 0007]
owner:
claimed_at:
branch:
blocked_reason:
reviewer:
---

# P1-1 — Repository scaffolding

## Goal
Create buildable, lint-clean skeletons for `backend/` (Rust) and `frontend/` (React + TypeScript + Rspack) so later tasks have a stable base.

## Scope
**In:** Cargo project with the module layout from ARCHITECTURE.md (empty modules with doc comments), `rust-toolchain.toml`, `#![forbid(unsafe_code)]`, clippy lint config per CONVENTIONS.md, SQLx and redis-rs dependencies declared but unused code avoided; Rspack React TS app (strict TS, ESLint, Prettier, Vitest), CSS layer/token skeleton; `.gitignore`, `.env.example` (variables from DEVOPS.md), PR template with the required checklist.
**Out:** any business logic, auth, database, Redis, WebSocket code.

## Acceptance criteria
- [ ] `cargo build`, `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test` pass.
- [ ] Frontend `lint`, `tsc --noEmit`, `vitest run`, `build` pass.
- [ ] Module layout matches ARCHITECTURE.md section 3.
- [ ] `.env.example` lists every variable in DEVOPS.md with no real values.
- [ ] Any dependency added is justified in the PR description.

## Expected files / areas
`backend/**`, `frontend/**`, `.gitignore`, `.env.example`, `.github/pull_request_template.md`, `rust-toolchain.toml`.

## Docs to read first
AGENTS.md, STATUS.md, PROTECTED_DECISIONS.md, TASK_PROTOCOL.md, ARCHITECTURE.md, CONVENTIONS.md, DEVOPS.md.

## Plan
## Test plan
| Rule / criterion | Test name(s) | Level |
|---|---|---|
| Skeleton builds | `smoke_builds` (CI) | CI |

## Security and licensing notes
No secrets; pin toolchain; commit lockfiles.

## Verification
## Questions for owner
## Handoff
