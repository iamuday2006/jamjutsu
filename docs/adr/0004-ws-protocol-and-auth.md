# ADR-0004: WebSocket protocol and authentication handshake

Status: Accepted

## Context
Tokens in URLs leak via logs and referrers. Clients need reliable ordering and reconnect.

## Decision
Single `/ws` endpoint; JSON envelope `{v, type, id, payload}`; first message `auth` within 5 s; server-assigned monotonically increasing `seq` for playback; full `room.snapshot` on every (re)connect instead of event replay.

## Alternatives
Token in query string (leakage); per-room WS URLs with replay logs (more state, contradicts ephemerality); Socket.IO (extra protocol layer, not needed).

## Consequences
Simple reconnect logic, stateless servers. Clients must handle the auth-message handshake. Breaking changes require `v` bump.
