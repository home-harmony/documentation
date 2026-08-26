# 0016. Pluggable ExchangeRateProvider Trait with BNM and ECB Frankfurter Implementations

* **Status**: Accepted
* **Date**: 2026-08-26
* **Deciders**: Dmitri Astafiev

## Context and Problem Statement

To support multi-currency families, FamilyLedger requires daily official foreign exchange rates. 

Different target audiences and geographic regions rely on different authoritative central banks:
* For families in Moldova (or holding accounts in Moldovan Leu - MDL), the **National Bank of Moldova (BNM)** is the authoritative official source.
* For euro-denominated families in the EU, the **European Central Bank (ECB)** (accessible via the open Frankfurter API) is standard.
* Other jurisdictions may require alternative providers in future expansions.

Hard-coding a single central bank API directly into Lambda handlers violates Clean Architecture and restricts the platform to a single jurisdiction.

## Decision Drivers

* **Clean Architecture & Pluggability** — the currency fetching mechanism must be abstracted behind an asynchronous Rust trait located in the `infrastructure` crate.
* **Support for Primary Provider (BNM)** — accurately parse the official BNM daily export, including custom semicolon-delimited CSV formats, comma decimal separators, and non-unitary nominal currency multipliers.
* **Zero-Code Provider Switching** — allow switching or falling back between providers via environment variables in AWS Lambda.

## Considered Options

1. **Hardcoded API client in Lambda** — direct HTTP calls inside the handler. (Rejected: tightly couples implementation to a single provider).
2. **Pluggable `ExchangeRateProvider` Trait** — abstract provider trait in `infrastructure::db` with concrete provider implementations (`BnmProvider`, `FrankfurterProvider`). (Chosen).

## Decision Outcome

Chosen option: **Option 2 — Pluggable `ExchangeRateProvider` Trait**.

### Provider Trait Definition

```rust
// backend/infrastructure/src/exchange_rates/provider.rs
use async_trait::async_trait;
use chrono::{DateTime, Utc};
use domain::value_objects::CurrencyCode;
use rust_decimal::Decimal;

#[derive(Debug, Clone)]
pub struct ExchangeRateEntry {
    pub quote_currency: CurrencyCode,
    pub rate: Decimal, // 1 quote_currency = rate home_currency
    pub fetched_at: DateTime<Utc>,
}

#[async_trait]
pub trait ExchangeRateProvider: Send + Sync {
    /// Fetches official daily rates against the requested home currency.
    async fn fetch_rates(
        &self,
        home_currency: &CurrencyCode,
    ) -> Result<Vec<ExchangeRateEntry>, ProviderError>;
}
```

### Provider Implementations

#### 1. National Bank of Moldova (`BnmProvider`) — Default
* **URL**: `https://www.bnm.md/en/export-official-exchange-rates?date={DD.MM.YYYY}`
* **Format**: Semicolon-separated CSV with comma decimals (`20,1386`) and CRLF line endings.
* **Parsing Rule**: Each row contains `Currency;Code;Abbr;Nominal;Rate`. The actual rate per single unit of currency is:

$$\text{EffectiveRate} = \frac{\text{parse\_decimal}(\text{Rate})}{\text{parse\_decimal}(\text{Nominal})}$$

* **Example**: For `Japanese Yen;392;JPY;100;10,8418`, nominal is 100 and rate is 10.8418 MDL, yielding an effective rate of `0.108418 MDL` per 1 JPY.

#### 2. ECB Frankfurter (`FrankfurterProvider`) — Alternative / Fallback
* **URL**: `https://api.frankfurter.app/latest?from={home_currency}`
* **Format**: Standard JSON payload with floating-point/decimal `rates` map.
* **Parsing Rule**: Inverts quote rates where needed to maintain consistent `1 quote = X home` pricing.

### Provider Configuration via Environment Variables

The `exchange_rate_fetcher` Lambda selects the provider at startup:

```bash
EXCHANGE_RATE_PROVIDER=bnm           # Options: "bnm", "frankfurter"
EXCHANGE_RATE_FALLBACK=frankfurter   # Optional secondary provider if primary fails
```

### Positive Consequences

* Seamless support for the primary Moldovan Leu (MDL) market with official central bank rates.
* Instant portability to other markets (EUR, USD) by swapping environment variables.
* Clean separation of HTTP/CSV parsing logic from domain calculations.

### Negative Consequences / Trade-offs

* Maintaining custom parser logic for idiosyncratic bank export formats (e.g. semicolon-separated CSVs).

## Compliance & Invariants

* The `ExchangeRateProvider` trait **must reside in `infrastructure`**, never in `domain` (which has zero I/O).
* The `BnmProvider` parser **must explicitly handle** nominal scaling (`Nominal > 1`) and comma-to-dot decimal conversions.
* Failures in exchange rate fetching **must emit** a warning alert but not block transaction recording.
