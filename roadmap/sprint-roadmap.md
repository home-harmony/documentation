# Sprint Roadmap (S1 – S7) — FamilyLedger

Total Estimated Development Timeline: **~15 Weeks** (Part-time / Agentic acceleration).

---

## Sprint Schedule

```
Sprint 1 (Weeks 1–2)  ──► AWS Infra, Aurora DSQL (34 Migrations), EventBridge, Rust Workspace, Migrations Runner
Sprint 2 (Weeks 3–4)  ──► Identity & Family Context, Multi-Family Auth (X-Family-Id), Granular Permissions
Sprint 3 (Weeks 5–7)  ──► Payment Accounts & Snapshots, Ledger Core (Immutable/Amendments), FX Transfers
Sprint 4 (Weeks 8–10) ──► Debt Planner: Extensible Kinds, LoanPayment Child Entity, EventBridge Sync, Avalanche/Snowball
Sprint 5 (Weeks 11–12)──► Budget Approval Flow, Loan-Bound Insurance, Savings Goals, Daily FX Fetcher (BNM)
Sprint 6 (Weeks 13–14)──► Scheduled Automation, Notifications (Push/FCM), Offline Caching, OCC Hardening
Sprint 7 (Week 15)    ──► E2E Test Suite, Multi-Tenant Security Audit, Load Testing, Family Beta Launch
```

---

## Detailed Deliverables by Sprint

### [Sprint 1](sprint-1/sprint_1_summary.md) (2 Weeks) — ✅ Completed
* **AWS Infrastructure Provisioning**: Cognito User Pool (identity only), API Gateway HTTP API (`X-Family-Id` CORS), S3 + CloudFront hosting, Aurora DSQL cluster, custom EventBridge bus (`$0.00` idle cost; Kinesis removed per ADR-0017).
* **Rust Multi-Crate Workspace**: Clean Architecture setup (`domain/`, `infrastructure/`, `api/`, `lambdas/`) with pinned dependencies (`sqlx 0.9.0`, `aurora-dsql-sqlx-connector 0.2.2`).
* **Migration Runner Lambda (`migrate_runner`)**: Deploy embedded SQLx migrator and initial **34 individual SQL migrations** applying all 18 tables and 16 asynchronous indexes.

### [Sprint 2](sprint-2/sprint_2_plan.md) (2 Weeks) — 🚧 In Progress
* **Identity & Family Bounded Context**: `Family`, `FamilyMember`, and `InviteToken` aggregate roots, RPITIT ports, and `lambdas/family_service`.
* **Multi-Family Workspace Support**: Support users participating in multiple families via `X-Family-Id` context header (ADR-0011) and `GET /profile/families`.
* **Granular Role-Based Permissions**: Implement fixed roles (`Owner`, `Member`, `Child`) and configurable `Permission` JSON flags for `Role::Other` (ADR-0011).
* **3-Stack SAM Deployment & Angular Web UI**: Deploy Option B2 (`core` + `data` + `services`) with non-admin `DSQL_USER` mapping, and deliver the Sprint 2 Angular Web SPA ([`sprint_2_ui_plan.md`](sprint-2/sprint_2_ui_plan.md)) in `frontend-web/` (`spartan/ui`, Cognito Hosted UI PKCE, multi-family switcher, member governance, EN/RU i18n). *(Flutter Mobile UI remains deferred to `frontend-mobile/`.)*

### Sprint 3 (3 Weeks)
* **Payment Accounts Context**: `PaymentAccount` aggregate (Credit cards, Debit cards, Cash envelopes, Bank accounts) and `accounts_service`.
* **Monthly Balance Snapshots**: `AccountBalanceSnapshot` child entity and monthly closing balance calculation (ADR-0006).
* **Ledger Core Context**: Immutable transactions, double-entry financial movements, and accounting amendment flow (`POST /transactions/{id}/amend`, ADR-0012).
* **Cross-Currency Transfers**: Dual-amount recording (deducted source amount + received destination amount) and implicit FX spread calculation (ADR-0015).
* **Keyset Pagination**: Keyset cursor pagination with time-sortable UUID v7.
* **Domain Event Publishing**: Rust services publish typed domain events directly to EventBridge upon DSQL commit (ADR-0017).

### Sprint 4 (3 Weeks)
* **Debt Planner Context**: Loan registry and `RepaymentPlanCalculator` domain service.
* **Extensible Loan Kinds**: System-wide seed taxonomy and custom per-family loan kinds (`debt_loan_kinds`, ADR-0007).
* **LoanPayment Child Entity**: Dedicated child entity enforcing `principal_portion + interest_portion == amount` invariant (ADR-0008).
* **Event-Driven Ledgerization**: `LoanPaymentRecorded` $\rightarrow$ EventBridge $\rightarrow$ `ledger_service` creates `loan_payment` transaction $\rightarrow$ `LoanPaymentLedgerized` (ADR-0009).
* **Credit Card Debt Linking**: Explicit account linkage (`linked_account_id`) with event-driven balance synchronization from card spending (`TransactionRecorded`, ADR-0014).
* **Debt Payoff Engine**: Automated Avalanche and Snowball optimization algorithms factoring in composite `DebtServiceCost`. 95%+ unit test coverage.

### Sprint 5 (2 Weeks)
* **Recurring Payments Context**: Utilities, subscriptions, rent registry, and loan-bound insurance (`linked_loan_id`, ADR-0010).
* **Monthly Budget & Governance**: `MonthlyBudget` lifecycle (`Draft` $\rightarrow$ `Active`) with Owner review, limit adjustments, and formal approval endpoint (`POST /budget/{y}/{m}/approve`, ADR-0013).
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

---

## Post-MVP Future Investigations

Tracked for Phase 2 / post-MVP security & compliance spikes (`documentation/research/`):

* **STRIDE Threat Modeling**: Systematic Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, and Elevation of Privilege threat analysis across Cognito JWT + `X-Family-Id` multi-family isolation, Aurora DSQL / IAM roles, EventBridge event flows, Angular Web SPA (CloudFront CSP & OAuth2 PKCE), and future AWS Bedrock / MCP tool surfaces (`ADR-0018`).
* **GDPR & Privacy Compliance (incl. Child Protection)**: EU GDPR Article 8 parental consent & age-verification workflows for `Role::Child` accounts, data minimization for minor profiles, right to erasure (`DSAR` anonymization / crypto-shredding reconciled with soft-delete and immutable ledger `ADR-0012`), data export/portability, retention policies, and EU AWS region residency options.
