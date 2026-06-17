# FEAT-001 — Identity/Users Bounded Context — Spec

> **Tür:** Feature (epic) · **Durum:** 📝 Spec · **Öncelik:** P1–P3 (fazlı) · **Kaynak:** genişletilmiş use-case analizi (Identity/Users use-case spesifikasyonu).
> Bu belge **NE & NEDEN** + kabul kriterleri + kilitlenen kararları içerir. **NASIL** → [plan.md](plan.md).
> Bu feature, backlog'daki **FIX-013 (Auth & Authorization)**'ın genişletilmiş ve bounded-context'e oturtulmuş halidir.

## 1. Problem & Hedef
Sistemde kimlik/kullanıcı yok; tüm endpoint'ler anonim, Order'daki `Customer` gerçek bir hesaba bağlı değil. Hedef: **Keycloak tabanlı kimlik** + kullanıcı **profili**, **favoriler**, **sipariş geçmişi & yaşam döngüsü takibi**, **iade/refund** ve **rol bazlı admin** — mevcut mikroservis sınırlarını, DB-per-service ve Outbox/Saga (eventual consistency) desenini bozmadan.

## 2. Kilitlenen Mimari Kararlar (bu spec'in temeli)
| # | Karar | Gerekçe |
|---|---|---|
| D1 | Yeni **`Users.API`** servisi (Profile + Address + Favorites), vertical-slice (Carter+MediatR+Marten/Postgres) | Kullanıcı-merkezli iş verisine sahip bir yer gerek; Keycloak credential/rol'ün source-of-truth'u kalır |
| D2 | Kimlik = **Keycloak** (tek realm `eshop`); kayıt/login/şifre orada | Olgun IdP; credential'ı biz tutmayız |
| D3 | **JIT provisioning**: ilk authenticated istekte Users.API profili `sub`'tan oluşturur | Çift-yazma yok, basit |
| D4 | **`sub`** tüm sistemde korelasyon kimliği; Order `CustomerId = sub` + sipariş anında ad/e-posta **snapshot** | Cross-DB FK yok; tarihsel doğruluk |
| D5 | Sipariş yaşam döngüsü **Order.API'de durum makinesi + `OrderStatusHistory`** | Sipariş zaten Order aggregate'i |
| D6 | İade = **Order context'inde `Return` aggregate** (ileride ayrı servise çıkma yolu açık) | Refund order+stok ile sıkı bağlı |
| D7 | **Çift katman authz**: Gateway'de edge authn + kaba route authz + rate limit; her serviste JWT (JWKS) doğrulama + resource-based (owner) authz | Zero-trust |
| D8 | Durum güncelleme: **komut HTTP (Order.API) → Order.API event (RabbitMQ/Outbox)** | Sahiplik + mevcut desen |
| D9 | Favoriler **Users.API içinde bir slice** (ayrı servise çıkma sınırı temiz) | YAGNI; favori = kullanıcı verisi |

## 3. Kapsam
**Dahil (fazlı — bkz. plan):** Keycloak entegrasyonu + JWT doğrulama; Users.API (profil, adres, favoriler); Basket'in `sub`'a taşınması + auth'lu checkout; Order'da `CustomerId=sub` + snapshot + durum makinesi + sipariş geçmişi + admin durum endpoint'leri; iade/refund akışı; rol bazlı yetkilendirme.

**Hariç (non-goals, şimdilik):** Gerçek ödeme sağlayıcı entegrasyonu (bkz. açık karar O1); ayrı Inventory/Payment/Favorites servisleri; çoklu tenant/realm; kargo taşıyıcı entegrasyonu (webhook).

## 4. Aktörler & Roller
- **Misafir**, **Customer** (Keycloak `customer`), **Admin alt-rolleri**: `support-agent` (iade), `fulfillment-manager` (durum geçişleri), `catalog-manager` (ürün/kupon), `super-admin` (kullanıcı/rol). **System** (servis-servis).
- Yetki matrisi use-case §6'da; **owner kuralı**: kullanıcı yalnızca `resource.userId == token.sub` kaynak üzerinde işlem yapar (yatay yetki yükseltme engeli).

