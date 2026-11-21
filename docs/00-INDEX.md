# Documentation Index

| Doc | Purpose | Read when |
|---|---|---|
| [../AGENTS.md](../AGENTS.md) | **Constitution and precedence rules** | Always first |
| [STATUS.md](STATUS.md) | Living progress log, task board, blocked items | Always second |
| [TASK_PROTOCOL.md](TASK_PROTOCOL.md) | Mandatory task execution protocol | Before any work |
| [tasks/](tasks/) | Task cards (`TEMPLATE.md`, `P1-1-...`) | Claiming and doing work |
| [PROTECTED_DECISIONS.md](PROTECTED_DECISIONS.md) | Changes that need human approval | Before any structural change |
| [PRODUCT.md](PRODUCT.md) | Vision, users, scope, business rules BR-xx | Before any feature |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System design, modules, flows, scaling | Before backend/frontend structural work |
| [AUDIO_ARCHITECTURE.md](AUDIO_ARCHITECTURE.md) | WebRTC / LiveKit / SFU evaluation and decision | Before any audio work |
| [LICENSING.md](LICENSING.md) | Jamendo terms, license handling, YouTube prohibition | Before any music-source work |
| [DATA_MODEL.md](DATA_MODEL.md) | Postgres schema, Redis keys, TTLs, Lua atomicity | Before storage work |
| [API_AND_WS_PROTOCOL.md](API_AND_WS_PROTOCOL.md) | REST endpoints, WebSocket messages, error codes | Before API/WS work |
| [SECURITY.md](SECURITY.md) | Threat model, auth, authorization, checklist | Before every change |
| [TESTING.md](TESTING.md) | Test strategy, BR-xx → test matrix | Before writing/merging code |
| [CONVENTIONS.md](CONVENTIONS.md) | Rust, TS, CSS, git, error, logging conventions | While coding |
| [DEVOPS.md](DEVOPS.md) | Docker, Compose, CI, env vars, deployment | Infra work |
| [ROADMAP.md](ROADMAP.md) | Phased plan with acceptance criteria | Picking next work |
| [AGENT_WORKFLOW.md](AGENT_WORKFLOW.md) | One-page orientation pointing to the rules above | Quick orientation |
| [adr/README.md](adr/README.md) | ADR system: when, lifecycle, acceptance, index | Making or checking a decision |

## Research basis (verified 2026-11-21)

- Jamendo API Terms of Use: API usable freely for non-commercial uses; commercial uses require a quote via licensing@jamendo.com; Jamendo may charge or insert ads in future (https://devportal.jamendo.com/api_terms_of_use).
- Neon Auth is a managed service built on Better Auth; users and sessions live in the database's `neon_auth` schema. For backends, the JWT plugin issues EdDSA (Ed25519) tokens valid 15 minutes, verified via `<auth URL>/.well-known/jwks.json`, with `iss`/`aud` equal to the auth URL origin and `sub` as the user id (https://neon.com/docs/auth/guides/plugins/jwt, https://neon.com/docs/auth/authentication-flow).
- Neon pooled connections use PgBouncer in transaction mode (`-pooler` hostname); protocol-level prepared statements work, but session `SET`, `LISTEN/NOTIFY`, temp tables and session advisory locks do not; migrations should use a direct connection (https://neon.com/docs/connect/connection-pooling).
- Jamendo tracks carry individual Creative Commons licenses; some forbid commercial use, some require share-alike, all require attribution; the API exposes a per-track license field.
- LiveKit provides a Rust SDK (`livekit`, `livekit-api` crates) with Tokio required, server-side token generation, and audio publish/subscribe (https://github.com/livekit/rust-sdks). A reported issue (rust-sdks #1018) shows stereo publishing did not flag stereo to the server; re-verify before relying on stereo.
