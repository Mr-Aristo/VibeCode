# VibeCode

E-Commerce Microservices çözümü için **mühendislik dokümantasyonu ve planlama** çalışma alanı.

> Uygulama kodu ayrı bir repoda yaşar: [Mr-Aristo/ECommerce_Microservices](https://github.com/Mr-Aristo/ECommerce_Microservices)
> (bu repoda bilinçli olarak hariç tutulmuştur).

## İçerik

| Yol | Açıklama |
|---|---|
| [docs/architecture/](docs/architecture/) | Mimari dokümantasyon (Türkçe) — sistem, servisler, checkout, gateway, test |
| [docs/architecture_en/](docs/architecture_en/) | Aynı dokümantasyonun İngilizce sürümü |
| [docs/improvement-plan.md](docs/improvement-plan.md) | Kod incelemesine dayalı önceliklendirilmiş fix/geliştirme planı |
| [docs/fixes/](docs/fixes/) | Spec-driven düzeltme çalışma alanı (her FIX için spec + plan) |
| [CLAUDE.md](CLAUDE.md) | Proje için mühendislik ajanı talimatları/konvansiyonları |
| [.claude/skills/](.claude/skills/) | Projeye özel beceri (skill) tanımları |

## Spec-Driven Akış

Düzeltmeler [docs/fixes/](docs/fixes/) altında, önce **spec** (ne/neden + kabul kriterleri),
sonra **plan** (sıralı/atomik adımlar) yazılarak ve sırayla uygulanır. Ayrıntı:
[docs/fixes/README.md](docs/fixes/README.md).
