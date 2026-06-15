---
name: messaging-events
description: >
  Use this skill whenever work involves asynchronous communication between services in this
  e-commerce solution — publishing or consuming RabbitMQ messages through MassTransit. Trigger
  it for requests like "publish an event when X happens", "make service Y react to Z",
  "add a consumer", "notify the other service", "send a message to the queue", or any
  cross-service flow that should NOT be a synchronous HTTP call. It encodes where integration
  event contracts live (BuildingBlockMessaging), how to define them, how to register and write
  consumers, and how to publish. Use it together with `outbox-saga` whenever the event is part
  of the checkout flow.
---

# Messaging: Integration Events (MassTransit + RabbitMQ)

Services talk asynchronously by publishing integration events to RabbitMQ. MassTransit handles
serialization, exchange/queue topology, and retries. Contracts are shared so producer and
consumer agree on shape.

## Golden rules
- **Shared contracts live in `BuildingBlockMessaging`** (e.g. an `Events/` folder). Both the
  publisher and the consumer reference this project — never redefine the same event in two
  places.
- An integration event is an immutable `record` named `...Event`. It carries data, not behavior.
- Changing an existing event's fields is a **breaking change**: update every consumer in the
  same change, or add a new event instead.
- Prefer events for cross-service reactions; use HTTP/gRPC only when you need an immediate
  synchronous response.

## Step 1 — Define the event contract

In `BuildingBlockMessaging` (mirror the existing events such as `BasketCheckoutEvent`):

```csharp
public record OrderShippedEvent
{
    public Guid OrderId { get; init; }
    public string CustomerId { get; init; } = default!;
    public DateTime ShippedAt { get; init; }
}
```
Keep it flat and serializable. Use `init` setters / positional records consistent with the
existing events in that folder.

## Step 2 — Publish the event

From a MediatR handler or background service, inject MassTransit's `IPublishEndpoint`:

```csharp
public class ShipOrderHandler(IPublishEndpoint publishEndpoint, OrderDbContext db)
    : ICommandHandler<ShipOrderCommand, ShipOrderResult>
{
    public async Task<ShipOrderResult> Handle(ShipOrderCommand cmd, CancellationToken ct)
    {
        // ... domain work + db.SaveChangesAsync(ct) ...
        await publishEndpoint.Publish(new OrderShippedEvent
        {
            OrderId = cmd.OrderId, CustomerId = cmd.CustomerId, ShippedAt = DateTime.UtcNow
        }, ct);
        return new ShipOrderResult(true);
    }
}
```
> If publishing must be atomic with a DB write (i.e. losing the event is unacceptable), do NOT
> publish inline — use the Outbox pattern instead (`outbox-saga` skill). The checkout flow does
> exactly this.

## Step 3 — Write the consumer

In the consuming service, implement `IConsumer<TEvent>`:

```csharp
public class OrderShippedConsumer(/* deps */) : IConsumer<OrderShippedEvent>
{
    public async Task Consume(ConsumeContext<OrderShippedEvent> context)
    {
        var message = context.Message;
        // idempotent handling — see Step 5
        // map to a command and dispatch via MediatR if business logic is involved:
        // await sender.Send(message.Adapt<SomeCommand>());
    }
}
```
For non-trivial logic, the consumer should map the message to a MediatR command and send it,
keeping business logic in handlers (same as the checkout flow maps `BasketCheckoutEvent` to
`CreateOrderCommand`).

## Step 4 — Register the consumer

MassTransit only delivers to **registered** consumers. Find where MassTransit is configured
(typically a `AddMessageBroker`/`AddMassTransit` extension in `BuildingBlockMessaging` or the
service's `Program.cs`) and register the consumer + endpoint:

```csharp
services.AddMassTransit(config =>
{
    config.AddConsumer<OrderShippedConsumer>();
    config.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(/* from configuration: host, vhost, guest/guest */);
        cfg.ConfigureEndpoints(ctx);   // or explicit ReceiveEndpoint
    });
});
```
**Forgetting this registration is the #1 messaging bug** — the event publishes fine but is
silently never handled.

## Step 5 — Reliability concerns (always consider)

- **Idempotency**: a consumer may receive the same message more than once. Guard against double
  processing (check if the order/state already reflects the event before acting).
- **Retries / errors**: rely on MassTransit retry/error-queue config that already exists; don't
  swallow exceptions — a thrown exception lets MassTransit retry / dead-letter.
- **Ordering**: don't assume events arrive in order across different message types.

## Step 6 — Verify

```bash
dotnet build && dotnet test
docker compose up -d   # ensure RabbitMQ (messagebroker) is up
```
Trigger the publishing action and confirm the consumer fires. Inspect queues/exchanges in the
RabbitMQ management UI at `http://localhost:15672` (guest/guest).

## Checklist
- [ ] Event is a record in `BuildingBlockMessaging`, referenced by both sides.
- [ ] Publisher uses `IPublishEndpoint` (or the Outbox if atomicity is required).
- [ ] Consumer implements `IConsumer<T>` and delegates real logic to a MediatR handler.
- [ ] Consumer is registered with MassTransit.
- [ ] Idempotency and error/retry behavior considered.
- [ ] Verified against a running RabbitMQ broker.
