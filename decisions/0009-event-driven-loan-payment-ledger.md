# 0009. Event-Driven Loan Payment Ledgerization via EventBridge

* **Status**: Accepted
* **Date**: 2026-08-26
* **Deciders**: Dmitri Astafiev

## Context and Problem Statement

When a loan payment is recorded, two primary domain operations must occur across distinct bounded contexts:
1. **Debt Planner Context**: Record the payment in `debt_loan_payments` (with principal/interest split) and reduce the `debt_loans` balance.
2. **Ledger Core Context**: Insert a financial movement record into `ledger_transactions` (with kind `loan_payment`), deducting money from the corresponding `PaymentAccount` (source account) and updating budget envelope spent amounts.

These two operations belong to separate aggregate roots (`Loan` and `Transaction`), which are implemented in separate microservices (`debt_planner_service` and `ledger_service`) running as distinct AWS Lambda functions. 

Direct synchronous cross-table writes in Aurora DSQL create high Optimistic Concurrency Control (OCC) conflict risks across service boundaries, violate bounded context independence, and couple microservice deployments.

## Decision Drivers

* **Bounded Context Isolation** — `debt_planner_service` must not directly manipulate `ledger_transactions` or bypass the Ledger Core domain logic.
* **OCC Optimization & Fault Isolation** — failures or retries in ledger recording must not roll back or block loan payment registration.
* **Eventual Consistency & Idempotency** — at-least-once message delivery from AWS EventBridge must not cause duplicate payments or corrupted ledger balances.

## Considered Options

1. **Synchronous Cross-Service Write (or Two-Phase Commit)** — `debt_planner_service` writes to both `debt_loan_payments` and `ledger_transactions` in a single distributed transaction. (Rejected: tightly couples databases, high OCC contention, breaks bounded context separation).
2. **Event-Driven Choreography via AWS EventBridge** — `debt_planner_service` records the loan payment locally and emits `LoanPaymentRecorded`. `ledger_service` consumes the event, creates the immutable ledger entry, and emits `LoanPaymentLedgerized` to confirm linkage. (Chosen).

## Decision Outcome

Chosen option: **Option 2 — Event-Driven Choreography via AWS EventBridge**.

### End-to-End Event Flow

```
[ Client / User ]
       │  POST /loans/{id}/payments (with idempotency_key)
       ▼
┌──────────────────────────────────────────────────────────┐
│ debt_planner_service (Lambda)                            │
│  1. INSERT debt_loan_payments                            │
│  2. UPDATE debt_loans SET balance_value -= principal     │
│  3. Publish to EventBridge: LoanPaymentRecorded          │
└──────────────────────────────┬───────────────────────────┘
                               │
                               ▼ EventBridge Bus
┌──────────────────────────────────────────────────────────┐
│ ledger_service (Lambda Consumer)                         │
│  1. Deduplicate event by event.payment_id                │
│  2. INSERT ledger_transactions                           │
│     (kind='loan_payment', idempotency_key=payment_id,    │
│      source_account_id=..., amount=...)                  │
│  3. Publish to EventBridge: LoanPaymentLedgerized        │
└──────────────────────────────┬───────────────────────────┘
                               │
                               ▼ EventBridge Bus
┌──────────────────────────────────────────────────────────┐
│ debt_planner_service (Consumer)                          │
│  1. UPDATE debt_loan_payments                            │
│     SET linked_transaction_id = event.transaction_id     │
│     WHERE id = event.payment_id                          │
└──────────────────────────────────────────────────────────┘
```

### CDC Filter Rule for Credit Card Linked Loans

Because `ledger_transactions` changes are captured by native Aurora DSQL CDC to update linked credit card loans (see ADR-0014), the CDC fanout handler **must filter out** transactions where `kind = 'loan_payment'`. This prevents a loan payment from erroneously re-increasing or re-decrementing the loan balance.

### Positive Consequences

* Clean architectural decoupling: `debt_planner_service` and `ledger_service` remain fully independent Lambdas.
* Resilience against temporary database unavailability or cold-start spikes in downstream services.
* Full idempotency guaranteed by using `payment_id` as the transaction's `idempotency_key`.

### Negative Consequences / Trade-offs

* Eventual consistency: the ledger transaction appears milliseconds after the loan payment response returns 201 Created.
* Cross-service traceability requires tracking `linked_transaction_id` asynchronously.

## Compliance & Invariants

* `debt_planner_service` **must never** execute SQL queries against `ledger_transactions`.
* `ledger_service` **must** handle `LoanPaymentRecorded` events and ensure idempotent transaction creation using `payment_id`.
* The CDC fanout Lambda **must** ignore `ledger_transactions` with `kind IN ('loan_payment', 'reversal')` when synchronizing credit card balances.
