<!-- Adapted from QuantConnect/Documentation, 4f39bafb4bee339ec252276b2cfae98619e92ab0, skills/python/datasets/alternative-data-universes/SKILL.md.
Changes: renamed SKILL.md to guide.md; replaced skill frontmatter with scope and provenance.
The upstream body below is unchanged. Named skills refer to topics in this package. -->

> Upstream scope (alternative-data-universes): Use when selecting a dynamic Equity universe from QuantConnect/LEAN alternative-data classes with `add_universe(<AltClass>, selector)`. Covers Brain, CoinGecko, EODHD, Quiver Quantitative, and Smart Insider universes. Skip for Morningstar fundamentals, ETF constituents, or pure indicator-driven universes.

# Alternative Data Universe Classes

- `BrainCompanyFilingLanguageMetricsUniverseAll` — Brain Language Metrics on Company Filings
- `BrainStockRankingUniverse` — Brain ML Stock Ranking
- `BrainSentimentIndicatorUniverse` — Brain Sentiment Indicator
- `CoinGeckoUniverse` — Crypto Market Cap
- `EODHDUpcomingDividends` — Upcoming Dividends
- `EODHDUpcomingEarnings` — Upcoming Earnings
- `EODHDUpcomingIPOs` — Upcoming IPOs
- `EODHDUpcomingSplits` — Upcoming Splits
- `FDICBankFinancialsUniverse` — US Bank Financials
- `FINRAShortInterestUniverse` — OTC Transparency
- `FINRAShortSaleVolumeUniverse` — Regulation SHO
- `QuiverCNBCsUniverse` — CNBC Trading
- `QuiverLobbyingUniverse` — Corporate Lobbying
- `QuiverInsiderTradingUniverse` — Insider Trading
- `QuiverQuantCongressUniverse` — US Congress Trading
- `QuiverGovernmentContractUniverse` — US Government Contracts
- `SmartInsiderIntentionUniverse` — Corporate Buybacks
- `SmartInsiderTransactionUniverse` — Corporate Buybacks
- `USPTOPatentMaintenanceUniverse` — Patent Maintenance
