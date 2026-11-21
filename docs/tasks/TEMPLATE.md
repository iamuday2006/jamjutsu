---
id: <ID, e.g. P2-4>
title: <imperative title>
type: feature | spike | bug | chore | docs
status: ready            # ready | in_progress | blocked | in_review | done | cancelled
risk: low                # low | med | high (high requires independent reviewer)
phase: <n>
depends_on: []           # task ids that must be done first
rules: []                # business rules, e.g. [BR-03, BR-04]
adrs: []                 # related ADR numbers
owner:                   # agent id once claimed
claimed_at:              # UTC ISO timestamp, refresh every <24h
branch:
blocked_reason:
reviewer:                # required for risk:high
---

# <ID> — <title>

## Goal
One or two sentences: what outcome, and why it matters.

## Scope
**In:** ...
**Out (explicitly not in this task):** ...

## Acceptance criteria
- [ ] AC1 (testable)
- [ ] AC2

## Expected files / areas
List paths or modules likely touched (used for collision avoidance).

## Docs to read first
- AGENTS.md, STATUS.md, PROTECTED_DECISIONS.md, TASK_PROTOCOL.md
- <task-specific docs / ADRs>

## Plan
(Filled by the agent in Phase 4 before coding.)

## Test plan
| Rule / criterion | Test name(s) | Level |
|---|---|---|
| | | |

## Security and licensing notes
Relevant items from SECURITY.md / LICENSING.md checklists.

## Verification
(Commands run and results, filled in Phase 7.)

## Questions for owner
(Only when blocked; include recommended default.)

## Handoff
Done / Tests added / Docs-ADRs changed / Verification / Follow-ups / Gotchas
