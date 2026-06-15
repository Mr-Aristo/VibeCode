# Skill Authoring — Best Practices (condensed)

Condensed guidance for writing Claude Agent Skills in `.claude/skills/`. Source:
https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices

## Core principles

- **Concise wins.** The context window is shared. Only `name` + `description` are preloaded;
  `SKILL.md` is read when triggered, extra files on demand. Don't teach Claude what it already
  knows — every paragraph must earn its tokens.
- **Match degrees of freedom to the task.** High freedom → prose heuristics (many valid
  approaches). Medium → parameterized templates/pseudocode. Low → exact scripts ("run exactly
  this, don't modify") for fragile, high-stakes, must-be-consistent operations.

## Frontmatter

```yaml
---
name: outbox-saga            # ≤64 chars, lowercase + digits + hyphens; no "claude"/"anthropic"
description: …               # ≤1024 chars, third person, says WHAT it does AND WHEN to use it
---
```

- **`description` is the discovery mechanism.** Write in third person; include concrete triggers
  ("Use when the user mentions checkout, outbox, saga…"), not vague phrases ("helps with stuff").
- Prefer gerund or noun-phrase names (`writing-documentation`, `pdf-processing`); avoid
  `helper`/`utils`/`tools`.

## Structure (progressive disclosure)

- `SKILL.md` body **under 500 lines** — it's a table of contents pointing to detail files.
- Keep reference files **one level deep** from `SKILL.md` (deep link chains get truncated to a
  preview and Claude misses content).
- Add a short TOC to reference files longer than ~100 lines.
- Organize by domain/feature (`reference/finance.md`), not `doc1.md`.

## Workflows & feedback loops

- For complex tasks, give explicit sequential steps; for fragile ones, a checklist Claude can copy
  and tick off.
- Implement **validate → fix → repeat** loops (run a checker, fix errors, re-run) — they sharply
  improve output quality.

## Content rules

- **No time-sensitive info** ("before August 2025…"); put deprecated patterns under a collapsed
  "Old patterns" section.
- **Consistent terminology** — pick one term and use it everywhere.
- **Forward slashes** in all paths, even on Windows.
- **Don't offer too many options** — give a default + an escape hatch.
- **Fully-qualified MCP tool names** (`ServerName:tool_name`).
- **Don't assume tools are installed** — state the install step.

## Scripts (if any)

- Scripts should handle errors explicitly, not dump them on Claude.
- No "voodoo constants" — justify every value in a comment.
- Mark intent clearly: "**run** `x.py`" vs "**see** `x.py` for the algorithm".
- Use the plan → validate → execute pattern for batch/destructive operations.

## Iterate with evals

- Write evaluations **before** extensive docs: run the task without the skill, capture the gaps,
  build 3 scenarios, measure baseline, write the minimum instructions that close the gaps, iterate.
- Test on the models you'll actually use (Haiku/Sonnet/Opus differ in how much detail they need).

## Checklist

- [ ] `description` is specific, third-person, says what + when.
- [ ] `SKILL.md` under 500 lines; extra detail in one-level-deep files.
- [ ] Consistent terminology; no time-sensitive content; forward-slash paths.
- [ ] Concrete examples, not abstract ones.
- [ ] Scripts (if any) handle errors, document constants, mark run-vs-read.
- [ ] At least three eval scenarios; tested on target models.

> This project's existing skills live in [`.claude/skills/`](../skills/) and are indexed in
> [CLAUDE.md §7](../../CLAUDE.md). Follow their style when adding a new one.
