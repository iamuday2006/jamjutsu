# Product Specification

## 1. Vision and why

People want to listen to music together remotely without installing anything. Jam Room offers lightweight, temporary listening rooms: create, share a code, listen together, close. The room lives only as long as the host keeps it alive, so there is no queue to manage long-term, no history to curate, and minimal data retained.

**Why ephemeral:** simpler operations, lower storage cost, better privacy, and a clear mental model ("the room is the session").

## 2. Personas

- **Host:** authenticated user who creates a room, picks tracks, controls play/pause/seek/track change, ends the room.
- **Participant:** authenticated user who joins by code or link and listens. Cannot control playback.

## 3. Scope

### In scope (MVP)
- Neon Auth sign-in (email/OAuth providers as configured in Neon Auth).
- Create room (becomes host), receive room code + invite link.
- Join room by code/link; see participant list.
- Host playback commands: play, pause, seek, set track, end room.
- Synchronized listening for all participants (see AUDIO_ARCHITECTURE.md).
- Real-time events over WebSocket.
- Reconnection with grace periods.
- Room end: Redis cleanup, Postgres metadata finalized, participants disconnected.
- Track search/catalog via Jamendo with license-aware filtering and attribution.

### Out of scope (MVP)
- Persistent queues, playlists, playback history.
- Chat, reactions (possible later; must not break ephemerality).
- Host transfer (see Open Questions).
- Mobile native apps.
- Any non-licensed music source.
- Payments/monetization (blocked by Jamendo non-commercial API terms, see LICENSING.md).

## 4. Business rules (authoritative IDs)

Each rule **must** have automated tests (see TESTING.md matrix).

| ID | Rule |
|---|---|
| BR-01 | Only authenticated users can create or join rooms. |
| BR-02 | Every room has exactly one host: its creator. |
| BR-03 | Only the host may issue playback commands (play, pause, seek, set_track) and end the room. |
| BR-04 | Playback commands from non-hosts are rejected with `FORBIDDEN`, are never applied, and are never broadcast. |
| BR-05 | A room can be joined only while its status is `active`; ended rooms can never be joined or revived. |
| BR-06 | A room has a maximum participant count (default 50, configurable); joins beyond it fail with `ROOM_FULL`. |
| BR-07 | Ending a room (host action or auto-end) atomically: marks Postgres `ended` with `ended_at` and reason, deletes all Redis keys for the room, broadcasts `room.ended`, closes participant sockets. |
| BR-08 | If the host disconnects, the room enters a grace period (default 60 s). Host reconnect within grace restores control; expiry auto-ends the room. |
| BR-09 | A participant reconnecting within grace (default 30 s) keeps their presence without duplicates; a user has at most one active connection per room (newest wins, older closed with `SUPERSEDED`). |
| BR-10 | No queue or playback history is persisted. All Redis room keys have TTLs and are removed at room end. |
| BR-11 | Playback state has a monotonically increasing `seq`; every applied command increments it; clients ignore events with `seq` ≤ last seen. |
| BR-12 | Room codes are unguessable (CSPRNG, 8 chars Crockford base32) and unique among active rooms. |
| BR-13 | WebSocket connections must present a valid, unexpired Neon Auth JWT. Because tokens last only 15 minutes, a connection must be kept alive with `auth.refresh` (same `sub`); a missed refresh closes the socket at token expiry (4401). Banned users are rejected. |
| BR-14 | A user may host at most one active room at a time (default; see Open Questions). |
| BR-15 | Only tracks whose license permits the intended use (per LICENSING.md) can be played; attribution data is always delivered with the track. |
| BR-16 | Inbound WebSocket messages and REST create/join calls are rate-limited per user/IP. |
| BR-17 | Postgres stores only room metadata and aggregate participation stats, never live state. |

## 5. User flows

1. **Create:** sign in → `POST /api/v1/rooms` → room `active`, host joined → UI shows code + invite link.
2. **Join:** sign in → open link or enter code → `POST /api/v1/rooms/{code}/join` precheck → connect WebSocket → `room.snapshot`.
3. **Listen:** host sends `playback.set_track`/`play` → server validates, updates Redis atomically, broadcasts `playback.updated` → clients start synchronized playback.
4. **Host drops:** `host.disconnected` broadcast → grace timer → reconnect (`host.reconnected`) or `room.ended(reason=host_timeout)`.
5. **End:** host `room.end` → `room.ended` → sockets closed → Redis cleaned → Postgres finalized.

## 6. Open questions (defaults apply until a human decides)

| # | Question | Default |
|---|---|---|
| OQ-1 | Host transfer on host drop? | No; room ends after grace. |
| OQ-2 | Allow anonymous guests? | No; auth required (BR-01). |
| OQ-3 | Commercial/public launch? | Blocked until Jamendo commercial licensing is resolved (LICENSING.md). |
| OQ-4 | Multiple hosted rooms per user? | No (BR-14). |
| OQ-5 | Final audio mode (synchronized playback vs. SFU broadcast)? | Synchronized playback for MVP; SFU spike later (ADR-0003). |

## 7. Success metrics (non-binding)

- Join-to-audio latency p95 < 3 s.
- Playback sync drift between participants < 250 ms p95 after correction.
- Room cleanup success 100 percent (no orphaned Redis keys beyond TTL).
