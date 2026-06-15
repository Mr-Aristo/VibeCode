---
description: Execute the task list (tasks.md) step-by-step with build/test gates, on a feature branch, following project conventions.
---

## User Input

```text
$ARGUMENTS
```

`$ARGUMENTS` should identify the target spec/plan folder. If absent, infer the most recent one with
a `tasks.md` and confirm.

## Goal

Implement the tasks safely and incrementally, keeping the build and tests green, and produce a
focused, reviewable change.

## Steps

1. **Load** `tasks.md` (required), `plan.md`, and `spec.md`. If `tasks.md` is missing, suggest
   `/tasks` first.

2. **Baseline**: run `dotnet build` and `dotnet test` on the solution
   (`ECommerce_Microservices/ECommerce_Microservices.sln`). Record any **pre-existing** failures so
   they aren't blamed on this change.

3. **Branch** (via the `git-github` skill): create `fix/<scope>-<slug>` or `feat/<scope>-<slug>`
   off the latest base. Never work on `main`/`master` directly unless the user explicitly says so.

4. **Execute phase by phase**, respecting dependencies:
   - Complete each phase before the next. Run `[P]` tasks together only when they touch different
     files. Tasks on the same file run sequentially.
   - After each task, do its per-task verification and **mark it `[x]` in `tasks.md`**.
   - Follow [CLAUDE.md](../../CLAUDE.md): business logic in handlers, validation in validators,
     contracts updated end-to-end, migrations for EF changes, consumer/route registration, no
     package bumps without approval.

5. **Gates**:
   - After the core change: `dotnet build` must be green.
   - After tests: `dotnet test` must be green (modulo the recorded pre-existing failures — surface
     those honestly, don't hide them).
   - If a task fails, halt non-parallel work, diagnose the root cause (don't paper over it), fix,
     re-run.

6. **Finish**:
   - Update the backlog status in [docs/fixes/README.md](../../docs/fixes/README.md) (or feature
     index). Optionally invoke the `fix-documentor` agent.
   - Commit with a Conventional Commit (`type(scope): summary`), push the branch (the project uses
     SSH + PR-via-UI unless a token is configured — see the `git-github` skill).
   - Report: branch, build/test results (incl. pre-existing failures), commit, and the PR link/step.

## Rules

- Keep the diff minimal and focused on the tasks; don't reformat unrelated code.
- Report outcomes faithfully: if tests fail, say so with the output; if a step was skipped, say so.
- Definition of done = builds, tests pass, conventions followed, new contract/route/migration wired
  end-to-end (CLAUDE.md §1.7).
