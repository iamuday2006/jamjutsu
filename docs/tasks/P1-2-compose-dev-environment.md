---
id: P1-2
title: Docker Compose development environment
type: chore
status: ready
risk: low
phase: 1
depends_on: [P1-1]
rules: []
adrs: [0007]
owner:
claimed_at:
branch:
blocked_reason:
reviewer:
---

# P1-2 — Compose dev environment

## Goal
One command brings up everything needed to develop and test locally.

## Scope
**In:** `docker/compose.dev.yml` with Redis 7, local Postgres, `fake-auth` (small service that generates an Ed25519 key pair, serves `/.well-known/jwks.json`, and mints test JWTs with configurable `sub`, `exp`, `iss`, `aud`, ban flag), backend (cargo-watch) and frontend dev server; healthchecks; Dockerfiles for backend and frontend (multi-stage, non-root).
**Out:** production deployment, real Neon connectivity.

## Acceptance criteria
- [ ] `docker compose -f docker/compose.dev.yml up` starts all services healthy.
- [ ] Redis is not exposed beyond localhost.
- [ ] `fake-auth` JWKS is reachable by the backend via `NEON_AUTH_JWKS_URL` override.
- [ ] README documents the command and any prerequisites.

## Expected files / areas
`docker/**`, `tools/fake-auth/**` (or `backend/tests/support/`), `README.md`, `DEVOPS.md` (only if config changes).

## Docs to read first
DEVOPS.md, TESTING.md (fake-auth), SECURITY.md (auth facts).

## Plan
## Test plan
| Rule / criterion | Test name(s) | Level |
|---|---|---|
| fake-auth tokens verify with EdDSA | `fake_auth_jwks_matches_signed_tokens` | integration |

## Security and licensing notes
fake-auth keys are test-only and must never be usable outside dev/test configuration.

## Verification
## Questions for owner
## Handoff
