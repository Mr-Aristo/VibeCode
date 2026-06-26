# FIX-027 — Uygulama Planı

> İlgili spec: [spec.md](spec.md). Bu plan yalnızca spec onaylandıktan sonra uygulanır.

## Ön Koşullar
- [ ] spec.md kabul kriterleri okundu
- [ ] Çalışan dal hazır: `fix/payment-api-auth`
- [ ] Başlangıçta `dotnet build` yeşil (baseline)

## Adımlar (sıralı, atomik)

### Adım 1 — JWT auth altyapısını ekle
- **Dosya:** `Src/Services/PaymentAPI/Program.cs`
- **Eylem:**
  - `using BuildingBlock.Auth;` ekle.
  - `builder.Services.AddStandardJwtAuth(builder.Configuration);` (Marten/health kaydından sonra, `Build()`'den önce).
  - `app.Build()` sonrası pipeline'a `app.UseAuthentication(); app.UseAuthorization();` (Carter map'inden önce).
- **Doğrulama:** Build yeşil; uygulama açılışta JWKS Authority'yi çözebiliyor.

### Adım 2 — Yapılandırma anahtarlarını ekle
- **Dosya:** `Src/Services/PaymentAPI/appsettings.json`
- **Eylem:** `"Jwt": { "Authority": "http://localhost:8088/realms/eshop" }` bloğu ekle.
- **Dosya:** `docker-compose.override.yml` (`paymentapi` → `environment`)
- **Eylem:** `- Jwt__Authority=http://keycloak:8080/realms/eshop` satırını ekle (diğer servislerle aynı backchannel adresi).
- **Doğrulama:** Konfig diğer servislerle tutarlı.

### Adım 3 — Endpoint'i koru
- **Dosya:** `Src/Services/PaymentAPI/Endpoints/GetPayment.cs`
- **Eylem:** Zincire `.RequireAuthorization(p => p.RequireRole("support-agent","super-admin"))` ekle.
- **Doğrulama:** Build yeşil; uç `[Authorize]` metadata taşır.

## Son Doğrulama
- [ ] `dotnet build` yeşil
- [ ] `dotnet test` yeşil
- [ ] `GET /payments/{orderId}`: anonim → 401, yanlış rol → 403, `super-admin` → 200/404
- [ ] Yan etki taraması: consumer'lar (capture/refund) çalışıyor; health-check `/health` erişilebilir (auth gerektirmemeli)

## Rollback
Commit revert. `Jwt__Authority` env satırı da geri alınmalı.

## Done
- [ ] Backlog `README.md`'de durum ✅ yapıldı
