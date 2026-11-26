# FamilyLedger Documentation Portal

Welcome to the central documentation repository for **HomeHarmony** (`FamilyLedger` internal product/architecture name), a private, family-shared financial hub built on AWS Serverless and Rust.

> 🇷🇺 **Для пользователей и членов семьи**: Описание проекта простыми словами на русском языке доступно в файле **[README.ru.md](README.ru.md)**.
> 🤖 **AI IDE Setup**: Because `documentation/` and `backend/` are separate Git repositories ([ADR-0004](decisions/0004-multi-repo-organization.md)), open both folders together as a **multi-root workspace** (not the parent `home-harmony/` folder) so `AGENTS.md` and `.agents/skills.json` auto-discover cleanly.

---

## 📚 Documentation Index

```
documentation/
├── README.md                          # Engineering index
├── README.ru.md                       # Описание для всей семьи простыми словами (Russian)
├── AGENTS.md                          # AI Agent entry point, rules & skills index
├── ai-development-guide.md            # Developer playbook: how to use skills for each task type
├── .ai/                               # Cross-repo AI context, governance rules & 3 shared skills
├── architecture/
│   ├── system-overview.md             # Cloud architecture, AWS services & EventBridge domain events
│   ├── aws-deployment-and-teardown.md # End-to-end AWS bootstrap, clean recreation & teardown runbook
│   ├── aurora-dsql.md                 # Aurora DSQL constraints, OCC, IAM & migration rules
│   ├── database-schema.md             # Complete DDL table (18) & async index (18) catalog
│   ├── kinesis-cdc-streaming.md       # Event publishing analysis & Kinesis removal rationale (ADR-0017)
│   ├── api-design.md                  # REST API routes, multi-family X-Family-Id & role permissions
│   └── ai-native-platform-rethinking.md # Phased AI-native & SDUI architecture specification
│
├── domain/
│   ├── bounded-contexts.md            # The 9 bounded contexts & aggregate root models
│   ├── event-storming.md              # Domain events catalog, triggers & loan payment choreography
│   └── ubiquitous-language.md         # Domain glossary, invariants & True Disposable Surplus formulas
│
├── research/                          # Phase 1 factual primary-source research spikes (behavior specs live in ../backend/specs/)
│
├── decisions/                         # Architectural Decision Records (ADRs)
│   ├── README.md                      # ADR index (0001–0021) & MADR template
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
│   ├── 0014-credit-card-explicit-loan-link.md # ADR 0014: Credit Card Event-Driven Loan Link
│   ├── 0015-multi-currency-live-exchange-rates.md # ADR 0015: Multi-Currency & Dual-Amount Transfers
│   ├── 0016-pluggable-exchange-rate-providers.md # ADR 0016: Pluggable FX Providers (BNM / ECB)
│   ├── 0017-cdc-and-event-publishing-strategies.md # ADR 0017: Event Publishing Strategy (Direct EventBridge)
│   ├── 0018-ai-native-modular-platform-and-sdui.md # ADR 0018: AI-Native Modular Platform & SDUI
│   ├── 0019-declarative-test-specifications.md # ADR 0019: Declarative Behavior Specs & AI Workflow
│   ├── 0020-atomic-aggregate-update-port.md # ADR 0020 (Proposed): Atomic Aggregate Update Port & Targeted Writes
│   └── 0021-angular-spa-architecture-and-ui-library.md # ADR 0021: Angular SPA, spartan/ui & Cognito PKCE
│
├── roadmap/
│   ├── business-plan.md               # MVP problem, solution, multi-currency scope & cost model
│   ├── sprint-roadmap.md              # S1-S7 execution roadmap (34+ migrations, ~15 weeks)
│   ├── sprint-1/sprint_1_summary.md   # Sprint 1 completion summary (AWS, Rust, 34 migrations)
│   └── sprint-2/
│       ├── sprint_2_plan.md           # Sprint 2 backend vertical-slice plan (Identity, Auth, Option B2 SAM)
│       ├── sprint_2_ui_plan.md        # Sprint 2 UI vertical-slice plan (Angular SPA, spartan/ui, PKCE, EN/RU)
│       ├── sprint_2_review_fix_plan.md # Sprint 2 two-axis review remediation plan & tracker
│       └── aws-deployment-guide.md    # Sprint 2 AWS upgrade, DSQL role bootstrap & smoke-test guide
│
└── testing/
    └── testing-strategy.md            # Test pyramid: Unit, Testcontainers PG, Axum, Spec Traceability
```

---

## 🚀 Quick Navigation

### ☁️ AWS Bootstrap, Operations & Teardown
* **[AWS Deployment, Operations & Teardown Guide](architecture/aws-deployment-and-teardown.md)** — Authoritative Option B2 (3-stack DAG) product runbook for bootstrapping, day-2 service & migration deployments, clean recreation, and complete teardown across `dev` / `staging` / `prod`.
* **[Sprint 2 AWS Deployment & Verification Guide](roadmap/sprint-2/aws-deployment-guide.md)** — Focused PowerShell upgrade runbook for deploying Sprint 2 (`family_service`, `36` migrations, `familyledger_app` role bootstrap, live Cognito + `X-Family-Id` smoke tests).
* **[Sprint 1 Completion Summary](roadmap/sprint-1/sprint_1_summary.md)** — Delivered AWS infrastructure, Rust workspace, 34 Aurora DSQL migrations, and spec-driven governance.
* **[Sprint 2 Execution Plan (Backend)](roadmap/sprint-2/sprint_2_plan.md)** & **[Sprint 2 UI Plan (Angular SPA)](roadmap/sprint-2/sprint_2_ui_plan.md)** — Vertical-slice plans for `family_service` (Option B2 SAM) and the Angular Web SPA (`spartan/ui`, Cognito PKCE, EN/RU).

### 🏛️ Architecture & Domain Model
* **[System Architecture Overview](architecture/system-overview.md)** — Serverless topology, microservices inventory, and scheduled EventBridge flows.
* **[Ubiquitous Language & Invariants](domain/ubiquitous-language.md)** — Canonical glossary, financial formulas (`True Disposable Surplus`, `DebtServiceCost`), and 10 non-negotiable domain invariants.
* **[Bounded Contexts & Aggregates](domain/bounded-contexts.md)** — The 9 DDD bounded contexts and aggregate boundaries.
* **[REST API Design](architecture/api-design.md)** — Endpoints, `X-Family-Id` multi-family tenancy, role permission matrix, and RFC 9457 error contracts.
* **[Database Schema Catalog](architecture/database-schema.md)** — All 18 Aurora DSQL tables and 18 asynchronous indexes (`CREATE INDEX ASYNC`).
* **[Aurora DSQL Guide & Release Tracker](architecture/aurora-dsql.md)** — Architectural constraints, UUID v4/v7 strategy, OCC/read-only patterns, and [official Aurora DSQL release notes](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/release-notes.html) tracking.

### 🤖 AI-First & Spec-Driven Workflow
* **[AGENTS.md](AGENTS.md)** & **[AI Development Guide](ai-development-guide.md)** — Entry point and practical playbooks for our 7 specialized skills (`domain-and-adr`, `spec-driven-dev`, `rust-patterns`, `rust-testing`, `dsql-migrations`, `diagnosing-bugs`, `code-review`).
* **[Architectural Decision Records (ADRs 0001–0021)](decisions/README.md)** — Chronological record of architectural decisions.
* **[Testing Strategy](testing/testing-strategy.md)** — Layer 1 pure domain tests, Layer 2a Testcontainers PostgreSQL, Layer 2b Axum `oneshot`, and `spec_traceability`.

