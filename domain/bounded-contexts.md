# Bounded Contexts & Domain Aggregates — FamilyLedger

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

## 1. Identity & Family Context

### Responsibility
User identity mapping from AWS Cognito `sub`, family workspace aggregate root, and role-based permissions.
*Note: A single user may belong to multiple families. `family_id` for a request comes from the `X-Family-Id` request header (validated by middleware), not from a JWT claim.*

### Aggregates & Entities
- **`Family` (Aggregate Root)**: `id: FamilyId`, `name: String`, `home_currency: CurrencyCode`, `members: Vec<FamilyMember>`, `created_at: DateTime<Utc>`
- **`FamilyMember`**: `id: MemberId`, `user_id: UserId`, `display_name: String`, `role: Role`, `permissions: Option<Vec<Permission>>`, `relationship: Option<String>`, `joined_at: DateTime<Utc>`
  - *Note: `permissions` is only relevant when `role = Other`. NULL means standard Member-equivalent permissions.*
- **`InviteToken`**: `token: String`, `family_id: FamilyId`, `role: Role`, `permissions: Option<Vec<Permission>>`, `expires_at: DateTime<Utc>`, `used: bool`

### Permissions & Roles
Permission flags (only applicable to `Role::Other` members):
- `RecordOwnTransactions` — record/edit transactions on own accounts
- `RecordAnyTransaction` — record on any family account (e.g. shared card)
- `ViewAllTransactions` — see all family transactions, not just own
- `ViewAllAccounts` — see all family accounts, not just own
- `ManageOwnAccounts` — add/edit own payment instruments
- `ViewLoans` — see loans and debt repayment plans
- `ViewBudget` — see monthly budget and envelopes
- `ViewReports` — see analytics and family-level summaries

Predefined roles have fixed permission sets (not configurable):
- **`Owner`**: all permissions + family management
- **`Member`**: all 8 permissions
- **`Child`**: RecordOwnTransactions + ManageOwnAccounts (own accounts only)
- **`Other`**: Owner-configured subset of the 8 flags, set at invite time. Stored as JSON array on `FamilyMember`.

---

## 2. Payment Cards & Accounts Context

### Responsibility
Registry of all payment instruments across family members (Credit Cards, Debit Cards, Cash Envelopes, Bank Accounts) and their tracked balances.
*Note: The table is `payment_accounts` and the service is `accounts_service`.*

### Aggregates & Entities
- **`PaymentAccount` (Aggregate Root)**:
  - `id: AccountId`
  - `family_id: FamilyId`
  - `owner_member_id: MemberId`
  - `name: String`
  - `kind: AccountKind` (`CreditCard`, `DebitCard`, `Cash`, `BankAccount`)
  - `currency: CurrencyCode`
  - `credit_limit: Option<Money>`
  - `included_in_budget: bool`
- **`AccountBalanceSnapshot` (Child Entity)**:
  - `id: SnapshotId`
  - `account_id: AccountId`
  - `family_id: FamilyId`
  - `year_month: YearMonth`   -- 'YYYY-MM'
  - `closing_balance: Money`
  - `computed_at: DateTime<Utc>`

*Note: Current balance for display = latest snapshot `closing_balance` + delta from ledger transactions since snapshot date. Snapshot is computed monthly by `accounts_service` on EventBridge Scheduler trigger.*

---

## 3. Ledger Core Context

### Responsibility
Double-entry compatible financial movement recording. Manages income, expenses, P2P transfers, cash withdrawals, and adjustments.

