# 0006. Separate Account Balance into Snapshot Table

* **Status**: Accepted
* **Date**: 2026-08-26
* **Deciders**: Dmitri Astafiev

## Context and Problem Statement

The current `payment_accounts` table (formerly `cards_accounts`) stores `current_balance` as a mutable column that is overwritten on every transaction. This creates a second source of truth alongside `ledger_transactions`, which is the authoritative record of all financial movements. When a transaction is recorded, both the new `ledger_transactions` row and the `current_balance` field on the account must update atomically. Under Aurora DSQL's Optimistic Concurrency Control (OCC) model, these two simultaneous row updates can produce conflicts under concurrent load. Furthermore, overwriting the balance column makes it impossible to answer historical queries such as "what was my account balance in March?", and any missed update silently causes the stored balance to diverge from the ledger.

## Decision Drivers

* **Single source of truth** — `ledger_transactions` must be the sole authoritative record of money movements; derived projections must not duplicate that authority.
* **Aurora DSQL OCC compatibility** — concurrent writes to multiple rows in the same transaction increase conflict probability; the balance update must be decoupled from the transaction insert.
* **Balance history** — the product requires monthly balance trend views and the ability to reconstruct any historical balance.
* **Audit trail** — financial best practices and future compliance requirements expect full reconstructibility of any account state.

## Considered Options

1. **Keep `current_balance` on the account table** — status quo; balance updated atomically with each transaction insert.
2. **Derive balance on the fly** — remove `current_balance`; compute balance by summing `ledger_transactions` for the account on every read.
3. **Snapshot table with scheduled Lambda** — remove `current_balance` from `payment_accounts`; add a separate `account_balance_snapshots` table populated by a monthly scheduled Lambda. Current balance for display = latest snapshot `closing_balance` + sum of ledger transactions recorded since the snapshot date.

## Decision Outcome

Chosen option: **Option 3 — Snapshot table with scheduled Lambda**, because it eliminates OCC contention, preserves the full monthly balance history, and keeps the `payment_accounts` table free of mutable derived state while keeping current-balance reads fast (one snapshot lookup + a bounded ledger sum).

Option 1 is rejected because it maintains a mutable derived column that can diverge from the ledger and causes OCC conflicts.

Option 2 is retained as the fallback for real-time balance display within the current month (before the snapshot is computed), but not as the sole strategy, because unbounded ledger summation degrades as transaction volume grows.

### Schema Change

Remove `current_balance` from `payment_accounts`. Add:

```sql
CREATE TABLE account_balance_snapshots (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id      UUID          NOT NULL,
  family_id       UUID          NOT NULL,
  year_month      CHAR(7)       NOT NULL,  -- 'YYYY-MM'
  closing_balance NUMERIC(19,4) NOT NULL,
  currency        CHAR(3)       NOT NULL,
  computed_at     TIMESTAMPTZ   NOT NULL DEFAULT now(),
  UNIQUE (account_id, year_month)
);
```

**Balance derivation formula** (used by the API layer):

```
current_balance
  = latest_snapshot.closing_balance
  + SUM(ledger_transactions WHERE account_id = ? AND recorded_at > snapshot.year_month)
```

For accounts with no snapshot yet, fall back to summing the full ledger.

### Positive Consequences

* No OCC conflicts from simultaneous `current_balance` updates — the snapshot is computed asynchronously.
* Full monthly balance history enables trend queries ("how did my balance change over 6 months?").
* Single source of truth: the ledger is authoritative; the snapshot is an explicit, time-stamped projection.
* `payment_accounts` table is append-friendly and free of mutable financial state.

### Negative Consequences / Trade-offs

* Current balance display requires two queries (snapshot lookup + ledger delta sum) instead of a single column read.
* A scheduled Lambda must run at month-end for each family/account; operational overhead for that Lambda must be maintained.
* Newly created accounts have no snapshot until the first month-end run; balance derivation falls back to a full ledger scan for those accounts.

## Compliance & Invariants

* The `payment_accounts` table **must not** contain a `current_balance` column.
* All balance reads in the API layer **must** route through the ledger-derivation formula described above; direct balance fields on the account row are prohibited.
* The CDC Lambda **must** trigger the snapshot computation at month-end for every active account.
* The snapshot row is immutable after creation; re-computation for the same `(account_id, year_month)` must upsert using the `UNIQUE` constraint, not create a duplicate.
