# FIX-005 — CancellationToken yayılımı (repository SaveChanges)

> **Durum:** 📝 Spec hazır · **Öncelik:** P0 · **Boyut:** XS · **Bağımlılık:** —

## 1. Problem (NE)
`BasketRepository`, metod imzasında aldığı `CancellationToken`'ı Marten'in
`SaveChangesAsync` çağrısına geçirmiyor. İstemci isteği iptal ettiğinde DB yazımı iptal
edilemez; gereksiz iş yapılır ve iptal semantiği bozulur.

## 2. Kanıt
- `Src/Services/Basket/BasketAPI/Data/BasketRepository.cs:23` → `await session.SaveChangesAsync();`
- `Src/Services/Basket/BasketAPI/Data/BasketRepository.cs:29` → `await session.SaveChangesAsync();`
  (her ikisinde de parametredeki `cancellationToken` kullanılmıyor)

## 3. Etki / Neden önemli
- İptal edilen isteklerde gereksiz DB yazımı ve kaynak tüketimi.
- İptal semantiği tutarsız: bazı çağrılar CT'yi onurlandırırken bunlar etmiyor.
- Kod incelemesinde "sessiz" bir tutarsızlık; başka yerlere de yayılmış olabilir.

## 4. Kapsam
**Dahil:**
- `BasketRepository.StoreBasket` ve `DeleteBasket` içindeki `SaveChangesAsync` çağrılarına CT
  geçirme.
- Projede aynı kalıbın (CT alıp geçirmeyen `SaveChangesAsync`/async DB çağrısı) hızlı taranması.

**Hariç (non-goals):**
- Yeni CT parametreleri eklemek için imza değişiklikleri yapmak (mevcut imzalar yeterli).

## 5. Önerilen Çözüm (yaklaşım)
Mevcut parametreyi ilet:

```csharp
public async Task<ShoppingCard> StoreBasket(ShoppingCard basket, CancellationToken cancellationToken = default)
{
    session.Store(basket);
    await session.SaveChangesAsync(cancellationToken);
    return basket;
}

public async Task<bool> DeleteBasket(string userName, CancellationToken cancellationToken = default)
{
    session.Delete<ShoppingCard>(userName);
    await session.SaveChangesAsync(cancellationToken);
    return true;
}
```

## 6. Kabul Kriterleri
- [ ] `BasketRepository`'deki tüm async DB çağrıları CT'yi iletiyor.
- [ ] Tarama sonucunda bulunan benzer ihlaller ya düzeltildi ya da yeni FIX olarak not edildi.
- [ ] `dotnet build` ve `dotnet test` yeşil.

## 7. Riskler / Notlar
- Davranışsal risk yok; yalnızca iptal semantiğini doğru hale getirir.
- Tarama daha geniş kapsam ortaya çıkarırsa (örn. CheckoutHandler, başka servisler), bu FIX'i
  şişirmek yerine ayrı bir takip FIX'i açılır.
