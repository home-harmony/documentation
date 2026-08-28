# FamilyLedger — Design Review 2026-08-28

> **Status:** Decisions captured — implementation postponed.  
> **Session type:** Automated documentation audit + interactive design interview.  
> **Scope:** All documentation under `documentation/`, backend scaffold in `backend/`.

---

## Background

A full audit of all 28 documentation files and the backend code scaffold was performed.
The audit identified 25 issues ranging from critical contradictions to minor gaps.
13 issues were resolved in an interactive design session; the rest are tracked in the
[Remaining Open Issues](#remaining-open-issues) section below.

---

## Resolved Decisions

### Decision 1 — Role System Redesign

**Problem:** ADR-0007 referenced a non-existent `Admin` role (does not exist anywhere in
`bounded-contexts.md`, `ubiquitous-language.md`, or the DB schema). The current four-role
system (`Owner`, `Member`, `Child`, `Other`) also conflated `Other` with "custom permissions"
in an awkward way, and neither `FamilyMember` nor `InviteToken` structs in code carried a
`permissions` field.

**Decision:** Redesign the role system as follows:

- **Immutable system roles** — fixed permission sets, cannot be modified:
  - `Owner` — all permissions + family management
  - `Member` — all adult permissions
  - `Child` — `RecordOwnTransactions` + `ManageOwnAccounts` only
- **Custom roles** — Owner creates family-defined roles with a `Set<Permission>`.
  Custom roles replace the old `Other` concept entirely.
- **Role reassignment** — Owner can change a member's role at any time (e.g., promote
  a `Child` to `Member` when they come of age).

**Schema changes required:**

```sql
-- New table: family_roles
CREATE TABLE family_roles (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id   UUID         NOT NULL,
  name        VARCHAR(100) NOT NULL,
  permissions TEXT         NOT NULL DEFAULT '[]',  -- JSON array of Permission flags
  created_at  TIMESTAMPTZ  NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ,
  UNIQUE (family_id, name)
);

-- Modify family_members.role:
-- Was: VARCHAR(10) CHECK (role IN ('owner','member','child','other'))
-- Becomes: VARCHAR(10) CHECK (role IN ('owner','member','child'))
--   plus: role_id UUID (NULL for system roles, points to family_roles for custom roles)

-- Modify family_invite_tokens.role similarly.
```

**Docs to update:**
- `decisions/0007-loan-kind-dictionary.md` — change `Admin or Owner` → `Owner` for `POST /loan-kinds`
- `architecture/database-schema.md` — add `family_roles` table; update `family_members` and `family_invite_tokens`
- `domain/bounded-contexts.md` — update role definitions; remove `Other` role
- `domain/ubiquitous-language.md` — update role definitions

**Code to update:**
- `backend/domain` — `Role` enum: remove `Other` variant, add custom role support
- `backend/domain` — `FamilyMember` struct: add `permissions: Vec<Permission>` field
- `backend/domain` — `InviteToken` struct: add `permissions: Vec<Permission>` field

---

### Decision 2 — `LoanPaymentRecorded` Event Must Carry `source_account_id`

**Problem:** ADR-0008's code example for `Loan::record_payment()` emits `LoanPaymentRecorded`
**without** `source_account_id`. ADR-0009 correctly specifies the event as
`LoanPaymentRecorded { payment_id, loan_id, amount, source_account_id }`. Without
`source_account_id`, the downstream `ledger_service` cannot determine which account to debit
when creating the ledger entry.

**Decision:** `source_account_id` is a **required field** in `LoanPaymentRecorded`.
ADR-0008 code example is wrong; ADR-0009 is correct.

**Docs to update:**
- `decisions/0008-loan-payment-child-entity.md` — add `source_account_id` to the
  `LoanPaymentRecorded` event emission in the code example.

---

### Decision 3 — Balance Derivation Uses `occurred_at`

**Problem:** ADR-0006 references a column called `recorded_at` in the balance derivation
formula (`SUM(ledger_transactions WHERE recorded_at > snapshot.year_month)`). No such column
exists. The `ledger_transactions` table has:
- `occurred_at TIMESTAMPTZ` — user-supplied date when money moved (can be backdated)
- `created_at TIMESTAMPTZ` — DB insert timestamp

**Decision:** Balance derivation uses **`occurred_at`** as the temporal cutoff.

Rationale: A user may enter last week's grocery purchase today. The balance should reflect
reality (when money moved), not when the user opened the app. `created_at` is a system
audit field only.

**Corrected formula:**

