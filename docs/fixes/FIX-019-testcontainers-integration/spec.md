# FIX-019 — Testcontainers integration testleri

> **Durum:** 📝 · **Öncelik:** P4 · **Boyut:** L (ilk artım: S) · **Bağımlılık:** —

## 1. Problem (NE)
Tüm testler mock/EF-InMemory; gerçek altyapıya (Postgres/Marten, SQL Server, Redis, RabbitMQ) karşı
doğrulama yok. Persistence/serileştirme/şema davranışı gerçek DB'de test edilmiyor.

## 2. Kapsam
**Dahil (ilk artım):** Gerçek PostgreSQL container'ı (Testcontainers) ile Marten + `BasketRepository`
store→get round-trip testi. Docker yoksa **skip** (suite yeşil kalır).
**Hariç (sonraki artımlar):** WebApplicationFactory uçtan-uca testleri, SQL Server (Order) ve
RabbitMQ (checkout saga) integration testleri.

## 3. Çözüm
- Paketler: `Testcontainers.PostgreSql`, `Xunit.SkippableFact`.
- `PostgresContainerFixture` (`IAsyncLifetime`): container'ı başlatır; başlatılamazsa
  `DockerAvailable=false`.
- `[SkippableFact]` + `Skip.IfNot(fixture.DockerAvailable, ...)` → Docker yoksa atla.

## 4. Kabul Kriterleri
- [ ] Docker varken test gerçek Postgres'e karşı geçer.
- [ ] Docker yokken test **skip** edilir; `dotnet test` yeşil (yeni başarısızlık yok).
- [ ] Mevcut 32 birim testi etkilenmez.

## 5. Riskler
- Testcontainers Docker gerektirir; CI'da Docker yoksa skip (kasıtlı).
- İlk artım yalnızca Basket/Marten'i kapsar; tam kapsam sonraki artımlarda.
