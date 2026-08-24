# Database Schema Specification — Aurora DSQL

All tables and indexes conform to Aurora DSQL constraints:
- UUID v4 / v7 PKs using `gen_random_uuid()`.
- No DB Foreign Keys (`REFERENCES`).
- No native JSONB columns (JSON is stored as `TEXT` and cast `::jsonb` at query time).
- Soft-deletes (`deleted_at TIMESTAMPTZ NULL`).
- Asynchronous index creation (`CREATE INDEX ASYNC`).

---

## 1. Table Definitions

### 1.1 Identity & Family Context

#### `family_families`
```sql
CREATE TABLE family_families (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name            VARCHAR(200)  NOT NULL,
  home_currency   CHAR(3)       NOT NULL DEFAULT 'USD',
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at      TIMESTAMPTZ
);
```

#### `family_members`
```sql
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
```

#### `family_invite_tokens`
```sql
CREATE TABLE family_invite_tokens (
  token           VARCHAR(64)   PRIMARY KEY,
  family_id       UUID          NOT NULL,
  role            VARCHAR(10)   NOT NULL CHECK (role IN ('owner','member','child','other')),
  relationship    VARCHAR(50),
  created_by      UUID          NOT NULL,
  expires_at      TIMESTAMPTZ   NOT NULL,
  used            BOOLEAN       NOT NULL DEFAULT false
);
```

---

### 1.2 Payment Cards & Accounts Context

#### `cards_accounts`
```sql
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
```

---

### 1.3 Ledger Core Context

#### `ledger_categories`
```sql
CREATE TABLE ledger_categories (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id   UUID,                               -- NULL = system default
  name        VARCHAR(100) NOT NULL,
  kind        VARCHAR(10)  NOT NULL CHECK (kind IN ('income','expense','transfer')),
  icon        VARCHAR(50),
  color       VARCHAR(7),
  deleted_at  TIMESTAMPTZ
);
```

#### `ledger_transactions`
```sql
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
  tags                    TEXT          NOT NULL DEFAULT '[]',  -- JSON array as text
  description             TEXT,
  occurred_at             TIMESTAMPTZ   NOT NULL,
  created_at              TIMESTAMPTZ   NOT NULL DEFAULT now(),
  idempotency_key         UUID          NOT NULL UNIQUE,
  deleted_at              TIMESTAMPTZ
);
```

---

### 1.4 Debt Planner Context

#### `debt_loans`
```sql
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
```

#### `debt_loan_payments`
```sql
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
```

#### `debt_repayment_plans`
```sql
CREATE TABLE debt_repayment_plans (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id             UUID          NOT NULL,
  strategy              VARCHAR(20)   NOT NULL CHECK (strategy IN ('avalanche','snowball')),
  extra_budget_value    NUMERIC(19,4) NOT NULL,
  extra_budget_currency CHAR(3)       NOT NULL,
  generated_at          TIMESTAMPTZ   NOT NULL DEFAULT now(),
  estimated_payoff      DATE,
  interest_saved_value  NUMERIC(19,4),
  projection            TEXT          NOT NULL   -- JSON serialized Vec<MonthlyProjection>
);
```

---

### 1.5 Recurring Payments Context

#### `recurring_payments`
```sql
CREATE TABLE recurring_payments (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id            UUID          NOT NULL,
  name                 VARCHAR(200)  NOT NULL,
  kind                 VARCHAR(20)   NOT NULL CHECK (kind IN ('utility','subscription','insurance','rent','other')),
  amount_value         NUMERIC(19,4) NOT NULL,
  amount_currency      CHAR(3)       NOT NULL,
  is_variable_amount   BOOLEAN       NOT NULL DEFAULT false,
  frequency            VARCHAR(20)   NOT NULL CHECK (frequency IN ('monthly','quarterly','annual','irregular')),
  frequency_config     TEXT          NOT NULL DEFAULT '{}',  -- JSON config
  payment_account_id   UUID,
  category_id          UUID          NOT NULL,
  next_due_date        DATE,
  is_active            BOOLEAN       NOT NULL DEFAULT true,
  notes                TEXT,
  created_at           TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at           TIMESTAMPTZ
);
```

#### `recurring_payment_records`
```sql
CREATE TABLE recurring_payment_records (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  recurring_payment_id    UUID          NOT NULL,
  family_id               UUID          NOT NULL,
  actual_amount_value     NUMERIC(19,4) NOT NULL,
  actual_amount_currency  CHAR(3)       NOT NULL,
  paid_at                 DATE          NOT NULL,
  linked_transaction_id   UUID,
  created_at              TIMESTAMPTZ   NOT NULL DEFAULT now(),
  idempotency_key         UUID          NOT NULL UNIQUE
);
```

---

### 1.6 Budget & Planning Context

#### `budget_monthly_budgets`
```sql
CREATE TABLE budget_monthly_budgets (
  id                           UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id                    UUID    NOT NULL,
  year_month                   CHAR(7) NOT NULL,
  total_income_expected_value  NUMERIC(19,4),
  total_income_currency        CHAR(3),
  total_committed_value        NUMERIC(19,4) NOT NULL DEFAULT 0,
  total_discretionary_value    NUMERIC(19,4) NOT NULL DEFAULT 0,
  deleted_at                   TIMESTAMPTZ,
  UNIQUE (family_id, year_month)
);
```

#### `budget_envelopes`
```sql
CREATE TABLE budget_envelopes (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  budget_id             UUID          NOT NULL,
  family_id             UUID          NOT NULL,
  category_id           UUID          NOT NULL,
  kind                  VARCHAR(15)   NOT NULL CHECK (kind IN ('committed','discretionary')),
  recurring_payment_id  UUID,
  limit_value           NUMERIC(19,4) NOT NULL,
  limit_currency        CHAR(3)       NOT NULL,
  spent_value           NUMERIC(19,4) NOT NULL DEFAULT 0,
  alert_at_percent      SMALLINT      NOT NULL DEFAULT 80,
  alerted               BOOLEAN       NOT NULL DEFAULT false
);
```

#### `planning_savings_goals`
```sql
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

---

## 2. Asynchronous Indexes

Each index statement is defined in a separate migration file and executed with `CREATE INDEX ASYNC`:

```sql
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

