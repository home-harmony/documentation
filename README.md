# FamilyLedger — MVP Business Plan, Architecture & Domain Design

---

## Reference Documents

The following official AWS sources inform this document. Refer to them when implementing:

- [Aurora DSQL Connector for Rust (sqlx)](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/SECTION_program-with-dsql-connector-for-rust-sqlx.html) — IAM auth, OCC retry, connection pooling
- [Pagination Patterns in Aurora DSQL](https://aws.amazon.com/blogs/database/pagination-patterns-in-amazon-aurora-dsql/) — keyset vs OFFSET/LIMIT, 3,000-row transaction limit
- [DSQL SQL Dialect vs Standard PostgreSQL](https://aws.amazon.com/blogs/database/dsql-sql-dialect-how-amazon-aurora-dsql-differs-from-single-instance-postgresql/) — FK constraints, JSONB, SERIAL, async DDL, OCC
- [Change Data Capture in Aurora DSQL](https://aws.amazon.com/blogs/database/getting-started-with-change-data-capture-in-amazon-aurora-dsql/) — CDC to Kinesis, event structure, fanout patterns

---

## Part 1: Business Plan (MVP)

### Problem Statement

Modern families face a compound financial challenge: multiple active loans with varying interest rates and due dates, no unified visibility into household spending across cards and cash, and no structured mechanism to prioritize debt repayment. The result is chronic financial anxiety, missed optimization opportunities, and difficulty planning major future purchases.

### Solution

**FamilyLedger** is a private, family-shared financial hub that:
- Tracks all income and expenses across family members — by card, cash, or bank transfer
- Records all payment/credit card transactions and balances
- Manages peer-to-peer transfers and cash withdrawals within the family
- Tracks all recurring commitments — communal bills (water, electricity, gas, internet), subscriptions, and other fixed regular payments — so the family knows exactly how much is already spoken for each month before spending freely
- Manages all active loans/credits with repayment visualization
- Automatically generates a prioritized loan closure plan that accounts for recurring obligations first, then allocates the true disposable surplus to debt repayment
- Plans future major purchases against projected savings capacity

### Target Users (MVP)

A household of 2–10 members (adults, children, extended family) sharing finances, with 2+ active credits/loans and multiple payment cards. No financial expertise assumed.

### MVP Feature Scope

| Feature | Description | Priority |
|---|---|---|
| Family workspace | Invite members, roles (Owner/Member/Child/Other), shared view | P0 |
| Payment & credit card registry | Track all cards per member, balances, limits | P0 |
| Expense & income tracking | Manual entry with card/cash/transfer source, categories, tags | P0 |
| P2P and cash transactions | Family member-to-member transfers, cash withdrawals | P0 |
| **Recurring payment registry** | **Communal bills, subscriptions, fixed regular payments with schedule and amount** | **P0** |
| Loan registry | Track all loans: balance, rate, due date, monthly payment | P0 |
| Loan repayment plan | Auto-generated plan using true disposable surplus (after recurring obligations) | P0 |
| Monthly budget | Set limits per category, alerts; recurring payments pre-fill fixed envelopes | P1 |
| Future major expenses | Plan a purchase, see savings timeline | P1 |
| Reports & charts | Monthly summaries, by card, by member, by category | P1 |
| Notifications | Payment reminders for loans AND recurring bills, budget warnings, low card balance | P2 |

### MVP Success Metrics

- A family can register, add all cards + loans, and see a repayment plan within 20 minutes
- 80%+ of users enter at least one transaction per week over the first month
- Repayment plan seen as "actionable" in user feedback (NPS ≥ 40)

### Technology Choice Rationale

| Choice | Rationale |
|---|---|
| **AWS Lambda (Rust)** | Zero idle cost; Rust binaries give fast cold-starts (~200ms) |
| **Aurora DSQL** | Truly serverless SQL: scales to zero, bills only on active DPU usage, free tier covers 100K DPUs + 1 GB/month. Native CDC to Kinesis. |
| **Kinesis Data Streams** | Receives Aurora DSQL CDC events; Lambda reads from it for event fanout |
| **Angular (Web SPA)** | **Phase 1 — primary client.** Full-featured web application; well-suited for financial dashboards and complex tables. Delivered first. |
| **Flutter (Android)** | **Phase 2 — mobile.** Native Android app built after the web version is stable. Shares the same API. |
| **AWS Cognito** | Managed auth with family invite flows |
| **API Gateway** | HTTP API layer in front of Lambda |
| **EventBridge Scheduler** | Scheduled triggers (payment reminders, monthly budget close) |
| **S3 + CloudFront** | Angular SPA hosting with CDN |
| **`aurora-dsql-sqlx-connector`** | Official Rust connector: handles IAM token generation, pool refresh, OCC retry |
| **`sqlx migrate`** | Versioned SQL migrations embedded in binary — Flyway equivalent for Rust |

### Cost Estimate (MVP, Family Scale)

| Service | Estimated Monthly Cost |
|---|---|
| Aurora DSQL | **$0** (within free tier: 100K DPUs + 1 GB) |
| Kinesis (on-demand) | **$0** (negligible at family scale) |
| Lambda (first 1M requests free) | **$0** |
| API Gateway (first 1M free) | **$0** |
| Cognito (first 50K MAU free) | **$0** |
| S3 + CloudFront | ~$0.50 |
| EventBridge Scheduler | ~$0.10 |
| **Total** | **~$0.60/month** |

---

## Part 2: Aurora DSQL — Key Differences from Standard PostgreSQL

> Always consult the [DSQL SQL Dialect](https://aws.amazon.com/blogs/database/dsql-sql-dialect-how-amazon-aurora-dsql-differs-from-single-instance-postgresql/) article before writing schema or queries.

### What's the same
Standard DML (`SELECT`, `INSERT`, `UPDATE`, `DELETE`), standard DDL (`CREATE TABLE`, `ALTER TABLE`, `CREATE VIEW`), transaction control (`BEGIN`, `COMMIT`, `ROLLBACK`), core data types (numeric, char, date/time, boolean, bytea), joins, subqueries, aggregations, and all major PostgreSQL client drivers including `sqlx`.

### Critical differences that affect FamilyLedger's schema and code

| Difference | Impact | How we handle it |
|---|---|---|
| **Data stored in primary key order** (not a heap) | Sequential/monotonic PKs create write hot-spots across the distributed cluster | **UUID v4/v7 PKs everywhere** — random UUIDs spread writes evenly |
| **No `FOREIGN KEY` constraints** | Cannot enforce referential integrity at DB level | Enforced in Rust domain layer; every query includes `family_id` from JWT |
| **`JSONB` not supported as column type** | Cannot use `JSONB` for `tags`, `projection`, etc. | Store as `TEXT`, cast to `JSONB` at query time with `::jsonb` |
| **No `SERIAL`/`BIGSERIAL`** | Auto-increment columns unsupported | Use `gen_random_uuid()` for PKs; use `GENERATED ALWAYS AS IDENTITY` only for non-PK sequences |
| **No `TRUNCATE`** | Cannot truncate tables | Use `DELETE FROM table_name` in batches (respect 3,000-row limit per transaction) |
| **No triggers, no stored procedures** | No DB-side automation | All logic in Lambda / domain services |
| **No `ON DELETE CASCADE`** | Cascades not enforced | Soft-delete pattern: `deleted_at TIMESTAMPTZ NULL` |
| **`CREATE INDEX ASYNC` only** | Indexes built in background; not immediately available | Separate index migrations from data migrations; verify via `sys.jobs` |
| **One DDL statement per transaction** | Cannot mix DDL and DML | Migration files: one DDL op per file or run DDL outside transactions |
| **OCC, not row-level locking** | Concurrent writes may conflict at commit time → SQLSTATE 40001 | `aurora-dsql-sqlx-connector` `occ` feature handles retry automatically |
| **`SELECT FOR UPDATE` doesn't block** | Locking behavior differs; conflicts surface at commit, not at SELECT | Design for low contention; use idempotent transactions |
| **3,000-row transaction write limit** | Bulk inserts/deletes fail if they modify >3,000 rows in one transaction | Keep batch sizes ≤ 500 with individual commits per batch |
| **IAM auth, not passwords** | Connection strings and auth flow are different | `aurora-dsql-sqlx-connector` handles token generation automatically |
| **`BEGIN READ ONLY`** | Read-only transactions have zero OCC conflict risk | Use for all read-only Lambda handlers (transaction list, plan view, etc.) |

---

## Part 3: Domain Design (DDD + Event Storming)

### Bounded Contexts

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           FamilyLedger                                   │
│                                                                           │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │  Identity &      │  │  Ledger      │  │  Debt Planner            │   │
│  │  Family          │  │  Core        │  │  (Loan Optimizer)        │   │
│  └──────────────────┘  └──────────────┘  └──────────────────────────┘   │
│                                                                           │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │  Payment Cards   │  │  Recurring   │  │  Budget & Limits         │   │
│  │  & Accounts      │  │  Payments    │  │                          │   │
│  └──────────────────┘  └──────────────┘  └──────────────────────────┘   │
│                                                                           │
│  ┌──────────────────┐  ┌──────────────┐                                  │
│  │  Future Planning │  │Notifications │                                  │
│  └──────────────────┘  └──────────────┘                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Bounded Context 1: Identity & Family

**Responsibility:** User registration (via Cognito), family workspace, membership with role-based access.

#### Family Roles

```
Role:
  Owner       — created the family; full control including member removal and role assignment
  Member      — adult family member; can record transactions, manage their own cards
  Child       — restricted view; can record own transactions; cannot see loans or planning
  Other       — extended family (Grandma, Grandpa, Cousin, Aunt, etc.); same permissions as Member
                by default, but Owner can restrict per-member
```

Children see their own transactions and budget allowance. They cannot see loan details, repayment plans, or family-level reports — these are adult-only views enforced at the API level.

#### Aggregates

**Family** (Aggregate Root)
```
Family
  - id: FamilyId (UUID)
  - name: String
  - home_currency: CurrencyCode        // ISO 4217, e.g. "MDL", "USD", "EUR"
  - created_at: DateTime
  - members: Vec<FamilyMember>
  - invite_tokens: Vec<InviteToken>

FamilyMember
  - id: MemberId (UUID)
  - user_id: UserId (UUID, from Cognito sub)
  - display_name: String
  - role: Role { Owner | Member | Child | Other }
  - relationship: Option<String>       // e.g. "Grandma", "Cousin" — display label for Other
  - joined_at: DateTime
  - deleted_at: Option<DateTime>       // soft-delete

InviteToken
  - token: String                      // UUID, used as URL token
  - family_id: FamilyId
  - role: Role                         // pre-assigned role for this invite
  - relationship: Option<String>
  - expires_at: DateTime
  - used: bool
  - created_by: UserId
```

**Domain Events:** `FamilyCreated`, `MemberInvited`, `MemberJoined`, `MemberRoleChanged`, `MemberRemoved`

---

### Bounded Context 2: Payment Cards & Accounts

**Responsibility:** Registry of all payment instruments held by family members — credit cards, debit cards, and cash envelopes. Tracks balances and credit limits. Source of truth for "which instrument was used" on each transaction.

#### Aggregates

**PaymentAccount** (Aggregate Root)
```
PaymentAccount
  - id: AccountId (UUID)
  - family_id: FamilyId
  - owner_member_id: MemberId          // which family member holds this card/account
  - name: String                       // e.g. "Maib Visa Gold", "Moldo Indigo", "Cash EUR"
  - kind: AccountKind {
      CreditCard     // has credit limit; balance can be negative (debt)
      DebitCard      // bank account / debit card; balance ≥ 0
      Cash           // physical cash envelope
      BankAccount    // savings or current account without card
    }
  - currency: CurrencyCode
  - current_balance: Money             // tracked balance (updated by transactions)
  - credit_limit: Option<Money>        // for CreditCard: approved limit
  - last_four: Option<String>          // last 4 digits of card number (for display)
  - bank_name: Option<String>          // e.g. "Maib", "Moldindconbank"
  - color: HexColor                    // for UI display
  - is_included_in_budget: bool        // exclude business/child accounts if needed
  - created_at: DateTime
  - deleted_at: Option<DateTime>
```

**Domain Events:** `AccountRegistered`, `AccountBalanceUpdated`, `AccountRemoved`, `CreditLimitChanged`

---

### Bounded Context 3: Ledger Core

**Responsibility:** Records all financial movements. Every transaction references a source account and optionally a destination account (for transfers). This context is the heart of the system.

#### Transaction Types

```
TransactionKind:
  Income          — money comes in (salary, freelance payment, gift)
                    source: external | destination: one account
  Expense         — money goes out (purchase, subscription, utility)
                    source: one account | destination: external
  Transfer        — money moves between accounts within the family
                    (P2P between members, cash withdrawal from card, etc.)
                    source: one account | destination: one account
  CashWithdrawal  — special case of Transfer: card/bank → Cash account
```

#### Aggregates

**Transaction** (Aggregate Root)
```
Transaction
  - id: TransactionId (UUID v7)        // v7 = time-sortable; efficient for keyset pagination
  - family_id: FamilyId
  - recorded_by: MemberId
  - kind: TransactionKind
  - amount: Money
  - source_account_id: Option<AccountId>       // null for Income from external
  - destination_account_id: Option<AccountId>  // null for Expense to external
  - category_id: CategoryId
  - tags: Vec<String>
  - description: Option<String>
  - occurred_at: DateTime
  - created_at: DateTime
  - idempotency_key: UUID              // client-generated; prevents duplicates on API retry
  - deleted_at: Option<DateTime>       // soft-delete only
```

**Note on P2P transfers:** When MemberA sends money to MemberB (e.g., reimburses for groceries), this is a `Transfer` from MemberA's account to MemberB's account. Both accounts' balances are updated in the same transaction.

**Category** (Entity)
```
Category
  - id: CategoryId (UUID)
  - family_id: Option<FamilyId>        // None = system default
  - name: String
  - applicable_kinds: Vec<TransactionKind>
  - icon: String
  - color: HexColor
  - deleted_at: Option<DateTime>
```

**Domain Events:** `TransactionRecorded`, `TransactionUpdated`, `TransactionDeleted`, `AccountBalancesAdjusted`

---

### Bounded Context 4: Debt Planner

**Responsibility:** Loan registry and the automated repayment optimizer — the core differentiating feature.

#### Aggregates

**Loan** (Aggregate Root)
```
Loan
  - id: LoanId (UUID)
  - family_id: FamilyId
  - name: String                       // "Maib Car Loan"
  - lender: String
  - kind: LoanKind { Mortgage | CarLoan | PersonalLoan | CreditCard | Other }
  - linked_account_id: Option<AccountId>  // link to credit card account if applicable
  - principal: Money
  - current_balance: Money
  - annual_interest_rate: Decimal      // e.g. 0.19 for 19%
  - monthly_payment: Money             // minimum required
  - next_payment_date: Date
  - opened_at: Date
  - expected_payoff_date: Date
  - status: LoanStatus { Active | PaidOff | Paused }
  - payments: Vec<LoanPayment>
  - deleted_at: Option<DateTime>

LoanPayment
  - id: LoanPaymentId (UUID)
  - paid_at: Date
  - amount: Money
  - principal_portion: Money
  - interest_portion: Money
  - remaining_balance: Money
  - idempotency_key: UUID
```

**RepaymentPlan** (Aggregate Root — always regenerated, never incrementally mutated)
```
RepaymentPlan
  - id: PlanId (UUID)
  - family_id: FamilyId
  - strategy: RepaymentStrategy { Avalanche | Snowball }
  - monthly_income_estimate: Money           // basis for surplus calculation
  - monthly_recurring_obligations: Money     // snapshot from RecurringObligationSummary at generation time
  - monthly_discretionary_ceiling: Money     // living expense budget ceiling
  - total_loan_minimums: Money               // sum of all active loan minimums
  - extra_monthly_budget: Money              // = income − recurring − discretionary − minimums
                                             //   (can be overridden manually by the family)
  - generated_at: DateTime
  - projection: Vec<MonthlyProjection>       // stored as TEXT (JSONB at query time in DSQL)
  - estimated_payoff_date: Date
  - total_interest_saved: Money              // vs minimum-only baseline
```

**Domain Service — RepaymentPlanCalculator:**

The calculator receives three inputs to determine how much is truly available each month for extra loan payments:

```
True Disposable Surplus =
    Average Monthly Income
  − Total Monthly Recurring Obligations  ← from RecurringObligationSummary
  − Total Discretionary Budget Limit      ← living expenses ceiling
  − Sum of All Loan Minimum Payments
  ─────────────────────────────────────
  = Extra budget available for debt acceleration
```

The family sets a **discretionary budget ceiling** (what they plan to spend on groceries, dining, etc.). The calculator deducts this alongside recurring obligations so the plan is realistic — not optimistically assuming all income beyond minimums is free.

The UI pre-fills the `extra_budget` field from `GET /analytics/monthly-surplus` (which performs this calculation over the last 3 months of actual data). The family can override it manually.

- **Avalanche** (default): sort active loans by `annual_interest_rate` DESC → pay minimums on all → apply full extra budget to #1 → when #1 hits zero, roll freed minimum + extra to #2
- **Snowball**: sort by `current_balance` ASC → quick wins for motivation

**Domain Events:** `LoanRegistered`, `LoanPaymentRecorded`, `LoanPaidOff`, `RepaymentPlanGenerated`, `RepaymentStrategyChanged`

---

### Bounded Context 5: Recurring Payments

**Responsibility:** Registry of all fixed, predictable obligations the family pays on a regular schedule — utility bills (water, electricity, gas, internet, phone), subscriptions (streaming services, gym, software), rent, insurance premiums, and anything else that hits every month or quarter whether or not anyone thinks about it. This context is the foundation for computing the family's **true disposable income**: what remains after all recurring obligations are met.

The Recurring Payments context feeds two downstream consumers:
- **Budget context** — each active recurring payment automatically pre-fills its envelope in the monthly budget so the family never has to manually reserve it
- **Debt Planner context** — the repayment plan calculator reads the total committed recurring amount to compute the real surplus available for extra loan payments

#### Aggregates

**RecurringPayment** (Aggregate Root)
```
RecurringPayment
  - id: RecurringPaymentId (UUID)
  - family_id: FamilyId
  - name: String                         // "Maib Internet 100Mbps", "Netflix", "Water Chisinau"
  - kind: RecurringKind {
      Utility        // water, electricity, gas, heating, building maintenance fund
      Subscription   // streaming, gym, software, cloud, newspapers
      Insurance      // car, health, home, life
      Rent           // apartment, garage, storage
      Other          // anything else that doesn't fit above
    }
  - amount: Money                        // typical/expected amount (may vary for utilities)
  - is_variable_amount: bool             // true for utilities: amount changes each period
  - frequency: PaymentFrequency {
      Monthly { day_of_month: u8 }              // e.g. day=25 → due 25th of every month
      Quarterly { month_offset: u8, day: u8 }   // e.g. offset=0 → Jan/Apr/Jul/Oct
      Annual { month: u8, day: u8 }             // e.g. insurance premium once a year
      Irregular                                  // user records manually, no auto-reminder
    }
  - payment_account_id: Option<AccountId>  // which account this is typically paid from
  - category_id: CategoryId
  - next_due_date: Date                  // computed from frequency; updated after each payment
  - is_active: bool
  - notes: Option<String>               // e.g. "Account number: 123456789"
  - created_at: DateTime
  - deleted_at: Option<DateTime>

RecurringPaymentRecord
  - id: UUID
  - recurring_payment_id: RecurringPaymentId
  - actual_amount: Money               // recorded actual amount (may differ from expected)
  - paid_at: Date
  - linked_transaction_id: Option<TransactionId>  // if linked to a Ledger transaction
  - idempotency_key: UUID
```

**Monthly Committed Amount (Domain Service — `RecurringObligationSummary`):**

Computes the total monthly equivalent of all active recurring payments for a family. Quarterly and annual payments are prorated to a monthly figure:
- Monthly → full amount
- Quarterly → amount ÷ 3
- Annual → amount ÷ 12

This figure is published as `MonthlyCommittedAmountComputed` whenever the recurring registry changes.

**Domain Events:**

| Event | Trigger |
|---|---|
| `RecurringPaymentRegistered` | New recurring obligation added |
| `RecurringPaymentUpdated` | Amount or schedule changed |
| `RecurringPaymentRecorded` | Actual payment made for a period |
| `RecurringPaymentMissed` | Due date passed without a payment record |
| `RecurringPaymentDeactivated` | Subscription cancelled, utility no longer used |
| `MonthlyCommittedAmountComputed` | Total obligation sum recalculated |

---

### Bounded Context 6: Budget & Limits

**Responsibility:** Monthly spending envelopes per category. Two sources of data populate each month's budget:
1. **Recurring payments** — auto-seeded from active `RecurringPayment` records when a new month's budget is initialized; these envelopes are pre-filled and marked as committed
2. **Discretionary spending** — manually set limits for groceries, dining, entertainment, etc.

Reacts to `TransactionRecorded` and `RecurringPaymentRecorded` CDC events to update the `spent` counter.

```
MonthlyBudget
  - id: BudgetId (UUID)
  - family_id: FamilyId
  - month: YearMonth                         // "2026-06"
  - total_income_expected: Money             // estimated income for the month
  - total_committed: Money                   // sum of all recurring obligations (auto-computed)
  - total_discretionary_limit: Money         // sum of all non-recurring envelope limits
  - envelopes: Vec<BudgetEnvelope>

BudgetEnvelope
  - id: UUID
  - category_id: CategoryId
  - kind: EnvelopeKind {
      Committed      // pre-filled from a RecurringPayment; amount locked
      Discretionary  // manually set spending limit
    }
  - recurring_payment_id: Option<RecurringPaymentId>  // link for Committed envelopes
  - limit: Money
  - spent: Money                             // updated via CDC events
  - alert_at_percent: u8                     // default 80 for Discretionary; 100 for Committed
  - alerted: bool
```

**Budget initialization flow (when a new month begins):**
1. Copy all active `RecurringPayment` records → create `Committed` envelopes pre-filled with expected amounts
2. Copy prior month's `Discretionary` envelope limits as starting defaults
3. Owner/Member can adjust discretionary limits before or during the month

**Domain Events:** `BudgetInitialized`, `EnvelopeLimitSet`, `BudgetAlertTriggered`, `BudgetMonthClosed`, `CommittedEnvelopeSeeded`

---

### Bounded Context 7: Future Planning

**Responsibility:** Goal-based savings projections.

```
SavingsGoal
  - id: GoalId (UUID)
  - family_id: FamilyId
  - name: String                       // "Summer vacation 2027"
  - target_amount: Money
  - target_date: Date
  - current_saved: Money
  - monthly_contribution: Money        // computed or manual override
  - status: GoalStatus { OnTrack | Behind | Achieved | Paused }
  - deleted_at: Option<DateTime>
```

**Domain Events:** `GoalCreated`, `GoalContributionRecorded`, `GoalAchieved`, `GoalBehindSchedule`

---

### Bounded Context 8: Notifications & Reminders

Purely reactive — listens to EventBridge events from all contexts. Delivers push notifications via web push (Angular/browser) and FCM (Flutter/Android, Phase 2).

Triggered by: `LoanPaymentDue` (EventBridge Scheduler, 3 days before), `RecurringPaymentMissed` (Scheduler, 1 day after due), `BudgetAlertTriggered`, `GoalBehindSchedule`, `RepaymentPlanGenerated`, low card balance (computed in Cards context).

---

### Event Storming Narrative

```
TIMELINE ──────────────────────────────────────────────────────────────────────►

[User registers → creates family]
    └─► FamilyCreated (DB write)
        └─► CDC → Kinesis → Fanout Lambda → EventBridge

[Member registers Maib Visa credit card]
    └─► AccountRegistered (PaymentAccount)
        └─► CDC → Kinesis → Fanout Lambda → EventBridge

[Member pays for groceries with Maib Visa, $47]
    └─► TransactionRecorded (Expense, kind=Expense, source=Maib Visa, category=Groceries)
        ├─► AccountBalancesAdjusted (Maib Visa balance reduced by $47)
        ├─► Budget Lambda: envelope.spent += 47
        │     └─► if >80% → BudgetAlertTriggered → push notification
        └─► Reports Lambda: monthly summary updated

[Member withdraws $200 cash from ATM (Maib Visa → Cash MDL)]
    └─► TransactionRecorded (kind=Transfer/CashWithdrawal, source=Maib Visa, dest=Cash MDL)
        └─► AccountBalancesAdjusted (both accounts updated atomically)

[MemberA reimburses MemberB $50 (P2P transfer)]
    └─► TransactionRecorded (kind=Transfer, source=MemberA Card, dest=MemberB Card)
        └─► AccountBalancesAdjusted (both accounts)

[User registers recurring: "Orange Internet" 200 MDL/month, due 15th]
    └─► RecurringPaymentRegistered
        ├─► MonthlyCommittedAmountComputed (total obligations updated)
        │     └─► Budget Lambda: Committed envelope seeded for current month
        └─► RepaymentPlanGenerated (recalculate: surplus reduced by 200 MDL)
              // The plan now realistically accounts for this fixed cost

[EventBridge Scheduler: 14th of month — internet bill due tomorrow]
    └─► Notification Lambda → push reminder "Orange Internet 200 MDL due tomorrow"

[User pays internet bill, records expense linked to RecurringPayment]
    └─► TransactionRecorded (Expense, Utilities, linked recurring_payment_id)
        ├─► RecurringPaymentRecorded → next_due_date advanced by 1 month
        └─► Budget Committed envelope: spent += 200

[EventBridge Scheduler: 16th — internet bill overdue, no RecurringPaymentRecorded found]
    └─► RecurringPaymentMissed event
        └─► Notification Lambda → push alert "Orange Internet — payment not recorded"

[User registers car loan: 18% APR, $8,000 balance]
    └─► LoanRegistered
        └─► RepaymentPlanGenerated (auto-recalculate with updated minimums)

[User records loan payment: $250]
    └─► LoanPaymentRecorded → balance updated
        └─► if balance = 0 → LoanPaidOff
              └─► RepaymentPlanGenerated (roll freed payment to next loan)

[EventBridge Scheduler: 3 days before loan next_payment_date]
    └─► Notification Lambda → push loan payment reminder

[EventBridge Scheduler: 1st of new month]
    └─► Budget Lambda: initialize new month's budget
          ├─► seed Committed envelopes from all active RecurringPayments
          └─► copy prior month's Discretionary limits as defaults
                └─► BudgetInitialized event

[User adds savings goal: "New Car $12,000 by Dec 2027"]
    └─► GoalCreated → monthly contribution computed
        └─► if projection insufficient → GoalBehindSchedule → push notification
```

---

## Part 4: Technical Architecture

### System Overview

```
                       ┌──────────────────────────────────────────────────────────┐
  Flutter (Android)    │                      AWS Cloud                            │
  Angular (Web SPA)    │                                                            │
        │              │  CloudFront ──► S3 (Angular SPA)                          │
        │ HTTPS        │                                                            │
        ▼              │  API Gateway (HTTP API)                                   │
  ┌──────────┐         │        │                                                  │
  │ Cognito  │◄────────│────────┤ JWT validation (all routes)                      │
  │  Auth    │         │        ▼                                                  │
  └──────────┘         │  ┌──────────────────────────────────┐                    │
                       │  │       Rust Lambda Functions       │                    │
                       │  │  • family-service                 │                    │
                       │  │  • ledger-service                 │                    │
                       │  │  • cards-service                  │                    │
                       │  │  • debt-planner-service           │                    │
                       │  │  • budget-service                 │                    │
                       │  │  • planning-service               │                    │
                       │  │  • notification-service           │                    │
                       │  └───────────────┬──────────────────┘                    │
                       │                  │ read/write                             │
                       │          ┌───────▼────────┐                              │
                       │          │  Aurora DSQL   │──── CDC ──► Kinesis Streams  │
                       │          │  (serverless)  │                   │           │
                       │          └────────────────┘                   ▼           │
                       │                                    ┌─────────────────┐   │
                       │                                    │ CDC Fanout      │   │
                       │                                    │ Lambda          │   │
                       │                                    └────────┬────────┘   │
                       │                                             ▼            │
                       │                                    ┌─────────────────┐   │
                       │                                    │  EventBridge    │   │
                       │                                    └────┬──────┬─────┘   │
                       │                         ┌──────────────┘      └──────┐  │
                       │                         ▼                             ▼  │
                       │                  budget-lambda               notification│
                       │                  planning-lambda             lambda      │
                       │                                                          │
                       │  EventBridge Scheduler ──► notification-lambda           │
                       │  (payment reminders, monthly budget close)               │
                       └──────────────────────────────────────────────────────────┘
```

---

### Connecting Rust to Aurora DSQL

Use the **official AWS connector**: `aurora-dsql-sqlx-connector`. It handles IAM token generation, pool connection refresh, and OCC retry — all the DSQL-specific concerns, invisible to your application code.

```toml
# Cargo.toml
[dependencies]
aurora-dsql-sqlx-connector = { version = "0.1.2", features = ["pool", "occ"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-rustls", "macros", "migrate", "uuid", "chrono", "rust_decimal"] }
```

**Connecting (Lambda startup):**
```rust
use aurora_dsql_sqlx_connector::pool;

pub async fn create_pool() -> Result<sqlx::PgPool> {
    // Connector auto-detects region from hostname, handles IAM token generation
    // and refreshes tokens in the background at 80% of the 15-minute token lifetime
    pool::connect(
        "postgres://admin@your-cluster.dsql.us-east-1.on.aws/postgres"
    ).await.map_err(Into::into)
}
```

**OCC retry (all write operations):**
```rust
use aurora_dsql_sqlx_connector::{retry_on_occ, OCCRetryConfig};

// The closure is re-executed entirely on OCC conflict — must be idempotent
retry_on_occ(&OCCRetryConfig::default(), || async {
    let mut tx = pool.begin().await?;
    sqlx::query!(
        "INSERT INTO ledger_transactions (id, family_id, ...) VALUES ($1, $2, ...)",
        id, family_id, /* ... */
    ).execute(&mut *tx).await?;
    tx.commit().await?;
    Ok(())
}).await?;
```

**Read-only transactions (zero OCC conflict risk):**
```rust
let mut tx = pool.begin().await?;
sqlx::query!("SET TRANSACTION READ ONLY").execute(&mut *tx).await?;
// ... SELECT queries ...
tx.commit().await?;
```

---

### Pagination Strategy

> Reference: [Pagination Patterns in Aurora DSQL](https://aws.amazon.com/blogs/database/pagination-patterns-in-amazon-aurora-dsql/)

All list endpoints use **keyset (cursor-based) pagination**. OFFSET pagination is explicitly avoided — in Aurora DSQL's distributed architecture, `OFFSET N` causes the DB to scan and discard N rows, consuming DPUs proportional to depth, increasing cost with every page flip.

**Keyset pattern for transactions (sorted by occurred_at DESC, id DESC):**

```sql
-- First page
SELECT id, family_id, kind, amount_value, occurred_at
FROM ledger_transactions
WHERE family_id = $1
  AND deleted_at IS NULL
ORDER BY occurred_at DESC, id DESC
LIMIT 50;

-- Next page (cursor = last row's occurred_at + id, base64-encoded in API response)
SELECT id, family_id, kind, amount_value, occurred_at
FROM ledger_transactions
WHERE family_id = $1
  AND deleted_at IS NULL
  AND (occurred_at, id) < ($2, $3)   -- $2 = last_occurred_at, $3 = last_id
ORDER BY occurred_at DESC, id DESC
LIMIT 50;
```

**Why UUID v7 for transaction IDs:** UUID v7 is time-sortable. `(occurred_at, id)` gives a stable, unique sort that supports keyset pagination with no tie-breaking ambiguity. Random UUID v4 would not work as a reliable tiebreaker for ordering.

**Supporting composite index (created async):**
```sql
CREATE INDEX ASYNC idx_tx_family_time ON ledger_transactions (family_id, occurred_at, id)
  INCLUDE (kind, amount_value, amount_currency, category_id);
-- INCLUDE enables index-only scans, avoiding a round-trip to base table storage
```

**API cursor encoding:**
```rust
// Encode last row as opaque base64 cursor — clients treat it as an opaque token
fn encode_cursor(occurred_at: DateTime<Utc>, id: Uuid) -> String {
    let payload = serde_json::json!({"t": occurred_at.to_rfc3339(), "i": id.to_string()});
    base64::encode_config(payload.to_string(), base64::URL_SAFE_NO_PAD)
}

// Response shape
{
  "items": [...],
  "next_cursor": "eyJ0IjoiMjAyNi0wNi0wMVQxMjowMDowMFoiLCJpIjoiMTIzLi4uIn0",
  "has_more": true
}
```

**3,000-row write limit:** All batch operations (e.g., bulk import, end-of-month budget close) must use keyset-iterated batches with ≤ 500 rows per transaction and a separate commit per batch.

**Never use `COUNT(*)` for total page counts** — full table scans consume DPUs proportional to table size. Use the "fetch N+1" trick: request `page_size + 1`, if you get it back there's a next page.

---

### Database Schema (Aurora DSQL)

Key constraints applied throughout:
- All PKs are `UUID` using `gen_random_uuid()` — random UUIDs prevent write hot-spots on DSQL's key-ordered storage
- `JSONB` not supported as column type → store as `TEXT`, cast at query time with `::jsonb`
- No `FOREIGN KEY`, no `ON DELETE CASCADE`, no `SERIAL`, no `TRUNCATE`
- All indexes use `CREATE INDEX ASYNC`
- One DDL statement per migration file

```sql
-- ─── family context ───────────────────────────────────────────────────────
CREATE TABLE family_families (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name            VARCHAR(200)  NOT NULL,
  home_currency   CHAR(3)       NOT NULL DEFAULT 'USD',
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at      TIMESTAMPTZ
);

CREATE TABLE family_members (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id       UUID          NOT NULL,
  user_id         UUID          NOT NULL,        -- Cognito sub
  display_name    VARCHAR(100)  NOT NULL,
  role            VARCHAR(10)   NOT NULL CHECK (role IN ('owner','member','child','other')),
  relationship    VARCHAR(50),                    -- e.g. 'Grandma', 'Cousin'
  joined_at       TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at      TIMESTAMPTZ,
  UNIQUE (family_id, user_id)
);

CREATE TABLE family_invite_tokens (
  token           VARCHAR(64)   PRIMARY KEY,
  family_id       UUID          NOT NULL,
  role            VARCHAR(10)   NOT NULL CHECK (role IN ('owner','member','child','other')),
  relationship    VARCHAR(50),
  created_by      UUID          NOT NULL,
  expires_at      TIMESTAMPTZ   NOT NULL,
  used            BOOLEAN       NOT NULL DEFAULT false
);

-- ─── payment cards & accounts context ────────────────────────────────────
CREATE TABLE cards_accounts (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id           UUID          NOT NULL,
  owner_member_id     UUID          NOT NULL,
  name                VARCHAR(200)  NOT NULL,
  kind                VARCHAR(20)   NOT NULL CHECK (kind IN ('credit_card','debit_card','cash','bank_account')),
  currency            CHAR(3)       NOT NULL,
  current_balance     NUMERIC(19,4) NOT NULL DEFAULT 0,
  credit_limit        NUMERIC(19,4),             -- CreditCard only
  last_four           VARCHAR(4),
  bank_name           VARCHAR(100),
  color               VARCHAR(7),
  included_in_budget  BOOLEAN       NOT NULL DEFAULT true,
  created_at          TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at          TIMESTAMPTZ
);

-- ─── ledger context ───────────────────────────────────────────────────────
CREATE TABLE ledger_categories (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id   UUID,                               -- NULL = system default
  name        VARCHAR(100) NOT NULL,
  kind        VARCHAR(10)  NOT NULL CHECK (kind IN ('income','expense','transfer')),
  icon        VARCHAR(50),
  color       VARCHAR(7),
  deleted_at  TIMESTAMPTZ
);

CREATE TABLE ledger_transactions (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id               UUID          NOT NULL,
  recorded_by             UUID          NOT NULL,  -- member_id
  kind                    VARCHAR(20)   NOT NULL CHECK (kind IN ('income','expense','transfer','cash_withdrawal')),
  amount_value            NUMERIC(19,4) NOT NULL,
  amount_currency         CHAR(3)       NOT NULL,
  source_account_id       UUID,                    -- NULL for external income
  destination_account_id  UUID,                    -- NULL for external expense
  category_id             UUID          NOT NULL,
  tags                    TEXT          NOT NULL DEFAULT '[]',  -- stored as JSON text, cast ::jsonb at query time
  description             TEXT,
  occurred_at             TIMESTAMPTZ   NOT NULL,
  created_at              TIMESTAMPTZ   NOT NULL DEFAULT now(),
  idempotency_key         UUID          NOT NULL UNIQUE,
  deleted_at              TIMESTAMPTZ
);

-- ─── debt planner context ─────────────────────────────────────────────────
CREATE TABLE debt_loans (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id             UUID          NOT NULL,
  name                  VARCHAR(200)  NOT NULL,
  lender                VARCHAR(200),
  kind                  VARCHAR(30)   NOT NULL CHECK (kind IN ('mortgage','car_loan','personal_loan','credit_card','other')),
  linked_account_id     UUID,
  principal_value       NUMERIC(19,4) NOT NULL,
  principal_currency    CHAR(3)       NOT NULL,
  balance_value         NUMERIC(19,4) NOT NULL,
  balance_currency      CHAR(3)       NOT NULL,
  annual_interest_rate  NUMERIC(8,6)  NOT NULL,
  monthly_payment_value NUMERIC(19,4) NOT NULL,
  monthly_payment_curr  CHAR(3)       NOT NULL,
  next_payment_date     DATE          NOT NULL,
  opened_at             DATE          NOT NULL,
  expected_payoff_date  DATE,
  status                VARCHAR(20)   NOT NULL DEFAULT 'active' CHECK (status IN ('active','paid_off','paused')),
  created_at            TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at            TIMESTAMPTZ
);

CREATE TABLE debt_loan_payments (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  loan_id            UUID          NOT NULL,
  family_id          UUID          NOT NULL,
  paid_at            DATE          NOT NULL,
  amount_value       NUMERIC(19,4) NOT NULL,
  amount_currency    CHAR(3)       NOT NULL,
  principal_portion  NUMERIC(19,4) NOT NULL,
  interest_portion   NUMERIC(19,4) NOT NULL,
  remaining_balance  NUMERIC(19,4) NOT NULL,
  created_at         TIMESTAMPTZ   NOT NULL DEFAULT now(),
  idempotency_key    UUID          NOT NULL UNIQUE
);

CREATE TABLE debt_repayment_plans (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id             UUID          NOT NULL,
  strategy              VARCHAR(20)   NOT NULL CHECK (strategy IN ('avalanche','snowball')),
  extra_budget_value    NUMERIC(19,4) NOT NULL,
  extra_budget_currency CHAR(3)       NOT NULL,
  generated_at          TIMESTAMPTZ   NOT NULL DEFAULT now(),
  estimated_payoff      DATE,
  interest_saved_value  NUMERIC(19,4),
  projection            TEXT          NOT NULL   -- JSON serialized Vec<MonthlyProjection>; cast ::jsonb at query time
);

-- ─── recurring payments context ───────────────────────────────────────────
CREATE TABLE recurring_payments (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id            UUID          NOT NULL,
  name                 VARCHAR(200)  NOT NULL,
  kind                 VARCHAR(20)   NOT NULL CHECK (kind IN ('utility','subscription','insurance','rent','other')),
  amount_value         NUMERIC(19,4) NOT NULL,
  amount_currency      CHAR(3)       NOT NULL,
  is_variable_amount   BOOLEAN       NOT NULL DEFAULT false,
  frequency            VARCHAR(20)   NOT NULL CHECK (frequency IN ('monthly','quarterly','annual','irregular')),
  frequency_config     TEXT          NOT NULL DEFAULT '{}',  -- JSON: {"day_of_month":25} or {"month_offset":0,"day":1} etc.
  payment_account_id   UUID,
  category_id          UUID          NOT NULL,
  next_due_date        DATE,
  is_active            BOOLEAN       NOT NULL DEFAULT true,
  notes                TEXT,
  created_at           TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at           TIMESTAMPTZ
);

CREATE TABLE recurring_payment_records (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  recurring_payment_id    UUID          NOT NULL,
  family_id               UUID          NOT NULL,
  actual_amount_value     NUMERIC(19,4) NOT NULL,
  actual_amount_currency  CHAR(3)       NOT NULL,
  paid_at                 DATE          NOT NULL,
  linked_transaction_id   UUID,            -- optional link to ledger_transactions
  created_at              TIMESTAMPTZ   NOT NULL DEFAULT now(),
  idempotency_key         UUID          NOT NULL UNIQUE
);

-- ─── budget context ───────────────────────────────────────────────────────
CREATE TABLE budget_monthly_budgets (
  id                           UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id                    UUID    NOT NULL,
  year_month                   CHAR(7) NOT NULL,
  total_income_expected_value  NUMERIC(19,4),
  total_income_currency        CHAR(3),
  total_committed_value        NUMERIC(19,4) NOT NULL DEFAULT 0,   -- auto-computed from recurring
  total_discretionary_value    NUMERIC(19,4) NOT NULL DEFAULT 0,   -- sum of discretionary limits
  deleted_at                   TIMESTAMPTZ,
  UNIQUE (family_id, year_month)
);

CREATE TABLE budget_envelopes (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  budget_id             UUID          NOT NULL,
  family_id             UUID          NOT NULL,
  category_id           UUID          NOT NULL,
  kind                  VARCHAR(15)   NOT NULL CHECK (kind IN ('committed','discretionary')),
  recurring_payment_id  UUID,                    -- set for kind='committed'
  limit_value           NUMERIC(19,4) NOT NULL,
  limit_currency        CHAR(3)       NOT NULL,
  spent_value           NUMERIC(19,4) NOT NULL DEFAULT 0,
  alert_at_percent      SMALLINT      NOT NULL DEFAULT 80,
  alerted               BOOLEAN       NOT NULL DEFAULT false
);

-- ─── planning context ─────────────────────────────────────────────────────
CREATE TABLE planning_savings_goals (
  id                     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id              UUID          NOT NULL,
  name                   VARCHAR(200)  NOT NULL,
  target_amount_value    NUMERIC(19,4) NOT NULL,
  target_amount_currency CHAR(3)       NOT NULL,
  target_date            DATE          NOT NULL,
  current_saved_value    NUMERIC(19,4) NOT NULL DEFAULT 0,
  monthly_contribution   NUMERIC(19,4),
  status                 VARCHAR(20)   NOT NULL DEFAULT 'on_track' CHECK (status IN ('on_track','behind','achieved','paused')),
  created_at             TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at             TIMESTAMPTZ
);
```

**Indexes** (each in a separate migration file, one DDL op per file):
```sql
-- migration: 20240101000010_indexes.sql (split into one per file)
CREATE INDEX ASYNC idx_members_family ON family_members (family_id) WHERE deleted_at IS NULL;
CREATE INDEX ASYNC idx_accounts_family ON cards_accounts (family_id) WHERE deleted_at IS NULL;
CREATE INDEX ASYNC idx_tx_family_time ON ledger_transactions (family_id, occurred_at, id)
  INCLUDE (kind, amount_value, amount_currency, category_id, source_account_id, destination_account_id)
  WHERE deleted_at IS NULL;
CREATE INDEX ASYNC idx_tx_account_src ON ledger_transactions (source_account_id, occurred_at, id)
  WHERE deleted_at IS NULL;
CREATE INDEX ASYNC idx_tx_account_dst ON ledger_transactions (destination_account_id, occurred_at, id)
  WHERE deleted_at IS NULL;
CREATE INDEX ASYNC idx_loans_family ON debt_loans (family_id) WHERE deleted_at IS NULL;
CREATE INDEX ASYNC idx_loan_payments ON debt_loan_payments (loan_id, paid_at DESC, id DESC);
CREATE INDEX ASYNC idx_recurring_family ON recurring_payments (family_id) WHERE deleted_at IS NULL AND is_active = true;
CREATE INDEX ASYNC idx_recurring_due ON recurring_payments (next_due_date) WHERE deleted_at IS NULL AND is_active = true;
CREATE INDEX ASYNC idx_recurring_records ON recurring_payment_records (recurring_payment_id, paid_at DESC, id DESC);
CREATE INDEX ASYNC idx_goals_family ON planning_savings_goals (family_id) WHERE deleted_at IS NULL;
```

---

### Aurora DSQL Native CDC → Kinesis

> Reference: [Getting Started with CDC in Aurora DSQL](https://aws.amazon.com/blogs/database/getting-started-with-change-data-capture-in-amazon-aurora-dsql/)

Aurora DSQL streams every row-level change to Kinesis Data Streams. The **CDC Fanout Lambda** (Kinesis trigger) interprets these events and publishes semantic domain events to EventBridge for downstream contexts.

**CDC Fanout Lambda — routing table:**

| Table | Op | Domain Event Published |
|---|---|---|
| `ledger_transactions` | `c` (insert) | `TransactionRecorded` |
| `ledger_transactions` | soft-delete | `TransactionDeleted` |
| `cards_accounts` | `c` | `AccountRegistered` or `AccountBalanceUpdated` |
| `debt_loans` | `c` | `LoanRegistered` |
| `debt_loan_payments` | `c` | `LoanPaymentRecorded` |
| `debt_repayment_plans` | `c` | `RepaymentPlanGenerated` |
| `budget_envelopes` | `c` | `EnvelopeLimitSet` |
| all others | any | dropped silently |

**CDC limitations to handle:**
- INSERT and UPDATE both use `op: "c"` — distinguish by checking if UUID already exists in the target context's read model
- DELETE `before` contains only the primary key — use soft-delete (`deleted_at`) for all business entities to preserve row data in CDC events
- At-least-once delivery from Kinesis — all consumers must be idempotent; use `txId + table + PK` as dedup key
- Events may arrive slightly out of order — use `source.ts_ms` for ordering where needed

---

### Rust Lambda Structure

```
workspace/
  ├── domain/                   # Pure Rust: entities, value objects, domain services
  │     ├── family/             # Family, FamilyMember, InviteToken, Role
  │     ├── cards/              # PaymentAccount, AccountKind
  │     ├── ledger/             # Transaction, Category, TransactionKind
  │     ├── debt_planner/       # Loan, RepaymentPlan, RepaymentPlanCalculator
  │     ├── budget/             # MonthlyBudget, BudgetEnvelope
  │     └── planning/           # SavingsGoal, GoalProjectionCalculator
  │
  ├── infrastructure/           # I/O adapters — never referenced from domain/
  │     ├── dsql/               # sqlx queries per context
  │     │     ├── family_repo.rs
  │     │     ├── cards_repo.rs
  │     │     ├── ledger_repo.rs
  │     │     ├── debt_repo.rs
  │     │     └── pagination.rs  # shared keyset cursor encode/decode
  │     └── eventbridge/        # EventBridge publish helpers
  │
  ├── api/                      # Shared: JWT middleware, error mapping, response types
  │
  ├── migrations/               # Versioned SQL files (one DDL per file for DSQL)
  │     ├── 20240101000001_create_family_tables.sql
  │     ├── 20240101000002_create_cards_table.sql
  │     ├── 20240101000003_create_ledger_tables.sql
  │     ├── 20240101000004_create_debt_tables.sql
  │     ├── 20240101000005_create_budget_tables.sql
  │     ├── 20240101000006_create_planning_table.sql
  │     ├── 20240101000010_idx_members_family.sql    # one index per file
  │     ├── 20240101000011_idx_accounts_family.sql
  │     └── ... (one file per CREATE INDEX ASYNC)
  │
  └── lambdas/
        ├── migrate_runner/     # Runs sqlx migrations on deploy
        ├── family_service/
        ├── ledger_service/
        ├── cards_service/
        ├── debt_planner_service/
        ├── budget_service/
        ├── planning_service/
        ├── cdc_fanout/         # Kinesis trigger → EventBridge
        └── notification_service/
```

**Key Rust crates:**
```toml
aurora-dsql-sqlx-connector = { version = "0.2.1", features = ["pool", "occ"] }
sqlx = { version = "0.9.0", features = ["postgres", "runtime-tokio-rustls", "macros", "migrate", "uuid", "chrono", "rust_decimal"] }
axum = { version = "0.8.9", features = ["macros"] }
lambda_http = "1.2.1"
lambda_runtime = "1.2.1"
tower = "0.5"
tower-http = { version = "0.7.0", features = ["trace", "cors"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
rust_decimal = { version = "1", features = ["serde-with-str"] }
rust_decimal_macros = "1"
uuid = { version = "1", features = ["v4", "v7", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
base64 = "0.22"
jsonwebtoken = "10.4.0"
thiserror = "2.0.18"
anyhow = "1.0.82"
tokio = { version = "1", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

---

### REST API Design

All endpoints require Cognito JWT Bearer token. `family_id` is always extracted from JWT — never from request body or URL parameters.

Role-based access enforced per endpoint:

```
# Family
POST   /families                            # any authenticated user
POST   /families/{id}/invites               # Owner only
POST   /families/{id}/members               # accept invite (any auth user with valid token)
GET    /families/{id}/members               # Member+
PATCH  /families/{id}/members/{mid}/role    # Owner only

# Payment Accounts (Cards)
GET    /accounts                            # Member+; children see own only
POST   /accounts                            # Member+
PUT    /accounts/{id}
DELETE /accounts/{id}                       # soft-delete

# Ledger — Transactions
GET    /transactions?cursor=...&limit=50&category=...&account=...&kind=...&from=...&to=...
POST   /transactions                        # body includes idempotency_key
PUT    /transactions/{id}
DELETE /transactions/{id}                   # soft-delete

# Loans
GET    /loans                               # Member+ (not Child)
POST   /loans                              
PUT    /loans/{id}
DELETE /loans/{id}
POST   /loans/{id}/payments               

# Repayment Plan
GET    /loans/repayment-plan               
POST   /loans/repayment-plan/generate      # body: { strategy, extra_budget }

# Analytics
GET    /analytics/monthly-surplus?months=3  # income - expenses - loan minimums
GET    /analytics/by-account?from=...&to=...
GET    /analytics/by-category?from=...&to=...&month=...

# Budget
GET    /budget/{year}/{month}
PUT    /budget/{year}/{month}/envelopes

# Goals
GET    /goals
POST   /goals
PUT    /goals/{id}
DELETE /goals/{id}
POST   /goals/{id}/contributions
```

---

### Frontend Clients

#### Flutter (Android)

- **State management:** Riverpod
- **HTTP:** `dio` with auth token interceptor (auto-refreshes Cognito tokens)
- **Charts:** `fl_chart` (loan payoff curves, expense breakdowns, account balances)
- **Navigation:** `go_router`
- **Local cache:** `shared_preferences` (last-fetched snapshots for offline read)
- **Push notifications:** `firebase_messaging` (FCM)
- **Pagination:** infinite scroll consuming `next_cursor` from API

**Key screens:** Dashboard, Accounts (cards), Transactions (with filters), Loans, Repayment Plan, Budget, Goals

Children's profile shows only: own transactions, own account balance, personal allowance envelope. Loan and planning screens are hidden.

#### Angular (Web SPA)

- **HTTP:** `HttpClient` with `AuthInterceptor` injecting Cognito JWT
- **State:** NgRx Signals Store or standalone signals
- **Charts:** `ngx-charts` or `Chart.js` via `ng2-charts`
- **Tables:** Angular CDK Virtual Scroll for large transaction lists with cursor pagination
- **UI:** Angular Material
- **Auth:** `amazon-cognito-identity-js` or AWS Amplify Auth
- **Pagination:** infinite scroll + explicit "Load more" button, both consuming `next_cursor`

The web app is the primary device for financial planning sessions (debt plan, goal calculator, detailed reports). The Flutter app is for quick transaction entry on the go.

---

## Part 5: Schema Migrations (sqlx migrate — Flyway equivalent)

### How sqlx migrate works

Migration files are plain numbered `.sql` files. sqlx tracks applied migrations in `_sqlx_migrations` table (analogous to Flyway's `flyway_schema_history`). The `sqlx::migrate!()` macro embeds all files into the binary at compile time — no filesystem access needed at Lambda runtime.

```
migrations/
  20240101000001_create_family_tables.sql
  20240101000002_create_cards_table.sql
  ...
  20240101000010_idx_members_family.sql     ← one index per file (DSQL DDL rule)
  20240101000011_idx_accounts_family.sql
  20240115000001_add_relationship_to_member.sql  ← backward-compatible ALTER TABLE
```

### Migration runner Lambda

```rust
// lambdas/migrate_runner/main.rs
static MIGRATOR: sqlx::migrate::Migrator = sqlx::migrate!("../../migrations");

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let pool = aurora_dsql_sqlx_connector::pool::connect(
        &std::env::var("DSQL_ENDPOINT")?
    ).await?;
    MIGRATOR.run(&pool).await?;
    println!("All migrations applied.");
    Ok(())
}
```

GitHub Actions runs this Lambda **before** deploying any service Lambda:
```yaml
- name: Run DB migrations
  run: aws lambda invoke --function-name familyledger-migrate --payload '{}' /tmp/result.json && cat /tmp/result.json
- name: Deploy service Lambdas
  run: cargo lambda deploy ...
```

### Aurora DSQL migration rules

- **One DDL op per migration file** — DSQL allows only one DDL statement per transaction
- **Never mix DDL and DML** in the same migration file
- **Always `CREATE INDEX ASYNC`** — one index per file; verify completion via `SELECT * FROM sys.jobs` before considering the migration done
- **Backward-compatible changes only** — add nullable columns; rename via add+copy+drop sequence across three deployments
- **Never edit applied migrations** — create a new file to fix mistakes
- **No `ALTER TABLE ... ADD FOREIGN KEY`** — not supported; skip entirely
- **No `TRUNCATE`** — use `DELETE FROM table WHERE ...` in batches ≤ 500 rows if needed in a migration

### Compile-time SQL verification

```rust
// Fails at compile time if column doesn't exist — catches schema drift before deploy
let rows = sqlx::query!(
    "SELECT id, amount_value, amount_currency, occurred_at
     FROM ledger_transactions
     WHERE family_id = $1 AND deleted_at IS NULL
     AND (occurred_at, id) < ($2, $3)
     ORDER BY occurred_at DESC, id DESC
     LIMIT $4",
    family_id, last_occurred_at, last_id, page_size
).fetch_all(&pool).await?;
```

Run `cargo sqlx prepare` to generate `.sqlx/` offline snapshot. CI compiles against this snapshot without a live DB.

---

## Part 6: Testing Strategy

```
Test Pyramid
                    ▲
                   ╱ ╲   E2E (reqwest against deployed test stack)
                  ╱───╲
                 ╱ Integ╲  Handler tests (axum tower::ServiceExt, local PG)
                ╱────────╲
               ╱  Unit    ╲  Domain logic, pure Rust — no I/O
              ╱────────────╲
             fast ◄──────► slow
```

### Layer 1: Domain Unit Tests

The `domain/` crate has zero I/O dependencies. `cargo test -p domain` runs in seconds anywhere.

**What to test:**
- `RepaymentPlanCalculator` — avalanche vs snowball ordering, freed-payment rollover, interest-saved calculation
- `Money` arithmetic — always `Decimal`, never `f64`; addition, subtraction, zero checks, currency mismatch guards
- `BudgetEnvelope` alert threshold logic
- `GoalProjectionCalculator` timeline math
- All value object validation (negative money, invalid ISO codes, etc.)
- Transaction kind invariants (Transfer requires both source and destination accounts)

```rust
// domain/src/debt_planner/calculator_tests.rs
#[test]
fn avalanche_targets_highest_rate_first() { ... }

#[test]
fn freed_payment_rolls_to_next_loan_after_payoff() { ... }

#[test]
fn money_uses_decimal_not_float() {
    let a = Money::new(dec!(0.1), "USD");
    let b = Money::new(dec!(0.2), "USD");
    assert_eq!((a + b).amount, dec!(0.3));  // would fail with f64
}

#[test]
fn transfer_requires_both_accounts() {
    let result = Transaction::new_transfer(
        /* source_account_id */ None,  // missing
        /* destination_account_id */ Some(account_id),
        /* ... */
    );
    assert!(result.is_err());
}
```

### Layer 2a: Database Integration Tests (sqlx + testcontainers)

Test real SQL queries against a local PostgreSQL 16 container. Aurora DSQL is wire-compatible with PostgreSQL 16, so local PG validly represents DSQL query behavior. The DSQL-specific features (IAM auth, OCC, async indexes) are tested separately in E2E only.

```toml
# [dev-dependencies]
testcontainers = "0.15"
testcontainers-modules = { version = "0.3", features = ["postgres"] }
```

```rust
// infrastructure/tests/ledger_repo_test.rs
#[tokio::test]
async fn keyset_pagination_returns_correct_page_and_cursor() {
    let pool = setup_test_db().await;  // testcontainers PG, migrations applied

    // Insert 105 transactions with known timestamps
    insert_test_transactions(&pool, 105).await;

    // Page 1
    let page1 = ledger_repo::list_transactions(&pool, family_id, None, 50).await.unwrap();
    assert_eq!(page1.items.len(), 50);
    assert!(page1.next_cursor.is_some());

    // Page 2 using cursor from page 1
    let page2 = ledger_repo::list_transactions(&pool, family_id, page1.next_cursor, 50).await.unwrap();
    assert_eq!(page2.items.len(), 50);

    // Page 3 — only 5 items left
    let page3 = ledger_repo::list_transactions(&pool, family_id, page2.next_cursor, 50).await.unwrap();
    assert_eq!(page3.items.len(), 5);
    assert!(page3.next_cursor.is_none());

    // No duplicates across pages
    let all_ids: Vec<_> = [&page1.items, &page2.items, &page3.items]
        .concat().iter().map(|t| t.id).collect();
    let unique_ids: std::collections::HashSet<_> = all_ids.iter().collect();
    assert_eq!(all_ids.len(), unique_ids.len());
}

#[tokio::test]
async fn idempotent_transaction_insert() {
    let pool = setup_test_db().await;
    let key = Uuid::new_v4();

    ledger_repo::save_transaction(&pool, &make_tx(key)).await.unwrap();
    ledger_repo::save_transaction(&pool, &make_tx(key)).await.unwrap(); // duplicate

    let count: i64 = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM ledger_transactions WHERE idempotency_key = $1", key
    ).fetch_one(&pool).await.unwrap().unwrap();
    assert_eq!(count, 1);
}
```

CI setup:
```yaml
services:
  postgres:
    image: postgres:16
    env: { POSTGRES_PASSWORD: test, POSTGRES_DB: fl_test }
    ports: ['5432:5432']
    options: --health-cmd pg_isready --health-interval 5s --health-retries 5
```

### Layer 2b: Lambda Handler Integration Tests

**The key insight:** Lambda handlers written with `axum` are standard HTTP services. The `tower::ServiceExt::oneshot()` method lets you call them in-process without any AWS infrastructure, API Gateway, or network.

```rust
// lambdas/ledger_service/tests/handler_test.rs
use axum::body::Body;
use axum::http::{Request, StatusCode};
use tower::ServiceExt;

// Expose a test router that skips JWT middleware (auth tested in E2E)
pub fn create_test_router(pool: PgPool, events: Arc<dyn EventPublisher>) -> Router {
    Router::new()
        .route("/transactions", get(list_transactions).post(record_transaction))
        .route("/transactions/:id", put(update_transaction).delete(delete_transaction))
        .with_state(AppState { pool, events })
    // No JWT middleware — inject AuthClaims via request extensions
}

#[tokio::test]
async fn post_transaction_creates_record_and_updates_balance() {
    let pool = setup_test_db_with_seed(&[family_seed(), account_seed()]).await;
    let spy = Arc::new(SpyPublisher::default());
    let app = create_test_router(pool.clone(), spy.clone());

    let request = Request::builder()
        .method("POST")
        .uri("/transactions")
        .header("Content-Type", "application/json")
        .extension(AuthClaims { family_id: TEST_FAMILY_ID, user_id: TEST_USER_ID, role: Role::Member })
        .body(Body::from(serde_json::to_vec(&json!({
            "kind": "expense",
            "amount": { "value": "49.99", "currency": "USD" },
            "source_account_id": TEST_ACCOUNT_ID,
            "category_id": TEST_CATEGORY_ID,
            "occurred_at": "2026-06-01T12:00:00Z",
            "idempotency_key": Uuid::new_v4()
        })).unwrap()))
        .unwrap();

    let response = app.oneshot(request).await.unwrap();

    assert_eq!(response.status(), StatusCode::CREATED);

    // Verify DB state
    let tx = sqlx::query!("SELECT amount_value FROM ledger_transactions WHERE family_id = $1", TEST_FAMILY_ID)
        .fetch_one(&pool).await.unwrap();
    assert_eq!(tx.amount_value, dec!(49.99));

    // Verify account balance updated
    let account = sqlx::query!("SELECT current_balance FROM cards_accounts WHERE id = $1", TEST_ACCOUNT_ID)
        .fetch_one(&pool).await.unwrap();
    assert_eq!(account.current_balance, INITIAL_BALANCE - dec!(49.99));
}

#[tokio::test]
async fn child_role_cannot_see_loans_endpoint() {
    let pool = setup_test_db().await;
    let app = debt_planner_service::create_test_router(pool.clone());

    let request = Request::builder()
        .method("GET")
        .uri("/loans")
        .extension(AuthClaims { role: Role::Child, .. })
        .body(Body::empty()).unwrap();

    let response = app.oneshot(request).await.unwrap();
    assert_eq!(response.status(), StatusCode::FORBIDDEN);
}
```

**Mocking EventBridge:** use a `SpyPublisher` trait implementation that captures events in-memory:

```rust
#[async_trait]
pub trait EventPublisher: Send + Sync {
    async fn publish(&self, event: DomainEvent) -> Result<()>;
}

pub struct SpyPublisher { pub events: Arc<Mutex<Vec<DomainEvent>>> }

#[async_trait]
impl EventPublisher for SpyPublisher {
    async fn publish(&self, event: DomainEvent) -> Result<()> {
        self.events.lock().await.push(event);
        Ok(())
    }
}
```

### Layer 3: End-to-End Tests

Deploy to a dedicated `test` AWS environment (separate Cognito pool, separate Aurora DSQL cluster). Use `reqwest` against the real API Gateway URL. Marked `#[ignore]` so they only run in the deploy pipeline, not on every PR.

```rust
// tests/e2e/card_and_transaction_flow.rs
#[tokio::test]
#[ignore = "requires deployed test stack"]
async fn full_card_registration_expense_and_balance_update() {
    let api = std::env::var("TEST_API_URL").unwrap();
    let token = cognito_test_token().await;
    let client = reqwest::Client::new();

    // Register a debit card
    let card = client.post(format!("{api}/accounts"))
        .bearer_auth(&token)
        .json(&json!({ "name": "Maib Debit", "kind": "debit_card", "currency": "MDL",
                        "current_balance": "5000.00" }))
        .send().await.unwrap();
    assert_eq!(card.status(), 201);
    let card_id: Uuid = card.json::<Value>().await.unwrap()["id"].as_str()
        .unwrap().parse().unwrap();

    // Record an expense from that card
    let tx = client.post(format!("{api}/transactions"))
        .bearer_auth(&token)
        .json(&json!({ "kind": "expense", "amount": { "value": "150.00", "currency": "MDL" },
                        "source_account_id": card_id, "category_id": GROCERY_CATEGORY_ID,
                        "occurred_at": "2026-06-15T10:00:00Z",
                        "idempotency_key": Uuid::new_v4() }))
        .send().await.unwrap();
    assert_eq!(tx.status(), 201);

    // Verify balance updated
    let account = client.get(format!("{api}/accounts/{card_id}"))
        .bearer_auth(&token).send().await.unwrap().json::<Value>().await.unwrap();
    assert_eq!(account["current_balance"]["value"], "4850.00");
}
```

### Test Coverage Targets

| Layer | Command | Goal | Runs in CI when |
|---|---|---|---|
| Domain unit | `cargo test -p domain` | 95%+ | Every commit |
| DB integration | `cargo test -p infrastructure` | 80%+ | Every PR (needs Docker) |
| Handler integration | `cargo test -p ledger_service ...` | 80%+ | Every PR (needs Docker) |
| E2E | `cargo test --test e2e -- --ignored` | Critical paths | On deploy to test stage |

### CI Pipeline

```yaml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo test -p domain

  sqlx-check:
    runs-on: ubuntu-latest
    steps:
      - run: cargo install sqlx-cli --no-default-features --features postgres
      - run: cargo sqlx prepare --check   # verifies .sqlx/ snapshot is up to date

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: fl_test }
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo test -p infrastructure -p ledger_service -p cards_service
               -p debt_planner_service -p budget_service -p planning_service
        env:
          DATABASE_URL: postgres://postgres:test@localhost/fl_test

  build-lambdas:
    runs-on: ubuntu-latest
    steps:
      - run: cargo install cargo-lambda
      - run: cargo lambda build --release --arm64   # ARM = ~20% cheaper Lambda pricing

  e2e-tests:
    needs: [build-lambdas]
    if: github.ref == 'refs/heads/main'
    steps:
      - run: aws lambda invoke --function-name familyledger-migrate --payload '{}' /tmp/r.json
      - run: cargo lambda deploy ...
      - run: cargo test --test e2e -- --ignored
        env:
          TEST_API_URL: ${{ secrets.TEST_API_URL }}
          TEST_COGNITO_CLIENT_ID: ${{ secrets.TEST_COGNITO_CLIENT_ID }}
```

---

## Part 7: Implementation Roadmap

| Sprint | Deliverable |
|---|---|
| **S1** (2 weeks) | AWS infra: Cognito, API Gateway, Aurora DSQL cluster + Kinesis CDC stream, S3 + CloudFront. Rust workspace + `aurora-dsql-sqlx-connector` wired up. Migration runner Lambda working. Schema created. |
| **S2** (2 weeks) | Identity & Family context with all four roles. Angular: login + family setup. Flutter: login screen. Unit + handler tests. |
| **S3** (3 weeks) | Payment Accounts (Cards context). Ledger Core: all transaction types including P2P transfers, cash withdrawals, balance updates. CDC Fanout Lambda. Keyset pagination. Tests. Angular: accounts + transactions screens. Flutter: quick-entry transaction form. |
| **S4** (3 weeks) | Debt Planner: loan registry, payments, repayment plan calculator (avalanche + snowball) with 95%+ unit test coverage. Angular: loans + plan screens + payoff chart. Flutter: loans + plan view. |
| **S5** (2 weeks) | Budget (reacts to CDC events), Future Planning, analytics endpoints. Angular: budget envelopes + goals. Flutter: budget + goals screens. |
| **S6** (2 weeks) | Notifications (EventBridge Scheduler + FCM for Flutter + Web Push for Angular). Offline cache (Flutter SharedPreferences). OCC retry hardening. |
| **S7** (1 week) | E2E tests on test stack, security review, role-based access audit, load test with k6, family beta. |

**Total: ~15 weeks** at ~10 hours/week.

---

## Non-Negotiable Engineering Rules

1. **Money is always `rust_decimal::Decimal`, never `f64`** — see the `money_uses_decimal_not_float` unit test.
2. **All PKs are UUID v4 (`gen_random_uuid()`)** — random UUIDs are mandatory in DSQL to prevent write hot-spots on key-ordered storage.
3. **All list endpoints use keyset pagination** — OFFSET is banned; it consumes DPUs proportional to depth.
4. **Page size ≤ 50 for UI, batch size ≤ 500 for background jobs** — never approach the 3,000-row transaction write limit.
5. **`JSONB` columns → store as `TEXT`, cast as `::jsonb` at query time** — DSQL does not support JSONB as a column type.
6. **Soft-deletes everywhere** (`deleted_at TIMESTAMPTZ`) — preserves CDC event data for the audit trail; no hard DELETEs on business entities.
7. **Aurora DSQL native CDC replaces all outbox polling** — events flow through Kinesis → Fanout Lambda → EventBridge in near-real-time.
8. **All CDC consumers are idempotent** — use `txId + table + PK` as dedup key; Kinesis delivers at-least-once.
9. **`family_id` always comes from the Cognito JWT, never from user input** — the primary security and isolation boundary.
10. **Use `aurora-dsql-sqlx-connector` with `occ` feature** — OCC retry is automatic; never hand-roll it.
11. **Use `BEGIN READ ONLY` in all read-only Lambda handlers** — zero OCC conflict risk for SELECT workloads.
12. **One DDL statement per migration file** — DSQL allows one DDL per transaction; separate index migrations from table migrations.
13. **Never modify applied migration files** — create a new file; same rule as Flyway.
14. **95%+ unit test coverage on `domain/` crate** — this is where all money math and business rules live.
15. **Handler integration tests use real local PostgreSQL via testcontainers** — no mocked DB queries in integration tests.
16. **Children (`Role::Child`) cannot access loans, repayment plans, or family-level analytics** — enforced at API middleware level, not just UI.
