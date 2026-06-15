# FIX-004 — Uygulama Planı

> İlgili spec: [spec.md](spec.md).

## Ön Koşullar
- [ ] spec.md kabul kriterleri okundu
- [ ] Başlangıçta `dotnet build` yeşil

## Adımlar (sıralı, atomik)

### Adım 1 — Güvenli deserialize yardımcı metodu
- **Dosya:** `Src/Services/Basket/BasketAPI/Data/CachedBasketRepository.cs`
- **Eylem:** `JsonException`'ı yakalayan private bir yardımcı ekle:
  ```csharp
  private static ShoppingCard? TryDeserialize(string json)
  {
      try { return JsonSerializer.Deserialize<ShoppingCard>(json); }
      catch (JsonException) { return null; }
  }
  ```
- **Doğrulama:** Derlenir.

### Adım 2 — `GetBasket`'i graceful fallback ile güncelle
- **Dosya:** aynı dosya, `GetBasket` metodu
- **Eylem:** spec'teki akışı uygula: geçerli ise döndür; bozuk/null ise `cache.RemoveAsync` +
  repository'ye düş + cache'i yeniden doldur.
- **Doğrulama:** Derlenir; metod artık non-null döndürmeyi garanti eder.

### Adım 3 — (Opsiyonel) birim testi
- **Dosya:** `Tests/ECommerce_Tests/Basket/CachedBasketRepositoryTests.cs` (yeni)
- **Eylem:** `IDistributedCache` mock'u ile üç senaryo: geçerli cache, boş cache, bozuk cache.
- **Doğrulama:** Testler geçer.

## Son Doğrulama
- [ ] `dotnet build` yeşil
- [ ] `dotnet test` yeşil
- [ ] Bozuk cache senaryosu NRE atmıyor (test veya manuel)

## Rollback
- Commit revert. Risk yok (davranışsal olarak yalnızca hata yolunu iyileştirir).

## Done
- [ ] `docs/fixes/README.md`'de FIX-004 durumu ✅
