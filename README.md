# FamilyLedger Documentation Portal

Welcome to the central documentation repository for **FamilyLedger**, a private, family-shared financial hub built on AWS Serverless and Rust.

> 🇷🇺 **Для пользователей и членов семьи**: Описание проекта простыми словами на русском языке доступно в файле **[README.ru.md](README.ru.md)**.

---

## 📚 Documentation Index

```
documentation/
├── README.md                          # Engineering index
├── README.ru.md                       # Описание для всей семьи простыми словами (Russian)
├── architecture/
│   ├── system-overview.md             # Cloud architecture, AWS services & event streaming
│   ├── aurora-dsql.md                 # Aurora DSQL constraints, OCC, IAM & migration rules
│   ├── database-schema.md             # Complete DDL table (18) & async index (16) catalog
│   └── api-design.md                  # REST API routes, multi-family X-Family-Id & role permissions
│
├── domain/
│   ├── bounded-contexts.md            # The 9 bounded contexts & aggregate root models
│   ├── event-storming.md              # Domain events catalog, triggers & loan payment choreography
│   └── ubiquitous-language.md         # Domain glossary, invariants & True Disposable Surplus formulas
│
├── decisions/                         # Architectural Decision Records (ADRs)
│   ├── README.md                      # ADR index (0001–0016) & MADR template
│   ├── 0001-aurora-dsql-serverless-db.md   # ADR 0001: Amazon Aurora DSQL
│   ├── 0002-keyset-pagination.md           # ADR 0002: Keyset Cursor Pagination
│   ├── 0003-rust-ddd-clean-architecture.md # ADR 0003: Clean Architecture / DDD in Rust
│   ├── 0004-multi-repo-organization.md     # ADR 0004: Multi-Repo Umbrella Structure
│   ├── 0005-decimal-precision-money.md     # ADR 0005: Float-Free Decimal Money
│   ├── 0006-account-balance-snapshot.md    # ADR 0006: Monthly Account Balance Snapshots
│   ├── 0007-loan-kind-dictionary.md        # ADR 0007: Extensible Loan Kinds Dictionary
│   ├── 0008-loan-payment-child-entity.md   # ADR 0008: LoanPayment Child Entity & Invariants
│   ├── 0009-event-driven-loan-payment-ledger.md # ADR 0009: Event-Driven Ledgerization
│   ├── 0010-insurance-linked-recurring-payment.md # ADR 0010: Loan-Bound Insurance & DebtServiceCost
│   ├── 0011-multi-family-membership.md     # ADR 0011: Multi-Family Membership & X-Family-Id Header
│   ├── 0012-immutable-ledger-transactions.md # ADR 0012: Immutable Ledger & Reversals
│   ├── 0013-budget-owner-approval-flow.md  # ADR 0013: Monthly Budget Owner Review & Approval
│   ├── 0014-credit-card-explicit-loan-link.md # ADR 0014: Credit Card CDC Loan Link
│   ├── 0015-multi-currency-live-exchange-rates.md # ADR 0015: Multi-Currency & Dual-Amount Transfers
│   └── 0016-pluggable-exchange-rate-providers.md # ADR 0016: Pluggable FX Providers (BNM / ECB)
│
├── roadmap/
│   ├── business-plan.md               # MVP problem, solution, multi-currency scope & cost model
│   ├── sprint-roadmap.md              # S1-S7 execution roadmap (34 migrations, ~15 weeks)
│   └── sprint-1/
│       ├── sprint_1_plan.md           # Sprint 1 detailed execution plan (34 migrations)
│       ├── task_1_1_detailed_guide.md # Task 1.1 AWS SAM / CloudFormation guide
│       ├── task_1_2_detailed_guide.md # Task 1.2 Database & Streaming guide
│       └── task_2_1_detailed_guide.md # Task 2.1 Rust Multi-Crate Workspace guide
│
└── testing/
    └── testing-strategy.md            # Test pyramid: Unit, Testcontainers PG, Axum, E2E
```

---

## 🚀 Quick Navigation
