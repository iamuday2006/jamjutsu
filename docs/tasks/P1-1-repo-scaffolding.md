---
id: P1-1
title: Repository scaffolding (backend and frontend skeletons)
type: chore
status: in_progress
risk: low
phase: 1
depends_on: []
rules: []
adrs: [0001, 0005, 0007]
owner: Copilot SDK
claimed_at: 2026-11-21T16:40:23Z
branch: chore/P1-1-repo-scaffolding
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
- [ ] Backend exposes a minimal `/healthz` endpoint returning success.
- [ ] Module layout matches ARCHITECTURE.md section 3.
- [ ] `.env.example` lists every variable in DEVOPS.md with no real values.
- [ ] Any dependency added is justified in the PR description.

## Expected files / areas
`backend/**`, `frontend/**`, `.gitignore`, `.env.example`, `.github/pull_request_template.md`, `rust-toolchain.toml`.

## Docs to read first
AGENTS.md, STATUS.md, PROTECTED_DECISIONS.md, TASK_PROTOCOL.md, ARCHITECTURE.md, CONVENTIONS.md, DEVOPS.md.

## Plan
1. Add a minimal Axum backend crate with the documented module skeleton, pinned Rust toolchain, strict lints, and `/healthz`; keep domain, auth, persistence, Redis, and realtime logic out of scope.
2. Add a strict React + TypeScript frontend using Rspack, ESLint, Prettier, and Vitest, with the documented CSS layer/token foundation.
3. Add repository hygiene files, public-only `.env.example` values/placeholders, and the required PR checklist.
4. Add failing backend health and frontend render tests before implementing their minimal behavior.
5. Add dependencies only for the chosen runtime/tooling and the SQLx/Postgres and redis-rs integrations explicitly required by this card; commit lockfiles.

Expected changes: `backend/**`, `frontend/**`, `.gitignore`, `.env.example`, `.github/pull_request_template.md`, `rust-toolchain.toml`, this card, and `docs/STATUS.md`.

No business rules, database schema/migrations, credentials, or REST/WebSocket protocol behavior are introduced. Rollback is limited to removing the scaffold files; the documentation baseline and Neon project remain untouched.

## Test plan
| Rule / criterion | Test name(s) | Level |
|---|---|---|
| Backend health endpoint | `healthz_returns_ok` | integration |
| Frontend initial render | `app_renders_jam_room_heading` | unit |
| Skeleton builds and lint | `smoke_builds`, `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`, `lint`, `tsc --noEmit`, `vitest run`, `build` | CI |

## Security and licensing notes
No secrets; pin toolchain; commit lockfiles.

## Verification
## Questions for owner
## Handoff
