# 0014. Credit Card PaymentAccount Explicitly Linked to Loan for Debt Planning

* **Status**: Accepted
* **Date**: 2026-08-26
* **Deciders**: Dmitri Astafiev

## Context and Problem Statement

A credit card serves a dual role in household finance:
1. **Spending Instrument (`PaymentAccount`)**: Used for daily point-of-sale and online transactions across categories.
2. **Revolving Liability (`Loan`)**: Carries an unpaid balance with an associated Annual Percentage Rate (APR) and monthly minimum payment that must be eliminated via debt acceleration strategies (Avalanche or Snowball).

Some families pay off their credit cards in full every single month (using them strictly as payment instruments with zero interest incurred), whereas other families carry a revolving balance that needs structured debt payoff planning.

Automatically treating every `CreditCard` account as a loan would pollute the debt planner with non-interest-bearing transactional accounts. Conversely, requiring users to manually manage two disconnected balance numbers (one for the card, one for the loan) leads to severe balance drift.

## Decision Drivers

* **Single Source of Truth for Balances** — card spending recorded via ledger transactions must automatically update the outstanding debt balance without manual double-entry.
* **User Intent & Clarity** — families must explicitly declare whether a credit card carries revolving debt that requires structured payoff planning.
* **Bounded Context Decoupling** — preserve separation between `Payment Cards & Accounts` and `Debt Planner` while supporting asynchronous synchronization.

## Considered Options

1. **Disconnected Manual Entries** — user creates a `PaymentAccount` and a separate unlinked `Loan`. (Rejected: balances drift immediately upon new purchases or payments).
2. **Implicit Auto-Linking for All Cards** — every credit card account automatically surfaces as a loan. (Rejected: confuses users who pay full balances monthly and do not have revolving debt).
3. **Explicit Linkage with CDC-Driven Balance Sync (Option C)** — user creates a `Loan` with `kind = 'credit_card'` and explicitly populates `linked_account_id` referencing the corresponding `PaymentAccount`. Spending on the card automatically synchronizes to the loan balance via Aurora DSQL CDC. (Chosen).

## Decision Outcome

Chosen option: **Option 3 — Explicit Linkage with CDC-Driven Balance Sync**.

### Architecture and Data Flow

```
[ Family Member ]
       │ Records purchase on Credit Card
       ▼
┌──────────────────────────────────────────────────────────┐
│ ledger_service                                           │
│  INSERT ledger_transactions                              │
│  (kind='expense', source_account_id=card_account_id)     │
└──────────────────────────────┬───────────────────────────┘
                               │
                               ▼ Aurora DSQL Native CDC
┌──────────────────────────────────────────────────────────┐
│ CDC Fanout Lambda                                        │
│  1. Detects transaction on source_account_id             │
│  2. Checks if kind != 'loan_payment' AND kind != 'reversal'
│  3. Publishes: CreditCardSpendingUpdatedLoanBalance      │
└──────────────────────────────┬───────────────────────────┘
                               │
                               ▼ EventBridge
┌──────────────────────────────────────────────────────────┐
│ debt_planner_service                                     │
│  UPDATE debt_loans                                       │
│  SET balance_value = balance_value + event.amount        │
│  WHERE linked_account_id = event.source_account_id       │
└──────────────────────────────────────────────────────────┘
```

### Repayment Flow Integration

When a debt repayment is made toward the credit card:
1. User records payment via `POST /loans/{id}/payments`.
2. `debt_planner_service` records `LoanPayment` and reduces `debt_loans.balance_value` by `principal_portion`.
3. Event `LoanPaymentRecorded` triggers `ledger_service` to insert a `loan_payment` transaction into the ledger (see ADR-0009).
4. The CDC Fanout Lambda **filters out** transactions of `kind = 'loan_payment'` to prevent double-decrementing or looping the balance update.

### Positive Consequences

* Balances remain 100% consistent across spending tracking and debt payoff planning.
* Full flexibility: full-pay cardholders do not link their accounts; revolving debt holders link their cards and benefit from Avalanche/Snowball calculations.
* Leverages existing `linked_account_id UUID` column on `debt_loans`.

### Negative Consequences / Trade-offs

* Asynchronous balance propagation means the debt planner reflects new card charges within ~1-2 seconds (eventual consistency).

## Compliance & Invariants

* Linking a `Loan` to a `PaymentAccount` **requires** that the account `kind == 'credit_card'` and that account and loan currencies match.
* The CDC fanout pipeline **must filter out** `ledger_transactions` with `kind IN ('loan_payment', 'reversal')` when emitting balance synchronization events for linked credit cards.
