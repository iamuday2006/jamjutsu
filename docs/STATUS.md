# Project Status (living document)

Last updated: 2026-11-21 (UTC) by documentation agent.

## Summary
Phase 0 (documentation) complete. **No application code exists.** Next: Phase 1 scaffolding.

## Implemented
- Documentation system (this folder), ADRs 0001-0008, task protocol, Phase 1 task cards.

## Task board (derived from `docs/tasks/`; update with the card in the same commit)

| Task | Card | Status | Owner | Claimed at (UTC) | Risk |
|---|---|---|---|---|---|
| P1-1 Repo scaffolding | [card](tasks/P1-1-repo-scaffolding.md) | in_progress | Copilot SDK | 2026-11-21T16:40:23Z | low |
| P1-2 Compose dev environment | [card](tasks/P1-2-compose-dev-environment.md) | ready (after P1-1) | | | low |
| P1-3 CI + docs-check | [card](tasks/P1-3-ci-workflows.md) | ready (after P1-1) | | | med |
| P1-4 Config, error, observability | [card](tasks/P1-4-config-error-observability.md) | ready (after P1-1) | | | med |

Phase 2+ items live in `ROADMAP.md` and get cards when picked up.

## In progress
_None._

## Blocked / awaiting human decision
| Item | Needed from | Default meanwhile |
|---|---|---|
| ADR-0003 audio mode sign-off (sync playback MVP) | Owner | Proceed with Model A |
| OQ-1..OQ-5 in PRODUCT.md | Owner | Defaults listed there |
| Jamendo commercial licensing and rebroadcast permission (S3) | Owner | Non-commercial only |
| Original brief text ended mid-sentence at "What must never be changed with…" | Owner | Interpreted as "without approval"; see PROTECTED_DECISIONS.md |

## Open spikes
S1 CORS/URL behavior · S2 capture compatibility · S3 licensing answer · S4 SFU cost · S5 sync drift (see AUDIO_ARCHITECTURE.md).

## Known risks
- Jamendo API terms restrict to non-commercial use.
- Sync accuracy depends on client clocks and network; mitigation designed, unmeasured.
- Neon Auth JWTs are fixed at 15 minutes with no custom claims; the whole realtime layer depends on reliable `auth.refresh`.
- Neon Auth has no local emulator; tests rely on the `fake-auth` helper, so real-provider drift is only caught by the main-branch smoke test.
- Confirm `neon_auth.user.id` type and Rust Ed25519 JWKS support at P2-1/P2-2.
- Neon compute scale-to-zero cold starts can affect first-query latency.

## Handoff log
### 2026-11-21 — documentation agent — governance (ADR-0008)
- Done: `AGENTS.md` rewritten as central authority with precedence table and untrusted-input rule; ADR system formalized (`adr/README.md`, template, ADR-0008); Task Execution Protocol (`TASK_PROTOCOL.md`) with task-card template and Phase 1 cards; AGENT_WORKFLOW reduced to a pointer; PROTECTED_DECISIONS #16 expanded.
- Follow-up: P1-3 implements `docs-check` enforcement.

### 2026-11-21 — documentation agent — owner-directed change: Supabase → Neon
- Done: replaced Supabase Auth/Postgres with Neon Auth/Postgres across all docs; added ADR-0007; ADR-0005 partially superseded; PROTECTED_DECISIONS #6 updated with owner approval.
- Docs changed: AGENTS, README, 00-INDEX, PRODUCT (BR-13), ARCHITECTURE, DATA_MODEL, API_AND_WS_PROTOCOL, SECURITY, TESTING, CONVENTIONS, DEVOPS, ROADMAP, ADR-0005/0007.
- Known gaps: Neon pooler/Ed25519/user-id details to confirm at P2-1/P2-2.

### 2026-11-21 — documentation agent — P0-1
- Done: full documentation set.
- Tests added: none (no code).
- Decisions/ADRs: 0001-0006.
- Known gaps: spikes S1-S5 unresolved; open questions defaulted.
- How to verify: read `AGENTS.md` and follow the links.
