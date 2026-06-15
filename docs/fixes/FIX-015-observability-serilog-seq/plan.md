# FIX-015 — Uygulama Planı

> İlgili spec: [spec.md](spec.md). Branch: `feat/observability-serilog-seq`.

## Ön Koşullar
- [ ] master güncel, baseline `dotnet test` = 32/32

## Adımlar

### 1 — BuildingBlock'a Serilog paketleri
- **Dosya:** `Src/BuildingBlocks/BuildingBlock/BuildingBlock.csproj`
- Ekle: `Serilog.AspNetCore`, `Serilog.Sinks.Console`, `Serilog.Formatting.Compact`,
  `Serilog.Sinks.Seq`, `Serilog.Enrichers.Span`.

### 2 — Paylaşılan extension
- **Dosya (yeni):** `Src/BuildingBlocks/BuildingBlock/Logging/SerilogExtensions.cs`
- spec §5'teki `UseStandardSerilog`.

### 3 — Servisleri bağla (Catalog/Basket/Order)
- Catalog `Program.cs`: statik bootstrap logger'ı kaldır; `builder.Host.UseStandardSerilog("CatalogAPI")`.
- Basket `Program.cs`: `builder.Host.UseStandardSerilog("BasketAPI")` + `app.UseSerilogRequestLogging()`.
- Order.API `Program.cs`: `builder.Host.UseStandardSerilog("Order.API")` + `app.UseSerilogRequestLogging()`.
- Gerekli `using BuildingBlock.Logging;` / `using Serilog;`.

### 4 — Discount & Gateway
- Her iki `.csproj`'a `BuildingBlock` ProjectReference ekle.
- Her iki **Dockerfile**'a restore öncesi `COPY [...BuildingBlock.csproj...]` satırı ekle.
- `Program.cs`: `builder.Host.UseStandardSerilog("DiscountGrpc" / "YarpApiGateway")` + `UseSerilogRequestLogging()`.

### 5 — Seq (docker-compose)
- `docker-compose.yml`: `seq` servisi (`datalust/seq`).
- `docker-compose.override.yml`: `seq` env `ACCEPT_EULA=Y`, ports `5341:5341` + `8081:80`.
- Her servise env: `Seq__ServerUrl=http://seq:5341`.

### 6 — Doğrulama
- `dotnet build` + `dotnet test` (32/32).
- `docker compose config --quiet`.
- (Manuel) `docker compose up -d --build` → Seq UI `http://localhost:8081`.

## Son Doğrulama
- [ ] build + test yeşil
- [ ] compose geçerli
- [ ] Seq env yokken servisler açılıyor (console)

## Rollback
- Commit revert. Paket eklemeleri/compose dışında üretim davranışı değişmez.

## Done
- [ ] backlog README'de FIX-015 ✅
