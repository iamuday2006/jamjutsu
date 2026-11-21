# Task Execution Protocol

Authority: referenced by `AGENTS.md` section 7. This protocol is mandatory for every change, including docs-only changes. The repository is the shared memory; chat is not.

## 1. Task cards

Each unit of work is a Markdown file `docs/tasks/<ID>-<slug>.md` created from `docs/tasks/TEMPLATE.md`.

**ID scheme:** roadmap items `P<phase>-<n>` (see ROADMAP.md); spikes `S<n>`; bugs `B-<n>`; chores `C-<n>`. Roadmap items without a card are created from the roadmap text when first picked up.

**Card is the source of truth for a task's status.** `STATUS.md` holds a summary board derived from the cards; update both in the same commit.

**Sizing:** one concern per task; target a diff a reviewer can read in one sitting (about 400 changed lines excluding generated files and lockfiles). Split larger work into cards with dependencies.

**Risk tag:** `risk:low | risk:med | risk:high`. `high` = authentication/authorization, Lua scripts and atomicity, migrations, deployment/secrets, anything in PROTECTED_DECISIONS.md. High-risk tasks require an independent reviewer (a different agent or the owner).

## 2. States

```
ready ──► in_progress ──► in_review ──► done
  │            │               │
  │            ├──► blocked ◄──┘
  └──► cancelled
```

| State | Meaning |
|---|---|
| ready | Dependencies done, acceptance criteria clear (Definition of Ready met) |
| in_progress | Claimed by exactly one agent |
| blocked | Waiting on owner/spike/dependency; reason recorded |
| in_review | PR open, awaiting review |
| done | Merged, verified, handoff written |
| cancelled | Will not be done; reason recorded |

## 3. Definition of Ready

- Dependencies are `done`.
- Acceptance criteria are testable and listed.
- Business rules (BR-xx) and ADRs involved are listed.
- No unresolved Open Question that changes the approach (otherwise use the documented default and say so).
- Expected files/areas listed (for collision avoidance).

## 4. Phases (execute in order; do not skip)

### Phase 1 — Orient
Read `AGENTS.md`, `STATUS.md`, `PROTECTED_DECISIONS.md`, this protocol. Pull the latest main (`git pull --rebase`). Choose the highest-priority `ready` card whose dependencies are `done` and whose expected files do not overlap an `in_progress` card.

### Phase 2 — Claim
1. Edit the card: `status: in_progress`, `owner: <agent id>`, `claimed_at: <UTC ISO>`, `branch: <name>`.
2. Update the `STATUS.md` board.
3. Commit `chore(tasks): claim <ID>` and **push before doing any work**. If the push is rejected, pull, re-read the card; if someone else claimed it, pick another task.
4. **Claim lifetime:** 24 hours. Refresh `claimed_at` with a small commit if work continues. After 24 h without refresh, any agent may reclaim, noting `reclaimed from <agent>` in the card. Never discard the previous agent's branch; build on or reference it.

### Phase 3 — Preflight
- Read the docs and ADRs listed in the card plus the Topic docs for the area.
- Search the code for existing implementations to reuse (once code exists).
- Decide whether an ADR is needed (adr/README.md section 1). If yes and it touches a protected decision, stop and escalate (AGENTS.md section 8).
- Check `LICENSING.md` for anything involving music sources, `SECURITY.md` for anything involving inputs, auth, or data.

### Phase 4 — Plan
Write the plan in the card's **Plan** section before coding: approach, files to create/change, BR ids and the exact test names to add (matching TESTING.md prefixes), migration/protocol impacts, rollback considerations. If the plan needs files outside "Expected files", update the list and check for collisions. If the plan conflicts with docs, follow `AGENTS.md` conflict procedure.

### Phase 5 — Test first
Add failing tests for each acceptance criterion and each affected BR (including negative authorization cases). Commit them (red is fine on the branch). Spikes replace this phase with a written hypothesis and success criteria.

