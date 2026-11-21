# Jam Room

Real-time social music listening. Create a temporary room, share a code, listen together. The host controls playback; everyone else listens in sync.

> **Status:** Documentation phase. No application code yet. See [`docs/STATUS.md`](docs/STATUS.md).

## Docs

AI agents: start at [`AGENTS.md`](AGENTS.md) (the central authority), then follow its reading order; work happens through task cards ([`docs/TASK_PROTOCOL.md`](docs/TASK_PROTOCOL.md)) and decisions are recorded as ADRs ([`docs/adr/README.md`](docs/adr/README.md)). Humans: start at [`docs/00-INDEX.md`](docs/00-INDEX.md).

## Stack

Rust (Axum, Tokio, SQLx, redis-rs) · React + TypeScript + Rspack · Neon (Auth + Postgres) · Redis · Docker Compose · GitHub Actions

## Principle in one line

Redis holds live room state; Postgres keeps only durable metadata; WebSocket delivers events; the host alone controls playback.
