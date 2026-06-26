# FIX-026 — Uygulama Planı

> İlgili spec: [spec.md](spec.md). Bu plan yalnızca spec onaylandıktan sonra uygulanır.

## Ön Koşullar
- [ ] spec.md kabul kriterleri okundu ve `POST /orders` kararı verildi (kaldır / super-admin)
- [ ] Çalışan dal hazır: `fix/order-endpoint-authorization`
- [ ] Başlangıçta `dotnet build` yeşil (baseline)

## Adımlar (sıralı, atomik)

### Adım 1 — Okuma uçlarını koru
- **Dosya:** `Order.API/Endpoints/GetOrders.cs`, `GetOrdersByCustomer.cs`, `GetOrdersByName.cs`
- **Eylem:** Her endpoint zincirine ekle:
  `.RequireAuthorization(p => p.RequireRole("fulfillment-manager","support-agent","super-admin"))`
- **Doğrulama:** Build yeşil; uçlar artık `[Authorize]` metadata taşır.

### Adım 2 — Yazma uçlarını koru
- **Dosya:** `Order.API/Endpoints/UpdateOrder.cs`, `DeleteOrder.cs`
- **Eylem:**
  - `PUT /orders` → `.RequireAuthorization(p => p.RequireRole("fulfillment-manager","super-admin"))`
  - `DELETE /orders/{id}` → `.RequireAuthorization(p => p.RequireRole("super-admin"))`
- **Doğrulama:** Build yeşil.

### Adım 3 — `POST /orders` kararını uygula
- **Dosya:** `Order.API/Endpoints/CreateOrder.cs`
- **Eylem:** Karara göre ya endpoint'i kaldır (saga consumer'ı oluşturmaya devam eder) ya da
  `.RequireAuthorization(p => p.RequireRole("super-admin"))` ekle.
- **Doğrulama:** Kaldırıldıysa `BasketCheckoutEventHandler` → `CreateOrderCommand` yolu bozulmadı (consumer doğrudan MediatR kullanır, HTTP'ye bağlı değil).

### Adım 4 — Regresyon kontrolü
- **Eylem:** `MyOrders`, `Returns` (müşteri), `ManageOrderStatus` (admin) uçlarının değişmediğini doğrula.
- **Doğrulama:** Bu uçların testleri (varsa) yeşil.

## Son Doğrulama
- [ ] `dotnet build` yeşil
- [ ] `dotnet test` yeşil
- [ ] Manuel/otomatik: anonim → 401, yanlış rol → 403, doğru rol → 2xx (en az 1 oku + 1 yaz ucu için)
- [ ] Yan etki taraması: gateway `ordering-route` davranışı, Swagger/OpenAPI metadata, varsa Order endpoint testleri

## Rollback
Commit revert. Kontrat (auth gereksinimi) değişikliği olduğu için, revert sonrası uçlar tekrar anonim olur — geçici durum dokümante edilmeli.

## Done
- [ ] Backlog `README.md`'de durum ✅ yapıldı
