# FIX-030 — Uygulama Planı

> İlgili spec: [spec.md](spec.md). Bu plan yalnızca spec onaylandıktan sonra uygulanır.

## Ön Koşullar
- [ ] spec.md kabul kriterleri okundu; rota-bazlı limit değerleri kararlaştırıldı
- [ ] Güvenilen proxy aralığı belirlendi (ForwardedHeaders için)
- [ ] Çalışan dal hazır: `fix/ip-rate-limiting`
- [ ] Başlangıçta `dotnet build` + `dotnet test` yeşil (baseline)

## Adımlar (sıralı, atomik)

### Adım 1 — ForwardedHeaders
- **Dosya:** `Src/ApiGateways/YarpApiGateway/Program.cs`
- **Eylem:** `builder.Services.Configure<ForwardedHeadersOptions>(...)` (XForwardedFor) + pipeline başında `app.UseForwardedHeaders()`; `KnownProxies`/`KnownNetworks` ile güvenilen aralığı kısıtla.
- **Doğrulama:** Build yeşil; `HttpContext.Connection.RemoteIpAddress` proxy arkasında gerçek istemciyi yansıtır.

### Adım 2 — Partition'lı limiter politikaları
- **Dosya:** `Src/ApiGateways/YarpApiGateway/Program.cs`
- **Eylem:** `AddRateLimiter` içinde istemci-başı partition tanımla:
  ```csharp
  RateLimitPartition.GetFixedWindowLimiter(
      partitionKey: ctx.User.FindFirst("sub")?.Value
                    ?? ctx.Connection.RemoteIpAddress?.ToString() ?? "anon",
      _ => new FixedWindowRateLimiterOptions { Window = ..., PermitLimit = ... });
  ```
  Rota sınıfları için ayrı adlandırılmış politikalar (`checkout-strict`, `auth-moderate`, `catalog-loose`); `OnRejected` ile `429` + `Retry-After`.
- **Doğrulama:** Build yeşil.

### Adım 3 — Route'lara politika ata
- **Dosya:** `Src/ApiGateways/YarpApiGateway/appsettings.json`
- **Eylem:** İlgili route'lara `"RateLimiterPolicy"` ekle (ordering/basket/users/catalog). Healthcheck muafiyeti gerekiyorsa ayrı ele al.
- **Doğrulama:** Konfig YARP tarafından yüklenir; route eşleşmesi doğru.

### Adım 4 — Doğrulama
- **Eylem:** İki farklı IP/sub'dan istek: biri limiti tüketirken diğeri etkilenmemeli. Limit aşımında `429` + `Retry-After`.
- **Doğrulama:** Beklenen izolasyon + yanıt.

## Son Doğrulama
- [ ] `dotnet build` yeşil
- [ ] `dotnet test` yeşil
- [ ] İstemci-başı izolasyon kanıtlandı; hassas rotalar limitli; 429+Retry-After üretiliyor
- [ ] Yan etki taraması: `ordering-route` regresyonu, gateway auth pipeline sırası, healthcheck erişimi

## Rollback
Commit revert. Yalnız gateway katmanı etkilenir; servis kodu değişmez → düşük riskli geri alma.

## Done
- [ ] Backlog `README.md`'de durum ✅ yapıldı; FIX-014 bu FIX'e referans verecek şekilde güncellendi