```sql
-- Current balance for account X:
SELECT closing_balance FROM account_balance_snapshots
WHERE account_id = X
ORDER BY year_month DESC LIMIT 1;

-- Delta since last snapshot:
SELECT SUM(amount_value) FROM ledger_transactions
WHERE account_id = X
  AND occurred_at >= '<latest_snapshot_year_month>-01'
  AND deleted_at IS NULL;
```

**Docs to update:**
- `decisions/0006-account-balance-snapshot.md` — replace all `recorded_at` with `occurred_at`.

---

### Decision 4 — Account Balance Snapshot Freeze Flow

**Problem:** No mechanism was defined for confirming/freezing account balance snapshots.
The scheduler computes snapshots automatically, but there was no way for the Owner to lock
a period and prevent further backdated transactions from silently altering history.

**Decision:** Introduce a dedicated **`account_frozen_periods`** table. Semantics:

1. The last-day-of-month EventBridge Scheduler triggers snapshot computation →
   `accounts_service` writes/updates `account_balance_snapshots` (pure computed value, no status).
2. Before the Owner freezes: the snapshot may be recomputed any number of times.
3. Owner reviews and confirms → **inserts a row into `account_frozen_periods`**.
4. After freeze: any transaction with `occurred_at` inside the frozen `year_month` is
   **rejected at the API level** (for MVP — no re-computation option).

```sql
CREATE TABLE account_frozen_periods (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id  UUID        NOT NULL,
  family_id   UUID        NOT NULL,
  year_month  CHAR(7)     NOT NULL,  -- 'YYYY-MM', e.g. '2026-08'
  frozen_by   UUID        NOT NULL,  -- owner member_id
  frozen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (account_id, year_month)
);

CREATE INDEX ASYNC ON account_frozen_periods (family_id, year_month);
```

**API changes required:**
- New endpoint: `POST /accounts/{id}/freeze/{year_month}` (Owner only)
- Validation layer: before inserting a transaction, check `account_frozen_periods` for
  both `source_account_id` and `destination_account_id` against `occurred_at`.

**Docs to update:**
- `decisions/0006-account-balance-snapshot.md` — add freeze flow section
- `architecture/database-schema.md` — add `account_frozen_periods` table
- `architecture/api-design.md` — add freeze endpoint

---

### Decision 5 — Budget `Closed` Status Removed

**Problem:** The budget lifecycle was documented as `Draft → Active → Closed`, but no mechanism
was defined for the `Active → Closed` transition (no endpoint, no scheduler job, no event).
On reflection, `Closed` was found to be redundant: envelope limits are already locked upon
approval, and historical analysis works against any month's `Active` budget.

**Decision:** **Remove the `Closed` status entirely.** Budget lifecycle is now `Draft → Active`.

| Status | Meaning |
|--------|---------|
| `draft` | Envelopes freely editable; not yet in effect |
| `active` | Owner approved; envelope limits locked; spending tracked in real time |

Historical analysis of any past month is done by querying its `active` budget.

**Schema changes required:**

```sql
-- Was:
CHECK (status IN ('draft','active','closed'))

-- Becomes:
CHECK (status IN ('draft','active'))
```

> **Note:** Decide during Sprint 4 implementation whether to keep or remove `approved_by` /
> `approved_at` columns (useful for audit; no harm in keeping them).

**Docs to update:**
- `decisions/0013-budget-owner-approval-flow.md` — remove `Closed` state; update lifecycle diagram
- `architecture/database-schema.md` — fix `CHECK` constraint on `budget_monthly_budgets.status`
- `domain/bounded-contexts.md` — update Budget & Limits context description

---

### Decision 6 — `FamilyName` Maximum Length is 200

**Problem:** `family_families.name` DB column is `VARCHAR(200)`, but the `FamilyName` value
object in Rust validates a maximum of **100 characters**. Data inserted directly into the DB
(admin scripts, future migrations) could create names that the domain layer rejects on read.

**Decision:** **200 characters is the canonical limit.** The DB column is the source of truth.

**Code to update:**
- `backend/domain` — `FamilyName` value object: change `max_len` from 100 to 200.

---

### Decision 7 — `POST /loan-kinds` Requires `Owner` Role

**Problem:** ADR-0007 stated `POST /loan-kinds` requires `Admin or Owner` role. There is no
`Admin` role in the system.

**Decision:** `POST /loan-kinds` (creating family-custom loan kinds) requires **`Owner` role only**.

**Docs to update:**
- `decisions/0007-loan-kind-dictionary.md` — fix `Admin or Owner` → `Owner`.

---

### Decision 8 — Migration File Naming: SQLx Timestamp Format

**Problem:** `migrations/README.md` described Flyway-style naming (`V{timestamp}_{description}.sql`).
The `migrate_runner` Lambda uses SQLx's `sqlx::migrate!` macro, which does **not** understand
the `V` prefix. SQLx uses either sequential or timestamp format.

