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

### Aggregates & Entities
- **`Family` (Aggregate Root)**: `id: FamilyId`, `name: String`, `home_currency: CurrencyCode`, `members: Vec<FamilyMember>`, `created_at: DateTime<Utc>`
- **`FamilyMember`**: `id: MemberId`, `user_id: UserId`, `display_name: String`, `role: Role`, `relationship: Option<String>`, `joined_at: DateTime<Utc>`
- **`InviteToken`**: `token: String`, `family_id: FamilyId`, `role: Role`, `expires_at: DateTime<Utc>`, `used: bool`

### Roles
- **`Owner`**: Full control over family settings, member invites, and role changes.
- **`Member`**: Adult family member with full visibility into transactions, accounts, loans, and budgets.
- **`Child`**: Restricted member. Can only view and record own transactions. Cannot access debt planning, loans, or family analytics.
- **`Other`**: Extended family (e.g. Grandparent, Cousin). Configurable view permissions.

---

## 2. Payment Cards & Accounts Context

### Responsibility
Registry of all payment instruments across family members (Credit Cards, Debit Cards, Cash Envelopes, Bank Accounts) and their tracked balances.

### Aggregates
- **`PaymentAccount` (Aggregate Root)**:
  - `id: AccountId`
  - `family_id: FamilyId`
  - `owner_member_id: MemberId`
  - `name: String`
  - `kind: AccountKind` (`CreditCard`, `DebitCard`, `Cash`, `BankAccount`)
  - `currency: CurrencyCode`
  - `current_balance: Money`
  - `credit_limit: Option<Money>`
  - `included_in_budget: bool`

---

## 3. Ledger Core Context

### Responsibility
Double-entry compatible financial movement recording. Manages income, expenses, P2P transfers, and cash withdrawals.

### Aggregates & Entities
- **`Transaction` (Aggregate Root)**:
  - `id: TransactionId` (UUID v7, time-sortable)
  - `family_id: FamilyId`
  - `recorded_by: MemberId`
  - `kind: TransactionKind` (`Income`, `Expense`, `Transfer`, `CashWithdrawal`)
  - `amount: Money`
  - `source_account_id: Option<AccountId>`
  - `destination_account_id: Option<AccountId>`
  - `category_id: CategoryId`
  - `tags: Vec<String>`
  - `occurred_at: DateTime<Utc>`
  - `idempotency_key: Uuid`
- **`Category` (Entity)**: `id: CategoryId`, `family_id: Option<FamilyId>`, `name: String`, `kind: TransactionKind`, `color: HexColor`

---

## 4. Debt Planner Context

### Responsibility
Active loan registry and automated repayment plan generation (Avalanche vs. Snowball) factoring in true disposable surplus.

### Aggregates
- **`Loan` (Aggregate Root)**:
  - `id: LoanId`
  - `family_id: FamilyId`
  - `name: String`, `lender: String`
  - `kind: LoanKind` (`Mortgage`, `CarLoan`, `PersonalLoan`, `CreditCard`, `Other`)
  - `principal: Money`, `current_balance: Money`
  - `annual_interest_rate: Decimal`
  - `monthly_payment: Money` (minimum payment)
  - `next_payment_date: NaiveDate`
  - `status: LoanStatus` (`Active`, `PaidOff`, `Paused`)
  - `payments: Vec<LoanPayment>`
- **`RepaymentPlan` (Aggregate Root)**:
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
  - `is_active: bool`

---

## 6. Budget & Limits Context

### Responsibility
Monthly spending envelopes. Automatically pre-seeded with committed recurring obligations, with user-configured discretionary spending limits.

### Aggregates
- **`MonthlyBudget` (Aggregate Root)**:
  - `id: BudgetId`, `family_id: FamilyId`, `year_month: YearMonth`
  - `total_income_expected: Money`
  - `total_committed: Money` (sum of active recurring obligations)
  - `total_discretionary_limit: Money`
  - `envelopes: Vec<BudgetEnvelope>`
- **`BudgetEnvelope`**:
  - `id: Uuid`, `category_id: CategoryId`
  - `kind: EnvelopeKind` (`Committed`, `Discretionary`)
  - `limit: Money`, `spent: Money`
  - `alert_at_percent: u8`, `alerted: bool`

---

## 7. Future Planning Context

### Responsibility
Goal-based savings projection and tracking against family savings capacity.

### Aggregates
- **`SavingsGoal` (Aggregate Root)**:
  - `id: GoalId`, `family_id: FamilyId`, `name: String`
  - `target_amount: Money`, `target_date: NaiveDate`
  - `current_saved: Money`, `monthly_contribution: Option<Money>`
  - `status: GoalStatus` (`OnTrack`, `Behind`, `Achieved`, `Paused`)

---

## 8. Notifications Context

### Responsibility
Event-driven delivery of payment reminders, budget alerts, and overdue notices via Web Push and FCM.

