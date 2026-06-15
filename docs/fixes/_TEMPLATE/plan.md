# FIX-XXX — Uygulama Planı

> İlgili spec: [spec.md](spec.md). Bu plan yalnızca spec onaylandıktan sonra uygulanır.

## Ön Koşullar
- [ ] spec.md kabul kriterleri okundu
- [ ] Çalışan dal/branch hazır
- [ ] Başlangıçta `dotnet build` yeşil (baseline)

## Adımlar (sıralı, atomik)

### Adım 1 — <kısa başlık>
- **Dosya:** `path/to/file.cs`
- **Eylem:** <ne değişecek — minimal diff>
- **Doğrulama:** <bu adımdan sonra ne kontrol edilir>

### Adım 2 — …

## Son Doğrulama
- [ ] `dotnet build` yeşil
- [ ] `dotnet test` yeşil
- [ ] spec.md'deki tüm kabul kriterleri sağlandı
- [ ] Yan etki taraması: <etkilenen producer/consumer/route/migration>

## Rollback
<Hata olursa nasıl geri alınır — genelde commit revert; contract değişikliği varsa ek adımlar.>

## Done
- [ ] Backlog `README.md`'de durum ✅ yapıldı
