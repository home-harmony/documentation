# Task 3.1 Detailed Guide — Aurora DSQL Migration Files Authoring & Verification

> **Goal**: Author the complete database schema for FamilyLedger across all 6 bounded contexts as **34 individual SQL migration files** in `backend/migrations/`. Enforce all Amazon Aurora DSQL constraints (1 DDL per file, `CREATE INDEX ASYNC`, UUID v4/v7 PKs, explicit Foreign Keys, native `JSONB`), verify schema integrity against PostgreSQL 16 (via Testcontainers/local PG), and update `cargo sqlx prepare` metadata.

---

## 1. Aurora DSQL Migration Invariants & Rules Checklist

Before creating any SQL files, review the strict constraints governing Amazon Aurora DSQL:

```
┌───────────────────────────────────────┬────────────────────────────────────────────────────────────────────────┐
│ Aurora DSQL Constraint                │ Mandatory Implementation Strategy                                     │
├───────────────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
│ 1 DDL Statement Per File              │ Exactly 1 DDL statement per .sql file. Never combine multiple DDLs.    │
│ No Mixed DDL + DML                    │ Never put INSERT/UPDATE statements in table creation migration files.  │
│ Asynchronous Indexes                  │ Always use CREATE INDEX ASYNC in dedicated individual migration files. │
│ Primary Key Versioning (RFC 9562)     │ UUID v4 for security tokens & low-volume roots; UUID v7 for logs/times.│
│ Foreign Key Referential Integrity     │ Use FOREIGN KEY (col) REFERENCES parent(id) across all tables.        │
│ Semi-Structured Attributes            │ Store as native JSONB with default '[]'::jsonb or '{}'::jsonb.         │
│ Keyset (Cursor) Pagination Alignment  │ Composite indexes ordered by (occurred_at DESC) for stable pagination. │
│ Soft-Deletes Everywhere               │ deleted_at TIMESTAMPTZ NULL on all business-critical entities.         │
└───────────────────────────────────────┴────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Directory Structure & File Manifest

All migration files are located in `backend/migrations/` and use sequential Flyway-style timestamps (`20260616000001` through `20260616000034`):

```
backend/migrations/
├── 20260616000001_create_family_families.sql
├── 20260616000002_create_family_members.sql
├── 20260616000003_create_family_invite_tokens.sql
├── 20260616000004_create_payment_accounts.sql
├── 20260616000005_create_account_balance_snapshots.sql
├── 20260616000006_create_ledger_categories.sql
├── 20260616000007_create_ledger_transactions.sql
├── 20260616000008_create_debt_loan_kinds.sql
├── 20260616000009_create_debt_loans.sql
├── 20260616000010_create_debt_loan_payments.sql
├── 20260616000011_create_debt_repayment_plans.sql
├── 20260616000012_create_recurring_payments.sql
├── 20260616000013_create_recurring_payment_records.sql
├── 20260616000014_create_budget_monthly_budgets.sql
├── 20260616000015_create_budget_envelopes.sql
├── 20260616000016_create_planning_savings_goals.sql
├── 20260616000017_create_goal_contributions.sql
├── 20260616000018_create_exchange_rates.sql
├── 20260616000019_idx_members_family.sql
├── 20260616000020_idx_accounts_family.sql
├── 20260616000021_idx_balance_snapshots_account.sql
├── 20260616000022_idx_tx_family_time.sql
├── 20260616000023_idx_tx_account_src.sql
├── 20260616000024_idx_tx_account_dst.sql
├── 20260616000025_idx_loan_kinds_family.sql
├── 20260616000026_idx_loans_family.sql
├── 20260616000027_idx_loan_payments.sql
├── 20260616000028_idx_recurring_family.sql
├── 20260616000029_idx_recurring_due.sql
├── 20260616000030_idx_recurring_linked_loan.sql
├── 20260616000031_idx_recurring_records.sql
├── 20260616000032_idx_goals_family.sql
├── 20260616000033_idx_goal_contributions.sql
└── 20260616000034_idx_exchange_rates.sql
```

---

## 3. Complete File Contents Specification

### Part 1: Table Migrations (18 Files)

#### `20260616000001_create_family_families.sql`
```sql
CREATE TABLE family_families (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name            VARCHAR(200)  NOT NULL,
  home_currency   CHAR(3)       NOT NULL DEFAULT 'USD',
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at      TIMESTAMPTZ
);
```

#### `20260616000002_create_family_members.sql`
```sql
CREATE TABLE family_members (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id       UUID          NOT NULL REFERENCES family_families(id),
  user_id         UUID          NOT NULL,        -- Cognito sub
  display_name    VARCHAR(100)  NOT NULL,
  role            VARCHAR(10)   NOT NULL CHECK (role IN ('owner','member','child','other')),
  permissions     JSONB         NOT NULL DEFAULT '[]'::jsonb,  -- JSON array of permissions for Role::Other
  relationship    VARCHAR(50),                    -- e.g. 'Grandma', 'Cousin'
  joined_at       TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at      TIMESTAMPTZ,
  UNIQUE (family_id, user_id)
);
```

#### `20260616000003_create_family_invite_tokens.sql`
```sql
CREATE TABLE family_invite_tokens (
  token           UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id       UUID          NOT NULL REFERENCES family_families(id),
  role            VARCHAR(10)   NOT NULL CHECK (role IN ('owner','member','child','other')),
  permissions     JSONB         NOT NULL DEFAULT '[]'::jsonb,
  relationship    VARCHAR(50),
  created_by      UUID          NOT NULL REFERENCES family_members(id),
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
  expires_at      TIMESTAMPTZ   NOT NULL,
  used            BOOLEAN       NOT NULL DEFAULT false
);
```

#### `20260616000004_create_payment_accounts.sql`
```sql
CREATE TABLE payment_accounts (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id           UUID          NOT NULL REFERENCES family_families(id),
  owner_member_id     UUID          NOT NULL REFERENCES family_members(id),
  name                VARCHAR(200)  NOT NULL,
  kind                VARCHAR(20)   NOT NULL CHECK (kind IN ('credit_card','debit_card','cash','bank_account')),
  currency            CHAR(3)       NOT NULL,
  credit_limit        NUMERIC(19,4),
  last_four           VARCHAR(4),
  bank_name           VARCHAR(100),
  color               VARCHAR(7),
  included_in_budget  BOOLEAN       NOT NULL DEFAULT true,
  created_at          TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at          TIMESTAMPTZ
);
```

#### `20260616000005_create_account_balance_snapshots.sql`
```sql
CREATE TABLE account_balance_snapshots (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id      UUID          NOT NULL REFERENCES payment_accounts(id),
  family_id       UUID          NOT NULL REFERENCES family_families(id),
  year_month      CHAR(7)       NOT NULL,  -- 'YYYY-MM'
  closing_balance NUMERIC(19,4) NOT NULL,
  currency        CHAR(3)       NOT NULL,
  computed_at     TIMESTAMPTZ   NOT NULL DEFAULT now(),
  UNIQUE (account_id, year_month)
);
```

#### `20260616000006_create_ledger_categories.sql`
```sql
CREATE TABLE ledger_categories (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id   UUID REFERENCES family_families(id), -- NULL = system default
  name        VARCHAR(100) NOT NULL,
  kind        VARCHAR(10)  NOT NULL CHECK (kind IN ('income','expense','transfer')),
  icon        VARCHAR(50),
  color       VARCHAR(7),
  deleted_at  TIMESTAMPTZ
);
```

#### `20260616000007_create_ledger_transactions.sql`
```sql
CREATE TABLE ledger_transactions (
  id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id                   UUID          NOT NULL REFERENCES family_families(id),
  recorded_by                 UUID          NOT NULL REFERENCES family_members(id),
  kind                        VARCHAR(20)   NOT NULL CHECK (kind IN ('income','expense','transfer','cash_withdrawal','loan_payment','reversal')),
  amount_value                NUMERIC(19,4) NOT NULL,
  amount_currency             CHAR(3)       NOT NULL,
  source_account_id           UUID REFERENCES payment_accounts(id),
  destination_account_id      UUID REFERENCES payment_accounts(id),
  destination_amount_value    NUMERIC(19,4),
  destination_amount_currency CHAR(3),
  category_id                 UUID REFERENCES ledger_categories(id),
  amendment_of_id             UUID REFERENCES ledger_transactions(id),
  tags                        JSONB         NOT NULL DEFAULT '[]'::jsonb,
  description                 TEXT,
  occurred_at                 TIMESTAMPTZ   NOT NULL,
  created_at                  TIMESTAMPTZ   NOT NULL DEFAULT now(),
  idempotency_key             UUID          NOT NULL UNIQUE,
  deleted_at                  TIMESTAMPTZ
);
```

#### `20260616000008_create_debt_loan_kinds.sql`
```sql
CREATE TABLE debt_loan_kinds (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id   UUID REFERENCES family_families(id), -- NULL = system-wide kind
  code        VARCHAR(50)  NOT NULL,
  name        VARCHAR(200) NOT NULL,
  is_system   BOOLEAN      NOT NULL DEFAULT false,
  sort_order  SMALLINT     NOT NULL DEFAULT 0,
  deleted_at  TIMESTAMPTZ,
  UNIQUE (family_id, code)
);
```

#### `20260616000009_create_debt_loans.sql`
```sql
CREATE TABLE debt_loans (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id             UUID          NOT NULL REFERENCES family_families(id),
  name                  VARCHAR(200)  NOT NULL,
  lender                VARCHAR(200),
  loan_kind_id          UUID          NOT NULL REFERENCES debt_loan_kinds(id),
  linked_account_id     UUID REFERENCES payment_accounts(id),
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

#### `20260616000010_create_debt_loan_payments.sql`
```sql
CREATE TABLE debt_loan_payments (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  loan_id               UUID          NOT NULL REFERENCES debt_loans(id),
  family_id             UUID          NOT NULL REFERENCES family_families(id),
  paid_at               DATE          NOT NULL,
  amount_value          NUMERIC(19,4) NOT NULL,
  amount_currency       CHAR(3)       NOT NULL,
  principal_portion     NUMERIC(19,4) NOT NULL,
  interest_portion      NUMERIC(19,4) NOT NULL,
  remaining_balance     NUMERIC(19,4) NOT NULL,
  linked_transaction_id UUID REFERENCES ledger_transactions(id),
  created_at            TIMESTAMPTZ   NOT NULL DEFAULT now(),
  idempotency_key       UUID          NOT NULL UNIQUE
);
```

#### `20260616000011_create_debt_repayment_plans.sql`
```sql
CREATE TABLE debt_repayment_plans (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id             UUID          NOT NULL REFERENCES family_families(id),
  strategy              VARCHAR(20)   NOT NULL CHECK (strategy IN ('avalanche','snowball')),
  extra_budget_value    NUMERIC(19,4) NOT NULL,
  extra_budget_currency CHAR(3)       NOT NULL,
  generated_at          TIMESTAMPTZ   NOT NULL DEFAULT now(),
  estimated_payoff      DATE,
  interest_saved_value  NUMERIC(19,4),
  projection            JSONB         NOT NULL
);
```

#### `20260616000012_create_recurring_payments.sql`
```sql
CREATE TABLE recurring_payments (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id            UUID          NOT NULL REFERENCES family_families(id),
  name                 VARCHAR(200)  NOT NULL,
  kind                 VARCHAR(20)   NOT NULL CHECK (kind IN ('utility','subscription','insurance','rent','other')),
  amount_value         NUMERIC(19,4) NOT NULL,
  amount_currency      CHAR(3)       NOT NULL,
  is_variable_amount   BOOLEAN       NOT NULL DEFAULT false,
  frequency            VARCHAR(20)   NOT NULL CHECK (frequency IN ('monthly','quarterly','annual','irregular')),
  frequency_config     JSONB         NOT NULL DEFAULT '{}'::jsonb,
  payment_account_id   UUID REFERENCES payment_accounts(id),
  category_id          UUID          NOT NULL REFERENCES ledger_categories(id),
  linked_loan_id       UUID REFERENCES debt_loans(id),
  next_due_date        DATE,
  is_active            BOOLEAN       NOT NULL DEFAULT true,
  notes                TEXT,
  created_at           TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at           TIMESTAMPTZ
);
```

#### `20260616000013_create_recurring_payment_records.sql`
```sql
CREATE TABLE recurring_payment_records (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  recurring_payment_id    UUID          NOT NULL REFERENCES recurring_payments(id),
  family_id               UUID          NOT NULL REFERENCES family_families(id),
  actual_amount_value     NUMERIC(19,4) NOT NULL,
  actual_amount_currency  CHAR(3)       NOT NULL,
  paid_at                 DATE          NOT NULL,
  linked_transaction_id   UUID REFERENCES ledger_transactions(id),
  created_at              TIMESTAMPTZ   NOT NULL DEFAULT now(),
  idempotency_key         UUID          NOT NULL UNIQUE
);
```

#### `20260616000014_create_budget_monthly_budgets.sql`
```sql
CREATE TABLE budget_monthly_budgets (
  id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id                   UUID          NOT NULL REFERENCES family_families(id),
  year_month                  CHAR(7)       NOT NULL,  -- 'YYYY-MM'
  total_income_expected_value NUMERIC(19,4) NOT NULL,
  total_income_currency       CHAR(3)       NOT NULL,
  status                      VARCHAR(20)   NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','active')),
  created_at                  TIMESTAMPTZ   NOT NULL DEFAULT now(),
  approved_at                 TIMESTAMPTZ,
  approved_by                 UUID REFERENCES family_members(id),
  UNIQUE (family_id, year_month)
);
```

#### `20260616000015_create_budget_envelopes.sql`
```sql
CREATE TABLE budget_envelopes (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  budget_id           UUID          NOT NULL REFERENCES budget_monthly_budgets(id),
  category_id         UUID          NOT NULL REFERENCES ledger_categories(id),
  limit_amount_value  NUMERIC(19,4) NOT NULL,
  limit_currency      CHAR(3)       NOT NULL,
  order_index         SMALLINT      NOT NULL DEFAULT 0,
  alert_threshold     SMALLINT      NOT NULL DEFAULT 80, -- percentage (80 = 80%)
  UNIQUE (budget_id, category_id)
);
```

#### `20260616000016_create_planning_savings_goals.sql`
```sql
CREATE TABLE planning_savings_goals (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id           UUID          NOT NULL REFERENCES family_families(id),
  name                VARCHAR(200)  NOT NULL,
  target_amount_value NUMERIC(19,4) NOT NULL,
  target_currency     CHAR(3)       NOT NULL,
  target_date         DATE,
  linked_account_id   UUID REFERENCES payment_accounts(id),
  color               VARCHAR(7),
  icon                VARCHAR(50),
  status              VARCHAR(20)   NOT NULL DEFAULT 'in_progress' CHECK (status IN ('in_progress','achieved','cancelled')),
  created_at          TIMESTAMPTZ   NOT NULL DEFAULT now(),
  deleted_at          TIMESTAMPTZ
);
```

#### `20260616000017_create_goal_contributions.sql`
```sql
CREATE TABLE goal_contributions (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  goal_id               UUID          NOT NULL REFERENCES planning_savings_goals(id),
  family_id             UUID          NOT NULL REFERENCES family_families(id),
  amount_value          NUMERIC(19,4) NOT NULL,
  amount_currency       CHAR(3)       NOT NULL,
  contributed_at        DATE          NOT NULL,
  linked_transaction_id UUID REFERENCES ledger_transactions(id),
  created_at            TIMESTAMPTZ   NOT NULL DEFAULT now(),
  idempotency_key       UUID          NOT NULL UNIQUE
);
```

#### `20260616000018_create_exchange_rates.sql`
```sql
CREATE TABLE exchange_rates (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  base_currency  CHAR(3)       NOT NULL,
  quote_currency CHAR(3)       NOT NULL,
  rate           NUMERIC(18,8) NOT NULL,
  fetched_at     TIMESTAMPTZ   NOT NULL,
  UNIQUE (base_currency, quote_currency, fetched_at)
);
```

---

### Part 2: Asynchronous Index Migrations (16 Files)

#### `20260616000019_idx_members_family.sql`
```sql
CREATE INDEX ASYNC idx_members_family ON family_members (family_id);
```

#### `20260616000020_idx_accounts_family.sql`
```sql
CREATE INDEX ASYNC idx_accounts_family ON payment_accounts (family_id);
```

#### `20260616000021_idx_balance_snapshots_account.sql`
```sql
CREATE INDEX ASYNC idx_balance_snapshots_account ON account_balance_snapshots (account_id, year_month);
```

#### `20260616000022_idx_tx_family_time.sql`
```sql
CREATE INDEX ASYNC idx_tx_family_time ON ledger_transactions (family_id, occurred_at DESC);
```

#### `20260616000023_idx_tx_account_src.sql`
```sql
CREATE INDEX ASYNC idx_tx_account_src ON ledger_transactions (source_account_id, occurred_at DESC);
```

#### `20260616000024_idx_tx_account_dst.sql`
```sql
CREATE INDEX ASYNC idx_tx_account_dst ON ledger_transactions (destination_account_id, occurred_at DESC);
```

#### `20260616000025_idx_loan_kinds_family.sql`
```sql
CREATE INDEX ASYNC idx_loan_kinds_family ON debt_loan_kinds (family_id);
```

#### `20260616000026_idx_loans_family.sql`
```sql
CREATE INDEX ASYNC idx_loans_family ON debt_loans (family_id, status);
```

#### `20260616000027_idx_loan_payments.sql`
```sql
CREATE INDEX ASYNC idx_loan_payments ON debt_loan_payments (loan_id, paid_at DESC);
```

#### `20260616000028_idx_recurring_family.sql`
```sql
CREATE INDEX ASYNC idx_recurring_family ON recurring_payments (family_id, is_active);
```

#### `20260616000029_idx_recurring_due.sql`
```sql
CREATE INDEX ASYNC idx_recurring_due ON recurring_payments (next_due_date) WHERE is_active = true;
```

#### `20260616000030_idx_recurring_linked_loan.sql`
```sql
CREATE INDEX ASYNC idx_recurring_linked_loan ON recurring_payments (linked_loan_id) WHERE linked_loan_id IS NOT NULL;
```

#### `20260616000031_idx_recurring_records.sql`
```sql
CREATE INDEX ASYNC idx_recurring_records ON recurring_payment_records (recurring_payment_id, paid_at DESC);
```

#### `20260616000032_idx_goals_family.sql`
```sql
CREATE INDEX ASYNC idx_goals_family ON planning_savings_goals (family_id, status);
```

#### `20260616000033_idx_goal_contributions.sql`
```sql
CREATE INDEX ASYNC idx_goal_contributions ON goal_contributions (goal_id, contributed_at DESC);
```

#### `20260616000034_idx_exchange_rates.sql`
```sql
CREATE INDEX ASYNC idx_exchange_rates ON exchange_rates (base_currency, quote_currency, fetched_at DESC);
```

---

## 4. Step-by-Step Implementation & Verification Guide

### Step 1: Create the `backend/migrations/` Directory
```powershell
# Run from workspace root
New-Item -ItemType Directory -Force -Path "backend/migrations"
```

### Step 2: Generate All 34 Migration Files
Create each file listed in Section 3 under `backend/migrations/`.

### Step 3: Run Database Migrations via SQLx
Ensure Docker or local PostgreSQL is running:
```powershell
# Set DATABASE_URL for local development
$env:DATABASE_URL = "postgres://postgres:postgres@localhost:5432/familyledger"

# Run all migrations
sqlx migrate run --source backend/migrations
```

### Step 4: Generate SQLx Offline Query Cache
To ensure CI and build targets can verify SQL queries compile-time without a live database:
```powershell
cd backend
cargo sqlx prepare -- --all-targets
```

---

## 5. Acceptance Criteria & Definition of Done

- [ ] Exactly 34 migration files exist in `backend/migrations/`.
- [ ] Every file contains **exactly one DDL statement**.
- [ ] No DML (`INSERT`, `UPDATE`) is present in any migration file.
- [ ] All table PKs use `UUID PRIMARY KEY DEFAULT gen_random_uuid()`.
- [ ] All indexes use `CREATE INDEX ASYNC`.
- [ ] Foreign key constraints (`REFERENCES`) are present and valid across all related tables.
- [ ] `sqlx migrate run` applies all 34 migrations in order without error.
- [ ] `cargo sqlx prepare -- --all-targets` runs and generates `.sqlx/` cache cleanly.

