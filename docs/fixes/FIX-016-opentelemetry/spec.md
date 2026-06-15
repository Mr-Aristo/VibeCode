# FIX-016 — OpenTelemetry (traces + metrics) + log korelasyonu

> **Durum:** 📝 · **Öncelik:** P3 · **Boyut:** L · **Bağımlılık:** FIX-015

## 1. Problem (NE)
Distributed tracing ve metrics yok. Async checkout akışını (Basket → RabbitMQ → Order) uçtan uca
tek bir iz olarak göremiyoruz; loglarda `TraceId` yok, log↔trace bağlanamıyor.

## 2. Etki
Mikroservis + Outbox/Saga'da bir isteğin servisler arası yolculuğunu görmek tek pratik debug yolu.

## 3. Kapsam
**Dahil:**
- Paylaşılan `BuildingBlock.Observability.AddStandardOpenTelemetry(serviceName)`.
- Tracing: ASP.NET Core + HttpClient instrumentation + `AddSource("MassTransit")` + `AddSource("Npgsql")` + OTLP exporter.
- Metrics: ASP.NET Core + HttpClient + Runtime + OTLP exporter.
- Serilog'a `TraceId/SpanId` enrichment (özel hafif enricher — paketsiz).
- docker-compose'a **.NET Aspire Dashboard** (OTLP toplayıcı/görselleştirici) + servislere `OTEL_EXPORTER_OTLP_ENDPOINT`.

**Hariç (non-goals):**
- gRPC client span'leri (beta paket gerekir) — sonra.
- Production trace backend (Grafana Tempo / Jaeger) — exporter hedefi değiştirilerek eklenir, kod değişmez.

## 4. Önerilen Çözüm
```csharp
services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(serviceName))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("MassTransit")   // checkout event akışı (Basket→Order)
        .AddSource("Npgsql")        // Catalog/Basket (Marten/Postgres)
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddOtlpExporter());
```
- OTLP hedefi env'den okunur (`OTEL_EXPORTER_OTLP_ENDPOINT`). Host-dev'de env yoksa exporter no-op olur (sorunsuz).
- Serilog enricher `Activity.Current`'tan `TraceId/SpanId` ekler → log↔trace korelasyonu.

## 5. Kabul Kriterleri
- [ ] 5 serviste `AddStandardOpenTelemetry`.
- [ ] `dotnet build` + `dotnet test` yeşil (32/32).
- [ ] `docker compose config` geçerli; `otel-dashboard` tanımlı.
- [ ] (Manuel) `docker compose up` sonrası Aspire Dashboard'da (`:18888`) bir `POST /basket/checkout` **tek trace** olarak Basket→RabbitMQ→Order boyunca görünür; loglarda `TraceId` var.

## 6. Riskler
- OTel paket sürüm uyumu → stabil 1.x kullanıldı, beta yok.
- Aspire Dashboard dev içindir (kalıcı depolama yok). Prod için OTel Collector → Tempo/Loki/Prometheus.
