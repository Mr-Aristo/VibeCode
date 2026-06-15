# FIX-004 — `CachedBasketRepository` null güvenliği

> **Durum:** 📝 Spec hazır · **Öncelik:** P0 · **Boyut:** S · **Bağımlılık:** —

## 1. Problem (NE)
`CachedBasketRepository.GetBasket`, cache'te bir değer bulduğunda
`JsonSerializer.Deserialize<ShoppingCard>(...)` sonucunu doğrudan döndürüyor. Deserialize
`null` dönebilir; metod imzası ise non-null `ShoppingCard` vaat ediyor. Bozuk/uyumsuz cache
verisinde `NullReferenceException` veya beklenmedik null akışı oluşur.

## 2. Kanıt
- `Src/Services/Basket/BasketAPI/Data/CachedBasketRepository.cs:14-20`
  ```csharp
  var cachedBasket = await cache.GetStringAsync(userName, cancellationToken);
  if (!string.IsNullOrEmpty(cachedBasket))
      return JsonSerializer.Deserialize<ShoppingCard>(cachedBasket);  // null olabilir
  ```

## 3. Etki / Neden önemli
- Bozuk cache içeriği (kısmi yazım, şema değişikliği, manuel müdahale) tüm GetBasket akışını
  çökertebilir.
- Sessiz null, ileride NRE olarak uzak bir noktada patlar — teşhisi zor.

## 4. Kapsam
**Dahil:**
- Deserialize sonucunun null/hatalı olması durumunda güvenli davranış (cache'i atlayıp
  repository'ye düşme — graceful degradation).

**Hariç (non-goals):**
- Cache anahtarlama/TTL stratejisini değiştirmek.
- Serileştirme formatını değiştirmek.

## 5. Önerilen Çözüm (yaklaşım)
Deserialize başarısız veya null ise cache'i yok say, repository'den taze veriyi al ve cache'i
yeniden doldur:

```csharp
var cachedBasket = await cache.GetStringAsync(userName, cancellationToken);
if (!string.IsNullOrEmpty(cachedBasket))
{
    var deserialized = TryDeserialize(cachedBasket);
    if (deserialized is not null)
        return deserialized;
    // bozuk cache: temizle ve repository'ye düş
    await cache.RemoveAsync(userName, cancellationToken);
}

var basket = await repository.GetBasket(userName, cancellationToken);
await cache.SetStringAsync(userName, JsonSerializer.Serialize(basket), cancellationToken);
return basket;
```

`TryDeserialize`, `JsonException`'ı yakalayıp `null` döndüren küçük bir yardımcıdır.

## 6. Kabul Kriterleri
- [ ] Geçerli cache verisi → cache'ten döner (mevcut davranış korunur).
- [ ] Cache yok → repository'den döner ve cache doldurulur (mevcut davranış korunur).
- [ ] Bozuk/null cache verisi → NRE atmadan repository'ye düşer ve cache yenilenir.
- [ ] `GetBasket` artık asla `null` döndürmez (repository `BasketNotFoundException` atar).
- [ ] `dotnet build` ve `dotnet test` yeşil.

## 7. Riskler / Notlar
- Repository `BasketNotFoundException` fırlatma davranışı korunur (sepet hiç yoksa).
- Performans etkisi ihmal edilebilir; yalnızca hatalı veri yolunda ekstra iş yapılır.
