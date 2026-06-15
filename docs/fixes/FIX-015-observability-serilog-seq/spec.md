# FIX-015 — Serilog standardizasyonu + Seq (merkezi structured logging)

> **Durum:** 📝 · **Öncelik:** P3 · **Boyut:** M · **Bağımlılık:** —

## 1. Problem (NE)
Loglama tutarsız: yalnızca **CatalogAPI** Serilog kullanıyor (console, compact JSON). Basket,
Order, Discount, Gateway varsayılan `ILogger` ile düz metin konsola yazıyor. Merkezi bir log
toplama yok; loglar her container'ın stdout'unda dağınık ve sorgulanamıyor.

## 2. Kanıt
- `Serilog` referansları yalnızca `Src/Services/CatalogAPI/*`.
- Diğer 4 servisin `Program.cs`'inde Serilog yok.

## 3. Etki / Neden önemli
Mikroservis sisteminde bir isteği/checkout'u servisler arası takip etmek için tutarlı,
yapılandırılmış ve **merkezi** loglar şart. Şu an her servis farklı formatta, tek yerde
toplanmıyor → teşhis zor.

## 4. Kapsam
**Dahil:**
- `BuildingBlock`'ta paylaşılan `UseStandardSerilog(serviceName)` extension'ı (tek kaynak).
- 5 servisin (Catalog, Basket, Order, Discount, Gateway) bu extension'ı kullanması.
- Compact JSON console + opsiyonel **Seq** sink; `ServiceName` + `TraceId/SpanId` enrichment.
- docker-compose'a `seq` container'ı + servislere `Seq__ServerUrl` env'i.

**Hariç (non-goals):**
- OpenTelemetry traces/metrics → **FIX-016** (ayrı).
- Log tabanlı alerting / dashboard kurulumu.

## 5. Önerilen Çözüm (yaklaşım)
`BuildingBlock/Logging/SerilogExtensions.cs`:
```csharp
public static IHostBuilder UseStandardSerilog(this IHostBuilder host, string serviceName) =>
    host.UseSerilog((ctx, lc) =>
    {
        lc.MinimumLevel.Information()
          .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
          .Enrich.FromLogContext()
          .Enrich.WithProperty("ServiceName", serviceName)
          .WriteTo.Console(new RenderedCompactJsonFormatter());
          // Not: TraceId/SpanId enrichment (log-trace korelasyonu) FIX-016'da OpenTelemetry ile eklenecek.
        var seqUrl = ctx.Configuration["Seq:ServerUrl"];
        if (!string.IsNullOrWhiteSpace(seqUrl)) lc.WriteTo.Seq(seqUrl);  // Seq opsiyonel
    });
```
- Seq **opsiyonel**: env yoksa yalnızca console (host-dev bozulmaz). Docker'da `Seq__ServerUrl=http://seq:5341`.
- Discount & Gateway `BuildingBlock`'a referans verecek (+ Dockerfile COPY) ki aynı extension'ı kullansınlar.

## 6. Kabul Kriterleri
- [ ] 5 servisin tümü `UseStandardSerilog` + `UseSerilogRequestLogging` kullanıyor.
- [ ] Seq env yokken servisler sorunsuz açılıyor (console).
- [ ] `docker compose config` geçerli; `seq` container'ı tanımlı.
- [ ] `dotnet build` + `dotnet test` yeşil (32/32 korunur).
- [ ] (Manuel) `docker compose up` sonrası `http://localhost:8081` (Seq UI) tüm servislerin loglarını `ServiceName`/`TraceId` ile gösteriyor.

## 7. Riskler / Notlar
- `BuildingBlock` her şey tarafından referanslanıyor → Serilog paketleri transitively yayılır (zararsız).
- Seq down ise Serilog.Sinks.Seq buffer'lar/drop eder, uygulamayı çökertmez.
- Gateway'e `BuildingBlock` ref'i MediatR/FluentValidation'ı transitively getirir (kullanılmaz, kabul edilebilir).