**Decision:** Use **SQLx timestamp format**: `{YYYYMMDDHHMMSS}_{description}.sql`

This is the default produced by `sqlx migrate add <description>`.

Example: `20260101000001_create_family_families.sql`

**Docs to update:**
- `migrations/README.md` — remove Flyway `V` prefix; document SQLx timestamp format with example.

---

### Decision 9 — Migration Count is 35

**Problem:** `migrations/README.md` said 25 files; `sprint-roadmap.md` and the main `README.md`
said 34 files. The true base count is 18 table DDLs + 16 async indexes = 34. With the addition
of one new index from Decision 13, the final count is **35**.

| Migration group | Count |
|-----------------|-------|
| Table DDLs (original 18) | 18 |
| Async indexes (original 16) | 16 |
| `ledger_transactions(category_id)` async index (new) | 1 |
| **Total** | **35** |

> Note: `account_frozen_periods` (Decision 4) and `family_roles` (Decision 1) add 2 more
> table DDLs + their indexes. The final Sprint 1 migration count after all schema changes
> will be higher than 35.

**Docs to update:**
- `migrations/README.md` — correct count from 25 → 35 (pre-schema-change baseline).
- `sprint-roadmap.md` — note the updated count.

---

### Decision 10 — `spent_value` Removed from `budget_envelopes` (MVP)

**Problem:** `budget_envelopes.spent_value` was updated via CDC on every transaction, creating a
hot-row OCC conflict risk. Incremental updates (`spent_value += amount`) also fail to correctly
handle reversals and amendments without extra logic.

**Decision (MVP):** **Remove `spent_value` column** from `budget_envelopes`. Compute envelope
spending on-the-fly via a `SUM` query at read time:

```sql
SELECT SUM(lt.amount_value)
FROM ledger_transactions lt
WHERE lt.category_id = <envelope.category_id>
  AND lt.family_id = <family_id>
  AND lt.occurred_at >= '<year>-<month>-01'
  AND lt.occurred_at <  '<year>-<next_month>-01'
  AND lt.deleted_at IS NULL;
```

This works correctly for reversals and amendments automatically (they are soft-deleted or
have `amendment_of_id` set and are excluded via `deleted_at IS NULL`).

**Post-MVP path:** If query latency becomes a concern, introduce a materialized projection
that recomputes the full monthly SUM on each CDC event (not incremental) and caches it.
See ADR-0018.

**Docs to update:**
- `architecture/database-schema.md` — remove `spent_value` column from `budget_envelopes`.
- `domain/bounded-contexts.md` — update `BudgetEnvelope` entity description.

---

### Decision 11 — CDC Deduplication via DynamoDB (Write-Last Pattern)

**Problem:** The engineering rules mandated CDC consumers deduplicate by `txId + table + PK`,
but no idempotency store was defined in the schema.

**Decision:** Use a **DynamoDB table** for CDC event idempotency tracking.

```
DynamoDB table: cdc_idempotency_keys
  Partition key: event_id (string) — hash of Aurora DSQL CDC txId + table + pk
  Attribute: processed_at (string, ISO-8601)
  TTL attribute: ttl_epoch (number) — processed_at + 7 days (DynamoDB native TTL)
```

**Write-last contract (mandatory for correctness):**

```
1. CHECK   DynamoDB: if event_id exists → skip (already processed)
2. PROCESS all Aurora DSQL writes (entity idempotency_key UNIQUE constraints prevent
           duplicate DB rows if step 3 was missed on a prior run)
3. WRITE   DynamoDB event_id mark  ← LAST, after all Aurora DSQL writes succeed
```

If the Lambda crashes between step 2 and step 3: Kinesis retries → step 1 finds no entry →
step 2 re-runs → entity UNIQUE constraints reject duplicates harmlessly → step 3 completes.

**Post-MVP path (Sprint 6):** Add `in_flight` / `done` state to the DynamoDB record for
stuck-event monitoring and alerting (saga pattern). See ADR-0017.

**New ADR required:** ADR-0017 — CDC Consumer Idempotency via DynamoDB Write-Last Pattern.

---

### Decision 12 — Notification Device Token Storage Deferred to Sprint 6

**Problem:** The notification service is documented in `system-overview.md` and
`domain/bounded-contexts.md` but there is no database table for storing FCM tokens (Flutter)
or Web Push subscriptions (Angular). Push notifications cannot be delivered without these.

**Decision:** Deferred to Sprint 6 as a prerequisite task.

**Sprint 6 planning note:** Before implementing notification delivery, decide between:
- A new Aurora DSQL table `notification_device_tokens` (member_id, platform, token, active, created_at)
- A separate DynamoDB table (lower latency, no relational joins needed for token lookup)

