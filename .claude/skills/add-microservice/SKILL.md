---
name: add-microservice
description: >
  Use this skill when scaffolding a brand-new microservice in this e-commerce solution from
  scratch — for example "add a Payment service", "create a Shipping microservice", "we need a
  new Notification service", or any request to introduce a new bounded context that owns its own
  data and participates in the system. Trigger it even if the user only describes the
  responsibility ("a service that handles refunds") without saying "microservice". It encodes the
  project's two structural styles (vertical-slice API vs layered Clean Architecture), the
  required wiring (Carter, MediatR, FluentValidation, persistence, MassTransit, health checks),
  docker-compose entry, and gateway route, so the new service is consistent and actually
  reachable rather than a half-wired stub.
---

# Add a New Microservice

A new service must (1) own its data, (2) follow one of the project's two structural styles,
(3) be wired for the cross-cutting concerns the others use, and (4) be reachable (compose +
gateway). Half-wired services that compile but aren't registered/routed are the main failure
mode — finish all of Section 5.

## Step 1 — Define the bounded context
- Name and single responsibility (one reason to change).
- What data it owns (its own DB — never share another service's store).
- How others interact with it: HTTP, gRPC (sync), and/or RabbitMQ events (async).

## Step 2 — Choose the structure (match an existing service)
- **Simple CRUD-ish service** → single API project with **vertical slices**, like
  `CatalogAPI`/`BasketAPI`. Copy that layout.
- **Rich domain / complex rules** → layered **Clean Architecture** like `Order`:
  `X.API`, `X.Application`, `X.Domain`, `X.Infrastructure`. Copy that layout.

When unsure, default to the vertical-slice API style (lower ceremony) unless the domain is
clearly complex.

## Step 3 — Create the project(s)
Place under `Src/Services/<Name>/`. Reference the building blocks the others reference:
- `BuildingBlock` (CQRS abstractions, pipeline behaviors, exceptions, pagination).
- `BuildingBlockMessaging` (only if it publishes/consumes events).

```bash
dotnet new web -n PaymentAPI -o Src/Services/Payment/PaymentAPI
dotnet sln add Src/Services/Payment/PaymentAPI/PaymentAPI.csproj
# add references:
dotnet add Src/Services/Payment/PaymentAPI reference Src/BuildingBlocks/BuildingBlock/BuildingBlock.csproj
```

## Step 4 — Wire the cross-cutting concerns
Mirror the chosen reference service's `Program.cs` / DI extension. Typically:
- **Carter**: `AddCarter()` + `app.MapCarter()`.
- **MediatR**: register handlers from the assembly + the `BuildingBlock` pipeline behaviors
  (validation, logging).
- **FluentValidation**: `AddValidatorsFromAssembly(...)`.
- **Mapster**: scan/register mappings if used.
- **Persistence**: pick per `data-access` skill — Marten (`AddMarten`) for document data, or EF
  Core (`AddDbContext` + migrations) for relational data.
- **Messaging** (if needed): MassTransit + RabbitMQ via the shared registration (see
  `messaging-events`).
- **Health checks**: `AddHealthChecks()` + `app.MapHealthChecks("/health")` (include DB/broker
  checks like the other services).

## Step 5 — Make it reachable (do NOT skip)
1. **docker-compose.yml**: add the service (and any new infra container it needs) following the
   existing service entries. Assign the **next free port** in the 600x series (Catalog 6000,
   Basket 6001, Discount 6002, Order 6003 → new service 6004...). Wire env vars for DB/broker
   connection strings the same way as the others. Add `depends_on` for its DB/broker.
2. **Gateway**: add a YARP route so it's reachable via the gateway (see `gateway-routing`),
   e.g. `/payment-service/{**catch-all}` → the new service.
3. **launch profile**: add a local HTTP port in the 500x series for `dotnet run` debugging.

## Step 6 — Add a first feature
Use `add-feature-slice` to add at least one real endpoint (and `messaging-events` if it reacts
to events) so the service does something verifiable, not just `/health`.

## Step 7 — Verify
```bash
dotnet build
dotnet test
docker compose up -d --build
curl http://localhost:<dockerPort>/health        # direct
curl http://localhost:5004/<name>-service/health # via gateway
```

## Checklist
- [ ] Owns its own DB; no access to another service's store.
- [ ] Structure matches an existing service (vertical-slice OR layered).
- [ ] References `BuildingBlock` (and `BuildingBlockMessaging` if it uses events).
- [ ] Carter + MediatR + FluentValidation + persistence + health checks wired.
- [ ] Added to docker-compose with a unique port and correct env/depends_on.
- [ ] Gateway route added; reachable through the gateway.
- [ ] Has at least one real feature; builds, tests pass, `/health` responds.
