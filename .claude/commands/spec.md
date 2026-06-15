---
description: Create a spec (what & why + acceptance criteria) for a feature or fix, from a natural-language description. Self-contained — writes into docs/specs/features or docs/fixes.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding. It is the feature/fix description.

## Goal

Produce a clear **specification** — focused on WHAT and WHY, not HOW — that a human can approve
before any planning or implementation. This mirrors the spec-driven workflow already used in
[docs/fixes/](../../docs/fixes/).

## Steps

1. **Classify** the request:
   - **Fix** (bug / improvement of existing behavior) → target `docs/fixes/`.
   - **Feature** (new capability) → target `docs/specs/features/`.

2. **Allocate an ID & folder** (do NOT use git branches to find the number):
   - Fixes: find the max `FIX-NNN` under `docs/fixes/`, use N+1 → `docs/fixes/FIX-NNN-<slug>/`.
   - Features: find the max `NNN` under `docs/specs/features/`, use N+1 → `docs/specs/features/NNN-<slug>/`.
   - `<slug>`: 2–4 kebab-case words capturing the essence (e.g. `feature-flag-mismatch`,
     `add-payment-service`).

3. **Write `spec.md`** using the project template
   ([docs/fixes/_TEMPLATE/spec.md](../../docs/fixes/_TEMPLATE/spec.md) for fixes). Sections:
   - **Problem (NE)** — observed behavior / missing capability.
   - **Evidence** — concrete `file:line` references (for fixes) or the user need (for features).
   - **Impact / why it matters.**
   - **Scope** — *included* and *excluded (non-goals)*.
   - **Proposed approach** — the design decision (still WHAT/WHY-level; defer detailed HOW to `/plan`).
   - **Acceptance criteria** — verifiable, measurable; always include "`dotnet build` & `dotnet test` green"
     and "conventions preserved (CLAUDE.md), no scope creep".
   - **Risks / notes** — contract changes, backward compatibility, migrations.

4. **Quality gate.** Before finishing, validate the spec:
   - No implementation leakage in Problem/Scope (HOW belongs in `/plan`).
   - Requirements testable and unambiguous; scope clearly bounded.
   - Make informed guesses for unspecified details and record them under Risks/notes.
   - **Limit clarifying questions to the 3 most critical** (scope > security > UX > details). If
     more arise, pick reasonable defaults and document them.

5. **Update the backlog index** ([docs/fixes/README.md](../../docs/fixes/README.md) for fixes):
   add the row with status 📝.

6. **Report**: the spec path, classification, ID, and the next step (`/plan`, or `/advisors`
   to stress-test the spec first).

## Guidelines

- Focus on **WHAT** users need and **WHY**. Avoid tech stack / code structure here.
- Acceptance criteria must be verifiable without dictating implementation.
- Defer to [CLAUDE.md](../../CLAUDE.md) for conventions and the "definition of done".
- This command writes a spec only — it does not implement.
