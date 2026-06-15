# FIX-012 — Secrets externalization (docker-compose)

> **Durum:** 📝 · **Öncelik:** P2 · **Boyut:** S · **Bağımlılık:** —

## 1. Problem (NE)
DB şifreleri, SQL Server SA şifresi ve RabbitMQ kimlik bilgileri `docker-compose.override.yml`
içinde düz metin/hardcode. Repo'da gerçek sır taşıma riski + ortamlar arası parametrelenememe.

## 2. Etki
Sırların repoya girmesi güvenlik riskidir; ayrıca prod/farklı ortamda değer değiştirmek için
dosya düzenlemek gerekir.

## 3. Kapsam
**Dahil:** docker-compose secret'larını `${VAR}` ile parametrize; gitignore'lu `.env` + commit'lenen
`.env.example`.
**Hariç:** `appsettings.json` local dev default'ları (dev-only; prod env/user-secrets ile ezilir);
gerçek secret-store (Vault/KeyVault) entegrasyonu.

## 4. Çözüm
- `.env.example` (şablon) + `.env` (gitignore'lu; compose otomatik okur).
- override.yml: `POSTGRES_USER/PASSWORD`, `SQLSERVER_SA_PASSWORD`, `RABBITMQ_USER/PASS` ve bunları
  kullanan connection string'ler → `${...}`.

## 5. Kabul Kriterleri
- [ ] Repo'da ham secret yok (compose `${...}` kullanıyor).
- [ ] `.env` gitignore'da; `.env.example` commit'li.
- [ ] `docker compose config` `.env`'den doğru substitute ediyor.
- [ ] `.env` yokken kopyalama talimatı net (`.env.example` → `.env`).

## 6. Riskler
- `.env` yoksa compose boş substitution yapar → kopyalama adımı README/talimatta belirtilmeli.
- Bu bir ön koşul: ileride Keycloak admin creds de buraya taşınmalı (FIX-013/Identity).
