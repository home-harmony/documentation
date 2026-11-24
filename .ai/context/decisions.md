# Running Decision Log

Chronological summary complementing formal Architecture Decision Records in [`documentation/decisions/`](../../decisions/README.md).

---

## 2026-06-16 — Core Stack & Workspace Foundations (`ADR-0001` – `ADR-0005`)
- **Decisions**: Adopt Amazon Aurora DSQL (`0001`), enforce keyset pagination with UUID v7 (`0002`), organize Rust backend with Clean Architecture / DDD (`0003`), use independent Git repositories under the `home-harmony/` umbrella (`0004`), and mandate `rust_decimal::Decimal` for all money (`0005`).
- **See**: [`0001`](../../decisions/0001-aurora-dsql-serverless-db.md) – [`0005`](../../decisions/0005-decimal-precision-money.md)

## 2026-08-26 — Domain Model: Balances, Loans, Multi-Family & Budgets (`ADR-0006` – `ADR-0016`)
- **Decisions**: Separate monthly account balance snapshots (`0006`), extensible loan kinds (`0007`), `LoanPayment` child entity enforcing principal+interest split (`0008`), EventBridge loan ledgerization (`0009`), loan-bound insurance in `DebtServiceCost` (`0010`), multi-family membership via verified `X-Family-Id` (`0011`), immutable ledger via `Reversal` amendments (`0012`), Owner budget approval flow (`0013`), credit card link to loan (`0014`), multi-currency live FX & dual-amount transfers (`0015`), and pluggable FX providers (`0016`).
- **See**: [`0006`](../../decisions/0006-account-balance-snapshot.md) – [`0016`](../../decisions/0016-pluggable-exchange-rate-providers.md)

## 2026-09-05 — Direct EventBridge Event Publishing (`ADR-0017`)
- **Decision**: Publish domain events directly to Amazon EventBridge and remove Kinesis Data Streams to keep idle cost at `$0.00`.
- **Alternatives rejected**: Kinesis Data Streams CDC pipeline (fixed hourly shard cost).
- **See**: [`0017-cdc-and-event-publishing-strategies.md`](../../decisions/0017-cdc-and-event-publishing-strategies.md)

## 2026-11-14 — AI-Native Modular Platform & SDUI (`ADR-0018`)
- **Decision**: Adopt phased Dual-Surface (`REST + MCP`) architecture with a 3-tier `$0`-first cost funnel (`AI = OFF` MVP) and typed SDUI action cards backed by AWS Bedrock only on explicit user trigger.
- **Alternatives rejected**: Chat-first always-on LLM orchestrator; full backend-generated UI everywhere.
- **See**: [`0018-ai-native-modular-platform-and-sdui.md`](../../decisions/0018-ai-native-modular-platform-and-sdui.md)

## 2026-11-20 — Declarative Behavior Specifications & AI-First Optimization (`ADR-0019`)
- **Decision**: Introduce self-contained `backend/specs/` (Formats `T`/`U`/`P`) with human-owned `Approved` status, 4 automated CI guardrails in `backend/domain/tests/spec_traceability.rs` (blocking tests on `Draft` rules, `Origin: observed` approvals, and open-question approvals), blind test generation, and 7 zero-overlap skills across `documentation/` and `backend/`.
- **See**: [`0019-declarative-test-specifications.md`](../../decisions/0019-declarative-test-specifications.md)

## 2026-11-24 - Option B2 3-Stack SAM Layout (`core` + `data` + `services`)
- **Decision**: Split AWS SAM into three stacks with a one-way import DAG (`services` -> `core` & `data`): `core` (Cognito, S3, CloudFront), `data` (EventBridge bus, IAM roles, migrate runner), `services` (HTTP API + Lambdas). `FamilyLedgerApi` moved from `core` to `services`; `core` is deployed first to avoid two API Gateways and invoke-URL churn.
- **See**: [`backend/infrastructure/sam/README.md`](../../../backend/infrastructure/sam/README.md) and [`0020-atomic-aggregate-update-port.md`](../../decisions/0020-atomic-aggregate-update-port.md) (*`Proposed`*: atomic `update` port + snapshot-diff writes on Aurora DSQL).

