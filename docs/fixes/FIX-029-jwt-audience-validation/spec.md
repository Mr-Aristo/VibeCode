# FIX-029 — JWT audience doğrulamasının etkinleştirilmesi

> **Durum:** 📝 Spec yazıldı, onay bekliyor · **Öncelik:** P2 (güvenlik — MEDIUM) · **Boyut:** S · **Bağımlılık:** FIX-013.1 (paylaşılan JWT altyapısı, ✅), Keycloak realm yapılandırması

## 1. Problem (NE)
Paylaşılan JWT doğrulaması, audience kontrolünü yalnızca `Jwt:Audience` yapılandırılmışsa açıyor;
ancak **hiçbir** servis/compose `Jwt:Audience` tanımlamıyor. Sonuç: tüm servislerde audience
doğrulaması **kapalı**. Aynı Keycloak `eshop` realm'i içinde başka bir client için kesilmiş
bir token, bütün servislerce kabul edilir (audience confusion / token yeniden kullanımı).

## 2. Kanıt
- `Src/BuildingBlocks/BuildingBlock/Auth/AuthenticationExtensions.cs:30`
  ```csharp
  ValidateAudience = !string.IsNullOrWhiteSpace(audience),
  ```
  ve `:19` `var audience = configuration["Jwt:Audience"];`
- `Jwt:Audience` taraması: Catalog/Basket/Order/Users/Payment `appsettings.json` ve
  `docker-compose.override.yml` — **hiçbirinde tanımlı değil** (yalnız `Jwt:Authority` var).

## 3. Etki / Neden önemli
- Tek realm'de birden çok client olduğunda (örn. düşük yetkili public bir client), o client için
  alınan access token, audience kontrolü olmadığından korunan servisleri de açar.
- "Doğru issuer ama yanlış hedef kitle" senaryosunda yetki sınırı zayıflar (A07 — kimlik/oturum hataları).

## 4. Kapsam
**Dahil:**
- Her korunan servis için beklenen audience'ı yapılandırma (`Jwt:Audience`) ve doğrulamayı açma.
- Keycloak tarafında token'ların doğru `aud` taşımasını sağlama (audience mapper veya client scope) — yapılandırma/dokümantasyon düzeyinde.
**Hariç (non-goals):**
- Per-scope/claim bazlı ince yetkilendirme.
- `RequireHttpsMetadata`/TLS sertleştirmesi (FIX-014).

## 5. Önerilen Çözüm (yaklaşım)
1. Her servisin `appsettings.json`'una ve docker-compose ortamına `Jwt:Audience` ekle. İki seçenek
   (plan aşamasında netleşir):
   - **Tek paylaşılan audience** (örn. `eshop-api`) — tüm kaynak servisler aynı audience'ı bekler; Keycloak'ta bir client scope / audience mapper ile token'lara eklenir. (Basit, önerilen.)
   - **Servis-başı audience** — daha dar, ama Keycloak'ta daha fazla yapılandırma.
2. `AuthenticationExtensions` zaten `ValidateAudience`'ı audience varlığına göre açıyor → kod değişikliği gerekmeyebilir; yalnız yapılandırma. (Gerekirse `ValidAudiences` çoğul desteği eklenir.)
3. Gateway edge doğrulaması da aynı audience'ı bekleyecek şekilde hizalanır.

## 6. Kabul Kriterleri
- [ ] Her korunan serviste `Jwt:Audience` tanımlı ve `ValidateAudience` etkin.
- [ ] Beklenen audience'ı taşıyan token kabul; farklı/eksik audience'lı token `401`.
- [ ] Keycloak realm export'u (`keycloak/realms`) token'lara doğru `aud` ekleyecek şekilde yapılandırılmış (mapper/scope) ve dokümante.
- [ ] Mevcut uçtan uca akışlar (login → basket → checkout → order) regresyonsuz çalışır.
- [ ] `dotnet build` ve `dotnet test` yeşil.

## 7. Riskler / Notlar
- **Yanlış yapılandırma fail-closed:** audience uyuşmazsa tüm istekler 401 olur — Keycloak mapper'ı kod değişikliğiyle birlikte (aynı PR) hizalanmalı, aksi halde ortam kırılır.
- Realm import dosyası (`keycloak/realms/*.json`) güncellenmeli; dev'de `start-dev --import-realm` kullanılıyor.
- Token `aud` davranışı Keycloak sürümüne (26.0) göre doğrulanmalı.
