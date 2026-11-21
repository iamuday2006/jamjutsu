---
id: P1-3
title: GitHub Actions CI and docs-check governance job
type: chore
status: ready
risk: med
phase: 1
depends_on: [P1-1]
rules: []
adrs: [0008]
owner:
claimed_at:
branch:
blocked_reason:
reviewer:
---

# P1-3 — CI workflows and docs-check

## Goal
Automate every gate in DEVOPS.md and enforce the governance rules from ADR-0008.

## Scope
**In:** workflows `ci-backend`, `ci-frontend`, `docker`, `docs-check`; `scripts/check-docs.sh` and `scripts/check-rule-coverage.sh` (stub matching TESTING.md prefixes). `docs-check` verifies: every ADR file is in `docs/adr/README.md` with a matching status; ADR numbering contiguous; superseded ADRs link forward; markdown links resolve; `docs/STATUS.md` changed when code changed; every `docs/tasks/*.md` has valid front matter and a legal status; no two cards `in_progress` with the same owner and overlapping expected files; claims older than 24 h are flagged.
**Out:** deployment workflows.

## Acceptance criteria
- [ ] Workflows run on PR and push to main with least-privilege `permissions:` and pinned third-party actions.
- [ ] Backend job uses Redis and Postgres service containers.
- [ ] `docs-check` fails on each violation above (demonstrated by a fixture test).
- [ ] Caching configured.

## Expected files / areas
`.github/workflows/**`, `scripts/**`.

## Docs to read first
DEVOPS.md, TESTING.md section 4, adr/README.md section 7, TASK_PROTOCOL.md.

## Plan
## Test plan
| Rule / criterion | Test name(s) | Level |
|---|---|---|
| docs-check catches violations | `docs_check_rejects_unindexed_adr`, `docs_check_rejects_bad_card_status`, `docs_check_flags_stale_claim` | script tests |

## Security and licensing notes
Secrets only via GitHub secrets; never echoed. Review third-party action pinning.

## Verification
## Questions for owner
## Handoff
