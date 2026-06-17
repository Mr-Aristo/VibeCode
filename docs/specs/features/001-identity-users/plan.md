# FEAT-001 — Identity/Users — Uygulama Planı

> İlgili spec: [spec.md](spec.md). **NASIL** + fazlı uygulama. Her faz kendi branch'i + MR'ı olur (epic, tek MR değil).

## Yaklaşım & Mimari Değişimler
- **Yeni servis `Users.API`** (vertical slice; Catalog/Basket gibi: Carter + MediatR + FluentValidation + Marten/Postgres + health + Serilog/OTel). `add-microservice` skill'ine göre scaffold.
- **Keycloak** docker-compose'a; tek realm `eshop`; `eshop-spa` (public, Auth Code + PKCE) + resource-server doğrulama.
- **Paylaşılan JWT doğrulama** `BuildingBlock`'ta (`AddStandardJwtAuth`) → tüm API'ler + gateway.
- **Gateway**: JWT authn + route-level authz + token forward.
- Yeni event'ler `BuildingBlockMessaging`'de (additive), Outbox üzerinden.

---

## Faz 1 — Auth + Profil + Sipariş Döngüsü (P1) — branch: `feat/identity-phase1-auth-profile`

### 1. Keycloak (altyapı)
- `docker-compose`: `keycloak` (+ realm import). `.env`'e `KEYCLOAK_ADMIN`/`KEYCLOAK_ADMIN_PASSWORD` (FIX-012 deseni).
- Realm `eshop`: client `eshop-spa` (public+PKCE), realm roller `customer`/`support-agent`/`fulfillment-manager`/`catalog-manager`/`super-admin`. Realm export `realms/eshop-realm.json` (reproducible).
- Compose'da Keycloak healthcheck + dependent'lar `service_healthy`.

### 2. Paylaşılan JWT doğrulama (`BuildingBlock`)
- Paket: `Microsoft.AspNetCore.Authentication.JwtBearer`.
- `AddStandardJwtAuth(config)`: Authority=Keycloak realm URL, JWKS, `issuer`+`audience`+`exp` doğrula, `realm_access.roles` → `ClaimsPrincipal` rollerine düzleştir. Owner authz için bir policy/helper (`userId == sub`).
- Her API: `AddStandardJwtAuth` + `UseAuthentication/UseAuthorization`. Gateway: aynı authn + route authz (YARP `AuthorizationPolicy`).

### 3. Users.API (yeni servis)
- Slice'lar: `Profile` (GET/PUT `/me/profile`), `Address` (`/me/addresses` CRUD). `UserProfile.Id = sub`.
- **JIT provisioning**: authenticated istek geldiğinde profil yoksa `sub`+`email`'den oluştur (middleware veya handler).
- Marten/Postgres (`usersdb`), health, Serilog/OTel, gateway route `/users-service/**` + `.env`/compose girişi + Dockerfile.

