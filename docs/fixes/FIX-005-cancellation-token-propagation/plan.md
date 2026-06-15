# FIX-005 — Uygulama Planı

> İlgili spec: [spec.md](spec.md).

## Ön Koşullar
- [ ] spec.md kabul kriterleri okundu
- [ ] Başlangıçta `dotnet build` yeşil

## Adımlar (sıralı, atomik)

### Adım 1 — `BasketRepository`'de CT geçir
- **Dosya:** `Src/Services/Basket/BasketAPI/Data/BasketRepository.cs:23,29`
- **Eylem:** İki `SaveChangesAsync()` çağrısını `SaveChangesAsync(cancellationToken)` yap.
- **Doğrulama:** Derlenir.

### Adım 2 — Projede benzer ihlalleri tara
- **Eylem:** `SaveChangesAsync()` (parametresiz) çağrılarını ve CT alıp iletmeyen async DB
  çağrılarını ara (Basket, Order, Discount).
- **Doğrulama:** Bulgular listelenir.

### Adım 3 — Kapsam kararı
- **Eylem:** Bulunan ihlaller küçük ve aynı nitelikteyse bu FIX kapsamında düzelt. Geniş/riskli
  ise `docs/fixes/README.md`'ye yeni bir FIX olarak ekle (kapsam şişirme).
- **Doğrulama:** Karar belgede netleşti.

## Son Doğrulama
- [ ] `dotnet build` yeşil
- [ ] `dotnet test` yeşil
- [ ] `BasketRepository` CT iletiyor

## Rollback
- Commit revert. Risk yok.

## Done
- [ ] `docs/fixes/README.md`'de FIX-005 durumu ✅
