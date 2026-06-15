# FIX-002 — Uygulama Planı

> İlgili spec: [spec.md](spec.md).

## Ön Koşullar
- [ ] spec.md kabul kriterleri okundu
- [ ] **Geçiş anında bekleyen outbox/kuyruk mesajı olmadığı doğrulandı** (serileştirme riski)
- [ ] Başlangıçta `dotnet build` yeşil

## Adımlar (sıralı, atomik)

### Adım 1 — `IntegrationEvent`'i kararlı kimliğe çevir
- **Dosya:** `Src/BuildingBlocks/BuildingBlockMessaging/Events/IntegrationEvent.cs`
- **Eylem:**
  ```csharp
  public record IntegrationEvent
  {
      public Guid Id { get; init; } = Guid.NewGuid();
      public DateTime OccuredOn { get; init; } = DateTime.UtcNow;
      public string EventType => GetType().AssemblyQualifiedName!;
  }
  ```
- **Doğrulama:** `BuildingBlockMessaging` derlenir.

### Adım 2 — `id` kullanımlarını tara ve güncelle
- **Eylem:** Repo genelinde `.id` / `event.id` kullanımı ara (özellikle Basket ve Order
  consumer/handler'ları). Varsa `.Id`'ye çevir.
- **Doğrulama:** Çözümde `id` (küçük) referansı kalmadı.

### Adım 3 — Serileştirme uyumunu doğrula
- **Eylem:** MassTransit JSON serileştirmesinin `Id`/`OccuredOn` alanlarını sorunsuz
  yazıp okuduğunu kontrol et. Eski `id` alanlı payload uyumu gerekiyorsa
  `[JsonPropertyName("id")]` gibi bir uyumluluk eki değerlendir (yalnızca gerekiyorsa).
- **Doğrulama:** Bir checkout uçtan uca denenir (lokal), event üretilir ve tüketilir.

### Adım 4 — Türetilmiş event'leri doğrula
- **Eylem:** `BasketCheckoutEvent`, `BasketCheckoutSucceededEvent`, `BasketCheckoutFailedEvent`
  derlenir; bu kayıtlar temel alanları override etmiyor (sadece miras alıyor).
- **Doğrulama:** Derleme yeşil.

## Son Doğrulama
- [ ] `dotnet build` yeşil
- [ ] `dotnet test` yeşil
- [ ] Aynı event örneğinde `Id` sabit (gerekirse hızlı bir birim test ekle)
- [ ] Lokal checkout akışı çalışıyor (event üret/tüket)

## Rollback
- Commit revert. **Dikkat:** Bu sürümle üretilmiş kalıcı payload varsa geri alımda alan adı
  uyumu tekrar kontrol edilmeli.

## Done
- [ ] `docs/fixes/README.md`'de FIX-002 durumu ✅
