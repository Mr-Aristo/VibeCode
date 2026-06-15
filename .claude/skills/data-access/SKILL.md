---
name: data-access
description: >
  Use this skill for any persistence or data-access work in this e-commerce solution: storing or
  querying documents with Marten (Catalog, Basket on PostgreSQL), adding or changing EF Core
  entities and creating migrations (Order on SQL Server, Discount on SQLite), or caching basket
  reads with Redis. Trigger it for requests like "add a field to the product/order/coupon",
  "create a migration", "the schema needs to change", "query products by X", "cache the basket",
  "why isn't my entity saving", or any DB read/write. It encodes which service uses which
  persistence technology and the exact commands and patterns so you don't mix Marten and EF Core
  or forget a migration.
---

# Data Access: Marten, EF Core & Redis

Each service owns its own database and uses ONE persistence technology. Never mix them in a
single service, and never read another service's database.

| Service | Tech | Store | Access type |
|---|---|---|---|
| Catalog | Marten | PostgreSQL | `IDocumentSession` |
| Basket | Marten (+ Redis cache) | PostgreSQL + Redis | `IDocumentSession`, `IDistributedCache` |
| Order | EF Core | SQL Server | `DbContext` + migrations |
| Discount | EF Core | SQLite | `DbContext` + migrations |

First, identify the target service, then follow the matching section.

---

## A. Marten (Catalog, Basket)

Marten is a document database over PostgreSQL — you store and load whole objects (aggregates),
no migrations needed for shape changes (it's schemaless JSONB).

**Store / update:**
```csharp
public class StoreBasketHandler(IDocumentSession session) : ICommandHandler<StoreBasketCommand, StoreBasketResult>
{
    public async Task<StoreBasketResult> Handle(StoreBasketCommand cmd, CancellationToken ct)
    {
        session.Store(cmd.Cart);            // upsert by identity (e.g. UserName)
        await session.SaveChangesAsync(ct);
        return new StoreBasketResult(cmd.Cart.UserName);
    }
}
```

**Load / query:**
```csharp
var cart = await session.LoadAsync<ShoppingCart>(userName, ct);          // by id
var products = await session.Query<Product>()
                            .Where(p => p.Category.Contains(category))
                            .ToListAsync(ct);
```

Notes:
- The document identity is configured where Marten is registered (`AddMarten(...)`) — e.g.
  Basket carts keyed by `UserName`. Follow the existing identity config.
- Adding a field to a Marten document needs **no migration** — old documents simply lack it
  (defaults apply). For backfills, write a one-off script if needed.
- Pagination/filtering helpers may live in `BuildingBlock`; reuse them.

---

## B. EF Core (Order, Discount)

EF Core is relational — **every schema change requires a migration**.

**1. Change/add the entity** (Order entities live in `Order.Domain`; EF config &
`OrderDbContext` in `Order.Infrastructure`):
```csharp
public class Order : Aggregate<OrderId>   // follow the existing base type
{
    public CustomerId CustomerId { get; private set; } = default!;
    public OrderName OrderName { get; private set; } = default!;
    // new property:
    public string? TrackingNumber { get; private set; }
}
```
Keep value objects / strongly-typed ids consistent with the existing domain model. Configure new
properties in the corresponding `IEntityTypeConfiguration<T>` if conventions don't cover them.

**2. Create the migration** (run from the directory containing the DbContext / startup; adjust
`--project`/`--startup-project` to the service):
```bash
dotnet ef migrations add AddTrackingNumberToOrder \
  --project Src/Services/Order/Order.Infrastructure \
  --startup-project Src/Services/Order/Order.API \
  --output-dir Migrations
```
For Discount, target the DiscountGrpc project (SQLite).

**3. Apply it.** Migrations may be applied automatically at startup (a `Database.Migrate()` /
migration hosted service) — check the service's startup. If so, just restart the service /
container. Otherwise:
```bash
dotnet ef database update --project ... --startup-project ...
```

Notes:
- Never edit an already-applied migration; create a new one.
- Inspect the generated migration before applying — EF sometimes infers unwanted drops.
- For SQL Server in Docker, ensure the `orderdb` container is up first; for SQLite the file is
  local to the Discount service.

---

## C. Redis cache (Basket)

Basket caches reads to avoid hitting Postgres on every basket fetch (often via a cached
repository decorator over `IBasketRepository`, using `IDistributedCache`).

```csharp
public class CachedBasketRepository(IBasketRepository inner, IDistributedCache cache) : IBasketRepository
{
    public async Task<ShoppingCart> GetBasket(string userName, CancellationToken ct = default)
    {
        var cached = await cache.GetStringAsync(userName, ct);
        if (!string.IsNullOrEmpty(cached))
            return JsonSerializer.Deserialize<ShoppingCart>(cached)!;

        var basket = await inner.GetBasket(userName, ct);
        await cache.SetStringAsync(userName, JsonSerializer.Serialize(basket), ct);
        return basket;
    }
    // Store/Delete must update or invalidate the cache entry too.
}
```
Cache-consistency rule: **any write (store/delete) must update or invalidate the cached entry**,
or clients will read stale baskets. Keep the cache key scheme consistent with existing code.

---

## Verify
```bash
dotnet build && dotnet test
docker compose up -d        # bring up the relevant db (catalogdb/basketdb/orderdb) + redis
```
Then exercise a read and a write path and confirm persistence (and, for Basket, that the cache
is updated on write).

## Checklist
- [ ] Correct tech for the service (Marten vs EF Core); no mixing.
- [ ] Marten: identity/registration respected; no needless migration.
- [ ] EF Core: entity changed AND migration created, reviewed, and applied/auto-applied.
- [ ] Redis: writes invalidate/update the cache; key scheme consistent.
- [ ] No cross-service database access.
- [ ] Builds and tests pass.
