---
id: P1-4
title: Config, error and observability modules
type: feature
status: ready
risk: med
phase: 1
depends_on: [P1-1]
rules: []
adrs: [0001, 0007]
owner:
claimed_at:
branch:
blocked_reason:
reviewer:
---

# P1-4 — Config, error, observability

## Goal
Typed fail-fast configuration, a single `AppError` with stable machine codes, structured logging, and health endpoints.

## Scope
**In:** `config` (env parsing for DEVOPS.md variables incl. `DATABASE_URL`, `DATABASE_URL_DIRECT`, `NEON_AUTH_URL`, grace/capacity defaults), `error` (`AppError` to HTTP/WS code mapping per API_AND_WS_PROTOCOL.md), `observability` (tracing JSON logs, request id), `/healthz`, `/readyz` (Redis and Postgres checks with timeouts).
**Out:** auth, rooms, realtime.

## Acceptance criteria
- [ ] Missing/invalid config fails at startup with a clear message and no secret values printed.
- [ ] Every `AppError` variant maps to a documented code; internal errors never leak details.
- [ ] `/readyz` returns 503 when Redis or Postgres is unreachable; does not flap on a slow Neon cold start (documented timeout).
- [ ] No `unwrap`/`expect` outside tests and startup validation.

## Expected files / areas
`backend/src/{config,error,observability,http}/**`.

## Docs to read first
ARCHITECTURE.md (modules, failure modes), CONVENTIONS.md, DEVOPS.md, DATA_MODEL.md (Neon connections).

## Plan
## Test plan
| Rule / criterion | Test name(s) | Level |
|---|---|---|
| Config validation | `config_fails_on_missing_required`, `config_never_logs_secrets` | unit |
| Error mapping | `app_error_maps_to_documented_codes`, `internal_error_hides_details` | unit |
| Readiness | `readyz_503_when_redis_down`, `readyz_503_when_db_down` | integration |

## Security and licensing notes
Secrets must be redacted in `Debug` output; use a secret wrapper type.

## Verification
## Questions for owner
## Handoff
