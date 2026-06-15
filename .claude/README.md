# `.claude/` — Engineering Workspace Configuration

Project-specific Claude Code configuration for the **E-Commerce Microservices** solution
(.NET 9). Adapted from a professional reference setup, tailored to this project's conventions
(vertical slice / Clean Architecture, MediatR CQRS, Carter, Marten/EF Core, MassTransit
Outbox+Saga, gRPC) and its spec-driven workflow.

## Layout

```
.claude/
├── agents/        # Subagents (Task tool) — dev, explorer, documentor, advisor panel
├── commands/      # Slash commands — self-contained spec-driven workflow
├── docs/          # Authoring references (skills, prompting)
├── skills/        # Project skills (add-feature-slice, outbox-saga, …) — see ../CLAUDE.md §7
└── settings.json  # Permissions allow-list
```

## Agents (`agents/`)

Invoked via the Task tool (`subagent_type`). Each is a focused persona.

| Agent | Use for |
|---|---|
| [backend-dev](agents/backend-dev.md) | Implementing features/fixes/refactors in any service, following project conventions |
| [feature-explorer](agents/feature-explorer.md) | Read-only investigation: "how does X work", "where is Y", before touching code |
| [fix-documentor](agents/fix-documentor.md) | Documenting a completed feature/fix into `docs/specs` or `docs/fixes` |
| [advisor-skeptic](agents/advisor-skeptic.md) | Stress-testing a design/plan/PR for flaws and edge cases |
| [advisor-pragmatist](agents/advisor-pragmatist.md) | Scope/shippability: smallest correct change, what to cut |
| [advisor-architect](agents/advisor-architect.md) | Long-term fit: boundaries, contracts, consistency with the architecture |

The three advisors form a **board of advisors** — run them in parallel on a proposal
(see [`/advisors`](commands/advisors.md)).

## Commands (`commands/`)

A self-contained, spec-driven flow. No external scripts; everything lives in this repo's
`docs/`. Inspired by GitHub Spec Kit, adapted to our `docs/fixes` (fixes) and
`docs/specs/features` (features).

```mermaid
graph LR
    SPEC[/spec] --> PLAN[/plan] --> TASKS[/tasks] --> IMPL[/implement]
    SPEC -.review.-> ADV[/advisors]
    PLAN -.review.-> ADV
```

| Command | Purpose |
|---|---|
| [`/spec`](commands/spec.md) | Create a spec (feature → `docs/specs/features`, fix → `docs/fixes`) — what & why + acceptance criteria |
| [`/plan`](commands/plan.md) | Turn a spec into an implementation plan (how), respecting conventions |
| [`/tasks`](commands/tasks.md) | Break a plan into ordered, atomic, checkboxed tasks |
| [`/implement`](commands/implement.md) | Execute tasks step-by-step with `dotnet build`/`dotnet test` gates |
| [`/advisors`](commands/advisors.md) | Run the advisor panel in parallel on a spec/plan/PR/diff |

## Conventions

- Authoritative project rules live in [../CLAUDE.md](../CLAUDE.md). Agents and commands defer to it.
- Architecture reference: [../docs/architecture/](../docs/architecture/) (TR) and
  [../docs/architecture_en/](../docs/architecture_en/) (EN).
- Spec-driven fixes workflow: [../docs/fixes/README.md](../docs/fixes/README.md).
