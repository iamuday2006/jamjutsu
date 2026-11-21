# Data Model

Principle: **Redis = live, Postgres = durable metadata only** (BR-10, BR-17).

## 1. PostgreSQL (Neon)

Migrations live in `backend/migrations/` (SQLx, forward-only, reviewed). Users are owned by Neon Auth, which stores them in the managed `neon_auth` schema (`neon_auth.user`, etc.) of the same database. We **never write to or alter** the `neon_auth` schema and never copy passwords. Our tables store the Neon Auth user id (JWT `sub`) as plain `text` **without a foreign key**, so a managed-schema change cannot break our migrations. Confirm the exact id type of `neon_auth.user.id` at task P2-2 and adjust the column type if needed.

```sql
create type room_status as enum ('active', 'ended', 'failed');
create type room_end_reason as enum ('host_ended', 'host_timeout', 'expired', 'system');

create table public.rooms (
  id                 uuid primary key default gen_random_uuid(),
  code               text not null,
  host_id            text not null,   -- Neon Auth user id (JWT sub); no FK to neon_auth by design
  status             room_status not null default 'active',
  created_at         timestamptz not null default now(),
  ended_at           timestamptz,
  end_reason         room_end_reason,
  peak_participants  int not null default 0,
  unique_participants int not null default 0,
  constraint ended_consistency check (
    (status = 'active' and ended_at is null and end_reason is null) or
    (status <> 'active' and ended_at is not null)
  )
);
-- code is unique only among active rooms (codes may be reused after end)
create unique index rooms_active_code_uq on public.rooms(code) where status = 'active';
create index rooms_host_idx on public.rooms(host_id, created_at desc);

-- Optional aggregate stats (no per-event history)
create table public.room_participation (
  room_id    uuid not null references public.rooms(id) on delete cascade,
  user_id    text not null,   -- Neon Auth user id (JWT sub); no FK to neon_auth by design
  first_joined_at timestamptz not null default now(),
  total_seconds   int not null default 0,
  primary key (room_id, user_id)
);
```

**Security:** the backend connects with a dedicated application role (least privilege: DML on our tables only, no access to `neon_auth` writes) and enforces authorization in application code. Do **not** enable the Neon Data API / RLS-over-HTTP for these tables; browsers never talk to Postgres directly.

**Connections (Neon):**
- Runtime queries use the **pooled** connection string (hostname with `-pooler`; PgBouncer in transaction mode). Protocol-level prepared statements are supported, which is what SQLx uses, so SQLx is compatible; SQL-level `PREPARE`/`EXECUTE` is not.
- **Not available on pooled connections:** session-level `SET`, `LISTEN/NOTIFY`, temp tables, session advisory locks. Our design needs none of them (Redis provides pub/sub and locks). Do not introduce them.
- **Migrations run over the direct (non-pooled) connection**, because SQLx's migrator takes a session-level advisory lock. Use a separate `DATABASE_URL_DIRECT` for migrations.
- Configure pool `max_lifetime`, `idle_timeout`, `acquire_timeout` and health checks to survive compute restarts. If scale-to-zero is enabled on the Neon compute, the first query after idle can be slow; set generous connect timeouts and keep `/readyz` from flapping on a cold start. Avoid scale-to-zero on production.
- Neon branches isolate data (including auth data) per branch; use a dedicated branch per environment.

**Never store:** queue, track lists, playback events, chat, participant presence.

## 2. Redis

Keys use hash tag `{code}` so all room keys map to one cluster slot.

| Key | Type | Fields / value | TTL |
|---|---|---|---|
| `room:{code}:meta` | hash | `room_id`, `host_id`, `status`, `created_at_ms`, `max_participants`, `seq` | 6 h, refreshed on activity |
| `room:{code}:playback` | hash | `track_json`, `status`, `position_ms`, `updated_at_ms`, `seq` | same as meta |
| `room:{code}:participants` | hash | `user_id` → JSON `{display_name, conn_id, joined_at_ms, last_seen_ms, state}` | same as meta |
| `room:{code}:host_grace` | string | `1` | 60 s (config) |
| `room:{code}:pgrace:{user_id}` | string | `1` | 30 s (config) |
| `user:{user_id}:hosted_room` | string | room code (BR-14) | same as meta |
| `room:{code}:events` | pub/sub channel | JSON events | n/a |
| `lock:sweeper` | string | instance id | 10 s |

### Atomic operations (Lua scripts in `backend/src/redis_store/scripts/`)

- `create_room.lua`: fails if `user:{id}:hosted_room` exists or code collides; writes all keys with TTL.
- `join_room.lua`: checks status and capacity, upserts participant, supersedes old `conn_id`, returns snapshot.
- `apply_playback.lua`: verifies caller == `host_id`, validates command, updates playback, `INCR seq`, returns new state.
- `leave_room.lua` / `expire_participant.lua`.
- `end_room.lua`: idempotent delete of all room keys and host mapping; returns whether it deleted anything.

Every script has an integration test against real Redis.

## 3. Consistency rules

- Create: Postgres insert → Redis script; on Redis failure mark row `failed`.
- End: Postgres update (`where status='active'`, idempotent) → Redis `end_room.lua` → publish `room.ended`.
- Crash between steps: sweeper reconciles (Postgres `active` rooms without Redis meta are finalized as `expired`/`system`).
- Redis TTL expiry without explicit end: sweeper finalizes Postgres as `expired`.
