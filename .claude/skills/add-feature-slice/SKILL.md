---
name: add-feature-slice
description: >
  Use this skill whenever adding a new endpoint, command, query, or feature to any service in
  this .NET 9 e-commerce solution (Catalog, Basket, Order, etc.). Trigger it for ANY request
  phrased as "add an endpoint", "create a GET/POST/PUT/DELETE for X", "let users do Y",
  "implement the feature that...", or any new read/write operation — even if the user does not
  say the words "CQRS", "Carter", or "MediatR". This skill encodes the project's exact vertical
  slice convention (Carter endpoint -> MediatR command/query -> FluentValidation -> Mapster ->
  Marten/EF persistence) so new code matches existing code instead of drifting.
---

# Add a Feature Slice (Endpoint + Handler)

A "feature slice" is one folder containing everything for a single operation. This is the
default unit of work for Catalog, Basket, and Discount (and `Order.Application` for Order).
Mirror the closest existing slice in the target service before writing.

## Step 0 — Locate the convention

1. Find the service's `Features/` (or feature) folder.
2. Open the most similar existing slice (a query if you're adding a query; a command if a write).
3. Note: namespace, Carter route prefix, how the handler resolves persistence (`IDocumentSession`
   for Marten / `DbContext` for EF Core), and how it returns responses.

## Step 1 — Decide command vs query

- **Writes** (create/update/delete, anything with side effects) → a `Command` returning a result record.
- **Reads** → a `Query` returning a response record. Reads must not mutate state.

## Step 2 — Create the slice files

Group these in one folder, e.g. `Features/<FeatureName>/`:

```
Features/CreateProduct/
  CreateProductCommand.cs     # command + response records, MediatR IRequest
  CreateProductHandler.cs     # IRequestHandler, business logic + persistence
  CreateProductValidator.cs   # FluentValidation rules (one per command)
  CreateProductEndpoint.cs    # Carter ICarterModule mapping HTTP -> MediatR
```

### Command + response (records)

```csharp
public record CreateProductCommand(string Name, string Category, decimal Price)
    : ICommand<CreateProductResult>;   // ICommand<T> lives in BuildingBlock (wraps IRequest<T>)

public record CreateProductResult(Guid Id);
```
For queries use the `IQuery<TResponse>` abstraction from `BuildingBlock` the same way.

### Validator (auto-invoked by the validation pipeline behavior)

```csharp
public class CreateProductValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name).NotEmpty();
        RuleFor(x => x.Price).GreaterThan(0);
    }
}
```
Do NOT call the validator manually in the handler — the `ValidationBehavior` registered in
`BuildingBlock` runs it automatically before the handler. Just write the rules.

### Handler

```csharp
public class CreateProductHandler(IDocumentSession session)   // Marten example
    : ICommandHandler<CreateProductCommand, CreateProductResult>
{
    public async Task<CreateProductResult> Handle(CreateProductCommand cmd, CancellationToken ct)
    {
        var product = cmd.Adapt<Product>();      // Mapster mapping
        session.Store(product);
        await session.SaveChangesAsync(ct);
        return new CreateProductResult(product.Id);
    }
}
```
For EF Core services (Order/Discount) inject the `DbContext`, add the entity, and
`SaveChangesAsync`. All business rules belong here — never in the endpoint.

### Carter endpoint (thin: HTTP <-> MediatR only)

```csharp
public class CreateProductEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost("/product-create", async (CreateProductRequest req, ISender sender) =>
        {
            var result = await sender.Send(req.Adapt<CreateProductCommand>());
            return Results.Created($"/products/{result.Id}", result.Adapt<CreateProductResponse>());
        })
        .WithName("CreateProduct")
        .Produces<CreateProductResponse>(StatusCodes.Status201Created)
        .ProducesProblem(StatusCodes.Status400BadRequest)
        .WithSummary("Create Product");
    }
}
```
Endpoints contain NO business logic — they map request → command, send via `ISender`, and map
the result → response. Keep request/response records separate from the command/result records
when the HTTP shape differs from the internal contract.

## Step 3 — Routing conventions

Match the existing endpoint naming style of the service (e.g. Catalog uses `/products`,
`/product-create`, `/product-update`, `/product-delete/{id}`). Don't invent a new convention.

## Step 4 — Wire it up

- Carter auto-discovers `ICarterModule`s if `AddCarter()`/`MapCarter()` is already registered
  (it is). New validators are auto-registered if `AddValidatorsFromAssembly` is used (check the
  service's `Program.cs` / DI extension and follow what's there).
- If the endpoint should be reachable through the gateway, also use the `gateway-routing` skill.

## Step 5 — Verify

```bash
dotnet build
dotnet test
```
Then sanity-check the route (locally or via gateway). Confirm validation rejects bad input
with a 400 and the happy path returns the right status code.

## Checklist
- [ ] Command/query + response are `record` types using `BuildingBlock` abstractions.
- [ ] Validator written (rules only; not called manually).
- [ ] Business logic + persistence in the handler, not the endpoint.
- [ ] Mapster used for mapping; Marten `IDocumentSession` or EF `DbContext` matches the service.
- [ ] Endpoint follows the service's existing route naming.
- [ ] Builds and tests pass.
