# Architecture

## 1. Style

**Modular monolith**: one deployable Rust binary, internal modules with strict boundaries, one React SPA. No microservices. Extraction of a module into a service requires an ADR and evidence of need.

## 2. Context diagram

```
 Browser (React SPA)
   │  HTTPS REST            │  WebSocket (wss)           │ Neon Auth (SDK)
   ▼                        ▼                            ▼
 ┌───────────────────────── Axum backend (Rust) ──────────────────────────┐
 │ http  │ auth │ rooms │ realtime │ playback │ tracks │ persistence │ obs │
 └───────────┬──────────────────┬─────────────────────────┬───────────────┘
             │                  │                         │
        Redis (live state,   Neon Postgres            Jamendo API
        pub/sub, locks)      (room metadata, stats)   (licensed tracks)
```

Neon Auth (managed Better Auth, data in the database's `neon_auth` schema) handles sign-in in the browser using its own session cookie. For our separate Rust API the frontend obtains a short-lived (15 min) EdDSA-signed JWT from the auth SDK and sends it as a Bearer token. The backend verifies JWTs against the Neon Auth JWKS endpoint; it never stores passwords and never writes to the `neon_auth` schema.

## 3. Backend modules

| Module | Responsibility | May depend on |
|---|---|---|
| `config` | Typed env config, validated at startup | none |
| `error` | `AppError`, HTTP/WS error mapping | none |
| `auth` | Neon Auth JWT verification (cached JWKS, EdDSA), `AuthUser` extractor | config, error |
| `rooms` | Domain types (`RoomCode`, `RoomStatus`), room lifecycle service, authorization rules | error |
| `playback` | `PlaybackState`, command validation, seq logic (pure, no I/O) | rooms (types) |
| `realtime` | WS upgrade, connection registry, message codec, Redis pub/sub fan-out, heartbeat | auth, rooms, playback |
| `tracks` | Jamendo client, license evaluation, attribution model | config, error |
| `persistence` | SQLx repositories (rooms, stats), migrations | error |
| `redis_store` | Redis repositories and Lua scripts for room state | error |
| `http` | Router, REST handlers, middleware (rate limit, tracing, CORS) | all services |
| `observability` | tracing, metrics, health | config |

**Rules:**
- Domain logic (`rooms`, `playback`) is **pure and synchronous** where possible: input state + command → result/events. This makes BR rules unit-testable without I/O.
- Handlers are thin: parse → authorize → call service → map response.
- No module reaches into another's internals; use the public API of the module.
- Traits are introduced only at real I/O boundaries needed for substitution (e.g., `Clock`). Prefer real Redis/Postgres in integration tests via containers instead of mock traits.

## 4. Core flows

### 4.1 Create room
1. `POST /api/v1/rooms` with JWT.
2. Check BR-14 (user has no active hosted room via `user:{id}:hosted_room` in Redis).
3. Generate code (BR-12); insert Postgres row `status=active`.
4. Write Redis room keys with TTL (see DATA_MODEL.md) in a single Lua/MULTI script.
5. If step 4 fails, mark Postgres row `failed`/delete (compensation); no half-created rooms.
6. Return room code and invite link.

### 4.2 Join + realtime
1. Client precheck `POST /rooms/{code}/join` → verifies active, capacity (BR-05/06).
2. Client opens `wss://.../ws`, sends first message `auth` with JWT (token is not placed in URL to avoid log leakage).
3. Server verifies JWT, registers connection, atomically adds participant in Redis (superseding older connection, BR-09), publishes `participant.joined`.
4. Server sends `room.snapshot` (room, participants, playback, server time).

### 4.3 Playback command
1. Host sends `playback.*` with client `msg_id`.
2. Server checks user is host (BR-03) using Redis room state, not client claims.
3. Lua script atomically validates, updates playback hash, increments `seq` (BR-11).
4. Publish `playback.updated` on `room:{code}:events`; every instance fans out to local sockets.

### 4.4 Room end
Single `end_room` operation (BR-07): Postgres update first (idempotent), then Redis delete via script, publish `room.ended`, close sockets. Must be idempotent so retries and concurrent triggers (host click + grace expiry) are safe.

### 4.5 Disconnect handling
- Heartbeat: client ping every 15 s; server drops after 2 missed. 
- Host drop → `host_grace:{code}` key with TTL; a background expiry watcher (keyspace notifications or a periodic sweeper with a Redis lock) ends expired rooms.
- Participant drop → presence retained for the participant grace period, then removed with `participant.left`.

## 5. Horizontal scalability

- Backend instances are **stateless** apart from in-memory socket registries; all room truth is in Redis.
- Cross-instance fan-out: Redis pub/sub channel per room. Each instance subscribes only for rooms with local sockets.
- No sticky sessions required. Reconnects may land on any instance.
- Atomic state transitions use Lua scripts; hash-tag keys `{code}` so all room keys share a cluster slot.
- Sweeper/grace watcher uses a distributed lock (`SET NX PX`) so only one instance runs each sweep cycle; operations are idempotent regardless.
- Limits: configure max connections per instance and message size caps; scale out behind a load balancer supporting WebSocket.

## 6. Failure modes

| Failure | Behavior |
|---|---|
| Redis down | Reject new rooms/joins (503); existing sockets get `error(UNAVAILABLE)` and are closed; no data invented. |
| Postgres down | Room creation fails; active rooms continue; end-room finalization retried with backoff and a Redis `pending_finalize` marker. |
| Jamendo down | Search returns 502 with cached results where allowed; playing rooms continue. |
| Instance crash | Clients reconnect to another instance; state intact in Redis. |

## 7. Frontend architecture

- React + TypeScript (strict) + Rspack.
- Feature folders: `auth/`, `rooms/`, `player/`, `realtime/`, `ui/`.
- Realtime client: single WebSocket manager with reconnect/backoff, typed message codec (generated or hand-maintained to match API_AND_WS_PROTOCOL.md), seq-based event ordering.
- State: React context + reducer for room state; no global state library unless justified by ADR.
- Player: HTMLAudioElement-based synchronized playback (ADR-0003) with drift correction using server time offset.
- CSS: native CSS with custom properties (design tokens), `@layer` for ordering, CSS Modules for component scope, mobile-first responsive, `prefers-reduced-motion` and `prefers-color-scheme` respected, accessible focus states.
- Neon Auth SDK used for sign-in and for obtaining JWTs only; all room operations go through our backend. A token manager refreshes the JWT before expiry (15 min lifetime) and feeds both REST calls and the WebSocket `auth.refresh` message.

## 8. Cross-cutting

- Observability: `tracing` with JSON logs, request/room IDs as span fields, Prometheus-style metrics (active rooms, sockets, command latency, errors).
- Errors: single `AppError` enum, stable machine codes (API_AND_WS_PROTOCOL.md).
- Config: 12-factor env vars (DEVOPS.md).
