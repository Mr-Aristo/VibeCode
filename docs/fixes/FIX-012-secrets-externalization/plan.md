# FIX-012 — Uygulama Planı

> İlgili spec: [spec.md](spec.md). Branch: `feat/security-and-integration-tests`.

## Adımlar
1. `.gitignore`'a `.env` ekle.
2. `.env.example` oluştur (dev default değerlerle).
3. `.env` oluştur (yerel; gitignore'lu).
4. `docker-compose.override.yml`: secret'ları `${POSTGRES_USER/PASSWORD}`, `${SQLSERVER_SA_PASSWORD}`,
   `${RABBITMQ_USER/PASS}` ile parametrize (env'ler + connection string'ler).
5. `docker compose config` ile substitution doğrula.

## Son Doğrulama
- [ ] compose geçerli · `.env` ignore'da · ham secret yok

## Rollback
- Commit revert.

## Done
- [ ] backlog FIX-012 ✅
