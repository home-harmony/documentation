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
- **`Family` (Aggregate Root)**: `id: FamilyId`, `name: FamilyName`, `home_currency: CurrencyCode`, `members: Vec<FamilyMember>`, `invite_tokens: Vec<InviteToken>`, `created_at: DateTime<Utc>`, `deleted_at: Option<DateTime<Utc>>`
- **`FamilyMember`**: `id: MemberId`, `family_id: FamilyId`, `user_id: UserId`, `display_name: MemberDisplayName`, `role: Role`, `permissions: Vec<Permission>`, `relationship: Option<Relationship>`, `joined_at: DateTime<Utc>`, `deleted_at: Option<DateTime<Utc>>`
  - *Note: `permissions` is only relevant when `role = Guest` (empty `[]` for predefined roles, whose permissions are derived directly from `Role`).*
- **`InviteToken`**: `id: InviteId`, `token_hash: InviteTokenHash`, `family_id: FamilyId`, `role: Role`, `permissions: Vec<Permission>`, `relationship: Option<Relationship>`, `created_by: MemberId`, `created_at: DateTime<Utc>`, `expires_at: DateTime<Utc>`, `used: bool`

### Permissions & Roles
Permission flags (only applicable to `Role::Guest` members):
- `RecordOwnTransactions` — record/amend/delete transactions on own accounts
- `RecordAnyTransaction` — record/amend/delete on any family account and manage custom categories
- `ViewAllTransactions` — see all family transactions, not just own
- `ViewAllAccounts` — see all family accounts, not just own
- `ManageOwnAccounts` — add/edit/delete own payment instruments
- `ViewLoans` — see loans and debt repayment plans
- `ViewBudget` — see monthly budget and envelopes
- `ViewReports` — see analytics and family-level summaries

Predefined roles have fixed permission sets (not configurable):
- **`Owner`**: all 8 ledger/view permissions + workspace read (`DomainAction::ViewFamilyWorkspace`) + family management (`DomainAction::ManageFamily`)
- **`Member`**: all 8 ledger/view permissions + workspace read (`DomainAction::ViewFamilyWorkspace`)
- **`Child`**: `RecordOwnTransactions` + `ManageOwnAccounts` (own accounts only) + workspace read (`DomainAction::ViewFamilyWorkspace`)
- **`Guest`**: workspace read (`DomainAction::ViewFamilyWorkspace`) + Owner-configured subset of the 8 flags, set at invite time (stored as JSON array on `FamilyMember`; UI label `"Guest"` / `«Гость»`).

---

## 2. Payment Cards & Accounts Context

### Responsibility
Registry of all payment instruments across family members (Credit Cards, Debit Cards, Cash Envelopes, Bank Accounts) and their derived balances.
*Note: The table is `payment_accounts` and the service is `accounts_service`.*

### Aggregates & Entities
- **`PaymentAccount` (Aggregate Root)**:
  - `id: AccountId` (UUID v4)
  - `family_id: FamilyId`
  - `owner_member_id: MemberId` (immutable after creation)
  - `name: AccountName` (`1..=200` chars)
  - `kind: AccountKind` (`CreditCard`, `DebitCard`, `Cash`, `BankAccount` — immutable after creation)
  - `currency: CurrencyCode` (immutable after creation)
  - `opening_balance: SignedMoney` (immutable baseline set once at creation, defaults to `0` — never mutated on transactions)
  - `credit_limit: Option<Money>` (permitted strictly when `kind == CreditCard`)
  - `last_four: Option<LastFour>` (4 ASCII digits)
  - `bank_name: Option<BankName>` (`1..=100` chars)
  - `color: Option<HexColor>` (`#RRGGBB`)
  - `included_in_budget: bool`
  - `created_at: DateTime<Utc>`, `deleted_at: Option<DateTime<Utc>>`
- **`AccountBalanceSnapshot` (Child Entity)**:
  - `id: SnapshotId` (`AccountBalanceSnapshotId`, UUID v7)
  - `account_id: AccountId`
  - `family_id: FamilyId`
  - `year_month: YearMonth`   -- 'YYYY-MM'
  - `closing_balance: SignedMoney` -- signed decimal + `CurrencyCode` (folds in `opening_balance` on the first snapshot)
  - `computed_at: DateTime<Utc>`

*Note: Current balance for display = latest snapshot `closing_balance` + delta from active (`deleted_at IS NULL`) ledger transactions since snapshot month-end (or `opening_balance` + full active ledger sum before the first snapshot). Snapshot computation runs monthly on EventBridge Scheduler trigger (wired in Sprint 6).*

---

## 3. Ledger Core Context

