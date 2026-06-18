# FEAT-001 — Implementation Record (Phase 1 + Phase 2)

> What was actually built, branch-by-branch. Source of truth for the implemented Identity/Users feature.
> Repo: `Mr-Aristo/ECommerce_Microservices`. Build+test gate on every increment: `dotnet test` (32 pass / 1 Testcontainers skip when Docker is off).

## New services
| Service | Purpose | Storage | Notes |
|---|---|---|---|
| **Keycloak** (container) | Identity provider (OIDC/JWT) | — | Realm `eshop` imported (`keycloak/realms/eshop-realm.json`); client `eshop-spa` (public+PKCE); roles `customer`, `support-agent`, `fulfillment-manager`, `catalog-manager`, `super-admin` |
| **UsersAPI** | Profile, addresses, favorites | PostgreSQL (Marten) | `Id = sub`; JIT provisioning on first authenticated request |
| **PaymentAPI** | Simulated payment/refund | PostgreSQL (Marten) | Mock: captures on checkout, refunds on return approval |
| **NotificationAPI** | Mock notifications (consumer-only) | — (no DB) | Consumes order/return events and logs notifications (event fan-out demo) |

## Cross-cutting (BuildingBlock)
- `AddStandardJwtAuth` — Keycloak bearer validation (Authority/Audience from config, `realm_access.roles` → role claims).
- `ClaimsPrincipalExtensions` — `GetUserId()` (sub), `GetEmail()`, `GetUserName()`.
- Auth wired into Basket, Order, UsersAPI, PaymentAPI(none), and the gateway (`UseAuthentication/UseAuthorization`).

## Phase 1 — `feat/identity-phase1-auth-profile` (merged #42)
1. Shared Keycloak JWT validation (BuildingBlock) + Keycloak in docker-compose + realm import + `Jwt__Authority` per service.
2. **UsersAPI**: `GET/PUT /users/me/profile` (JIT), `POST/GET /users/me/addresses` — all `RequireAuthorization`.
3. **Basket**: identity from token `sub` (routes dropped `{userName}`); `ShoppingCard` keyed by sub; checkout sets `CustomerId=Guid(sub)`/`UserName` from token; all endpoints `RequireAuthorization`. `BasketCheckoutEvent` schema unchanged.
4. **Order**: `OrderStatus` expanded (additive); aggregate state machine (`Confirm/Process/Ship/Deliver/Cancel`); `OrderStatusHistory` (JSON column, migration `AddOrderStatusHistory`); `GET /me/orders` (owner); admin `POST /orders/{id}/confirm|process|ship|deliver|cancel` (fulfillment-manager/super-admin); `OrderStatusChanged` integration event.
5. **Gateway**: `/users-service` route; `AuthorizationPolicy: default` on basket/ordering/users routes (edge defense-in-depth).

## Phase 2
### Favorites — `feat/identity-phase2-favorites` (merged #43)
- UsersAPI: `POST/DELETE/GET /users/me/favorites` (owner via sub, idempotent add); stored on `UserProfile`.
- Follow-up: enrich list with Catalog product summaries (needs a Catalog batch-by-ids read).

### Returns — `feat/identity-phase2-returns` (merged #44)
- Order `Return` aggregate (JSON column, migration `AddOrderReturn`).
- Customer `POST /me/orders/{id}/returns` (Delivered + 14-day window, owner-checked).
- Support `POST /orders/{id}/returns/approve|reject|refund` (support-agent/super-admin).
- Events: `ReturnRequested`, `ReturnApproved`, `OrderReturned`.

### Payment (mock) — `feat/identity-phase2-payment` (merged #45)
- **PaymentAPI** consumers: `BasketCheckoutEvent` → capture (mock); `ReturnApproved` → refund (mock) → publish `RefundCompleted`.
- Order consumes `RefundCompleted` → `CompleteRefund` → `Returned`. **Closes the return/refund loop automatically** (the manual `/returns/refund` endpoint remains as an admin fallback).
- `GET /payments/{orderId}` for visibility.

### Catalog admin authz — `feat/identity-phase2-catalog-authz` (merged #46)
- Catalog wired with `AddStandardJwtAuth` + `UseAuthentication/UseAuthorization`; `Jwt__Authority` in appsettings + compose.
- `CreateProduct`/`UpdateProduct`/`DeleteProduct` now `RequireAuthorization(RequireRole("catalog-manager","super-admin"))`. Browse/read endpoints stay anonymous.

### Notifications — `feat/identity-phase2-notifications` (merged #47)
- **NotificationAPI**: consumer-only, no DB/Carter. Consumes `OrderStatusChanged`, `ReturnRequested`, `OrderReturned` and logs mock notifications (demonstrates RabbitMQ event fan-out alongside Payment/Order consumers).

## Open decisions adopted (O1–O3)
- **O1** → mock Payment service (above). **O2** → 14-day window, manual approval; partial/per-item returns deferred. **O3** → stock stays in Catalog (restock wiring deferred to a Catalog consumer).

## Not yet done (Phase 2/3 backlog)
- Discount coupon-management authz (gRPC; Catalog product management is done).
- Stock restock on return (Catalog consumer of `ReturnApproved`).
- Favorites→Catalog enrichment; per-item partial returns; checkout-time payment authorize (currently capture-only).
- Phase 3: GDPR delete/export, carrier webhook auto-deliver, user-specific coupons, Inventory/Payment as fuller services.

## Runtime verification (pending — Docker)
`docker compose up -d --build` → Keycloak `:8088` (realm eshop), token via `eshop-spa`, then exercise gateway `:5004`: profile (JIT), basket+checkout, order lifecycle, return → auto-refund → Returned. Seq `:8081`, Aspire dashboard `:18888`.
