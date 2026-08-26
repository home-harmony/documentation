# 0015. Multi-Currency Display with Live Exchange Rates and Dual-Amount Cross-Currency Transfers

* **Status**: Accepted
* **Date**: 2026-08-26
* **Deciders**: Dmitri Astafiev

## Context and Problem Statement

Modern families frequently hold monetary instruments in multiple currencies (e.g., local currency salary accounts in MDL, savings in EUR, and investment or subscription cards in USD). 

This introduces two distinct architectural and financial challenges:
1. **Aggregated Household Financial Visibility**: The True Disposable Surplus, overall net worth, monthly budget totals, and debt projections must all be presented in a single coherent reference currency (`family.home_currency`).
2. **Cross-Currency Internal Transfers**: When money is transferred between accounts of different currencies (e.g., exchanging MDL for EUR in a mobile banking app), the actual deducted amount and the actual credited amount reflect the commercial bank's exchange rate and hidden FX spread. Applying an idealized mid-market exchange rate would cause real bank account balances to drift from the ledger.

## Decision Drivers

* **Real-World Balance Exactness** — account balances must mirror actual bank statements down to the exact cent, regardless of foreign exchange margins.
* **Aggregated Clarity** — household-level calculations (debt surplus, total committed expenses) must convert multi-currency items to `home_currency` deterministically.
* **Simplicity of History for MVP** — store only the latest daily exchange rates; avoid complex historical FX snapshot tables for MVP.

## Considered Options

1. **Strict Single-Currency System** — prohibit multi-currency accounts entirely. (Rejected: completely unsuitable for multi-currency households).
2. **Auto-Converted Single-Amount Transfers** — user inputs only source amount; system calculates destination amount via mid-market rate. (Rejected: causes account balance drift because banks never exchange at mid-market rates).
3. **Home Base Currency + Dual-Amount Cross-Currency Transfers + Live Daily Rates** — user enters both source and destination amounts for cross-currency transfers; live daily rates provide converted values for aggregated surplus views. (Chosen).

## Decision Outcome

Chosen option: **Option 3 — Home Base Currency + Dual-Amount Cross-Currency Transfers + Live Daily Rates**.

### 1. Dual-Amount Cross-Currency Transfers

For cross-currency transfers within `ledger_transactions`:
* `amount_value` & `amount_currency`: The exact amount deducted from the source account (in source currency).
* `destination_amount_value` & `destination_amount_currency`: The exact amount credited to the destination account (in destination currency).
* **Implicit FX Spread**: Derived on demand as:

$$\text{FXSpread} = \text{source\_in\_home\_curr} - \text{destination\_in\_home\_curr}$$

### 2. Live Exchange Rates Table (`exchange_rates`)

Only the latest exchange rate per currency pair is stored (no historical timeseries table for MVP):

```sql
CREATE TABLE exchange_rates (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  base_currency  CHAR(3)       NOT NULL,
  quote_currency CHAR(3)       NOT NULL,
  rate           NUMERIC(18,8) NOT NULL,  -- 1 quote_currency = rate base_currency
  fetched_at     TIMESTAMPTZ   NOT NULL,
  UNIQUE (base_currency, quote_currency)
);
```

### 3. Aggregated Display Conversions

When calculating True Disposable Surplus or family-level net worth:
1. Every account balance snapshot, recurring payment, or loan is retrieved in its native currency.
2. If `item.currency != family.home_currency`, the amount is converted:

$$\text{ValueInHomeCurrency} = \text{item.amount} \times \text{exchange\_rate}(\text{home\_currency}, \text{item.currency})$$

### Positive Consequences

* Internal transfers between multi-currency accounts perfectly match real bank transactions with zero account balance drift.
* High visibility into foreign exchange margins and bank conversion fees.
* Consolidated single-currency reporting for family budget and debt optimization.

### Negative Consequences / Trade-offs

* Users must transcribe both source and destination numbers when logging a multi-currency transfer.
* Historical multi-currency reports are projected using the current day's exchange rate rather than historical spot rates.

## Compliance & Invariants

* When `source_account.currency != destination_account.currency`, both `destination_amount_value` and `destination_amount_currency` **must be non-null**.
* The `exchange_rates` table **must be refreshed daily** by the `exchange_rate_fetcher` Lambda.
* If exchange rates are older than 48 hours, the UI **must display** a "Stale Exchange Rate" warning banner.
