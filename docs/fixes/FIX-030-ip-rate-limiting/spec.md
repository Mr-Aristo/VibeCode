# FIX-030 — IP-bazlı rate limiting + hassas rotalarda throttling

> **Durum:** 📝 Spec yazıldı, onay bekliyor · **Öncelik:** P2 (güvenlik — MEDIUM) · **Boyut:** M · **Bağımlılık:** FIX-014 (rate limit genişletme — bu FIX onu somutlaştırır), FIX-025 (gateway compose)

## 1. Problem (NE)
Gateway rate limiter'ı **partition'sız tek kova** olarak tanımlı (tüm istemciler aynı 5 istek/10sn
limitini paylaşır) ve yalnızca `ordering-route`'a uygulanıyor. Bu hem yanlış (bir saldırgan kovayı
doldurup **herkesi** kilitler), hem eksik (kimlik/sepet/checkout rotalarında hiç limit yok →
brute-force / credential-stuffing / sipariş suistimaline açık). IP-bazlı throttling yok.

## 2. Kanıt
- `Src/ApiGateways/YarpApiGateway/Program.cs:23-30`
  ```csharp
  rateLimiterOptions.AddFixedWindowLimiter("fixed", options => {
      options.Window = TimeSpan.FromSeconds(10);
      options.PermitLimit = 5;   // partition yok → tüm istemciler için ortak kova
  });
  ```
- `Src/ApiGateways/YarpApiGateway/appsettings.json` — yalnız `ordering-route` `"RateLimiterPolicy": "fixed"` taşır; `catalog`/`basket`/`users` route'larında limit yok.

## 3. Etki / Neden önemli
- **Paylaşılan kova = DoS çarpanı:** Tek istemci limiti tüketince meşru tüm kullanıcılar 429 alır.
- **Korunmasız hassas uçlar:** Checkout ve kimlik-yakın rotalar limitlenmediği için otomatik kötüye kullanım (brute-force, sipariş spam'i) engellenmez.
- Kullanıcı bu denetimi açıkça talep etti (rate limit + IP throttling).

## 4. Kapsam
**Dahil:**
- Gateway'de `PartitionedRateLimiter` ile **istemci-başı** (IP veya kimlik doğrulanmışsa `sub`) partition.
- Limit politikalarını checkout + kimlik-yakın + diğer rotalara uygulama (rota bazında uygun pencere/limit).
- Proxy/LB arkasında doğru istemci IP'si için `ForwardedHeaders` (X-Forwarded-For) yapılandırması.
**Hariç (non-goals):**
- Servis-içi (gateway dışı) rate limiting.
- WAF / bot yönetimi / dağıtık (Redis-backed) rate limit store — gelecek kalem.
- HSTS/HTTPS yönlendirme (FIX-014'ün diğer yarısı).

## 5. Önerilen Çözüm (yaklaşım)
1. `AddFixedWindowLimiter("fixed", ...)` yerine `AddPolicy`/`GlobalLimiter` ile
   `RateLimitPartition.GetFixedWindowLimiter(partitionKey, ...)` kullan; `partitionKey`:
   kimlik doğrulanmışsa `sub` claim'i, değilse `RemoteIpAddress`.
2. Rota-bazlı politikalar:
   - `checkout`/`ordering` → sıkı (örn. düşük pencere/limit).
   - `basket`, `users` → orta.
   - `catalog` (public okuma) → gevşek ama yine de tavanlı.
   - YARP `appsettings.json` route'larına ilgili `RateLimiterPolicy` adlarını ekle.
3. `app.UseForwardedHeaders(...)` ile gerçek istemci IP'sini çöz; **yalnız güvenilen proxy ağına**
   izin ver (aksi halde X-Forwarded-For spoof'u partition'ı atlatır).
4. `429` yanıtı için anlamlı `Retry-After` üret.

## 6. Kabul Kriterleri
- [ ] Limit istemci-başı izole: bir IP/sub limiti tüketince **diğer** istemciler etkilenmez.
- [ ] Checkout + kimlik-yakın + (uygun) diğer rotalar limit taşır; limit aşımında `429` + `Retry-After`.
- [ ] Proxy arkasında doğru istemci IP'si kullanılır; spoof'a karşı güvenilen-proxy kısıtı var.
- [ ] Mevcut `ordering-route` davranışı korunur/iyileştirilir (regresyon yok).
- [ ] `dotnet build` ve `dotnet test` yeşil.

## 7. Riskler / Notlar
- **FIX-014 ile ilişki:** FIX-014 "HTTPS redirection/HSTS + rate limit genişletme" kalemiydi; bu FIX onun rate-limit yarısını somutlaştırır. FIX-014 backlog'da buna referans verecek şekilde güncellenmeli.
- `ForwardedHeaders` yanlış yapılandırması ya IP'yi maskeler (herkes aynı partition) ya da spoof'a açar; güvenilen proxy aralığı net tanımlanmalı.
- Bellek-içi limiter tek gateway örneği varsayar; yatay ölçeklemede dağıtık store gerekir (non-goal, not düşülür).
- Test ortamında healthcheck/Swagger gibi uçların limitten muaf tutulması değerlendirilmeli.
