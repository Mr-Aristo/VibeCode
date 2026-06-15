# FIX-001 — Uygulama Planı

> İlgili spec: [spec.md](spec.md).

## Ön Koşullar
- [ ] spec.md kabul kriterleri okundu
- [ ] Başlangıçta `dotnet build` yeşil (baseline)

## Adımlar (sıralı, atomik)

### Adım 1 — Sihirli string için sabit tanımla
- **Dosya:** `Src/Services/Order/Order.Application/FeatureFlags.cs` (yeni)
- **Eylem:**
  ```csharp
  namespace Order.Application;

  public static class FeatureFlags
  {
      public const string OrderFulfillment = "OrderFulfillment";
  }
  ```
- **Doğrulama:** Derlenir.

### Adım 2 — Handler'ı sabite hizala
- **Dosya:** `Src/Services/Order/Order.Application/OrdersCQRS/EventHandlers/Domain/OrderCreateEventHandler.cs:20`
- **Eylem:** `IsEnabledAsync("OrderFullfillment")` → `IsEnabledAsync(FeatureFlags.OrderFulfillment)`.
  Satır 9'daki XML yorumdaki yazımı da düzelt.
- **Doğrulama:** Derlenir; string literal kalmadı.

### Adım 3 — appsettings'i hizala
- **Dosya:** `Src/Services/Order/Order.API/appsettings.json:11`
- **Eylem:** `"OrderFullfilment": false` → `"OrderFulfillment": false`.
- **Doğrulama:** JSON geçerli; `FeatureManagement` bölümünün altında.

### Adım 4 — docker-compose'u hizala
- **Dosya:** `docker-compose.override.yml`
- **Eylem:** `FeatureManagement__OrderFullfilment=false` → `FeatureManagement__OrderFulfillment=false`.
- **Doğrulama:** YAML geçerli.

### Adım 5 — Yan etki taraması
- **Eylem:** Repo genelinde eski yazımları ara: `OrderFullfil`, `OrderFullfillment`,
  `OrderFulfilment`. Kalan tüm referansları kanonik ada çevir.
- **Doğrulama:** Arama sonucu yalnızca `OrderFulfillment` döndürüyor.

## Son Doğrulama
- [ ] `dotnet build` yeşil
- [ ] `dotnet test` yeşil
- [ ] Config/kod/compose'da tek tip ad
- [ ] (Opsiyonel manuel) `OrderFulfillment=true` ile event yayınlanıyor

## Rollback
- Commit revert. Kontrat/şema değişikliği yok, ek adım gerekmez.

## Done
- [ ] `docs/fixes/README.md`'de FIX-001 durumu ✅
