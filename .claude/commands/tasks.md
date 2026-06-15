---
description: Break an implementation plan into an ordered, atomic, checkboxed task list (tasks.md) that is immediately executable.
---

## User Input

```text
$ARGUMENTS
```

`$ARGUMENTS` should identify the target spec/plan folder. If absent, infer the most recent one with
a `plan.md` and confirm.

## Goal

Generate `tasks.md` next to the plan: a dependency-ordered, atomic task list where each task is
specific enough to execute without extra context.

## Steps

1. **Load** `plan.md` (required) and `spec.md` (for acceptance criteria). If `plan.md` is missing,
   suggest running `/plan` first.

2. **Derive tasks** from the plan's sequenced steps, touched files, contracts, data, and wiring.

3. **Write `tasks.md`** with phases:
   - **Phase 1 — Setup** (any scaffolding/prereqs).
   - **Phase 2 — Core change** (the handlers/consumers/entities/endpoints).
   - **Phase 3 — Wiring & contracts** (DI, MassTransit consumer, YARP route, migration).
   - **Phase 4 — Tests** (per the `testing` skill — handler/validator/consumer/domain).
   - **Phase 5 — Verify & document** (`dotnet build`/`dotnet test`, update backlog status,
     optionally invoke `fix-documentor`).

4. **Task format (required):**
   ```text
   - [ ] T001 [P?] Description with exact file path
   ```
   - Checkbox `- [ ]`, sequential ID (T001…), optional `[P]` if parallelizable (different files,
     no dependency on an incomplete task), clear action, **exact file path**.
   - Examples:
     - `- [ ] T001 Add FeatureFlags constant in Src/Services/Order/Order.Application/FeatureFlags.cs`
     - `- [ ] T004 [P] Add validator test in Tests/ECommerce_Tests/Order/...`

5. **Dependencies & parallelism**: add a short "Dependencies" section (which tasks block which) and
   note safe parallel groups. Tasks touching the same file must be sequential.

6. **Report**: the `tasks.md` path, total task count, parallelizable groups, and the suggested
   first executable slice. Next step: `/implement`.

## Rules

- Every task MUST have a checkbox, ID, and file path — no vague tasks.
- Keep tasks atomic: one logical change each, independently verifiable.
- Tests are part of done (CLAUDE.md §7); include them unless the spec explicitly excludes tests.
