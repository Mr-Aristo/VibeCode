# VibeCode — Setup Guide

This is the agent workspace for continuing development of the **.NET 9 e-commerce
microservices** solution. Place your project source inside this `VibeCode` folder; the agent
reads `CLAUDE.md` and the skills automatically.

## Final folder layout

```
VibeCode/
├── CLAUDE.md                         # Main agent prompt (architecture, conventions, rules)
├── SETUP.md                          # This file
├── .claude/
│   └── skills/
│       ├── add-feature-slice/SKILL.md
│       ├── add-microservice/SKILL.md
│       ├── messaging-events/SKILL.md
│       ├── outbox-saga/SKILL.md
│       ├── grpc-service/SKILL.md
│       ├── data-access/SKILL.md
│       ├── gateway-routing/SKILL.md
│       └── testing/SKILL.md
│
└── <your project goes here>          # e.g. clone/copy your repo into VibeCode/
    ├── Src/
    │   ├── ApiGateways/YarpApiGateway
    │   ├── BuildingBlocks/...
    │   └── Services/...
    ├── Tests/ECommerce_Tests
    └── docker-compose.yml
```

So the end state is: `VibeCode/CLAUDE.md`, `VibeCode/.claude/skills/...`, and your source
(its `Src/`, `Tests/`, `docker-compose.yml`, etc.) all directly under `VibeCode/`.

## The 8 skills (folder name = skill name)

| Skill name | Trigger when you want to... |
|---|---|
| `add-feature-slice` | Add an endpoint/feature (Carter + MediatR + FluentValidation + Mapster) |
| `add-microservice`  | Scaffold a brand-new service end-to-end (compose + gateway included) |
| `messaging-events`  | Publish/consume RabbitMQ events with MassTransit |
| `outbox-saga`       | Change the checkout flow / outbox dispatcher / saga result events |
| `grpc-service`      | Add or call a gRPC service (Discount) |
| `data-access`       | Marten docs, EF Core entities/migrations, Redis cache |
| `gateway-routing`   | Add/change a YARP route or rate limit |
| `testing`           | Write or fix tests |

Each skill's folder name matches the `name:` field in its `SKILL.md` frontmatter — keep them
in sync if you rename anything.

## How to use it

1. Copy your repository into `VibeCode/` (next to `CLAUDE.md`).
2. Open `VibeCode` as the working directory for your agent (Claude Code or any agent that reads
   `CLAUDE.md` + `.claude/skills/`).
3. Just describe the task in plain language ("add an endpoint to cancel an order", "make a
   Payment service that reacts to checkout"). The skill descriptions are written to auto-trigger
   on these phrasings, and `CLAUDE.md` tells the agent which skill to consult.

## Notes
- `.claude/` is the standard skills location. If your agent tool expects a different path (e.g.
  a plain `skills/` folder), just move the `skills/` directory there — the `SKILL.md` contents
  stay the same.
- If your real file tree differs from the README layout, tell the agent the actual paths once;
  the skills assume the structure described in the project README.
