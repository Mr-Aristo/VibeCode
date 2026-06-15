# FIX-002 — `IntegrationEvent` kimlik kararlılığı

> **Durum:** 📝 Spec hazır · **Öncelik:** P0 · **Boyut:** S · **Bağımlılık:** —

## 1. Problem (NE)
`IntegrationEvent` temel kaydındaki `id` ve `OccuredOn` üyeleri, her **okumada** yeniden
hesaplanıyor. Aynı event nesnesinin `id`'si iki kez okunduğunda iki farklı `Guid` döner;
`OccuredOn` her erişimde o anki zamanı verir.

## 2. Kanıt
- `Src/BuildingBlocks/BuildingBlockMessaging/Events/IntegrationEvent.cs`
  ```csharp
  public Guid id => Guid.NewGuid();              // expression-bodied → her erişimde YENİ
  public DateTime OccuredOn => DateTime.UtcNow;  // her erişimde farklı
  public string EventType => GetType().AssemblyQualifiedName;
  ```

## 3. Etki / Neden önemli
Bir entegrasyon event'inin kimliği **sabit** olmalıdır. Mevcut hâliyle:
- Korelasyon / izleme (tracing) bozulur — aynı event farklı id'lerle görünür.
- İdempotency / deduplication imkânsız — id'ye güvenilemez (bkz. FIX-008).
- Loglama yanıltıcı olur — `OccuredOn` event üretim zamanını değil okuma zamanını gösterir.

## 4. Kapsam
**Dahil:**
- `id` ve `OccuredOn`'u tek sefer atanan, kararlı değerlere çevirme.
- PascalCase'e (`Id`) geçiş ve etkilenen kullanımları güncelleme.

**Hariç (non-goals):**
- Yeni alan (CorrelationId vb.) eklemek — bu FIX-016 kapsamında.
- Event sözleşmelerinin (BasketCheckoutEvent vb.) alanlarını değiştirmek.

## 5. Önerilen Çözüm (yaklaşım)
Expression-bodied property'leri `init`-only auto-property'lere çevir:

```csharp
public record IntegrationEvent
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public DateTime OccuredOn { get; init; } = DateTime.UtcNow;
    public string EventType => GetType().AssemblyQualifiedName!;  // bu türetilmiş kalabilir
}
```

`Guid.NewGuid()` / `DateTime.UtcNow` artık nesne **oluşturulurken** bir kez değerlenir.
`EventType` türetilmiş olmaya devam edebilir (tipten hesaplanır, kararlıdır).

## 6. Kabul Kriterleri
- [ ] Aynı event örneğinin `Id`'si tekrar tekrar okununca aynı değer.
- [ ] `OccuredOn` event üretim anını yansıtır ve değişmez.
- [ ] Türetilmiş tüm event'ler (`BasketCheckoutEvent`, `BasketCheckoutSucceededEvent`,
      `BasketCheckoutFailedEvent`) sorunsuz derlenir.
- [ ] Serileştirme (MassTransit/JSON) bozulmaz; mevcut outbox payload'ları okunabilir.
- [ ] `dotnet build` ve `dotnet test` yeşil.

## 7. Riskler / Notlar
- **Serileştirme uyumu:** `id` → `Id` yeniden adlandırması, kalıcı (Marten outbox) veya kuyrukta
  bekleyen eski payload'larda alan adı uyumsuzluğu yaratabilir. Geçiş anında bekleyen mesaj
  olmaması önerilir; gerekirse alan adını `id` bırakıp yalnızca `init`'e çevirme alternatifi
  değerlendirilir (daha düşük risk).
- Bu, paylaşılan bir sözleşme (`BuildingBlockMessaging`) değişikliğidir; tüm
  producer/consumer'lar yeniden derlenmelidir.
