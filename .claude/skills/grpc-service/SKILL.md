---
name: grpc-service
description: >
  Use this skill whenever work involves gRPC in this e-commerce solution — the DiscountGrpc
  service or any service that calls it (Basket calls Discount over gRPC for coupon pricing).
  Trigger it for requests like "add a gRPC method", "change the coupon service", "make Basket
  call Discount", "update the .proto", "add a new grpc client", or anything about synchronous,
  contract-first, low-latency service-to-service calls. It encodes the proto-first workflow,
  how the server implements the generated base class, and how a client is registered and
  injected. Use HTTP/events instead when you don't need an immediate synchronous response.
---

# gRPC Service & Client Integration

gRPC is used for **synchronous, contract-first** calls where the caller needs an immediate
answer. In this solution, `DiscountGrpc` exposes coupon operations and `BasketAPI` calls it to
compute discount-aware pricing. gRPC is proto-first: the `.proto` is the source of truth and
both server and client are generated from it.

## When gRPC vs messaging
- **gRPC**: caller needs the result now (e.g. "what's the discount for this product?"). Tight
  coupling on a request/response contract is acceptable.
- **Events (MassTransit)**: fire-and-react, no immediate response needed → use `messaging-events`.

## Step 1 — Edit the contract (.proto)

The `.proto` lives in `DiscountGrpc` (e.g. `Protos/discount.proto`). Mirror existing messages
and the service definition:

```proto
service DiscountProtoService {
  rpc GetDiscount (GetDiscountRequest) returns (CouponModel);
  rpc CreateDiscount (CreateDiscountRequest) returns (CouponModel);
  // add new rpc here
}

message CouponModel {
  int32 id = 1;
  string productName = 2;
  string description = 3;
  int32 amount = 4;
}
```
Rules:
- **Never reuse or renumber an existing field tag** — append new fields with new numbers.
- Adding a new `rpc` or an optional field is backward-compatible; removing/renaming is breaking.
- The `.csproj` references the proto with the correct `GrpcServices` mode (`Server` in
  DiscountGrpc, `Client` in BasketAPI). If you add a proto consumed by a new client, add the
  proto reference to that project's `.csproj` with `GrpcServices="Client"`.

## Step 2 — Build to regenerate

```bash
dotnet build
```
This regenerates the base class (`DiscountProtoService.DiscountProtoServiceBase`) and client
stubs. Don't hand-edit generated code.

## Step 3 — Implement on the server (DiscountGrpc)

Override the generated base class. Discount uses EF Core over SQLite, so inject its
`DbContext`:

```csharp
public class DiscountService(DiscountContext db, IMapper /*Mapster or AutoMapper per existing code*/ mapper)
    : DiscountProtoService.DiscountProtoServiceBase
{
    public override async Task<CouponModel> GetDiscount(GetDiscountRequest request, ServerCallContext context)
    {
        var coupon = await db.Coupons.FirstOrDefaultAsync(c => c.ProductName == request.ProductName)
                     ?? new Coupon { ProductName = "No Discount", Amount = 0, Description = "No Discount Desc" };
        return coupon.Adapt<CouponModel>();   // or the mapper the project already uses
    }
}
```
Register the gRPC service in `Program.cs` (`builder.Services.AddGrpc()` +
`app.MapGrpcService<DiscountService>()`) — follow what's already there.

## Step 4 — Call it from a client (BasketAPI)

The client is registered as a typed gRPC client pointing at the Discount address (from
configuration):

```csharp
builder.Services.AddGrpcClient<DiscountProtoService.DiscountProtoServiceClient>(o =>
{
    o.Address = new Uri(builder.Configuration["GrpcSettings:DiscountUrl"]!);
});
```
Then inject and call it (often wrapped in a small `DiscountService`/grpc-client wrapper):

```csharp
public class DiscountGrpcClient(DiscountProtoService.DiscountProtoServiceClient client)
{
    public async Task<CouponModel> GetDiscount(string productName, CancellationToken ct = default)
        => await client.GetDiscountAsync(new GetDiscountRequest { ProductName = productName }, cancellationToken: ct);
}
```
In Basket, this is used during basket storage/pricing to apply coupons before saving.

## Step 5 — Configuration & ports

- DiscountGrpc Docker port `6002`, local profile `5002`. The Basket client address comes from
  configuration (`appsettings.json` / compose env), not a hard-coded literal.
- gRPC may need HTTP/2; keep the existing Kestrel/endpoint configuration intact.

## Step 6 — Verify

```bash
dotnet build && dotnet test
docker compose up -d
```
Exercise the Basket flow that triggers a discount lookup and confirm the coupon amount is
applied. If the call fails, check: client address config, proto field tags match, and the
service is mapped in `Program.cs`.

## Checklist
- [ ] `.proto` edited without reusing/renumbering field tags; change is backward-compatible or
      flagged as breaking.
- [ ] Solution rebuilt to regenerate stubs (no hand-edited generated code).
- [ ] Server overrides the generated base class and is mapped in `Program.cs`.
- [ ] Client registered via `AddGrpcClient` with address from configuration.
- [ ] Verified end-to-end through the Basket pricing path.
