# FEAT-001 — Identity/Users — Uygulama Planı

> İlgili spec: [spec.md](spec.md). **NASIL** + fazlı uygulama. Her faz kendi branch'i + MR'ı olur (epic, tek MR değil).

## Gap Analizi Düzeltmeleri (mevcut kod incelemesi)
Kodu inceleyince plana eklenen/düzeltilen noktalar:
- **G1 — OrderStatus enum yetersiz.** Mevcut: `Draft, Pending, Completed, Cancelled`. Yaşam döngüsü için **genişletilecek**: `Pending, Confirmed, Processing, Shipped, Delivered, Cancelled, Failed, ReturnRequested, Returned`. `Draft`/`Completed` kullanımları map'lenir; EF `Status` (string) dönüşümü + **migration** gerekir.
- **G2 — `BasketCheckoutEvent` kontratı DEĞİŞMEZ (additive gereksiz).** Event zaten `CustomerId (Guid)` + `UserName` taşıyor. Plan'daki "yeni `UserId` alanı" **iptal**. Bunun yerine `CustomerId = Guid(sub)` ve `UserName = preferred_username/email` artık **client'tan değil token'dan** doldurulur — şema aynı, **kaynak** değişir (kırıcı değişiklik yok). `CheckoutBasketCommandValidator`'daki `CustomerId`/`UserName` client-zorunluluğu kaldırılır; sunucu set eder.
- **G3 — Basket route'ları `{userName}` parametresini bırakır.** Kimlik token `sub`'tan; `GET /basket`, `DELETE /basket`, `/basket-store`, `/basket/checkout` `sub`'ı token'dan alır; `ShoppingCard.UserName = sub` sunucu tarafında set edilir (client'a güvenilmez). Gateway route'ları güncellenir.
- **G4 — Order `CustomerId = Guid(sub)` + snapshot.** Consumer `CustomerId`'yi event'ten alır; ad/e-posta snapshot'ı event'in `FirstName/LastName/EmailAddress` alanlarından (`Customer.Create(id,name,email)` yeterli). **Varsayım:** Keycloak `sub` = UUID → `Guid.Parse` edilebilir.
- **G5 — `StoreBasketEndpoints` metadata hatalı** (`WithName("CreateProduct")`/"Create Product") — Basket auth değişiklikleriyle düzeltilecek.

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

### 4. Basket — `sub`'a taşıma + auth'lu checkout (bkz. G2/G3/G5)
- Route'lar `{userName}` parametresini bırakır; kimlik token `sub`'tan. `ShoppingCard.UserName = sub` sunucu set eder. Checkout/store `RequireAuthorization("customer")`.
- Misafir sepeti: anonim sepet + login'de merge (O5 önerisi).
- **`BasketCheckoutEvent` şeması değişmez:** `CheckoutBasketHandler` `CustomerId=Guid(sub)` ve `UserName`'i token'dan doldurur; validator'dan client-zorunlulukları kaldırılır.
- `StoreBasketEndpoints` metadata hatası düzeltilir (G5).

### 5. Order — CustomerId=sub + durum makinesi + geçmiş (bkz. G1/G4)
- `CustomerId = Guid(sub)` + ad/e-posta **snapshot** (event alanlarından). Checkout consumer map'ler.
- **OrderStatus enum genişletilir** (G1: `Pending,Confirmed,Processing,Shipped,Delivered,Cancelled,Failed,ReturnRequested,Returned`) + EF/migration.
- **Durum makinesi** aggregate'te enforce; **`OrderStatusHistory`** child + **migration**.
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
