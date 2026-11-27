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
- **See**: [`backend/infrastructure/sam/README.md`](../../../backend/infrastructure/sam/README.md) and [`0020-atomic-aggregate-update-port.md`](../../decisions/0020-atomic-aggregate-update-port.md) (*`Accepted`*: atomic `update` port + snapshot-diff writes on Aurora DSQL).

## 2026-11-26 — Angular Web SPA Architecture, `spartan/ui` & Cognito PKCE (`ADR-0021`)
- **Decision**: Build `frontend-web/` with Angular (standalone/signals/zoneless-ready), **`spartan/ui` 1.0 + Tailwind CSS**, **Cognito Hosted UI (OAuth 2.0 Authorization Code + PKCE)** sending `IdToken` (`email_verified == true`) + `X-Family-Id`, and **EN + RU i18n** with a typed RFC 9457 `error.code` dictionary compliant with CloudFront strict CSP.
- **Alternatives rejected**: Sub-1.0 lightweight Angular UI libraries lacking charts (`Guild of Gleks UI`, `ng-oat`, `Frakton NG`, `eagami/ui`, `Revo UI`); heavyweight suites + custom SRP auth screens in Sprint 2.
- **See**: [`0021-angular-spa-architecture-and-ui-library.md`](../../decisions/0021-angular-spa-architecture-and-ui-library.md) and [`sprint_2_ui_plan.md`](../../roadmap/sprint-2/sprint_2_ui_plan.md).

## 2026-11-27 — Frontend Bounded-Context Slices, Suffixless Role Folders & 3-Layer CSS (`ADR-0022`)
- **Decision**: Organize `frontend-web/` into progressive DDD bounded-context slices (`core/` $\leftarrow$ `shared/` $\leftarrow$ `features/<context>/`), Angular 20+ suffixless filenames inside explicit role folders (`pages/`, `components/`, `services/`, `stores/`, `guards/`, `interceptors/`, `pipes/`, `models/`, `utils/`), multi-file non-trivial components with 3-layer styling (`spartan/ui` primitives $\rightarrow$ `@layer components` `.fl-*` $\rightarrow$ component native CSS + OKLCH `var(--*)`), and a symmetric `frontend-web/.ai/` rules/skills structure.
- **See**: [`0022-frontend-web-structure-and-styling.md`](../../decisions/0022-frontend-web-structure-and-styling.md).

## 2026-11-27 — Sprint 3 Phase 0 Domain & Accounting Alignment (`P0-1` – `P0-13`)
- **Decisions**:
  - Rename `Role::Other` (`'other'`) end-to-end to **`Role::Guest`** (`'guest'`, UI `"Guest"` / `«Гость»`) across Ubiquitous Language, specs, Aurora DSQL constraints, Rust backend, and Angular SPA.
  - Propagate `member_id` into `MemberAccess` / `AuthClaims` and enrich `GET /profile/families` with `member_id` + `permissions`.
  - Keep `original.deleted_at = NULL` on `TransactionAmendment` (`ADR-0012`) and swap both accounts and dual amounts on cross-currency `Reversal` rows so `ADR-0006` snapshot balance derivation never double-negates or mixes currencies; derive deterministic SHA-256 companion `idempotency_key`s for `Reversal` rows.
  - Add `SignedMoney` (signed balances/snapshots/`FXSpread`), `PositiveMoney` (`> 0` transaction amounts), and immutable `opening_balance` on `payment_accounts`; seed global system `ledger_categories` (`family_id IS NULL`) via an idempotent Rust step in `migrate_runner` (zero DML in `.sql` migrations).
- **See**: [`sprint_3_plan.md`](../../roadmap/sprint-3/sprint_3_plan.md), [`0006`](../../decisions/0006-account-balance-snapshot.md), [`0011`](../../decisions/0011-multi-family-membership.md), and [`0012`](../../decisions/0012-immutable-ledger-transactions.md).

