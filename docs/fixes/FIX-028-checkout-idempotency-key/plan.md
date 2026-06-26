# FIX-028 — Uygulama Planı

> İlgili spec: [spec.md](spec.md). Bu plan yalnızca spec onaylandıktan sonra uygulanır.

## Ön Koşullar
- [ ] spec.md kabul kriterleri okundu; anahtarsız davranış kararı verildi (zorunlu/opsiyonel)
- [ ] Çalışan dal hazır: `fix/checkout-idempotency-key`
- [ ] Başlangıçta `dotnet build` + `dotnet test` yeşil (baseline)

## Adımlar (sıralı, atomik)

### Adım 1 — IdempotencyRecord dokümanı
- **Dosya:** `Src/Services/Basket/BasketAPI/Models/IdempotencyRecord.cs` (yeni)
- **Eylem:** `Key` (Id), `CheckoutId`, `CreatedAt`, serileştirilmiş sonuç alanlarıyla küçük bir Marten dokümanı tanımla; `Program.cs`'de `opts.Schema.For<IdempotencyRecord>()` kaydet.
- **Doğrulama:** Build yeşil; şema kaydı çakışmıyor.

### Adım 2 — Endpoint'te anahtarı oku
- **Dosya:** `Basket/CheckoutBasket/CheckoutBasketEndpoints.cs`
- **Eylem:** `HttpRequest`/`[FromHeader("Idempotency-Key")]` ile anahtarı al, `CheckoutBasketCommand`'e taşı (yoksa karara göre 400 veya türet).
- **Doğrulama:** Build yeşil; mevcut `sub` kimlik ataması korunur.

### Adım 3 — Handler'da idempotent kontrol + yarış koruması
- **Dosya:** `Basket/CheckoutBasket/CheckoutBasketHandler.cs`
- **Eylem:**
  - Handler başında anahtarla `IdempotencyRecord` ara; varsa kayıtlı sonucu döndür (yeni checkout başlatma).
  - `ShoppingCard` için Marten optimistic concurrency etkinleştir; çakışmada anlamlı hata.
  - Başarılı checkout'ta `IdempotencyRecord`'u sepet/outbox ile aynı `SaveChangesAsync` içinde yaz.
  - `:8-9` TODO yorumunu güncelle/kaldır.
- **Doğrulama:** Build yeşil.

### Adım 4 — Testler
- **Dosya:** `Tests/ECommerce_Tests/Basket/CheckoutIdempotencyTests.cs` (yeni)
- **Eylem:** (a) aynı anahtarla iki çağrı → tek outbox + aynı sonuç; (b) eşzamanlı iki checkout → tek `CheckoutPending`.
- **Doğrulama:** Testler yeşil.

## Son Doğrulama
- [ ] `dotnet build` yeşil
- [ ] `dotnet test` yeşil (yeni testler dahil)
- [ ] Outbox/Saga regresyonu yok: başarı→sepet sil, hata→`Active`
- [ ] Yan etki taraması: `BasketCheckoutEvent` kontratı, `PaymentCaptureConsumer` idempotency'si

## Rollback
Commit revert. Yeni doküman şeması migration gerektirmez (Marten); revert güvenli.

## Done
- [ ] Backlog `README.md`'de durum ✅ yapıldı
