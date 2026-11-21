# Roadmap

Each item below becomes a task card in `tasks/` (see TASK_PROTOCOL.md) when first picked up; Phase 1 cards already exist. Task IDs (`P<phase>-<n>`) are referenced from STATUS.md. Each task lists dependencies (dep), acceptance criteria (AC), and rules covered.

## Phase 0 — Documentation (DONE)
P0-1 Documentation system created.

## Phase 1 — Foundations
- **P1-1 Repo scaffolding.** Cargo project (`backend/`), Rspack React TS app (`frontend/`), lint/format configs, toolchain pins, `.gitignore`, `.env.example`. AC: both build, hello-world health endpoint, `cargo test` and `vitest` run.
- **P1-2 Compose dev environment** (dep P1-1). AC: one command starts Redis, Postgres, `fake-auth` (JWKS + test-JWT helper), backend, frontend.
- **P1-3 CI workflows** (dep P1-1). AC: all gates in DEVOPS.md run on PR; rule-coverage script stub added.
- **P1-4 Config, error, observability modules.** AC: typed config fail-fast; `AppError` mapping; JSON tracing; `/healthz`, `/readyz`.

## Phase 2 — Auth and rooms (REST)
- **P2-1 Neon Auth JWT verification + `AuthUser` extractor** (BR-01, BR-13 partial). AC: EdDSA-only, cached JWKS, iss/aud origin checks, banned-user rejection; confirm Rust JWT library supports Ed25519 JWKs (record in ADR).
- **P2-2 Neon Postgres migrations + room repository** (BR-17). AC: migrations over direct connection, runtime over pooled connection; confirm `neon_auth.user.id` type; app role has no write access to `neon_auth`.
- **P2-3 Redis store + Lua scripts** (create/join/leave/end) (BR-06, BR-07, BR-10, BR-14).
- **P2-4 Room REST endpoints** create/get/join-precheck/end (BR-02, 03, 05, 12, 14).
- **P2-5 Frontend auth + create/join UI** (Neon Auth SDK, in-memory JWT token manager with proactive refresh).

## Phase 3 — Realtime
- **P3-1 WS upgrade, auth and `auth.refresh` messages, codec, heartbeat** (BR-13, BR-16).
- **P3-2 Presence + snapshot + join/leave events** (BR-09).
- **P3-3 Redis pub/sub fan-out and multi-instance test.**
- **P3-4 Frontend realtime manager** (reconnect/backoff, seq handling).

## Phase 4 — Playback
- **P4-1 Playback domain (pure) + `apply_playback.lua`** (BR-03, 04, 11) with property tests.
- **P4-2 WS playback commands + broadcast.**
- **P4-3 Time sync + client drift correction** (spike S5 informs constants).
- **P4-4 Frontend player UI** (host controls, listener view, autoplay gate).

## Phase 5 — Music source
- **P5-1 Spike S1/S3** (CORS, licensing answers recorded in docs).
- **P5-2 Jamendo client + license model** (BR-15), search endpoint, attribution UI.

## Phase 6 — Resilience
- **P6-1 Host grace + auto-end** (BR-08) with injectable clock.
- **P6-2 Participant grace, supersede** (BR-09).
- **P6-3 Sweeper + reconciliation** (Postgres/Redis consistency, TTL expiry).
- **P6-4 Graceful shutdown/draining.**

## Phase 7 — Hardening and release
- **P7-1 Rate limiting and abuse controls** (BR-16).
- **P7-2 E2E suite (Playwright), load test.**
- **P7-3 Security review against SECURITY.md checklist; dependency audit.**
- **P7-4 Staging deployment, runbooks, observability dashboards.**

## Phase 8 — Optional: SFU broadcast (gated)
Only after S2, S3, S4 are resolved and owner approves ADR-0003 follow-up. LiveKit token endpoint, room lifecycle hooks, host publish, listener subscribe, feature flag.

## Release gate
All BR-xx tests pass; security checklist complete; licensing mode confirmed (non-commercial unless Jamendo agreement exists).
