# ADR-0002: Redis as source of truth for active room state

Status: Accepted

## Context
Rooms are temporary; low-latency atomic updates and cross-instance fan-out are needed; persisting live state is unwanted.

## Decision
All live state (room meta, playback, presence, grace markers) lives in Redis with TTLs; atomic transitions via Lua scripts; pub/sub for cross-instance events. Postgres stores only metadata and aggregates.

## Alternatives
Postgres for live state (slow, heavy write churn, unwanted history); in-process memory only (breaks horizontal scaling and crash recovery).

## Consequences
Redis is a critical dependency (failure behavior in ARCHITECTURE.md section 6). Need sweeper reconciliation between stores. Hash-tagged keys keep cluster compatibility.
