# 0008. LoanPayment as Explicit Child Entity of the Loan Aggregate

* **Status**: Accepted
* **Date**: 2026-08-26
* **Deciders**: Dmitri Astafiev

## Context and Problem Statement

In the initial domain documentation, `payments: Vec<LoanPayment>` was referenced inside the `Loan` aggregate root, but `LoanPayment` was never explicitly specified as a domain entity or given a defined struct. Meanwhile, the `debt_loan_payments` persistence table was given its own UUID primary key, an `idempotency_key`, and separate `principal_portion` and `interest_portion` monetary columns. 

Without an explicit `LoanPayment` entity in the domain model, critical business invariants cannot be enforced within domain boundaries:
1. The mathematical invariant `principal_portion + interest_portion == amount` would have to be checked in an ad-hoc manner in handlers or SQL triggers.
2. The remaining balance projection after a payment cannot be validated consistently.
3. Event deduplication and linkage to ledger transactions (`linked_transaction_id`) lack a clear entity lifecycle.

## Decision Drivers

* **Domain Invariant Enforcement** — financial calculations such as payment split must be guaranteed by domain constructors and methods, never relying solely on database constraints.
* **DDD Aggregate Integrity** — all mutations to loan state (balance reduction, payoff transitions) must be mediated by the `Loan` aggregate root through child entity interactions.
* **Auditability & Traceability** — each payment record must maintain its own unique identity and correlation with downstream ledger movements.

## Considered Options

1. **Anonymous struct / raw tuple inside `Vec`** — rejected because it prevents domain-level method attachment, validation, and serialization consistency.
2. **Value Object representation** — rejected because loan payments have independent lifecycle identities (UUID PKs), are referenced across bounded contexts, and must be deduplicated via unique idempotency keys.
3. **Explicit Child Entity within `Loan` Aggregate Root** — chosen.

## Decision Outcome

Chosen option: **Option 3 — Explicit Child Entity within `Loan` Aggregate Root**. `LoanPayment` is formalized as an entity whose lifecycle is managed strictly through methods on the `Loan` aggregate root.

### Domain Model Definition

```rust
// backend/domain/src/entities/loan_payment.rs
use chrono::{DateTime, NaiveDate, Utc};
use uuid::Uuid;
use crate::value_objects::Money;

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct LoanPayment {
    pub id: Uuid,
    pub loan_id: Uuid,
    pub family_id: Uuid,
    pub paid_at: NaiveDate,
    pub amount: Money,
    pub principal_portion: Money,
    pub interest_portion: Money,
    pub remaining_balance: Money,
    pub linked_transaction_id: Option<Uuid>,
    pub idempotency_key: Uuid,
    pub created_at: DateTime<Utc>,
}
```

### Invariant Enforcement on `Loan` Aggregate

Payments must only be created via the `Loan::record_payment(...)` method:

```rust
impl Loan {
    pub fn record_payment(
        &mut self,
        paid_at: NaiveDate,
        amount: Money,
        principal_portion: Money,
        interest_portion: Money,
        idempotency_key: Uuid,
    ) -> Result<(LoanPayment, Vec<DomainEvent>), DomainError> {
        // Enforce split invariant: principal + interest must equal total amount
        let calculated_total = principal_portion.add(&interest_portion)?;
        if calculated_total != amount {
            return Err(DomainError::InvalidPaymentSplit {
                expected: amount.to_string(),
                actual: calculated_total.to_string(),
            });
        }

        // Currencies must match loan balance currency
        if amount.currency() != self.current_balance.currency() {
            return Err(DomainError::CurrencyMismatch {
                expected: self.current_balance.currency().to_string(),
                actual: amount.currency().to_string(),
            });
        }

        // Compute new balance
        let new_balance = self.current_balance.sub(&principal_portion)?;
        self.current_balance = new_balance.clone();

        let payment_id = Uuid::new_v4();
        let payment = LoanPayment {
            id: payment_id,
            loan_id: self.id,
            family_id: self.family_id,
            paid_at,
            amount,
            principal_portion,
            interest_portion,
            remaining_balance: new_balance.clone(),
            linked_transaction_id: None,
            idempotency_key,
            created_at: Utc::now(),
        };

        self.payments.push(payment.clone());

        // Check payoff transition
        let mut events = vec![DomainEvent::LoanPaymentRecorded {
            family_id: self.family_id,
            loan_id: self.id,
            payment_id,
            amount: payment.amount.clone(),
            principal_portion: payment.principal_portion.clone(),
            interest_portion: payment.interest_portion.clone(),
            remaining_balance: new_balance.clone(),
            occurred_at: Utc::now(),
        }];

        if self.current_balance.is_zero() {
            self.status = LoanStatus::PaidOff;
            events.push(DomainEvent::LoanPaidOff {
                family_id: self.family_id,
                loan_id: self.id,
                occurred_at: Utc::now(),
            });
        }

        Ok((payment, events))
    }
}
```

### Positive Consequences

* Mathematical invariants are strictly protected inside domain code with dedicated unit tests.
* The `Loan` aggregate maintains absolute consistency over its total remaining balance and status transitions (`Active` -> `PaidOff`).
* Clear correlation points exist for cross-context ledger integration (`linked_transaction_id`).

### Negative Consequences / Trade-offs

* Loading the `Loan` aggregate for repayment recording requires loading or projecting payment state.
* Adds a new entity file and error variants (`DomainError::InvalidPaymentSplit`) to the domain crate.

## Compliance & Invariants

* `LoanPayment` instances must never be constructed directly outside of `Loan::record_payment`.
* `principal_portion + interest_portion == amount` is non-negotiable and must be verified before emitting `LoanPaymentRecorded`.
* The `debt_loan_payments` table row must strictly map to the `LoanPayment` entity structure.
