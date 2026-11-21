# Architecture Decision Records (ADR System)

ADRs are the project's decision memory. A future agent with no history must be able to answer "why is it like this?" from here.

Authority: Accepted ADRs rank 4 in the precedence table in `AGENTS.md` (above topic docs, below `PROTECTED_DECISIONS.md`).

## 1. When an ADR is required

Write one when a change does any of the following:

- Adds, removes, or replaces a dependency, external service, or infrastructure component.
- Changes architecture, module boundaries, data model, storage rules, the WS/REST protocol, auth, or security posture.
- Resolves an Open Question (PRODUCT.md) or a spike (AUDIO_ARCHITECTURE.md S1-S5).
- Touches, or argues against, any item in `PROTECTED_DECISIONS.md`.
- Chooses between two or more reasonable approaches that a later agent might re-litigate.
- Adds a music source or changes licensing posture.

Not needed for: bug fixes, refactors inside a module, naming, test additions, or anything already decided by an Accepted ADR.

## 2. Lifecycle

```
Proposed ──► Accepted ──► Superseded by ADR-NNNN
    │            └──────► Deprecated
    └──► Rejected
```

| Status | Meaning |
|---|---|
| Proposed | Under review. Implement only the documented default; do not build on it. |
| Accepted | In force. Immutable except for the status line and a "Superseded by" pointer. |
| Rejected | Considered and declined; kept so it is not re-proposed blindly. |
| Superseded by ADR-NNNN | Replaced; history only. |
| Deprecated | No longer relevant, no replacement. |

## 3. Who can accept

- **Human owner only:** ADRs touching `PROTECTED_DECISIONS.md`, security/auth, licensing/legal, paid services or cost, or `AGENTS.md`.
- **Independent agent review:** other ADRs may be accepted when a *different* agent (or the owner) approves in the PR. The author never accepts their own ADR.

## 4. How to write one

1. Next number = highest existing + 1 (4 digits). Copy `0000-template.md` to `NNNN-kebab-title.md`.
2. Fill every section. Keep it under about one page. State the decision in one sentence at the top of "Decision".
3. List real alternatives with why they lost.
4. List **affected docs, rules, and tests**. Update those docs in the same PR (so no document contradicts an Accepted ADR).
5. Add a row to the index below, in the same PR.
6. If it replaces an earlier ADR, edit only the old ADR's status line to `Superseded by ADR-NNNN` and link both ways.
7. Task cards reference ADRs in their `adrs:` field; the PR description links the ADR.

## 5. Rules

- One decision per ADR. Do not bundle.
- Never rewrite history: do not edit the body of an Accepted ADR. Supersede it.
- Number gaps are not allowed; do not reuse numbers.
- A merged PR that contradicts an Accepted ADR is a defect; fix the code or supersede the ADR.

## 6. Index

| ADR | Title | Status |
|---|---|---|
| [0001](0001-modular-monolith.md) | Modular monolith | Accepted |
| [0002](0002-redis-ephemeral-state.md) | Redis as source of truth for active room state | Accepted |
| [0003](0003-audio-delivery-model.md) | Audio delivery model (synchronized playback first, SFU gated) | Proposed (owner sign-off requested) |
| [0004](0004-ws-protocol-and-auth.md) | WebSocket protocol and authentication handshake | Accepted |
| [0005](0005-persistence-and-sqlx.md) | SQLx for Postgres access | Accepted; partially superseded by 0007 |
| [0006](0006-music-source-jamendo.md) | Jamendo as initial music source; no YouTube extraction | Accepted (non-commercial) |
| [0007](0007-neon-postgres-and-auth.md) | Neon Postgres and Neon Auth replace Supabase | Accepted (owner-directed) |
| [0008](0008-documentation-governance.md) | Documentation governance: AGENTS.md authority, ADR system, task protocol | Accepted (owner-directed) |

## 7. Enforcement

CI (`docs-check`, roadmap task P1-3) verifies: every ADR file appears in this index with a matching status; numbering is contiguous; superseded ADRs link forward; PRs that change architecture-labelled paths reference an ADR.
