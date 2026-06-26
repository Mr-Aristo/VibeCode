# FIX-029 — Uygulama Planı

> İlgili spec: [spec.md](spec.md). Bu plan yalnızca spec onaylandıktan sonra uygulanır.

## Ön Koşullar
- [ ] spec.md kabul kriterleri okundu; audience stratejisi seçildi (tek paylaşılan vs servis-başı)
- [ ] Çalışan dal hazır: `fix/jwt-audience-validation`
- [ ] Keycloak realm export dosyası konumu doğrulandı (`keycloak/realms/*.json`)
- [ ] Başlangıçta `dotnet build` + `dotnet test` yeşil (baseline)

## Adımlar (sıralı, atomik)

### Adım 1 — Keycloak audience'ını yapılandır
- **Dosya:** `keycloak/realms/<eshop-realm>.json`
- **Eylem:** Access token'lara seçilen `aud` değerini ekleyen audience mapper / client scope tanımla.
- **Doğrulama:** `start-dev --import-realm` ile alınan token'da (`jwt.io`/debug) doğru `aud` görünür.

### Adım 2 — (Gerekirse) çoklu audience desteği
- **Dosya:** `Src/BuildingBlocks/BuildingBlock/Auth/AuthenticationExtensions.cs`
- **Eylem:** Tek audience yeterliyse değişiklik yok. Birden çok kabul edilen audience gerekiyorsa `TokenValidationParameters.ValidAudiences`'ı config dizisinden doldur.
- **Doğrulama:** Build yeşil; mevcut davranış audience boşken bozulmaz.

### Adım 3 — Servis yapılandırmalarını güncelle
- **Dosya:** Catalog/Basket/Order/Users/Payment `appsettings.json` + `docker-compose.override.yml` ilgili servis ortamları + gateway.
- **Eylem:** `Jwt:Audience` (ve compose'da `Jwt__Audience`) ekle.
- **Doğrulama:** Tüm servisler tutarlı audience bekler.

### Adım 4 — Uçtan uca doğrulama
- **Eylem:** Login → `GET /me/profile`, `/basket`, checkout, `/me/orders` akışını doğru audience'lı token ile çalıştır; yanlış audience'lı token ile 401 al.
- **Doğrulama:** Beklenen kabul/ret davranışı.

## Son Doğrulama
- [ ] `dotnet build` yeşil
- [ ] `dotnet test` yeşil
- [ ] Doğru audience → 2xx, yanlış/eksik audience → 401 (en az 2 servis için)
- [ ] Yan etki taraması: gateway edge authz, tüm servis appsettings tutarlılığı, realm import

## Rollback
Commit revert + realm export'u eski haline al. Mapper ve config birlikte geri alınmalı (yarı-uygulama 401 fırtınası yaratır).

## Done
- [ ] Backlog `README.md`'de durum ✅ yapıldı
