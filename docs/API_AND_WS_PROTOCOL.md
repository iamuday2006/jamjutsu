# API and WebSocket Protocol (v1)

All REST endpoints are under `/api/v1`, JSON, `Authorization: Bearer <Neon Auth JWT>` (EdDSA, 15-minute lifetime; see SECURITY.md) unless noted.

## 1. REST

| Method | Path | Auth | Purpose | Success | Errors |
|---|---|---|---|---|---|
| GET | `/healthz` | none | Liveness | 200 | n/a |
| GET | `/readyz` | none | Readiness (Redis + Postgres) | 200 | 503 |
| POST | `/rooms` | yes | Create room (BR-01/02/12/14) | 201 `{room_id, code, invite_url}` | 409 `ALREADY_HOSTING`, 429, 503 |
| GET | `/rooms/{code}` | yes | Public room info (status, participant count, host display name) | 200 | 404 `ROOM_NOT_FOUND` |
| POST | `/rooms/{code}/join` | yes | Precheck joinability | 200 `{ws_url}` | 404, 409 `ROOM_ENDED`, 409 `ROOM_FULL` |
| DELETE | `/rooms/{code}` | host | End room (BR-03/07) | 204 | 403 `FORBIDDEN`, 404 |
| GET | `/tracks/search?q=&limit=` | yes | License-aware Jamendo search (BR-15) | 200 `{tracks[]}` | 502 `UPSTREAM_ERROR` |

Error body: `{ "error": { "code": "FORBIDDEN", "message": "human readable", "request_id": "..." } }`

## 2. WebSocket

Endpoint: `GET /ws` (upgrade). **No token in the URL.** The first client message must be `auth` within 5 s or the server closes with code 4401.

### Envelope

```json
{ "v": 1, "type": "playback.set_track", "id": "client-msg-uuid", "payload": { } }
```

Server events include `seq` (room event/playback sequence where relevant) and `ts` (server ms).

### Client → server

| type | Who | Payload |
|---|---|---|
| `auth` | any | `{ token, room_code }` |
| `ping` | any | `{ t }` |
| `time.sync` | any | `{ client_t0 }` |
| `playback.set_track` | host | `{ track_id, autoplay }` (server resolves and license-checks the track; clients never supply URLs) |
| `playback.play` | host | `{ position_ms? }` |
| `playback.pause` | host | `{}` |
| `playback.seek` | host | `{ position_ms }` |
| `room.end` | host | `{}` |
| `room.leave` | any | `{}` |

### Server → client

| type | Payload |
|---|---|
| `room.snapshot` | `{ room, participants[], playback, server_time_ms, you: {user_id, role} }` |
| `participant.joined` / `participant.left` | `{ user_id, display_name }` |
| `playback.updated` | `{ state: {track, status, position_ms, updated_at_ms, seq} }` |
| `host.disconnected` | `{ grace_seconds }` |
| `host.reconnected` | `{}` |
| `room.ended` | `{ reason }` then socket closes (1000) |
| `time.sync.reply` | `{ client_t0, server_t }` |
| `pong` | `{ t }` |
| `ack` | `{ id }` for accepted commands |
| `error` | `{ id?, code, message }` |

The host client sends only `track_id`. The server resolves metadata from Jamendo, evaluates the license (BR-15), and stores the resolved `track` (`id, title, artist, audio_url, duration_s, license, attribution_url`) in playback state. Clients therefore can never inject arbitrary audio URLs, which also prevents the room from being used to push malicious or unlicensed content.

### Close codes

| Code | Meaning |
|---|---|
| 1000 | Normal / room ended |
| 4401 | Unauthorized / auth timeout / token expired |
| 4403 | Forbidden |
| 4404 | Room not found |
| 4409 | Superseded by newer connection (BR-09) |
| 4429 | Rate limited |
| 1013 | Try again later (overloaded) |

### Error codes

`UNAUTHENTICATED`, `FORBIDDEN`, `ROOM_NOT_FOUND`, `ROOM_ENDED`, `ROOM_FULL`, `ALREADY_HOSTING`, `INVALID_COMMAND`, `STALE_STATE`, `LICENSE_NOT_ALLOWED`, `RATE_LIMITED`, `UPSTREAM_ERROR`, `UNAVAILABLE`, `INTERNAL`.

### Ordering and reliability

- Server applies commands atomically; `seq` strictly increases (BR-11). Clients drop events with `seq` <= last applied.
- On reconnect, the client always receives a fresh `room.snapshot`; no event replay is required.
- Max message size 8 KiB; unknown `type` → `INVALID_COMMAND`; protocol `v` mismatch → close 4400.
- Rate limit default: 20 messages/s per connection (hosts) and 5/s (participants).

### Token expiry

Neon Auth JWTs live only **15 minutes** (fixed), so refresh is a normal part of every session, not an edge case. The server tracks the JWT `exp`. Clients **must** send `auth.refresh {token}` (a fresh token from the auth SDK) roughly 2 minutes before expiry. The server verifies the new token, requires the same `sub` as the connection, and replaces the stored `exp`. If no valid refresh arrives by `exp`, the server closes with 4401; the client reconnects with a fresh token and receives a new `room.snapshot` (BR-09 grace applies, so presence is not lost). Add `auth.refresh` to the client-to-server table: payload `{ token }`, any role; server replies `ack`.