### 4. Basket — `sub`'a taşıma + auth'lu checkout
- Sepet anahtarı `username` → `sub` (token'dan). Checkout `RequireAuthorization("customer")`.
- Misafir sepeti: anonim sepet + login'de merge (O5 önerisi).
- **`BasketCheckoutEvent`'e `UserId` (sub) ekle — additive** (mevcut alanlar korunur). `CheckoutBasketHandler` `sub`'ı doldurur.

### 5. Order — CustomerId=sub + durum makinesi + geçmiş
- `Customer` → `CustomerId = sub` + ad/e-posta **snapshot**. Checkout consumer `UserId`→`CustomerId` map'ler.
- **Durum makinesi** (`Pending→Confirmed→Processing→Shipped→Delivered`, `Cancelled`/`Failed`) aggregate'te enforce; **`OrderStatusHistory`** child + **migration**.
- Endpoint'ler: `GET /me/orders`, `GET /me/orders/{id}` (owner); admin `POST /orders/{id}/process|ship|deliver|cancel` (rol authz). Her geçiş `OrderStatusChanged` event (Outbox, additive).

### 6. Gateway + Observability
- Yeni route'lar: `/users-service/**`; korumalı route'lara authz policy.
- Serilog/OTel: `sub`/`userId` enrichment (mevcut `ActivityEnricher`'a claim ekleme).

### 7. Test
- Unit: JWT policy/owner kuralı, durum makinesi geçiş validasyonu, JIT provisioning handler.
- Integration (Testcontainers, FIX-019 deseni): Users profil round-trip; Order durum geçişi.

**Faz 1 DoD:** build+test yeşil; `docker compose up` ile Keycloak+Users.API dahil ayağa kalkar; login→token→checkout→sipariş→durum takibi uçtan uca; korumasız erişim 401/403.

---

## Faz 2 — Favoriler + İade/Refund + Admin rolleri (P2) — branch: `feat/identity-phase2-...`
> **Ön koşul:** Açık kararlar **O1 (ödeme/refund anlamı), O2 (iade politikası), O3 (stok sahipliği)** netleşmeli.
- **Favoriler** (Users.API slice): `(userId, productId)` idempotent; listede Catalog'tan ürün özeti (read-time HTTP); kırık referans zarif.
- **İade/Refund**: Order'da `Return` aggregate + durum makinesi (Requested→UnderReview→Approved/Rejected→…→Refunded→Closed); endpoint `POST /me/orders/{id}/returns` (owner, Delivered+pencere), Support onay/red. Choreography saga: `ReturnRequested`/`ReturnApproved`/`OrderReturned`. Stok restock (O3'e göre, öneri ItemReceived). Refund (O1'e göre — Opsiyon 2: durum + manuel).
- **Admin alt-rolleri** authz matrisine göre; ürün/kupon endpoint'leri role-korumalı (Catalog/Discount).
- **Bildirim**: `OrderStatusChanged` tüketen notification süreci (opsiyonel).

## Faz 3 — İleri (P3) — ayrı branch'ler
- KVKK: hesap silme/veri indirme (Keycloak+Users.API koordinasyon saga; soft-delete+audit).
- Kargo webhook ile otomatik teslim; kullanıcıya özel kupon; Favorites/Inventory/Payment ayrı servis kararları.

---

## Kontrat Değişiklikleri
| Kontrat | Değişiklik | Uyumluluk |
|---|---|---|
| `BasketCheckoutEvent` | `+UserId (sub)` | **Additive**; versiyonla, tüm consumer'ları kontrol et |
| Yeni event'ler | `OrderStatusChanged`, (Faz2) `ReturnRequested/Approved`, `OrderReturned` | Yeni — yeni consumer'lar |
| Yeni endpoint'ler | Users.API `/me/*`; Order `/me/orders`, `/orders/{id}/...` | Gateway route + authz |
| Keycloak | Yeni altyapı (OIDC/JWKS) | — |

## Migration'lar
- **Order (SQL Server):** `OrderStatusHistory` tablosu; `Customer`→`CustomerId=sub`+snapshot alanları; (Faz2) `Return`.
- **Users.API (Postgres/Marten):** doküman şeması (migration gerekmez, Marten).

## Constitution Check (CLAUDE.md)
- [ ] İş mantığı handler'da (endpoint'te değil); her yazma command, okuma query.
- [ ] Servis sınırları korunur; `sub` HTTP/event ile taşınır, cross-DB okuma yok.
- [ ] Checkout Outbox üzerinden; senkron'a çevrilmez.
- [ ] EF entity değişimi → migration; yeni MassTransit consumer kaydı; yeni endpoint → gateway route.
- [ ] Paket sürümleri onaysız bump edilmez (yeni paketler eklenir, mevcutlar korunur).

## Riskler & Rollback
- `username→sub` migrasyonu: mevcut sepet verisi; geçiş planı (dev'de temiz başlangıç kabul edilebilir).
- Keycloak kurulum kırılganlığı: realm import + healthcheck ile reproducible.
- Rollback: faz bazlı branch revert; contract değişiklikleri additive olduğundan geri alım güvenli.

## Sonraki adım
`/tasks` ile Faz 1'i atomik göreve böl → `/implement`. Faz 2 öncesi **O1–O3** kararları alınır.
