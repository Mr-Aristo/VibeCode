# FIX-026 — Order.API endpoint yetkilendirme boşlukları

> **Durum:** 📝 Spec yazıldı, onay bekliyor · **Öncelik:** P2 (güvenlik / CRITICAL) · **Boyut:** S · **Bağımlılık:** FIX-013/FEAT-001 (auth altyapısı, ✅ mevcut)

## 1. Problem (NE)
Order.API'nin birçok endpoint'i hiç yetkilendirme (`RequireAuthorization`) taşımıyor.
Gateway `ordering-route` yalnızca "kimliği doğrulanmış" kullanıcı ister, rol kontrol etmez;
bu yüzden **herhangi bir giriş yapmış müşteri** tüm siparişleri okuyabiliyor, değiştirebiliyor
ve silebiliyor. Order.API portu (6003) doğrudan erişilebilirse aynı işlemler **anonim** yapılabilir.

## 2. Kanıt
Korumasız uçlar (`Src/Services/Order/Order.API/Endpoints/`):
- `GetOrders.cs:19` — `GET /orders` → tüm siparişleri sayfalı döker, `RequireAuthorization` yok.
- `GetOrdersByCustomer.cs:10` — `GET /orders/by-customer/{customerId}` → rastgele GUID kabul eder, yetki yok (IDOR).
- `GetOrdersByName.cs:11` — `GET /orders/by-name/{orderName}` → yetki yok (IDOR).
- `CreateOrder.cs:18` — `POST /orders` → yetki yok (sahte sipariş oluşturma).
- `UpdateOrder.cs:12` — `PUT /orders` → yetki yok (sipariş kurcalama).
- `DeleteOrder.cs:10` — `DELETE /orders/{id}` → yetki yok (sipariş silme).

Doğru korunmuş kardeş uçlar (referans deseni):
- `MyOrders.cs:18` — `.RequireAuthorization()` + token `sub`'una göre filtreleme.
- `ManageOrderStatus.cs:11` — `.RequireAuthorization(p => p.RequireRole("fulfillment-manager","super-admin"))`.
- `Returns.cs:19,25` — müşteri + admin rol ayrımı.

## 3. Etki / Neden önemli
- **Dikey yetki yükseltme:** sıradan müşteri fiilen sipariş veritabanında admin (oku/değiştir/sil).
- **Toplu PII sızıntısı:** `GET /orders` tüm müşterilerin adı, e-postası, adresi, sipariş kalemleri ve tutarlarını ifşa eder (KVKK/GDPR yükümlülüğü).
- **Veri bütünlüğü kaybı:** `PUT`/`DELETE` ile başkalarının siparişleri bozulabilir/silinebilir.

## 4. Kapsam
**Dahil:** Order.API içindeki 6 korumasız endpoint'e uygun `RequireAuthorization` politikalarının eklenmesi.
**Hariç (non-goals):**
- Order.API portunun dışa açılmaması / gateway-bypass sertleştirmesi (prod konusu; ayrı kalem).
- Yeni rol/claim tanımlama (mevcut Keycloak rolleri `fulfillment-manager`, `support-agent`, `super-admin` kullanılır).
- Rate limit, audience doğrulama (FIX-030, FIX-029).

## 5. Önerilen Çözüm (yaklaşım)
Endpoint'leri sınıflandır ve mevcut rol desenini uygula:

| Endpoint | Politika | Gerekçe |
|---|---|---|
| `GET /orders` | `RequireRole("fulfillment-manager","support-agent","super-admin")` | Tüm siparişleri listeleme = admin işi |
| `GET /orders/by-customer/{id}` | aynı admin rolleri | Müşteriler `/me/orders` kullanır |
| `GET /orders/by-name/{name}` | aynı admin rolleri | Operasyon/destek aracı |
| `PUT /orders` | `RequireRole("fulfillment-manager","super-admin")` | Sipariş düzenleme = admin |
| `DELETE /orders/{id}` | `RequireRole("super-admin")` | Yıkıcı işlem; en dar rol |
| `POST /orders` | **Kaldır veya** `RequireRole("super-admin")` | Sipariş normalde checkout saga consumer'ı ile oluşur; HTTP ucu gereksiz/risklidir |

`POST /orders` için tercih: HTTP endpoint'ini kaldırmak (saga zaten oluşturuyor). Geriye dönük
kullanım varsa `super-admin`'e kilitle — karar plan aşamasında netleştirilir.

## 6. Kabul Kriterleri
- [ ] 6 endpoint'in her biri uygun `RequireAuthorization` politikası taşır (veya `POST /orders` kaldırılır).
- [ ] Yetkisiz/anonim istek `401`, yanlış rol `403` döner; doğru rol `2xx`.
- [ ] `GET /me/orders` ve müşteri iade akışı eskisi gibi çalışır (regresyon yok).
- [ ] `dotnet build` ve `dotnet test` yeşil.
- [ ] Mevcut konvansiyonlar korundu, kapsam dışına çıkılmadı.

## 7. Riskler / Notlar
- **Kontrat değişikliği:** Daha önce anonim erişen istemciler artık token+rol gerektirir; bu kasıtlı sertleştirmedir, dokümante edilmeli.
- Keycloak `eshop` realm'inde admin rollerinin tanımlı olduğu doğrulanmalı (FEAT-001 ile geldi).
- `POST /orders` kaldırılırsa olası test/Swagger bağımlılıkları taranmalı.
