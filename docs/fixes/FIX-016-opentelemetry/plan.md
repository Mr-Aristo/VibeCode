# FIX-016 — Uygulama Planı

> İlgili spec: [spec.md](spec.md). Branch: aynı observability branch'i.

## Adımlar
1. **BuildingBlock.csproj** — OTel paketleri (stabil 1.x): `OpenTelemetry.Extensions.Hosting`,
   `OpenTelemetry.Instrumentation.AspNetCore`, `OpenTelemetry.Instrumentation.Http`,
   `OpenTelemetry.Instrumentation.Runtime`, `OpenTelemetry.Exporter.OpenTelemetryProtocol`.
2. **BuildingBlock/Observability/OpenTelemetryExtensions.cs** *(yeni)* — `AddStandardOpenTelemetry(serviceName)` (spec §4).
3. **SerilogExtensions.cs** — `Activity.Current`'tan `TraceId/SpanId` ekleyen hafif enricher.
4. **5 servis** — `builder.Services.AddStandardOpenTelemetry("X")` (+ `using BuildingBlock.Observability;`).
5. **docker-compose** — `otel-dashboard` (aspire-dashboard) servisi; her servise
   `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-dashboard:18889`.
6. **Doğrulama** — `dotnet build`+`test` (32/32); `docker compose config`.

## Son Doğrulama
- [ ] build+test yeşil · compose geçerli · env yokken servisler açılıyor (exporter no-op)

## Rollback
- Commit revert.

## Done
- [ ] backlog FIX-016 ✅