---

### Decision 13 — Add `ledger_transactions(category_id)` Async Index

**Problem:** The on-the-fly SUM query (Decision 10) filters `ledger_transactions` by
`category_id`. Without an index, this is a full-table scan scoped to `family_id` only
(via the existing composite index). As the ledger grows, this query degrades.

**Decision:** Add a new async index:

```sql
CREATE INDEX ASYNC ON ledger_transactions (category_id)
WHERE deleted_at IS NULL;
```

`debt_repayment_plans(family_id)` index was also evaluated and **skipped** — the table
holds at most one active plan per family and is too small to warrant a dedicated index.

**Docs to update:**
- `architecture/database-schema.md` — add this index to section 2 (Async Indexes).

---

## New ADRs Required

### ADR-0017 — CDC Consumer Idempotency via DynamoDB Write-Last Pattern

**Status:** To be written.  
**Summary:** Documents the DynamoDB-based deduplication store for Aurora DSQL CDC consumers,
the write-last contract, and the post-MVP saga pattern path.

### ADR-0018 — Budget Envelope Spent Value Computation Strategy

**Status:** To be written.  
**Summary:** Documents the decision to compute `spent_value` on-the-fly via SUM for MVP,
the rationale (no OCC hot rows, correct reversal/amendment handling), and the post-MVP
materialized projection path.

---

## Remaining Open Issues

These issues were identified during the audit but not resolved in this session.
They are tracked here for future follow-up.

| # | File(s) | Issue | Severity |
|---|---------|-------|----------|
| A | Repo name / docs | Repo is `home-harmony`; product is `FamilyLedger`. Not a technical problem, but naming should be deliberate. | Low |
| B | `database-schema.md` | `family_invite_tokens.token` is `VARCHAR(64)` PK (not UUID), but the domain creates it as `Uuid::new_v4().to_string()` (36 chars). The non-UUID PK violates the stated UUID PK engineering rule. Acceptable for security tokens, but should be documented as a conscious exception. | Low |
| C | `domain/events/mod.rs` | `DomainEvent` enum has 5 events; `event-storming.md` catalogs 24 events. Expected gap for Sprint 1 scaffold, but the file should have a `// TODO: Sprint N` marker. | Expected |
| D | `backend/domain/errors.rs` | `DomainError` missing `InvalidPaymentSplit { expected, actual }` variant required by ADR-0008. Must be added before Debt Planner implementation (Sprint 4). | Sprint 4 |
| E | `backend/Cargo.toml` | Only `migrate_runner` in workspace `members`. All service lambdas will need to be added. | Expected |
| F | `database-schema.md` | `family_invite_tokens` has no `created_at` column, making it impossible to audit when an invite was generated. Consider adding `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`. | Minor |
| G | `database-schema.md` | `UNIQUE(family_id, code)` on `debt_loan_kinds`: for system seeds (`family_id IS NULL`), uniqueness of NULL-keyed rows needs explicit validation in Aurora DSQL — behavior may differ from standard PostgreSQL. Write a targeted migration test. | Sprint 4 |
| H | `architecture/api-design.md` | `GET /analytics/monthly-surplus` needs "Average Monthly Income" but the calculation path is not specified. Clarify whether it uses `budget_monthly_budgets.total_income_expected_value` or aggregates `ledger_transactions WHERE kind = 'income'`. | Sprint 5 |
| I | `architecture/api-design.md` / `domain/bounded-contexts.md` | Multi-currency budget envelope aggregation logic is not specified. When envelopes are in different currencies, how is the budget total computed? Which exchange rate is used (rate on transaction date vs. current rate)? | Sprint 4 |
| J | `backend/Cargo.toml` | `lambda_http = "1.2.1"` and `lambda_runtime = "1.2.1"` appear to be aspirational/placeholder versions (latest published versions are in the 0.x range at time of writing). Verify and pin to real published versions before Sprint 2. | Sprint 2 |

---

## Summary of Schema Changes

The table below lists all schema modifications implied by the decisions above.

| Change | Type | Decision |
|--------|------|----------|
| Add `family_roles` table | New table | #1 |
| Modify `family_members.role` column + add `role_id` | Alter table | #1 |
| Modify `family_invite_tokens.role` column | Alter table | #1 |
| Add `account_frozen_periods` table + index | New table | #4 |
| `budget_monthly_budgets.status` CHECK: remove `'closed'` | Alter table | #5 |
| Remove `budget_envelopes.spent_value` column | Alter table | #10 |
| Add `ledger_transactions(category_id)` async index | New index | #13 |

---

_Document generated: 2026-08-28_
