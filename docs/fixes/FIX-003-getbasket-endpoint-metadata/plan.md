# FIX-003 — Uygulama Planı

> İlgili spec: [spec.md](spec.md).

## Ön Koşullar
- [ ] spec.md kabul kriterleri okundu
- [ ] Başlangıçta `dotnet build` yeşil

## Adımlar (sıralı, atomik)

### Adım 1 — Metadata'yı düzelt
- **Dosya:** `Src/Services/Basket/BasketAPI/Basket/GetBasket/GetBasketEndpoints.cs:19-23`
- **Eylem:**
  - `.WithName("GetProductById")` → `.WithName("GetBasket")`
  - `.WithSummary("Get Product By Id")` → `.WithSummary("Get Basket")`
  - `.WithDescription("Get Product By Id")` → `.WithDescription("Get the basket for the given user name")`
- **Doğrulama:** Derlenir.

### Adım 2 — (Opsiyonel) yazım hatası
- **Dosya:** aynı dosya, satır 15
- **Eylem:** `var respose = ...` → `var response = ...` ve `Results.Ok(respose)` → `Results.Ok(response)`.
- **Doğrulama:** Derlenir.

### Adım 3 — Yan etki taraması
- **Eylem:** `"GetProductById"` route adına referans (`LinkGenerator`, `CreatedAtRoute`,
  `RedirectToRoute`) olup olmadığını ara. Basket içinde başka kullanım varsa kontrol et.
- **Doğrulama:** Kırılan referans yok.

## Son Doğrulama
- [ ] `dotnet build` yeşil
- [ ] `dotnet test` yeşil (`GetBasketEndpointTests` geçer)
- [ ] OpenAPI metadata doğru

## Rollback
- Commit revert. Risk yok.

## Done
- [ ] `docs/fixes/README.md`'de FIX-003 durumu ✅
