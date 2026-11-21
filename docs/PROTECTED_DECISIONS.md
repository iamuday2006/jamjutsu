# Protected Decisions

These must **never be changed without explicit written approval from the human owner** (record approval in an ADR). An agent that believes one of these is wrong must stop and raise it, not work around it.

## Architecture
1. Redis is the source of truth for **active** room state; Postgres never holds live state.
2. Ephemeral sessions: no persisted queues, playback history, or live presence.
3. Modular monolith; no microservices or new deployable services.
4. Event-driven realtime over WebSocket with the protocol in API_AND_WS_PROTOCOL.md; breaking protocol changes require a version bump (`v`) and ADR.

## Authorization and security
5. Playback commands and room termination are **host-only**, enforced server-side from server-held state (BR-03/04).
6. **Neon Auth** is the identity provider (changed from Supabase Auth by owner direction, ADR-0007); the backend verifies JWTs on every REST call and WS connection, and never writes to the `neon_auth` schema.
7. Clients never supply audio URLs; the server resolves tracks from approved sources.
8. Tokens never appear in URLs or logs; secrets never in the repo or the frontend bundle.

## Legal and product
9. **No YouTube / YouTube Music extraction or redistribution.**
10. Only sources with a documented license basis (LICENSING.md); per-track license checks and attribution are mandatory.
11. No commercial launch while on Jamendo's non-commercial API terms.

## Engineering process
12. Every business rule keeps its automated tests; tests may be strengthened but not deleted or weakened to make a change pass.
13. Rust type safety: no `unsafe`, no panics in request paths.
14. Forward-only migrations; never edit an applied migration.
15. Room codes remain CSPRNG-generated and unguessable (BR-12).
16. The governance system (ADR-0008): `AGENTS.md` as the highest authority with its precedence table, the ADR system (`adr/README.md`), and the Task Execution Protocol (`TASK_PROTOCOL.md`). Edits to `AGENTS.md` need owner approval; Accepted ADR bodies are immutable (supersede instead); work without a claimed task card is not allowed.

## How to request a change
Open an ADR (`adr/0000-template.md`) with status `Proposed`, explain the motivation, alternatives, and impact on rules/tests/docs, set the related task card to `blocked`, and wait for human approval before implementing. Only the owner can accept ADRs that touch this list.