## 5. User Story'ler & Kabul Kriterleri (özet — tam liste use-case §2)
**P1 (MVP):**
- **A1 Login** (email+şifre → JWT: `sub`,`email`,roller). Yanlış şifre → 401; süresi dolan token → 401 + refresh.
- **A2 Kayıt** (Keycloak'ta kullanıcı; JIT ile ilk login'de Users.API profili).
- **B1 Profil** görüntüle/güncelle — yalnız kendi (`userId==sub`), aksi 403/404.
- **B2 Adres** ekle; checkout'ta seçilebilir.
- **D1 Sipariş geçmişi**: `GET /me/orders` yalnız kendi siparişleri.
- **D2 Sipariş takibi**: durum + `OrderStatusHistory` zaman çizelgesi (+ kargo/takip no).
- **F1 Fulfillment**: durum ilerletme (process→ship(+tracking)→deliver); geçersiz geçiş reddedilir, `OrderStatusChanged` yayınlanır.
- **F2** tüm siparişleri görme (Support/SuperAdmin).

**P2:** Favoriler (C1 — idempotent, kırık referans zarif); İade akışı (E1–E3, politika kararı sonrası); admin alt-rolleri; F3 ürün/kupon yönetimi role-korumalı; F4 rol yönetimi; A3/A4 şifre sıfırlama/çıkış; D3 bildirim.

**P3:** B3 hesap silme/veri indirme (KVKK); C2 favoriden sepete; otomatik teslim (kargo webhook); kullanıcıya özel kupon.

## 6. Genel Kabul Kriterleri
- [ ] Korumalı endpoint'ler geçerli JWT olmadan 401; yanlış rol 403; owner ihlali 403/404.
- [ ] `sub` uçtan uca taşınır; hiçbir servis başka servisin DB'sini okumaz.
- [ ] `BasketCheckoutEvent` değişikliği **additive** (mevcut consumer kırılmaz); event şema versiyonlama.
- [ ] Outbox/Saga deseni korunur; yeni event'ler de Outbox üzerinden yayınlanır.
- [ ] Mevcut konvansiyonlar (vertical slice / Clean Arch, CQRS, Carter, sağlık check, gateway route) korunur.
- [ ] Her faz sonunda `dotnet build` + `dotnet test` yeşil; yeni endpoint gateway'e route'lanır.
- [ ] Loglar/trace'ler `userId/sub` ile zenginleştirilir.

## 7. Açık Kararlar (uygulamadan önce gereken) `[KARAR]`
| # | Karar | Etki | Öneri | Bloklar |
|---|---|---|---|---|
| **O1** | **Ödeme/refund'un anlamı** (sistemde payment yok) | İade tasarımını tümüyle belirler | Önce "refund = iş durumu (Returned) + manuel finans"; Wallet/payment yol haritasına | **Faz 2 (iade)** |
| O2 | İade politikası: pencere (öneri 14g), kısmi iade (öneri evet), onay (öneri manuel/Support), iade kargosu | İade kuralları | use-case §8.2 önerileri | Faz 2 |
| O3 | Stok sahipliği: Catalog mı, ayrı Inventory mi? (restock buna bağlı) | İade restock + gelecekte stok | Kısa vade Catalog, büyürse Inventory | Faz 2 (restock) |
| O4 | "Delivered" kim onaylar? (manuel/müşteri/kargo webhook) | Teslim geçişi | Başta Fulfillment manuel + müşteri onayı | Faz 1 (varsayılan kabul edilebilir) |
| O5 | Misafir sepeti: anonim + login'de merge mi? | Basket UX | Anonim sepet, checkout'ta login zorunlu, login'de merge | Faz 1 |

> **Faz 1 bu kararlardan bloklanmaz** (O4/O5 için öneriler kabul edilebilir varsayım). **O1–O3, Faz 2 (iade) öncesi netleşmeli.**

## 8. Riskler
- `BasketCheckoutEvent` kontrat değişimi → tüm consumer'lar; additive + versiyonlama zorunlu.
- Basket anahtarı `username→sub` migrasyonu → mevcut sepetler/migrasyon stratejisi.
- Keycloak realm/rol kurulumu reproducible olmalı (realm export/import; .env ile secrets).
- Order durum makinesi + Return → yeni migration'lar (SQL Server).
