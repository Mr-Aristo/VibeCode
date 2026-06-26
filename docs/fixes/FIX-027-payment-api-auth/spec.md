# FIX-027 — PaymentAPI kimlik doğrulama & yetkilendirme

> **Durum:** 📝 Spec yazıldı, onay bekliyor · **Öncelik:** P2 (güvenlik / HIGH) · **Boyut:** S · **Bağımlılık:** FIX-013/FEAT-001 (`AddStandardJwtAuth`, ✅ mevcut)

## 1. Problem (NE)
PaymentAPI'de kimlik doğrulama altyapısı **hiç kurulmamış**. `AddStandardJwtAuth`,
`UseAuthentication`, `UseAuthorization` çağrılmıyor ve `GET /payments/{orderId}` endpoint'i
yetkilendirme taşımıyor. Ödeme/müşteri kaydı (CustomerId, Amount, Status) korumasız okunabiliyor.

## 2. Kanıt
- `Src/Services/PaymentAPI/Program.cs:26-35` — pipeline'da auth yok (`AddStandardJwtAuth` / `UseAuthentication` / `UseAuthorization` çağrısı bulunmuyor).
- `Src/Services/PaymentAPI/Endpoints/GetPayment.cs:8` — `app.MapGet("/payments/{orderId:guid}", ...)` üzerinde `RequireAuthorization` yok.
- `Src/Services/PaymentAPI/appsettings.json` — `Jwt:Authority` yapılandırması yok (diğer servislerde var).
- `docker-compose.override.yml` — `paymentapi` servisi için `Jwt__Authority` env değişkeni tanımlı değil.
- Gateway `appsettings.json` — PaymentAPI için route/cluster yok (gateway arkasında değil), fakat port `6005` host'a açık.

## 3. Etki / Neden önemli
- Ağa erişebilen herhangi biri (veya küme içi başka bir servis) `orderId` (= `CheckoutId`) bilirse ödeme kaydını **kimlik doğrulaması olmadan** okur.
- Ödeme + müşteri ilişkisi ifşası → PII/finansal veri sızıntısı.
- Hafifletici: GUID kimlik tahmini zor; tam kart/CVV saklanmıyor (`PaymentDataSanitizer` ile redakte). Yine de "kimliksiz ödeme verisi okuma" kabul edilemez.

## 4. Kapsam
**Dahil:**
- PaymentAPI'ye standart JWT auth altyapısını ekleme.
- `GET /payments/{orderId}` ucunu role kilitleme.
- `Jwt:Authority` yapılandırmasını appsettings + docker-compose'a ekleme.
**Hariç (non-goals):**
- PaymentAPI'yi gateway'in arkasına alma (ayrı kalem; FIX-022/025 ile gateway entegrasyonuna bağlı).
- Gerçek ödeme sağlayıcısı entegrasyonu (servis bilinçli olarak mock).
- NotificationAPI (mesaj-tüketen, HTTP yüzeyi yok) — bu FIX kapsamında değil.

## 5. Önerilen Çözüm (yaklaşım)
Diğer servislerle (Users/Catalog/Basket/Order) aynı paylaşılan deseni uygula:
- `Program.cs`: `builder.Services.AddStandardJwtAuth(builder.Configuration);` +
  pipeline'da `app.UseAuthentication(); app.UseAuthorization();`
- `GetPayment`: `.RequireAuthorization(p => p.RequireRole("support-agent","super-admin"))`
  (ödeme görünürlüğü destek/finans işidir; müşteriye gerekirse ayrı `/me/payments` ucu sonra eklenir — bu FIX'te değil).
- `appsettings.json`'a `"Jwt": { "Authority": "http://localhost:8088/realms/eshop" }`.
- `docker-compose.override.yml` `paymentapi` ortamına `Jwt__Authority=http://keycloak:8080/realms/eshop`.

## 6. Kabul Kriterleri
- [ ] PaymentAPI başlangıçta JWT auth kayıtlı; pipeline `UseAuthentication`/`UseAuthorization` içeriyor.
- [ ] `GET /payments/{orderId}` anonim → `401`, yanlış rol → `403`, doğru rol → `2xx`.
- [ ] `Jwt:Authority` hem appsettings hem docker-compose'da tanımlı ve diğer servislerle tutarlı.
- [ ] PaymentAPI consumer'ları (`PaymentCaptureConsumer`, `ReturnRefundConsumer`) etkilenmedi (auth yalnız HTTP yüzeyini kapsar).
- [ ] `dotnet build` ve `dotnet test` yeşil.

## 7. Riskler / Notlar
- **Kontrat değişikliği:** Daha önce anonim okuyan istemciler artık token+rol gerektirir (kasıtlı).
- `RequireHttpsMetadata=false` mevcut ortak ayardır (dev); prod TLS ayrı kalemde (FIX-014/HSTS).
- Marten/health-check yapılandırması aynen korunur; yalnızca auth eklenir.
