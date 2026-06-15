# FIX-019 — Uygulama Planı

> İlgili spec: [spec.md](spec.md). Branch: `feat/security-and-integration-tests`.

## Adımlar
1. Test projesine `Testcontainers.PostgreSql` + `Xunit.SkippableFact` ekle.
2. `Integration/BasketPersistenceIntegrationTests.cs`:
   - `PostgresContainerFixture : IAsyncLifetime` — postgres:17-alpine başlat; hata olursa `DockerAvailable=false`.
   - `[SkippableFact]` test: Marten DocumentStore + `BasketRepository` ile store→get; `TotalPrice` doğrula.
3. `dotnet test` ile doğrula (Docker yoksa skip → yeşil).

## Son Doğrulama
- [ ] build+test yeşil (Docker yoksa 1 skip)

## Rollback
- Commit revert.

## Done
- [ ] backlog FIX-019 ✅ (ilk artım)
