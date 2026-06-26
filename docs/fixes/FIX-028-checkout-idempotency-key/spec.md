# FIX-028 — Checkout idempotency-key + eşzamanlılık koruması

> **Durum:** 📝 Spec yazıldı, onay bekliyor · **Öncelik:** P2 (güvenlik / dayanıklılık — MEDIUM) · **Boyut:** M · **Bağımlılık:** FIX-008 (Order idempotency, ✅), Outbox/Saga akışı

## 1. Problem (NE)
`POST /basket/checkout` istemciden bir **idempotency anahtarı almıyor** ve eşzamanlılık koruması
(optimistic concurrency / benzersiz kısıt) yok. Her çağrı yeni `Guid.NewGuid()` checkoutId üretir;
ağ timeout'unda istemci retry'ı (sepet `Active`'e döndükten sonra) **ikinci bir checkout/sipariş**
yaratabilir. Geliştirici kodda bunu kendisi not düşmüş.

## 2. Kanıt
- `Src/Services/Basket/BasketAPI/Basket/CheckoutBasket/CheckoutBasketHandler.cs:8-9`
  ```
  //Todo: db and cache must be atomic
  //need to ad concurrency secure, code must be idempotent, because of retry and possible duplicates.
  ```
- Aynı dosya `:54` — `var checkoutId = Guid.NewGuid();` (her istekte yeni kimlik).
- `:49-52` — `CheckoutPending` koruması yalnız "zaten beklemede" durumunu yakalar; kontrol ile yazma arasında eşzamanlılık koruması yok (TOCTOU).

**Mevcut hafifleticiler (kapsamı sınırlar):**
- `PaymentAPI/Consumers/PaymentCaptureConsumer.cs:13` — `CheckoutId` üzerinde idempotent (çift ödeme yok).
- `Order` tarafı `OrderId = CheckoutId` deterministik (FIX-008 ✅) → mesaj tekrar teslimi çift sipariş yaratmaz.
- Yani açık **mesaj-tekrarı** değil, **istemci HTTP retry'ı**dır.

## 3. Etki / Neden önemli
- Ağ kesintisi/timeout sonrası dürüst bir istemci retry'ı, sepet o sırada `Active`'e dönmüşse ikinci bir checkout başlatabilir → mükerrer sipariş/iş.
- Eşzamanlı iki checkout isteği `CheckoutPending` kontrolünü ikisi de geçip iki outbox mesajı yazabilir (yarış).

## 4. Kapsam
**Dahil:**
- `POST /basket/checkout` için `Idempotency-Key` HTTP başlığı kabulü ve aynı anahtarla gelen tekrarda ilk sonucu döndürme.
- Basket yazımında optimistic concurrency (Marten version) veya benzersiz kısıtla yarış koruması.
**Hariç (non-goals):**
- DB + Redis arasındaki atomiklik (`CheckoutBasketHandler.cs:8` ikinci TODO) — ayrı dayanıklılık kalemi.
- Tam dağıtık deduplication / global idempotency store.
- Diğer yazma uçlarına idempotency genişletmesi.

## 5. Önerilen Çözüm (yaklaşım)
1. **İstemci anahtarı:** Endpoint `Idempotency-Key` başlığını okur (yoksa `400` veya sunucu üretir — plan aşamasında karar). Anahtar → `(sonuç, durum)` eşlemesi Marten'de küçük bir `IdempotencyRecord` dokümanında saklanır; aynı anahtar tekrar gelirse kayıtlı sonuç döner, yeni checkout başlatılmaz.
2. **Yarış koruması:** `ShoppingCard` üzerinde Marten optimistic concurrency (`[Version]` / `UseOptimisticConcurrency`) ya da `(UserName, Status=CheckoutPending)` için tekillik; çakışmada `409`/anlamlı hata.
3. Mevcut Outbox/Saga akışı korunur (anahtar yalnız giriş kapısına eklenir).

## 6. Kabul Kriterleri
- [ ] Aynı `Idempotency-Key` ile iki kez `POST /basket/checkout` → tek checkout/outbox mesajı, ikinci yanıt ilk sonucu yansıtır.
- [ ] Anahtarsız davranış net tanımlı ve dokümante (reddet veya türet).
- [ ] Eşzamanlı iki checkout → en fazla bir `CheckoutPending` geçişi (yarış kapandı).
- [ ] Outbox/Saga davranışı değişmedi; başarı→sepet sil, hata→`Active` akışı korundu.
- [ ] `dotnet build` ve `dotnet test` yeşil; yeni birim testleri (idempotent tekrar + yarış) eklendi.

## 7. Riskler / Notlar
- **Kontrat eki:** İstemcilerin `Idempotency-Key` göndermesi beklenir; geçiş için zorunluluk derecesi (zorunlu/opsiyonel) planda kararlaştırılır.
- Marten optimistic concurrency mevcut store yapılandırmasını etkileyebilir; `ShoppingCard` şemasıyla sınırlı tutulmalı.
- `IdempotencyRecord` için retention (eski anahtarların temizlenmesi) düşünülmeli (FIX-006 outbox retention deseniyle uyumlu).
