---
description: Turn an approved spec into an implementation plan (HOW), respecting project conventions. Self-contained — writes plan.md next to the spec.
---

## User Input

```text
$ARGUMENTS
```

`$ARGUMENTS` should identify the target spec (a FIX-NNN, a feature folder, or a path). If absent,
infer the most recently created spec under `docs/fixes/` or `docs/specs/features/` and confirm.

## Goal

Produce an **implementation plan** that describes HOW to satisfy the spec, grounded in this
project's conventions, so that `/tasks` can break it into atomic steps.

## Steps

1. **Load the spec** (`spec.md` in the target folder). If acceptance criteria are unclear, stop and
   ask — do not plan against an ambiguous spec.

2. **Locate the comparable existing pattern.** Before designing, find the closest sibling in the
   codebase (an existing endpoint, consumer, entity, migration) and the matching skill in
   `.claude/skills/`. The plan must mirror it.

3. **Write `plan.md`** next to the spec (template:
   [docs/fixes/_TEMPLATE/plan.md](../../docs/fixes/_TEMPLATE/plan.md)). Include:
   - **Technical approach** — the design, naming, and which service/layer each change lands in.
     Identify the structural style (vertical slice vs Clean Architecture) of the target service.
   - **Touched files** — concrete paths to create/edit.
   - **Contracts** — any integration event / gRPC `.proto` / endpoint shape change, with the full
     list of producers/consumers to update (flag breaking changes explicitly).
   - **Data** — entity changes and the required EF migration, or Marten schema implications.
   - **Wiring** — DI registration, MassTransit consumer registration, YARP route, health check.
   - **Sequenced steps** — ordered, atomic; each with a per-step verification.
   - **Final verification** — `dotnet build` + `dotnet test`; acceptance criteria mapping.
   - **Rollback** — how to revert; extra care if a contract/migration changed.
   - **Risks** — concurrency, backward compatibility, eventual-consistency (Outbox/Saga) impact.

4. **Constitution check** (project rules — see [CLAUDE.md](../../CLAUDE.md)). Verify the plan does not:
   - put business logic in the endpoint instead of the handler;
   - read another service's DB or call its `DbContext` from outside;
   - publish a checkout event outside the Outbox, or make checkout synchronous;
   - change a shared contract without updating all consumers;
   - add an EF entity change without a migration, or a gateway endpoint without a YARP route;
   - bump package versions (forbidden without explicit approval).
   If any rule is violated, revise the plan.

5. **Report**: the plan path and the next step (`/tasks`, or `/advisors` to review the plan).

## Guidelines

- The plan is HOW; keep WHAT/WHY in the spec.
- Prefer the smallest correct change that fits existing patterns (the Pragmatist's bar).
- If the plan grows large, note natural split points so `/tasks` can scope incrementally.