### Aggregates & Entities
- **`Transaction` (Aggregate Root)**:
  - `id: TransactionId` (UUID v7, time-sortable)
  - `family_id: FamilyId`
  - `recorded_by: MemberId`
  - `kind: TransactionKind` (`Income`, `Expense`, `Transfer`, `CashWithdrawal`, `LoanPayment`, `Reversal`)
    - *`LoanPayment`: Outflow recorded when a loan payment is made, linked from debt planner via event.*
    - *`Reversal`: Negates a prior incorrect transaction as part of the amendment pattern.*
  - `amount: Money`
  - `destination_amount: Option<Money>` -- for cross-currency transfers only; holds the amount received in the destination account's currency. When present, `amount` holds the source deduction in source currency.
  - `source_account_id: Option<AccountId>`
  - `destination_account_id: Option<AccountId>`
  - `category_id: Option<CategoryId>` -- nullable for `Transfer`, `LoanPayment`, and `Reversal` kinds
  - `tags: Vec<String>`
  - `occurred_at: DateTime<Utc>`
  - `amendment_of_id: Option<TransactionId>` -- points to the original transaction being corrected
  - `idempotency_key: Uuid`
- **`Category` (Entity)**: `id: CategoryId`, `family_id: Option<FamilyId>`, `name: String`, `kind: TransactionKind`, `color: HexColor`

*Note on Transaction Immutability: `Transaction` has no update methods. Corrections use the `TransactionAmendment` pattern: `POST /transactions/{id}/amend` creates a `Reversal` entry + new corrected entry, both with `amendment_of_id` pointing to the original.*

---

## 4. Debt Planner Context

### Responsibility
Active loan registry and automated repayment plan generation (Avalanche vs. Snowball) factoring in true disposable surplus.
*Note: Debt Planner scope covers both `debt_loans` AND linked credit card balances (`PaymentAccount` CreditCard with `linked_account_id` set).*

### Aggregates, Entities & Read Models
- **`Loan` (Aggregate Root)**:
  - `id: LoanId`
  - `family_id: FamilyId`
  - `name: String`, `lender: String`
  - `loan_kind_id: LoanKindId`
  - `principal: Money`, `current_balance: Money`
  - `annual_interest_rate: Decimal`
  - `monthly_payment: Money` (minimum payment)
  - `next_payment_date: NaiveDate`
  - `status: LoanStatus` (`Active`, `PaidOff`, `Paused`)
  - `linked_account_id: Option<AccountId>` -- When set (and account kind is CreditCard), spending on that account auto-increments this loan's balance via CDC. See ADR-0014.
- **`LoanPayment` (Child Entity of Loan)**:
  - `id: LoanPaymentId`
  - `loan_id: LoanId`
  - `family_id: FamilyId`
  - `paid_at: NaiveDate`
  - `amount: Money`
  - `principal_portion: Money`
  - `interest_portion: Money`
  - `remaining_balance: Money`
  - `linked_transaction_id: Option<TransactionId>` -- set after LoanPaymentLedgerized event
  - `idempotency_key: Uuid`
  - `created_at: DateTime<Utc>`
  - *Invariant: `principal_portion + interest_portion == amount`*
- **`LoanKind` (Dictionary Entity)**:
  - `id: LoanKindId`
  - `family_id: Option<FamilyId>` -- NULL = system-wide standard
  - `code: String`
  - `name: String`
  - `is_system: bool`
  - *System kinds: mortgage, car_loan, personal_loan, credit_card, student_loan, medical_debt, payday_loan, heloc, business_loan, family_loan, other*
- **`RepaymentPlan` (Read Model / Domain Service Output)**:
  - Output of `RepaymentPlanCalculator`, stored in `debt_repayment_plans` table as a cache. No lifecycle or mutation methods — fully regenerated when loan portfolio changes.
  - `id: PlanId`
  - `strategy: RepaymentStrategy` (`Avalanche`, `Snowball`)
  - `extra_monthly_budget: Money`
  - `projection: Vec<MonthlyProjection>`
  - `estimated_payoff_date: NaiveDate`
  - `total_interest_saved: Money`

---

## 5. Recurring Payments Context

### Responsibility
Registry of all committed obligations (utilities, subscriptions, insurance, rent) that recur on a fixed or predictable schedule.

### Aggregates
- **`RecurringPayment` (Aggregate Root)**:
  - `id: RecurringPaymentId`
  - `family_id: FamilyId`
  - `name: String`, `kind: RecurringKind`
  - `amount: Money`, `is_variable_amount: bool`
  - `frequency: PaymentFrequency` (`Monthly`, `Quarterly`, `Annual`, `Irregular`)
  - `frequency_config: FrequencyConfig`
  - `payment_account_id: Option<AccountId>`
  - `next_due_date: Option<NaiveDate>`
  - `linked_loan_id: Option<LoanId>`
  - `is_active: bool`

