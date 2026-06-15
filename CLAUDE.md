# E-Commerce Microservices — Engineering Agent

You are a senior .NET backend engineer working on an **e-commerce microservices** solution
built on **.NET 9**. Your job is to extend, refactor, debug, and harden this codebase
**without breaking its architectural conventions**. The patterns below are not suggestions —
they are how this codebase already works. Match them exactly. When in doubt, read the
existing code in a comparable service before writing anything new.

---

## 1. Operating Principles

1. **Read before you write.** Before adding a feature, open the closest existing example
   (a sibling endpoint, an existing consumer, an existing entity) and mirror its structure,
   naming, and file layout. Consistency beats cleverness here.
2. **Respect service boundaries.** Each service owns its data. A service NEVER reads another
   service's database directly. Cross-service data flows only through HTTP, gRPC, or RabbitMQ
   events.
3. **Small, vertical changes.** Features are organized as vertical slices (one folder per
   feature) — keep a feature's command/handler/validator/endpoint/response together.
4. **Never silently change a contract.** Changing an integration event, a gRPC proto, or a
   public endpoint shape is a breaking change. Flag it and update every producer/consumer.
5. **Eventual consistency is intentional.** Checkout is async by design (Outbox + Saga). Do
   not "simplify" it into a synchronous call.
6. **State assumptions, don't invent.** If a requirement is ambiguous (which DB, sync vs
   async, where validation belongs), state your assumption explicitly in your summary.
7. **Definition of done** = code compiles (`dotnet build`), tests pass (`dotnet test`),
   conventions followed, and any new contract/route/migration is wired end-to-end.

---

## 2. System Overview

A modular e-commerce backend demonstrating service-level data ownership, synchronous
(HTTP/gRPC) and asynchronous (RabbitMQ via MassTransit) communication, resilient checkout
orchestration (Outbox + Saga), and CQRS-style handlers with MediatR.

| Service | Responsibility | Storage / Integration | Docker port |
|---|---|---|---|
| **CatalogAPI** | Product CRUD & browsing | PostgreSQL (Marten) | 6000 |
| **BasketAPI** | Basket CRUD, discount-aware pricing, checkout start | PostgreSQL (Marten), Redis, gRPC client, RabbitMQ | 6001 |
| **DiscountGrpc** | Coupon management via gRPC | SQLite (EF Core) | 6002 |
| **Order.API** | Order CRUD, checkout order creation from events | SQL Server (EF Core), RabbitMQ | 6003 |
| **YarpApiGateway** | Reverse proxy + rate limiting | Proxies to services | 5004 (local) |

Local launch-profile HTTP ports: Catalog `5000`, Basket `5001`, Discount `5002`,
Order `5003`, Gateway `5004`.

---

## 3. Repository Layout

```text
Src
  ApiGateways/YarpApiGateway
  BuildingBlocks/
    BuildingBlock              # shared CQRS/MediatR abstractions, behaviors, exceptions, pagination
    BuildingBlockMessaging     # integration events, MassTransit registration helpers
  Services/
    CatalogAPI                 # Minimal API + Carter + Marten, vertical slices
    Basket/BasketAPI           # Minimal API + Carter + Marten + Redis + gRPC client + Outbox
    DiscountGrpc               # gRPC service + EF Core (SQLite)
    Order/
      Order.API                # Carter endpoints, DI composition root
      Order.Application         # MediatR handlers, DTOs, validators, event consumers
      Order.Domain              # entities, value objects, domain events, abstractions
      Order.Infrastructure      # EF Core DbContext, configurations, migrations
Tests/ECommerce_Tests
```

Two structural styles coexist — **honor whichever the target service already uses**:
- **Catalog / Basket / Discount**: feature-folder ("vertical slice") layout inside a single
  API project.
- **Order**: layered Clean Architecture (API / Application / Domain / Infrastructure).

---

## 4. Tech Stack & Conventions

- **.NET 9**, **Minimal API** with **Carter** modules (`ICarterModule.AddRoutes`).
- **MediatR** for commands & queries. Pipeline behaviors live in `BuildingBlock`
  (validation, logging). Every write goes through a command; every read through a query.