### Phase 6 — Implement
Smallest change that satisfies the criteria. Follow `CONVENTIONS.md`. No unrelated refactors, no drive-by dependency additions, no new persistence "just in case". Commit in small conventional-commit steps (`feat(rooms): ...`) referencing the task ID.

### Phase 7 — Verify
Run every gate that exists and record the commands and results in the card's **Verification** section:
- Backend: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`, `cargo deny check`.
- Frontend: lint, `tsc --noEmit`, `vitest run`, build.
- Rule coverage script (when present) and `docs-check`.
- Manual or scripted check of each acceptance criterion; for protocol/Redis changes, a run against real Redis/Postgres.
Failing gates are never waived. A flaky test must be fixed or quarantined with a new `B-` card, not ignored.

### Phase 8 — Document
Update in the same change: the card (acceptance criteria checked), `STATUS.md`, affected topic docs, `TESTING.md` matrix names, ADR + ADR index (if applicable), `.env.example`/`DEVOPS.md` (if config changed), `API_AND_WS_PROTOCOL.md` (if protocol changed).

### Phase 9 — Review
Open a PR using the PR template: task ID, summary, BR ids, tests added, ADR links, security checklist, docs updated. Set card `in_review`. Review rules:
- `risk:high`: independent reviewer required; reviewer checks against SECURITY.md and the card's criteria and records approval in the card.
- Others: self-review against the Definition of Done checklist is acceptable; independent review preferred.
- Reviewer may not be the author; reviewer findings are fixed before merge, not deferred silently.

### Phase 10 — Handoff and close
After merge: set `status: done`, fill **Handoff**, move the board entry to Done, list follow-ups as new cards (`ready` or `blocked`), and unblock dependents. Never leave a `done` card without a verified result.

## 5. Blocked procedure

1. Set `status: blocked`, add `blocked_reason` and a precise question in **Questions for owner** with your recommended default.
2. Add to `STATUS.md` → Blocked / awaiting human decision.
3. Push, then pick another `ready` task. Do not guess on protected, security, licensing, or cost questions.
4. If a default is documented and the area is low-risk, proceed with the default and record that assumption in the card.

## 6. Spike tasks

Spikes (S1-S5, etc.) produce knowledge, not product code. Output: a section appended to the relevant doc (for example `AUDIO_ARCHITECTURE.md`), measured results with dates, and an ADR if a decision follows. Throwaway code stays under `spikes/<ID>/` and is not merged into `backend/` or `frontend/`.

## 7. Bug tasks

Card includes reproduction, expected vs actual, root cause, and the regression test written **first**. If the bug reveals a missing BR test, add it to the matrix.

## 8. Failure and recovery

| Situation | Action |
|---|---|
| Gates fail and you cannot fix them within the task scope | Set `blocked`, describe failure and attempted fixes, open a `B-` card if it is a pre-existing defect |
| Merge conflict with another agent's work | Rebase; never overwrite their changes; if semantics conflict, stop and note in both cards |
| Discover task is larger than planned | Stop, split into new cards, close or narrow the current one |
| Discover a doc is wrong | Fix the doc per precedence rules in the same PR and note it in handoff |
| Security issue found | Do not publish details in public places; mark `risk:high`, escalate to owner |
| Out of time/context | Leave the card `in_progress` with a precise **Handoff** of what is done, what remains, and exact next steps, then push |

## 9. Handoff note (card section)

```
Done: ...
Tests added: <names>
Docs/ADRs changed: ...
Verification commands and results: ...
Known gaps / follow-up cards: ...
Gotchas for the next agent: ...
```

## 10. Hard prohibitions

- No work without a claimed card.
- No merging with failing gates or missing BR tests.
- No changes to `AGENTS.md`, `PROTECTED_DECISIONS.md`, or Accepted ADR bodies without the process above.
- No force-push to shared branches; no rewriting other agents' commits.
- No secrets in cards, commits, logs, or PRs.
