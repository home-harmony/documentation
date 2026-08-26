# FamilyLedger Documentation Portal

Welcome to the central documentation repository for **FamilyLedger**, a private, family-shared financial hub built on AWS Serverless and Rust.

---

## 📚 Documentation Index

```
documentation/
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
│       └── task_1_1_detailed_guide.md # Task 1.1 AWS SAM / CloudFormation guide
│
└── testing/
    └── testing-strategy.md            # Test pyramid: Unit, Testcontainers PG, Axum, E2E
```

---

## 🚀 Quick Navigation

| Area | Key Document | Summary |
| :--- | :--- | :--- |
| **System Architecture** | [system-overview.md](architecture/system-overview.md) | AWS Cloud layout, API Gateway, Rust Lambdas, Aurora DSQL, CDC, Schedulers |
| **Database Constraints** | [aurora-dsql.md](architecture/aurora-dsql.md) | Aurora DSQL rules: UUID PKs, no FKs, 1 DDL per file, OCC retry |
| **Database Schema** | [database-schema.md](architecture/database-schema.md) | Full SQL DDL for 18 tables and 16 asynchronous indexes (34 total) |
| **API Specification** | [api-design.md](architecture/api-design.md) | Endpoints, `X-Family-Id` auth, amendments, budget approvals |
| **Domain Model** | [bounded-contexts.md](domain/bounded-contexts.md) | Aggregates and entities across all 9 bounded contexts |
| **Event Storming** | [event-storming.md](domain/event-storming.md) | 24 domain events flow via DSQL CDC $\rightarrow$ Kinesis $\rightarrow$ EventBridge |
| **Glossary & Formulas** | [ubiquitous-language.md](domain/ubiquitous-language.md) | `True Disposable Surplus`, `DebtServiceCost`, and 10 strict invariants |
| **Architectural Decisions** | [ADR Index](decisions/README.md) | Record of all 16 accepted architectural decisions (0001–0016) |
| **Business Plan** | [business-plan.md](roadmap/business-plan.md) | Problem, multi-currency solution, target users, and monthly cost estimate |
| **Sprint Roadmap** | [sprint-roadmap.md](roadmap/sprint-roadmap.md) | Sprint schedule and deliverables for S1 through S7 |
| **Testing Strategy** | [testing-strategy.md](testing/testing-strategy.md) | 95%+ pure domain unit testing and Testcontainers PG tests |

---

## ⚡ Non-Negotiable Engineering Rules

1. **Money is always `rust_decimal::Decimal`** — never `f64` or `f32`.
2. **All primary keys must be random UUIDs** (UUID v4 or v7) using `gen_random_uuid()`.
3. **All list/search endpoints must use keyset (cursor) pagination** — `OFFSET` is banned.
4. **Strict limits**: UI page size $\le 50$, batch write size $\le 500$.
5. **No native JSONB column types** — store as `TEXT`, cast `::jsonb` during queries.
6. **Soft-deletes everywhere** (`deleted_at TIMESTAMPTZ NULL`) — no hard DELETEs on business data.
7. **No database polling for downstream events** — rely on Aurora DSQL native CDC $\rightarrow$ Kinesis $\rightarrow$ EventBridge.
8. **All CDC consumers must be idempotent** — deduplicate using `txId + table + PK`.
9. **Never trust `family_id` from client request bodies** — extract `user_id` from validated JWT and authenticate against `family_members` using the `X-Family-Id` header (ADR-0011).
10. **Transactions are immutable** — `PUT /transactions/{id}` is prohibited; corrections use the reversal pattern (`POST /transactions/{id}/amend`, ADR-0012).
11. **Use `aurora-dsql-sqlx-connector` (v0.2.2)** with `occ` feature enabled.
12. **Use `BEGIN READ ONLY` in read-only handlers** to eliminate OCC conflict overhead.
13. **One DDL statement per migration file** — separate tables and async indexes into distinct files.
14. **Never modify already applied migration files** — create new sequential migrations.
15. **Maintain 95%+ unit test coverage on the pure `domain/` crate**.
16. **DB integration tests run against real PostgreSQL** (via Testcontainers) — no mocked SQL queries.
17. **Role-based access control enforced at API layer** — `Role::Child` is blocked from loans, debt plans, and family reports; `Role::Other` permissions are verified per request.
