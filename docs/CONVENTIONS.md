# Coding Conventions

## Rust

- Edition 2021+ stable toolchain pinned in `rust-toolchain.toml`.
- `#![forbid(unsafe_code)]` in the backend crate.
- Lints: `clippy::pedantic` selectively; `-D warnings` in CI. No `unwrap`/`expect`/`panic!`/indexing that can panic in non-test code (deny via clippy lints); startup config validation may fail fast with a clear message.
- Errors: `thiserror` for typed errors in modules; a single `AppError` at the HTTP/WS boundary; `anyhow` only in `main` and tooling. Every error variant maps to a stable code (API_AND_WS_PROTOCOL.md). Never expose internal error text to clients.
- Types: newtypes for `RoomCode`, `UserId`, `ConnId`, `Seq`; enums for states; make illegal states unrepresentable. `serde` with `#[serde(tag = "type", content = "payload")]` for protocol messages and `deny_unknown_fields` on inbound.
- Async: Tokio only. No blocking calls in async context (use `spawn_blocking` for CPU-heavy work). Never hold a lock guard across `.await`. Every outbound call has a timeout. Use `tokio::select!` with cancellation for socket tasks; each connection has a bounded channel to apply backpressure (drop slow consumers).
- SQL: SQLx with compile-time checked queries (offline `sqlx-data.json`/`.sqlx` committed). Migrations are forward-only.
- Redis: only through `redis_store`; scripts versioned in files; no ad-hoc keys elsewhere.
- Logging: `tracing` with structured fields (`room_code`, `user_id`, `request_id`); no tokens/PII; `info` for lifecycle, `warn` for recoverable, `error` for actionable.
- Dependencies: minimal; each new crate needs justification in the PR description and passes `cargo deny`.
- Modules follow ARCHITECTURE.md dependency directions; no cyclic dependencies.
- Doc comments on public items; reference BR ids where a function enforces a rule (`/// Enforces BR-03`).

## TypeScript / React

- `strict: true`, no `any` (use `unknown` + narrowing), ESLint + Prettier, `noUncheckedIndexedAccess`.
- Function components + hooks; no class components.
- Protocol types mirror API_AND_WS_PROTOCOL.md; validate inbound WS messages at runtime (e.g., zod) before use.
- Side effects (WebSocket, audio) live in hooks/services, not in render.
- Accessibility: semantic HTML, keyboard navigation, ARIA live region for room events, visible focus.
- No secrets in the bundle; only the public Neon Auth URL and API base URL.

## CSS

- Native CSS with custom properties as design tokens (`--color-*`, `--space-*`, `--radius-*`, `--font-*`), `@layer reset, tokens, base, components, utilities`.
- CSS Modules per component; mobile-first with `min-width` media/container queries; `clamp()` for fluid type.
- Support `prefers-color-scheme` and `prefers-reduced-motion`. No inline styles except dynamic values.

## Git

- Trunk-based with short-lived branches: `feat/<area>-<short>`, `fix/…`, `docs/…`, `chore/…`.
- Conventional Commits (`feat(rooms): …`). Small PRs, one concern each.
- PR template requires: linked roadmap task, BR ids touched, tests added, docs updated, security checklist.

## Naming

- Rust: `snake_case` fns/modules, `UpperCamelCase` types. TS: `camelCase`, components `PascalCase`, files `kebab-case` or `PascalCase.tsx` for components.
- WS message types: `domain.action` lowercase (e.g., `playback.set_track`).
- Redis keys exactly as in DATA_MODEL.md.
