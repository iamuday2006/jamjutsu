# DevOps

## 1. Local development

`docker compose -f docker/compose.dev.yml up` starts: `redis` (7.x), `postgres` (plain local Postgres; production uses Neon), `fake-auth` (a small local JWKS server that signs EdDSA test JWTs, see TESTING.md, because Neon Auth is managed and has no local emulator), `backend` (cargo-watch), `frontend` (Rspack dev server). One command must give a working environment; document any deviation in README.

Compose services: healthchecks on all; Redis not published beyond localhost; named volumes only for Postgres.

## 2. Docker images

- Backend: multi-stage (`cargo-chef` for dependency caching) → minimal runtime (`debian-slim` or distroless), non-root user, read-only filesystem where possible, `HEALTHCHECK` hitting `/healthz`.
- Frontend: build with Rspack → static files served by nginx or the edge/CDN.
- Pin base image versions; scan with Trivy in CI.

## 3. Environment variables

| Var | Service | Description |
|---|---|---|
| `APP_ENV` | be | `local`/`staging`/`prod` |
| `BIND_ADDR` | be | e.g. `0.0.0.0:8080` |
| `DATABASE_URL` | be | Neon **pooled** connection string (`-pooler` host) for runtime queries |
| `DATABASE_URL_DIRECT` | be/ci | Neon **direct** connection string, used only for migrations |
| `REDIS_URL` | be | `redis://` or `rediss://` |
| `NEON_AUTH_URL` | be | Neon Auth base URL; issuer and audience are its origin (scheme + host, no path) |
| `NEON_AUTH_JWKS_URL` | be | Defaults to `<NEON_AUTH_URL>/.well-known/jwks.json`; override only for tests |
| `JAMENDO_CLIENT_ID` | be | Server-side only |
| `COMMERCIAL_MODE` | be | `false` default (LICENSING.md) |
| `ALLOWED_ORIGINS` | be | CORS and WS Origin allowlist |
| `ROOM_MAX_PARTICIPANTS`, `HOST_GRACE_SECS`, `PARTICIPANT_GRACE_SECS`, `ROOM_MAX_TTL_SECS` | be | Defaults per PRODUCT.md |
| `VITE`-style public vars (`PUBLIC_API_URL`, `PUBLIC_NEON_AUTH_URL`) | fe | Public values only |

Provide `.env.example`; real `.env` is gitignored. Config is parsed into a typed struct and fails fast on missing values.

## 3b. Secrets

Stored in the hosting platform's secret store and GitHub Actions secrets. Rotate on suspicion. Never echo in CI logs.

## 4. GitHub Actions

Workflows in `.github/workflows/`:

| Workflow | Trigger | Jobs |
|---|---|---|
| `ci-backend.yml` | PR, push main | fmt, clippy, test (Redis + Postgres service containers), sqlx offline check, `cargo deny`, `cargo audit` |
| `ci-frontend.yml` | PR, push main | install (lockfile), lint, typecheck, vitest, rspack build |
| `e2e.yml` | push main, nightly | Compose up, Playwright |
| `docker.yml` | push main, tags | Build, Trivy scan, push images |
| `docs-check.yml` | PR | Link check; ensure `docs/STATUS.md` changed when code changed |

Use caching (`Swatinem/rust-cache`, package manager cache), least-privilege `permissions:`, pinned action versions (SHAs for third-party).

## 5. Deployment (initial recommendation)

Single region, containers on a platform supporting WebSockets and rolling deploys; managed Redis; a Neon project with one branch per environment (compute and API hosted in the same region to minimize latency); static frontend on CDN. Rolling deploy: on SIGTERM, stop accepting new connections, send `1012 service restart` close to sockets so clients reconnect to healthy instances, drain within 30 s.

## 6. Observability and operations

Metrics: active rooms, connected sockets, command latency, WS errors, Redis/Postgres latency, sweeper runs. Alerts: error rate, Redis unavailability, orphaned `active` rooms in Postgres without Redis state. Runbooks go in `docs/runbooks/` as incidents teach us.