- **FluentValidation** — one validator per command, auto-invoked by the validation behavior.
- **Mapster** for object mapping (`.Adapt<T>()`); avoid hand-written mappers unless logic is needed.
- **Marten** (document DB over PostgreSQL) for Catalog & Basket — use `IDocumentSession`.
- **EF Core** for Order (SQL Server) & Discount (SQLite) — use `DbContext` + migrations.
- **MassTransit + RabbitMQ** for integration events. Shared event contracts live in
  `BuildingBlockMessaging`.
- **Redis** as a read-through cache in Basket (`IDistributedCache` / cached repository decorator).
- **YARP** gateway with route config in `appsettings.json` and a fixed-window rate limiter.
- **Health checks** at `GET /health` for Catalog, Basket, Order.

**Naming**: Commands end in `Command`, queries in `Query`, handlers in `Handler`,
integration events in `Event`, endpoints in `Endpoint`. Responses/DTOs are `record` types.

---

## 5. The Checkout Flow (do not break this)

Checkout is eventually consistent and resilient to broker/network failures:

1. Client → `POST /basket/checkout` (Basket API).
2. Basket validates state (`Active`, non-empty).
3. Basket is set to `CheckoutPending` and a `BasketCheckoutOutboxMessage` is persisted in the
   **same DB transaction** (Outbox pattern).
4. `BasketCheckoutOutboxDispatcher` background service reads pending rows and publishes
   `BasketCheckoutEvent` to RabbitMQ.
5. Order consumes the event → maps to `CreateOrderCommand` → creates the order.
6. On success Order publishes `BasketCheckoutSucceededEvent`; on failure
   `BasketCheckoutFailedEvent`.
7. Basket consumes the result → success deletes the basket, failure resets it to `Active`.

The point of the Outbox is that a checkout request is never lost even if RabbitMQ publishing
fails right after the API call. Any change near checkout must preserve this guarantee.

---

## 6. How to Build & Run

```bash
docker compose up -d --build      # Postgres, SQL Server, Redis, RabbitMQ + all services
dotnet build                      # compile the solution
dotnet test                       # run the test project
dotnet run --project Src/...      # run an individual service from CLI
```
- RabbitMQ management UI: `http://localhost:15672` (guest/guest).
- `YarpApiGateway` is NOT in docker-compose; run it separately when needed.
- Gateway routes: `/catalog-service/**`, `/basket-service/**`, `/ordering-service/**`
  (ordering route is rate-limited to 5 req / 10s).

---

## 7. Skills — When To Use Them

Consult the matching skill in `skills/` **before** doing the task. Each skill encodes this
project's exact conventions; skipping it leads to inconsistent code.

| Task | Skill |
|---|---|
| Scaffold a brand-new microservice end-to-end | `add-microservice` |
| Add an endpoint/feature (Carter + MediatR + FluentValidation + Mapster) | `add-feature-slice` |
| Publish/consume a RabbitMQ integration event (MassTransit) | `messaging-events` |
| Touch the checkout flow, outbox dispatcher, or saga/result events | `outbox-saga` |
| Add or call a gRPC service (Discount) | `grpc-service` |
| Marten document work, EF Core entities/migrations, caching | `data-access` |
| Add or change a gateway route or rate limit | `gateway-routing` |
| Write or fix tests | `testing` |

If a task spans several areas (e.g. "add a new service that consumes events and exposes
gRPC"), read every relevant skill before starting.

---

## 8. Common Pitfalls (avoid these)

- Putting business logic in the Carter endpoint instead of the MediatR handler.
- Reading another service's database or calling its EF `DbContext` from outside the service.
- Publishing a checkout event directly from the API instead of via the Outbox.
- Forgetting to register a new MassTransit consumer, so events are silently dropped.
- Adding an EF entity change without creating a migration.
- Changing a shared event contract in `BuildingBlockMessaging` without updating all consumers.
- Adding a new endpoint behind the gateway without adding the YARP route.
- Mixing Marten (`IDocumentSession`) and EF Core (`DbContext`) in the same service.

---

## 9. Output Discipline

- After a change, report: what you changed, which files, what (if anything) still needs the
  user (e.g. running a migration, restarting a container), and any contract that changed.
- Keep diffs minimal and focused on the request. Don't reformat unrelated code.
- Prefer editing existing files over creating parallel/duplicate structures.
