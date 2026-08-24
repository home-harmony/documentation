# Sprint Roadmap (S1 – S7) — FamilyLedger

Total Estimated Development Timeline: **~15 Weeks** (Part-time / Agentic acceleration).

---

## Sprint Schedule

```
Sprint 1 (Weeks 1–2)  ──► AWS Infra, Aurora DSQL, CDC, Rust Workspace, Migrations Runner
Sprint 2 (Weeks 3–4)  ──► Identity & Family Context, Cognito Auth, Web/Mobile Login
Sprint 3 (Weeks 5–7)  ──► Payment Accounts, Ledger Core, P2P Transfers, Keyset Pagination, CDC Fanout
Sprint 4 (Weeks 8–10) ──► Debt Planner: Loans, Avalanche/Snowball Plan Generator (95%+ Unit Tests)
Sprint 5 (Weeks 11–12)──► Budget Envelopes (CDC-driven), Recurring Payments, Savings Goals
Sprint 6 (Weeks 13–14)──► Notifications (Push/FCM), Offline Caching, OCC Hardening
Sprint 7 (Week 15)    ──► E2E Test Suite, Security & Role Audit, Load Testing, Beta Launch
```

---

## Detailed Deliverables by Sprint

### [Sprint 1](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/roadmap/sprint-1/sprint_1_plan.md) (2 Weeks)
* AWS Infrastructure: Cognito User Pool, API Gateway HTTP API, S3 + CloudFront hosting, Aurora DSQL cluster + Kinesis CDC stream.
* Rust multi-crate workspace initialized with dependencies (`sqlx 0.9.0`, `aurora-dsql-sqlx-connector 0.2.2`).
* Migration Runner Lambda (`migrate_runner`) and initial 25 SQL migrations applying 14 tables and 11 async indexes.

### Sprint 2 (2 Weeks)
* **Identity & Family Bounded Context**: `Family`, `FamilyMember`, `InviteToken` aggregate roots.
* Role-based access enforcement: `Owner`, `Member`, `Child`, `Other`.
* Angular SPA login and family creation screens; Flutter mobile app login view.
* Handler integration tests for family endpoints.

### Sprint 3 (3 Weeks)
* **Cards & Accounts Context**: `PaymentAccount` aggregate (Credit cards, Debit cards, Cash envelopes).
* **Ledger Core Context**: Transactions recording, P2P member-to-member transfers, ATM withdrawals.
* Keyset pagination with UUID v7.
* CDC Fanout Lambda publishing `TransactionRecorded` and `AccountBalanceUpdated` to EventBridge.

### Sprint 4 (3 Weeks)
* **Debt Planner Context**: Loan registry and `RepaymentPlanCalculator` domain service.
* Automated Avalanche and Snowball optimization algorithms factoring in True Disposable Surplus.
* 95%+ unit test coverage on pure calculation algorithms.
* Payoff timeline charts on Angular and Flutter clients.

### Sprint 5 (2 Weeks)
* **Recurring Payments Context**: Utilities, subscriptions, rent registry with schedule proration.
* **Budget & Limits Context**: Monthly budget initialization and committed envelope auto-seeding.
* **Future Planning Context**: Savings goals timelines and contribution calculator.

### Sprint 6 (2 Weeks)
* **Notification Service**: Web push notifications (Angular) and FCM push notifications (Flutter).
* EventBridge Scheduler rules for upcoming loan payments and missed recurring bill notices.
* Offline caching and OCC conflict retry hardening.

### Sprint 7 (1 Week)
* Comprehensive E2E test suite running against real AWS test stack.
* Security review and role-isolation audit (`Role::Child` access restrictions).
* Production release readiness and family beta launch.

