# 0013. Monthly Budget Requires Owner Review and Approval Before Activation

* **Status**: Accepted
* **Date**: 2026-08-26
* **Deciders**: Dmitri Astafiev

## Context and Problem Statement

Each month, a family's budget must account for known committed obligations (rent, utilities, insurance, subscriptions, and debt minimums) alongside discretionary spending limits (groceries, dining, entertainment). 

If a monthly budget is activated automatically without human review:
1. Changes in household circumstances (e.g., canceled subscriptions, anticipated bonuses, or seasonal utility spikes) are not reflected before spending tracking begins.
2. Family members may spend against incorrect envelope limits.
3. Household financial governance is weakened if the family head/owner cannot calibrate limits prior to locking them in.

## Decision Drivers

* **Household Financial Governance** — the family Owner must have explicit oversight and approval over household spending allocations.
* **Predictable Envelope Locking** — envelope spending limits should remain stable and locked once the month's plan is finalized to prevent accidental mid-month tampering.
* **Proactive Review Triggers** — automated scheduled reminders ensure the review process happens predictably before or on the 1st of each month.

## Considered Options

1. **Automatic Immediate Activation** — budget is created on the 1st and instantly becomes `active`. (Rejected: gives no opportunity for review).
2. **Draft -> Owner Review & Approval -> Active Lifecycle** — budget is pre-seeded in `draft` status, allowing Owner corrections; once approved, limits transition to `active` and lock. (Chosen).

## Decision Outcome

Chosen option: **Option 2 — Draft -> Owner Review & Approval -> Active Lifecycle**.

### Lifecycle States (`BudgetStatus`)

```
 ┌───────────────────────────────────────────────────────────┐
 │ 1st of Month (EventBridge Scheduler)                      │
 │ Auto-create MonthlyBudget from active RecurringPayments  │
 └─────────────────────────────┬─────────────────────────────┘
                               ▼
                        ┌─────────────┐
                        │    Draft    │ ◄── Owner can adjust envelope limits
                        └──────┬──────┘
                               │ POST /budget/{y}/{m}/approve (Owner only)
                               ▼
                        ┌─────────────┐
                        │   Active    │ ◄── Limits locked; CDC tracks spent vs limit
                        └──────┬──────┘
                               │ Last day of month / Next cycle
                               ▼
                        ┌─────────────┐
                        │   Closed    │ ◄── Read-only archive & analytics
                        └─────────────┘
```

### Schema Support in `budget_monthly_budgets`

```sql
-- Schema adjustments for budget_monthly_budgets
ALTER TABLE budget_monthly_budgets 
  ADD COLUMN status VARCHAR(10) NOT NULL DEFAULT 'draft' 
    CHECK (status IN ('draft', 'active', 'closed')),
  ADD COLUMN approved_by UUID,
  ADD COLUMN approved_at TIMESTAMPTZ;

-- Soft deletes on envelopes
ALTER TABLE budget_envelopes ADD COLUMN deleted_at TIMESTAMPTZ;
```

### Operational Workflow

1. **Auto-Seeding (`BudgetDraftCreated`)**: On the 1st of each month at 00:05 UTC, `EventBridge Scheduler` triggers `budget_service`. The service:
   * Gathers all active `recurring_payments` and loan minimums.
   * Generates a new `MonthlyBudget` in `draft` status with pre-seeded `committed` envelopes and previous month's `discretionary` envelopes.
   * Emits `BudgetDraftCreated` to EventBridge.
2. **Owner Notification**: `notification_service` delivers a push notification to the Family Owner: *"Your monthly budget draft is ready for review."*
3. **Review & Corrections**: The Owner inspects `GET /budget/{year}/{month}` and updates discretionary limits via `PUT /budget/{year}/{month}/envelopes` while status is `draft`.
4. **Approval (`BudgetApproved`)**: The Owner executes `POST /budget/{year}/{month}/approve`. The budget transitions to `active`, and envelope limits are locked.

### Positive Consequences

* Prevents unintentional or unvetted spending plans from governing the household.
* Gives clear visibility into recurring obligation changes month-over-month.
* Transparent audit record of who approved the budget and when.

### Negative Consequences / Trade-offs

* If the Owner delays approval, spending during early days of the month is tracked against a draft budget (UI displays a "Pending Owner Approval" banner).

## Compliance & Invariants

* `PUT /budget/{year}/{month}/envelopes` **must reject modifications** with `409 Conflict` if the budget `status == 'active'` or `'closed'`.
* Only members with `Role::Owner` **may call** `POST /budget/{year}/{month}/approve`.
* Non-Owner members have read-only access to draft budgets.
