# FIX-001 — Feature flag adı uyuşmazlığı

> **Durum:** 📝 Spec hazır · **Öncelik:** P0 · **Boyut:** XS · **Bağımlılık:** —

## 1. Problem (NE)
`OrderFullfilment` feature flag'i config üzerinden hiçbir zaman etkinleştirilemiyor. Sipariş
oluşturulduğunda `OrderCreatedEvent` domain event'i işlense de, flag her zaman `false`
döndüğü için `OrderDto` entegrasyon event'i hiçbir koşulda yayınlanmıyor.

## 2. Kanıt
- `Src/Services/Order/Order.API/appsettings.json:11` → `"OrderFullfilment": false` (tek `l`)
- `Src/Services/Order/Order.Application/OrdersCQRS/EventHandlers/Domain/OrderCreateEventHandler.cs:20`
  → `await featureManager.IsEnabledAsync("OrderFullfillment")` (çift `l`)
- `docker-compose.override.yml` → `FeatureManagement__OrderFullfilment=false` (config ile aynı, koddan farklı)

## 3. Etki / Neden önemli
Config anahtarı (`OrderFullfilment`) ile kodun aradığı anahtar (`OrderFullfillment`) yazımsal
olarak farklı. `IFeatureManager` eşleşmeyen anahtarı bulamaz ve `false` döner; dolayısıyla
özellik **kapatılamaz/açılamaz** — flag işlevsiz. Bu, sessiz ve fark edilmesi zor bir hatadır.

## 4. Kapsam
**Dahil:**
- Tek bir kanonik flag adına karar verme.
- Config (appsettings), kod ve docker-compose'u aynı ada hizalama.
- Sihirli string'i ortadan kaldırmak için bir `const` tanımı.

**Hariç (non-goals):**
- Feature flag altyapısını değiştirmek (Azure App Configuration vb.).
- `OrderFullfilment` özelliğinin davranışını değiştirmek.

## 5. Önerilen Çözüm (yaklaşım)
Kanonik ad olarak doğru İngilizce yazım **`OrderFulfillment`** seçilir. Sihirli string yerine
`Order.Application` içinde bir sabit tanımlanır ve hem handler hem config bu ada hizalanır:

```csharp
public static class FeatureFlags
{
    public const string OrderFulfillment = "OrderFulfillment";
}
```

> Not: Mevcut config değeri `false`; bu davranış (özellik kapalı) korunur. Bu FIX yalnızca
> adın tutarlılığını sağlar, varsayılan davranışı değiştirmez.

## 6. Kabul Kriterleri
- [ ] Config, kod ve docker-compose'da flag adı birebir aynı (`OrderFulfillment`).
- [ ] Handler sabiti (`FeatureFlags.OrderFulfillment`) kullanıyor, sihirli string yok.
- [ ] Config'te `OrderFulfillment=true` yapıldığında event yayınlanıyor (manuel doğrulama).
- [ ] `dotnet build` ve `dotnet test` yeşil.

## 7. Riskler / Notlar
- Kontrat değişikliği değil; yalnızca config anahtarı + sabit. Geriye dönük risk yok.
- Ortam değişkeni ile override eden CI/CD veya başka deployment dosyaları varsa onlar da
  güncellenmeli (yan etki taraması plan adımında).