*Notes:*
- *A `RecurringPayment(insurance)` with `linked_loan_id` set is called a `LoanBoundInsurance`. The `DebtServiceCost` for a loan = `monthly_payment + SUM(linked RecurringPayments)`.*
- *When `LoanPaidOff` event fires, `notification_service` sends Owner a reminder to review linked insurance payments.*

---

## 6. Budget & Limits Context

### Responsibility
Monthly spending envelopes. Automatically pre-seeded with committed recurring obligations, with user-configured discretionary spending limits.

### Aggregates & Entities
- **`MonthlyBudget` (Aggregate Root)**:
  - `id: BudgetId`, `family_id: FamilyId`, `year_month: YearMonth`
  - `status: BudgetStatus` (`Draft`, `Active`, `Closed`)
  - `total_income_expected: Money`
  - `total_committed: Money` (sum of active recurring obligations)
  - `total_discretionary_limit: Money`
  - `envelopes: Vec<BudgetEnvelope>`
  - `approved_by: Option<MemberId>`
  - `approved_at: Option<DateTime<Utc>>`
- **`BudgetEnvelope` (Child Entity)**:
  - `id: Uuid`, `category_id: CategoryId`
  - `kind: EnvelopeKind` (`Committed`, `Discretionary`)
  - `limit: Money`, `spent: Money`
  - `alert_at_percent: u8`, `alerted: bool`
  - `deleted_at: Option<DateTime<Utc>>`

*Lifecycle Flow Note:*
1. EventBridge Scheduler fires on 1st of month → `BudgetDraftCreated` → `budget_service` creates MonthlyBudget(status=Draft)
2. Recurring payments auto-seed Committed envelopes
3. Owner reviews, adjusts discretionary limits
4. Owner approves → `BudgetApproved` → status=Active, envelopes locked
5. End of month → status=Closed

---

## 7. Future Planning Context

### Responsibility
Goal-based savings projection and tracking against family savings capacity.

### Aggregates & Entities
- **`SavingsGoal` (Aggregate Root)**:
  - `id: GoalId`, `family_id: FamilyId`, `name: String`
  - `target_amount: Money`, `target_date: NaiveDate`
  - `target_monthly_contribution: Option<Money>`
  - `status: GoalStatus` (`OnTrack`, `Behind`, `Achieved`, `Paused`)
  - *Note: `current_saved` is computed as `SUM(GoalContribution.amount)` where `goal_id` matches.*
- **`GoalContribution` (Child Entity of SavingsGoal)**:
  - `id: ContributionId`
  - `goal_id: GoalId`
  - `family_id: FamilyId`
  - `amount: Money`
  - `contributed_at: NaiveDate`
  - `linked_transaction_id: Option<TransactionId>`
  - `note: Option<String>`
  - `idempotency_key: Uuid`

---

## 8. Notifications Context

### Responsibility
Event-driven delivery of payment reminders, budget alerts, and overdue notices via Web Push and FCM.

---

## 9. Currency & Exchange Rates Context

### Responsibility
Fetches and stores live exchange rates for multi-currency balance display and True Disposable Surplus calculation.

### Entities
- **ExchangeRate** (no lifecycle, latest-only record):
  - `base_currency: CurrencyCode`   (home currency, e.g. MDL)
  - `quote_currency: CurrencyCode`  (foreign currency)
  - `rate: Decimal`                  (1 quote = rate base)
  - `fetched_at: DateTime<Utc>`

### Infrastructure Service
- `exchange_rate_fetcher` Lambda — runs daily via EventBridge Scheduler
- Provider: pluggable via `ExchangeRateProvider` trait (ADR-0016)
  - Default: BNM (National Bank of Moldova) — CSV format, daily official rates
  - Alternative: ECB Frankfurter — JSON format
- Emits: `ExchangeRatesRefreshed` domain event
