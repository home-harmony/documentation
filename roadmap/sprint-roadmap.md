# Sprint Roadmap (S1 – S7) — FamilyLedger

Total Estimated Development Timeline: **~15 Weeks** (Part-time / Agentic acceleration).

---

## Sprint Schedule

```
Sprint 1 (Weeks 1–2)  ──► AWS Infra, Aurora DSQL (34 Migrations), CDC, Rust Workspace, Migrations Runner
Sprint 2 (Weeks 3–4)  ──► Identity & Family Context, Multi-Family Auth (X-Family-Id), Granular Permissions
Sprint 3 (Weeks 5–7)  ──► Payment Accounts & Snapshots, Ledger Core (Immutable/Amendments), FX Transfers
Sprint 4 (Weeks 8–10) ──► Debt Planner: Extensible Kinds, LoanPayment Child Entity, CDC Sync, Avalanche/Snowball
Sprint 5 (Weeks 11–12)──► Budget Approval Flow, Loan-Bound Insurance, Savings Goals, Daily FX Fetcher (BNM)
Sprint 6 (Weeks 13–14)──► Scheduled Automation, Notifications (Push/FCM), Offline Caching, OCC Hardening
Sprint 7 (Week 15)    ──► E2E Test Suite, Multi-Tenant Security Audit, Load Testing, Family Beta Launch
```

---

## Detailed Deliverables by Sprint

### [Sprint 1](sprint-1/sprint_1_plan.md) (2 Weeks)
* **AWS Infrastructure Provisioning**: Cognito User Pool, API Gateway HTTP API, S3 + CloudFront hosting, Aurora DSQL cluster + Kinesis CDC stream.
* **Rust Multi-Crate Workspace**: Clean Architecture setup (`domain/`, `infrastructure/`, `api/`, `lambdas/`) with pinned dependencies (`sqlx 0.9.0`, `aurora-dsql-sqlx-connector 0.2.2`).
* **Migration Runner Lambda (`migrate_runner`)**: Deploy embedded SQLx migrator and initial **34 individual SQL migrations** applying all 18 tables and 16 asynchronous indexes.

### Sprint 2 (2 Weeks)
* **Identity & Family Bounded Context**: `Family`, `FamilyMember`, and `InviteToken` aggregate roots.
* **Multi-Family Workspace Support**: Support users participating in multiple families via `X-Family-Id` context header (ADR-0011) and `GET /profile/families`.
* **Granular Role-Based Permissions**: Implement fixed roles (`Owner`, `Member`, `Child`) and configurable `Permission` JSON flags for `Role::Other` (ADR-0011).
* **Client Auth**: Angular SPA workspace switcher & login; Flutter mobile app login view.

### Sprint 3 (3 Weeks)
* **Payment Accounts Context**: `PaymentAccount` aggregate (Credit cards, Debit cards, Cash envelopes, Bank accounts) and `accounts_service`.
* **Monthly Balance Snapshots**: `AccountBalanceSnapshot` child entity and monthly closing balance calculation (ADR-0006).
* **Ledger Core Context**: Immutable transactions, double-entry financial movements, and accounting amendment flow (`POST /transactions/{id}/amend`, ADR-0012).
* **Cross-Currency Transfers**: Dual-amount recording (deducted source amount + received destination amount) and implicit FX spread calculation (ADR-0015).
* **Keyset Pagination**: Keyset cursor pagination with time-sortable UUID v7.
* **CDC Fanout Lambda**: Ingest DSQL CDC stream from Kinesis and publish semantic events to EventBridge.

### Sprint 4 (3 Weeks)
* **Debt Planner Context**: Loan registry and `RepaymentPlanCalculator` domain service.
* **Extensible Loan Kinds**: System-wide seed taxonomy and custom per-family loan kinds (`debt_loan_kinds`, ADR-0007).
* **LoanPayment Child Entity**: Dedicated child entity enforcing `principal_portion + interest_portion == amount` invariant (ADR-0008).
* **Event-Driven Ledgerization**: `LoanPaymentRecorded` $\rightarrow$ EventBridge $\rightarrow$ `ledger_service` creates `loan_payment` transaction $\rightarrow$ `LoanPaymentLedgerized` (ADR-0009).
* **Credit Card Debt Linking**: Explicit account linkage (`linked_account_id`) with CDC-driven balance synchronization from card spending (ADR-0014).
* **Debt Payoff Engine**: Automated Avalanche and Snowball optimization algorithms factoring in composite `DebtServiceCost`. 95%+ unit test coverage.

### Sprint 5 (2 Weeks)
* **Recurring Payments Context**: Utilities, subscriptions, rent registry, and loan-bound insurance (`linked_loan_id`, ADR-0010).
* **Monthly Budget & Governance**: `MonthlyBudget` lifecycle (`Draft` $\rightarrow$ `Active` $\rightarrow$ `Closed`) with Owner review, limit adjustments, and formal approval endpoint (`POST /budget/{y}/{m}/approve`, ADR-0013).
* **Future Planning Context**: Savings goals timelines and `GoalContribution` child entity tracking.
* **Currency & Exchange Rates Context**: `exchange_rate_fetcher` Lambda with pluggable `ExchangeRateProvider` trait fetching daily official rates from National Bank of Moldova (BNM) or ECB Frankfurter (ADR-0015, ADR-0016).

### Sprint 6 (2 Weeks)
* **Scheduled Automation**: EventBridge Scheduler rules for 1st-of-month budget draft creation, month-end account balance snapshots, and daily exchange rate fetching.
* **Notification Service**: Web push notifications (Angular) and FCM push notifications (Flutter) for bill reminders, budget threshold warnings (>80%), and loan payoff insurance cancellation notices.
* **Resilience & Caching**: Offline client caching and OCC conflict retry hardening with `aurora-dsql-sqlx-connector`.

### Sprint 7 (1 Week)
* **Comprehensive E2E Testing**: Automated test suite executing against real AWS test environment.
* **Security & Multi-Tenant Isolation Audit**: Verify strict role enforcement (`Role::Child` access blocks, `Role::Other` permission checks, and cross-family tenant boundary isolation).
* **Production Release Readiness**: Final performance validation, documentation verification, and family beta launch.