### Responsibility
Double-entry compatible financial movement recording and category management. Manages income, expenses, internal/cross-currency transfers, cash withdrawals, loan payment outflows, and immutable accounting amendments.

### Aggregates & Entities
- **`Transaction` (Aggregate Root)**:
  - `id: TransactionId` (UUID v7, time-sortable)
  - `family_id: FamilyId`
  - `recorded_by: MemberId`
  - `kind: TransactionKind` (`Income`, `Expense`, `Transfer`, `CashWithdrawal`, `LoanPayment`, `Reversal`)
    - *`CashWithdrawal`: Outflow from a non-cash account (`CreditCard`, `DebitCard`, `BankAccount`) into a `Cash` account (supports same-currency and cross-currency ATM withdrawals).*
    - *`LoanPayment`: Outflow recorded when a loan payment is made, linked from Debt Planner via EventBridge.*
    - *`Reversal`: Negates a superseded transaction as part of `TransactionAmendment` by swapping `source_account_id` ↔ `destination_account_id` (and `amount` ↔ `destination_amount` on cross-currency rows).*
  - `amount: PositiveMoney` (strictly `> 0`, in source currency — or destination currency for external `Income` / `Reversal` inflows)
  - `destination_amount: Option<PositiveMoney>` -- required iff source and destination account currencies differ on `Transfer`, `CashWithdrawal`, or their `Reversal`; forbidden when currencies match.
  - `source_account_id: Option<AccountId>`
  - `destination_account_id: Option<AccountId>`
  - `category_id: Option<CategoryId>` -- required for `Income` (`CategoryKind::Income`) and `Expense` (`CategoryKind::Expense`); optional (`CategoryKind::Transfer`) for `Transfer` and `CashWithdrawal`; `None` for `LoanPayment` and `Reversal`.
  - `tags: Vec<Tag>` (`<= 20` unique trimmed tags, `1..=40` chars each)
  - `description: Option<TransactionDescription>` (`1..=500` chars)
  - `occurred_at: DateTime<Utc>`, `created_at: DateTime<Utc>`
  - `amendment_of_id: Option<TransactionId>` -- points to the superseded original transaction
  - `idempotency_key: Uuid` (`UNIQUE (family_id, idempotency_key)`)
  - `deleted_at: Option<DateTime<Utc>>` -- set only by `DELETE /transactions/{id}` (never set on amended originals)
- **`Category` (Aggregate Root)**:
  - `id: CategoryId` (UUID v4), `family_id: Option<FamilyId>` (`None` = read-only global system category), `name: CategoryName` (`1..=100` chars), `kind: CategoryKind` (`Income`, `Expense`, `Transfer`), `icon: Option<CategoryIcon>` (`1..=50` chars), `color: Option<HexColor>`, `deleted_at: Option<DateTime<Utc>>`

*Note on Transaction Immutability: `Transaction` has no in-place update methods (`PUT /transactions/{id}` returns `405`). Corrections use `POST /transactions/{id}/amend`, which atomically inserts a `Reversal` entry (with deterministic derived `idempotency_key`) + a corrected `Transaction` (`Reversal.id < Corrected.id`), keeping `original.deleted_at = NULL` so snapshot balance math stays exact.*

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
  - `linked_account_id: Option<AccountId>` -- When set (and account kind is CreditCard), spending on that account auto-increments this loan's balance via EventBridge (`TransactionRecorded`). See ADR-0014.
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
  - `status: BudgetStatus` (`Draft`, `Active`)
  - `total_income_expected: Money`
  - `total_committed: Money` (sum of active recurring obligations)
  - `total_discretionary_limit: Money`
  - `envelopes: Vec<BudgetEnvelope>`
  - `approved_by: Option<MemberId>`
  - `approved_at: Option<DateTime<Utc>>`
- **`BudgetEnvelope` (Child Entity)**:
  - `id: BudgetEnvelopeId`, `budget_id: BudgetId`, `category_id: CategoryId`
  - `limit_amount: Money`, `order_index: i16`, `alert_threshold: i16`
  - *Note: `spent` is computed dynamically via `SUM(ledger_transactions)` at read time (no stored `spent_value` column) to avoid OCC hot rows and handle amendments accurately.*

*Lifecycle Flow Note:*
1. EventBridge Scheduler fires on 1st of month → `BudgetDraftCreated` → `budget_service` creates MonthlyBudget(status=Draft)
2. Recurring payments auto-seed Committed envelopes
3. Owner reviews, adjusts discretionary limits
4. Owner approves → `BudgetApproved` → status=Active, envelopes locked (historical months remain `Active` for read-only reporting)

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
