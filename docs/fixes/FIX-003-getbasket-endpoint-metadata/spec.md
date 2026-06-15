# FIX-003 — GetBasket endpoint OpenAPI metadata düzeltmesi

> **Durum:** 📝 Spec hazır · **Öncelik:** P0 · **Boyut:** XS · **Bağımlılık:** —

## 1. Problem (NE)
GetBasket endpoint'i, Catalog servisinden kopyalanmış yanlış OpenAPI/route metadata'sına sahip:
route adı, özet ve açıklama "Get Product By Id" diyor.

## 2. Kanıt
- `Src/Services/Basket/BasketAPI/Basket/GetBasket/GetBasketEndpoints.cs:19-23`
  ```csharp
  .WithName("GetProductById")
  .Produces<GetBasketResponse>(StatusCodes.Status200OK)
  .ProducesProblem(StatusCodes.Status400BadRequest)
  .WithSummary("Get Product By Id")
  .WithDescription("Get Product By Id");
  ```

## 3. Etki / Neden önemli
- Üretilen OpenAPI dokümantasyonu yanıltıcı (yanlış özet/açıklama).
- `WithName("GetProductById")` route adı, link/URL üretiminde (`LinkGenerator`) anlamsal
  karışıklık ve potansiyel ad çakışması yaratır.

## 4. Kapsam
**Dahil:** Bu endpoint'in `WithName`, `WithSummary`, `WithDescription` değerlerinin doğru
biçimde güncellenmesi.

**Hariç (non-goals):** Endpoint davranışını, route path'ini veya response tipini değiştirmek.

## 5. Önerilen Çözüm (yaklaşım)
Metadata'yı endpoint'in gerçek işlevine hizala:

```csharp
.WithName("GetBasket")
.Produces<GetBasketResponse>(StatusCodes.Status200OK)
.ProducesProblem(StatusCodes.Status400BadRequest)
.WithSummary("Get Basket")
.WithDescription("Get the basket for the given user name");
```

Ek olarak `respose` → `response` yazım hatası (satır 15) düzeltilebilir (opsiyonel, aynı dosyada).

## 6. Kabul Kriterleri
- [ ] Route adı `GetBasket`, özet/açıklama sepetle ilgili.
- [ ] Endpoint davranışı/route path'i değişmedi.
- [ ] `dotnet build` ve `dotnet test` yeşil (mevcut `GetBasketEndpointTests` etkilenmez).

## 7. Riskler / Notlar
- `WithName` değişimi, bu route adına `LinkGenerator`/`CreatedAtRoute` ile referans veren kod
  varsa onları etkiler. Yan etki taramasında kontrol edilmeli (muhtemelen yok).
