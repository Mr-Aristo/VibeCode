---
name: testing
description: >
  Use this skill whenever writing, fixing, or extending tests for this e-commerce solution, or
  when a code change should be covered by tests. Trigger it for "write tests for X", "add a unit
  test", "the test is failing", "test the handler/consumer/endpoint", "increase coverage", or
  any time you finish a feature and need to verify it the project's way. Tests live in
  Tests/ECommerce_Tests and run with `dotnet test`. This skill encodes what to test at each
  layer (handlers, validators, consumers, domain) and how to keep tests fast and isolated from
  real infrastructure.
---

# Testing

The test project is `Tests/ECommerce_Tests`, run with `dotnet test`. Before writing new tests,
open an existing test class to match the framework, naming, and assertion style already in use
(xUnit/NUnit + a mocking library such as Moq/NSubstitute + an assertion lib like FluentAssertions
are typical for this stack — follow whatever is present).

## What to test (by layer)

Because business logic lives in MediatR handlers and the domain — not in endpoints — the highest
value tests are at those layers.

1. **Command/query handlers** — the primary unit under test. Mock the persistence dependency
   (`IDocumentSession` for Marten services, `DbContext`/repository for EF services) and any gRPC
   client or `IPublishEndpoint`, then assert the handler's behavior and that it called the
   dependency correctly.
2. **Validators** — pure and trivial to test: feed valid and invalid commands to the
   `AbstractValidator` and assert pass/fail per rule. Cheap, high signal.
3. **Consumers** — test that an `IConsumer<TEvent>` maps the message correctly and dispatches the
   right command / produces the right result event. Especially important for the checkout result
   consumers (success → delete basket, failure → reset to Active) and idempotency.
4. **Domain** — test entity invariants, value objects, and domain events directly (no
   infrastructure needed). Fast and stable.
5. **Endpoints** — usually thin; only worth an integration test if there's routing/mapping worth
   verifying.

## Patterns

### Handler unit test (mock the dependency)
```csharp
[Fact]
public async Task Handle_StoresProduct_AndReturnsId()
{
    var session = Substitute.For<IDocumentSession>();   // or Moq Mock<IDocumentSession>
    var handler = new CreateProductHandler(session);

    var result = await handler.Handle(new CreateProductCommand("Pen", "Office", 5m), default);

    session.Received(1).Store(Arg.Any<Product>());
    await session.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
    result.Id.Should().NotBeEmpty();
}
```

### Validator test
```csharp
[Fact]
public void Validator_Rejects_NonPositivePrice()
{
    var result = new CreateProductValidator().Validate(new CreateProductCommand("Pen", "Office", 0m));
    result.IsValid.Should().BeFalse();
}
```

### Consumer test (checkout failure path)
```csharp
[Fact]
public async Task FailedConsumer_ResetsBasketToActive()
{
    // arrange repository/session mock returning a CheckoutPending basket
    // act: invoke Consume with a BasketCheckoutFailedEvent
    // assert: basket state set back to Active and persisted
}
```

## Rules
- **Isolate from real infrastructure** for unit tests — no real Postgres/SQL Server/RabbitMQ.
  Mock the abstractions. Reserve real infrastructure for explicitly-marked integration tests
  (MassTransit's in-memory test harness or Testcontainers are appropriate there).
- **One behavior per test**, descriptive name (`Method_State_ExpectedResult`).
- **Cover both paths**: happy path and the failure/validation path. For consumers, also cover
  **idempotency** (same event twice → no duplicate effect).
- Keep tests fast and deterministic — no `Thread.Sleep`, no real network, no shared mutable state.

## Definition of done for a feature
A new feature is not done until:
- [ ] Its handler has at least a happy-path and a failure/edge test.
- [ ] Its validator (if any) is tested.
- [ ] Any new consumer is tested, including idempotency.
- [ ] `dotnet test` passes locally.

## Verify
```bash
dotnet test
# or a single project / filter:
dotnet test --filter FullyQualifiedName~CreateProduct
```
