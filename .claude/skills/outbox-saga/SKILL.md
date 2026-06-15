---
name: outbox-saga
description: >
  Use this skill for ANY change that touches the checkout flow or its reliability machinery in
  this e-commerce solution: the basket checkout endpoint, the BasketCheckoutOutboxMessage, the
  BasketCheckoutOutboxDispatcher background service, the BasketCheckoutEvent, the order-creation
  consumer in Order, or the success/failure result events that finalize basket state. Trigger it
  whenever the user mentions "checkout", "outbox", "saga", "make checkout reliable", "what
  happens if the broker is down", "order isn't being created after checkout", or wants to add a
  new step to the eventually-consistent purchase pipeline. This flow is deliberately async and
  fault-tolerant — this skill exists so you extend it without breaking that guarantee.
---

# Outbox + Saga: The Resilient Checkout Flow

Checkout is **eventually consistent** and survives temporary broker/network failures. The core
trick: the basket state change and the "I need to publish an event" record are written in the
**same database transaction**, then a background dispatcher publishes the event separately. This
means a checkout is never lost even if RabbitMQ is unreachable at the moment of the API call.

## The flow (memorize this)

```
POST /basket/checkout (Basket API)
  └─ validate basket (Active, non-empty)
  └─ set basket -> CheckoutPending
  └─ store BasketCheckoutOutboxMessage      ← SAME transaction as the state change
        │
BasketCheckoutOutboxDispatcher (background service in Basket)
  └─ reads pending outbox rows
  └─ publishes BasketCheckoutEvent to RabbitMQ
  └─ marks the outbox row as processed
        │
Order service consumer
  └─ consumes BasketCheckoutEvent -> CreateOrderCommand -> creates order
  └─ success -> publish BasketCheckoutSucceededEvent
  └─ failure -> publish BasketCheckoutFailedEvent
        │
Basket service result consumer
  └─ success -> delete basket
  └─ failure -> set basket back to Active
```

## Why each piece exists (don't "optimize" them away)
- **Outbox row in the same transaction**: guarantees that if the basket was marked pending, the
  intent to publish is durably recorded too. No lost checkouts.
- **Dispatcher background service**: decouples publishing from the request; if the broker is
  down, the row stays pending and is retried on the next poll.
- **Result events (succeeded/failed)**: this is the saga's compensation — failure must reset the
  basket to `Active` so the customer can retry; success cleans it up.

If asked to "make checkout simpler" or "just call Order directly", explain that this would
reintroduce the lost-update / lost-message problem the design prevents. Offer to extend it
safely instead.

## Adding a new step to the saga

When the purchase pipeline needs a new stage (e.g. payment, inventory reservation):

1. Decide whether the new step belongs **before** order creation (a new gate in the saga) or
   **after** (a reaction to success). Most additions are reactions to existing events.
2. Define the new event contract(s) in `BuildingBlockMessaging` (see `messaging-events`).
3. If the new step's success/failure must be able to compensate (undo) earlier steps, add a
   corresponding failure event and a consumer that performs the compensation (mirror how
   `BasketCheckoutFailedEvent` resets the basket).
4. Keep each consumer **idempotent** — the same event may be delivered more than once.

## Working on the Outbox dispatcher

- It is a hosted/background service that **polls** for unprocessed rows. Preserve: pick up only
  unprocessed rows, publish, then atomically mark processed; on publish failure, leave the row
  for the next cycle (don't mark it processed).
- Don't delete a row before publish is confirmed.
- Keep the poll resilient: a single bad row shouldn't stop the whole loop — log and continue.

## Working on the order-creation consumer (Order service)

- It maps `BasketCheckoutEvent` → `CreateOrderCommand` and runs it through MediatR.
- On success it must publish `BasketCheckoutSucceededEvent`; on any failure
  `BasketCheckoutFailedEvent`. Never let an exception escape without emitting a result event,
  or the basket will be stuck in `CheckoutPending` forever.
- Make order creation idempotent (e.g. keyed on the basket/checkout id) so a redelivered event
  doesn't create duplicate orders.

## Basket result consumer

- `BasketCheckoutSucceededEvent` → delete basket.
- `BasketCheckoutFailedEvent` → set basket back to `Active`.
- Both must tolerate the basket already being in the target state (idempotency).

## Verify

```bash
dotnet build && dotnet test
docker compose up -d
```
Manual end-to-end test:
1. `POST /basket-store` to create a basket.
2. `POST /basket/checkout`.
3. Confirm an order appears via `GET /orders/by-customer/{customerId}` (allow for async delay).
4. Failure path: force the order consumer to fail and confirm the basket returns to `Active`.
5. Resilience: stop RabbitMQ, call checkout, confirm the outbox row stays pending and is
   published once RabbitMQ is back.

## Checklist
- [ ] Basket state change + outbox row are in ONE transaction.
- [ ] Dispatcher only marks rows processed after a confirmed publish.
- [ ] Order consumer always emits a success OR failure result event.
- [ ] Failure path compensates (basket → Active); success path cleans up (basket deleted).
- [ ] All consumers are idempotent.
- [ ] Verified happy path, failure path, and broker-down resilience.
