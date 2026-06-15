---
description: Run the board-of-advisors panel (Skeptic, Pragmatist, Architect) in parallel on a spec, plan, PR, or diff, then synthesize a recommendation.
---

## User Input

```text
$ARGUMENTS
```

`$ARGUMENTS` is the thing to review: a spec/plan path, a FIX-NNN, a PR number/URL, a diff, or a
design described inline. If absent, ask what to review.

## Goal

Get three independent perspectives on a proposal before committing to it, then synthesize.

## Steps

1. **Assemble the proposal context**: read the spec/plan, or gather the diff (`git diff`), or the
   PR. Keep it concise — pass the personas what they need, not the whole repo.

2. **Run the three advisors in parallel** (one message, three Task calls):
   - `advisor-skeptic` — flaws, edge cases, failure modes.
   - `advisor-pragmatist` — scope, smallest correct change, what to cut/defer.
   - `advisor-architect` — boundaries, contracts, pattern consistency, evolvability.

   Give each the same proposal context.

3. **Synthesize** their verdicts into a single recommendation:
   - Surface the **blockers** first (anything the Skeptic marked BLOCK, the Architect marked
     REDESIGN, or a contract/boundary violation).
   - Note **agreements** (high-confidence) and **disagreements** (call the trade-off explicitly).
   - Give a clear **recommendation**: proceed / proceed-with-changes / rework — and the concrete
     changes needed.

4. **Output**:
   ```
   ## Advisor panel — <subject>

   | Advisor | Verdict | Top point |
   |---|---|---|
   | Skeptic | … | … |
   | Pragmatist | … | … |
   | Architect | … | … |

   ### Synthesis
   - Blockers: …
   - Recommendation: …
   - If proceeding, change: …
   ```

## Notes

- The advisors are reviewers, not implementers — they do not modify code.
- Scale effort to the request: a small fix needs a light pass; a new service or a contract change
  warrants the full panel.
- This pairs naturally with `/spec` and `/plan` (review before implementing).
