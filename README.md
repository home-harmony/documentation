# FamilyLedger Documentation Portal

Welcome to the central documentation repository for **FamilyLedger**, a private, family-shared financial hub built on AWS Serverless and Rust.

---

## 📚 Documentation Index

```
documentation/
├── architecture/
│   ├── system-overview.md             # Cloud architecture, AWS services & event streaming
│   ├── aurora-dsql.md                 # Aurora DSQL constraints, OCC, IAM & migration rules
│   ├── database-schema.md             # Complete DDL table & async index catalog
│   └── api-design.md                  # REST API routes, role permissions & JWT auth
│
├── domain/
│   ├── bounded-contexts.md            # The 8 bounded contexts & aggregate root models
│   ├── event-storming.md              # Domain events catalog, triggers & timeline narrative
│   └── ubiquitous-language.md         # Domain glossary, invariants & True Disposable Surplus
│
├── decisions/                         # Architectural Decision Records (ADRs)
│   ├── README.md                      # ADR process & MADR template
│   ├── 0001-aurora-dsql-serverless-db.md   # ADR 0001: Amazon Aurora DSQL
│   ├── 0002-keyset-pagination.md           # ADR 0002: Keyset Cursor Pagination
│   ├── 0003-rust-ddd-clean-architecture.md # ADR 0003: Clean Architecture / DDD in Rust
│   ├── 0004-multi-repo-organization.md     # ADR 0004: Multi-Repo Umbrella Structure
│   └── 0005-decimal-precision-money.md     # ADR 0005: Float-Free Decimal Money
│
├── roadmap/
│   ├── business-plan.md               # MVP problem, solution, scope & $0.60/mo cost model
│   ├── sprint-roadmap.md              # S1-S7 execution roadmap (~15 weeks)
│   └── sprint-1/
│       ├── sprint_1_plan.md           # Sprint 1 detailed execution plan
│       └── task_1_1_detailed_guide.md # Task 1.1 AWS SAM / CloudFormation guide
│
└── testing/
    └── testing-strategy.md            # Test pyramid: Unit, Testcontainers PG, Axum, E2E
```

---

## 🚀 Quick Navigation

| Area | Key Document | Summary |
| :--- | :--- | :--- |
| **System Architecture** | [system-overview.md](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/architecture/system-overview.md) | AWS Cloud layout, API Gateway, Rust Lambdas, Aurora DSQL, CDC |
| **Database Constraints** | [aurora-dsql.md](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/architecture/aurora-dsql.md) | Aurora DSQL rules: UUID PKs, no FKs, 1 DDL per file, OCC retry |
| **Database Schema** | [database-schema.md](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/architecture/database-schema.md) | Full SQL DDL for 14 tables and 11 asynchronous indexes |
| **API Specification** | [api-design.md](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/architecture/api-design.md) | Endpoints, HTTP methods, and role-based permissions |
| **Domain Model** | [bounded-contexts.md](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/domain/bounded-contexts.md) | Aggregates and entities across all 8 bounded contexts |
| **Event Storming** | [event-storming.md](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/domain/event-storming.md) | Domain events flow via DSQL CDC $\rightarrow$ Kinesis $\rightarrow$ EventBridge |
| **Glossary & Formulas** | [ubiquitous-language.md](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/domain/ubiquitous-language.md) | True Disposable Surplus formula and domain invariants |
| **Architectural Decisions** | [ADR Index](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/decisions/README.md) | Record of accepted architectural decisions (0001–0005) |
| **Business Plan** | [business-plan.md](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/roadmap/business-plan.md) | Problem, solution, target users, and monthly cost estimate |
| **Sprint Roadmap** | [sprint-roadmap.md](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/roadmap/sprint-roadmap.md) | Sprint schedule and deliverables for S1 through S7 |
| **Testing Strategy** | [testing-strategy.md](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/testing/testing-strategy.md) | 95%+ pure domain unit testing and Testcontainers PG tests |

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
9. **Never trust `family_id` from client arguments** — always extract `family_id` from verified Cognito JWT.
10. **Use `aurora-dsql-sqlx-connector` (v0.2.2)** with `occ` feature enabled.
11. **Use `BEGIN READ ONLY` in read-only handlers** to eliminate OCC conflict overhead.
12. **One DDL statement per migration file** — separate tables and async indexes into distinct files.
13. **Never modify already applied migration files** — create new sequential migrations.
14. **Maintain 95%+ unit test coverage on the pure `domain/` crate**.
15. **DB integration tests run against real PostgreSQL** (via Testcontainers) — no mocked SQL queries.
16. **Role-based access control enforced at API layer** — `Role::Child` is blocked from loans, debt plans, and family reports.
