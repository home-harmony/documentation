# 0005. Exact Decimal Precision for Monetary Arithmetic

* **Status**: Accepted
* **Date**: 2026-06-16
* **Deciders**: Core Engineering Team

## Context and Problem Statement
Floating-point arithmetic (IEEE 754 `f32` / `f64`) introduces binary rounding inaccuracies (e.g. `0.1 + 0.2 = 0.30000000000000004`). In a financial ledger and debt planning system, floating-point drift can result in non-zero balance artifacts, inaccurate interest calculations, and audit failures.

## Decision Drivers
* Financial calculation correctness and audit compliance.
* Lossless serialization and deserialization across DB, API, and frontend.
* Strong type safety preventing accidental currency mismatch.

## Considered Options
1. **`rust_decimal::Decimal`** (128-bit fixed-point decimal arithmetic)
2. **Integer Cent Representation** (`i64` storing cents/micros)
3. **Floating Point** (`f64`)

## Decision Outcome
Chosen option: **`rust_decimal::Decimal`**, packaged within a domain value object `Money { amount: Decimal, currency: CurrencyCode }`.

### Positive Consequences
* Completely eliminates floating-point rounding errors.
* Direct compatibility with PostgreSQL / Aurora DSQL `NUMERIC(19,4)` column types via SQLx.
* Enables serialization with exact string representation (`"49.99"`).

### Negative Consequences / Trade-offs
* Requires explicit arithmetic methods and prevents raw primitive addition without currency validation.

## Compliance & Invariants
* The use of `f32` or `f64` for money, prices, balances, or interest rates is strictly banned.
* Enforced via domain unit tests and `Money` value object constructors.

