# 0010. Loan-Bound Insurance Modelled as a Linked RecurringPayment

* **Status**: Accepted
* **Date**: 2026-08-26
* **Deciders**: Dmitri Astafiev

## Context and Problem Statement

Many major personal loans (mortgages, auto financing, home equity loans) legally mandate insurance policies:
* **Mortgages**: Homeowners insurance, private mortgage insurance (PMI), title insurance, and property hazard escrow.
* **Auto Loans**: Comprehensive and collision insurance.
* **Personal Loans**: Payment protection insurance (PPI) or life insurance tie-ins.

These insurance premiums represent fixed, non-negotiable monthly obligations tied directly to loan existence. If they are omitted from debt calculations, the **True Disposable Surplus** is significantly overstated, leading the automated Avalanche and Snowball repayment engines to generate unrealistically aggressive repayment timelines.

## Decision Drivers

* **Ubiquitous Language & Domain Accuracy** — recurring bills (like insurance) naturally belong in the `Recurring Payments` bounded context, while loan liability tracking belongs in `Debt Planner`.
* **Accurate True Disposable Surplus** — calculation of disposable surplus must aggregate both direct loan minimums and mandatory loan-bound insurance.
* **Payoff Lifecycle Automation** — when a loan reaches zero balance (`LoanPaidOff`), users should be prompted to review or cancel policies like PMI.

## Considered Options

1. **Inline Insurance Fields on `Loan` Aggregate** — add `insurance_monthly: Option<Money>` directly to `Loan` and table `debt_loans`. (Rejected: mixes recurring billing logic into loan management and creates duplicate concepts for insurance policies).
2. **Linked `RecurringPayment` Entity (Approach B)** — keep insurance in the `Recurring Payments` context as a `RecurringPayment` with `kind = 'insurance'`, but add a `linked_loan_id: Option<LoanId>` foreign reference. (Chosen).

## Decision Outcome

Chosen option: **Option 2 — Linked `RecurringPayment` Entity**.

### Domain & Schema Changes

Add `linked_loan_id: Option<LoanId>` to `RecurringPayment` and table `recurring_payments`:

```sql
-- Modified recurring_payments table
ALTER TABLE recurring_payments ADD COLUMN linked_loan_id UUID;
CREATE INDEX ASYNC idx_recurring_linked_loan ON recurring_payments (linked_loan_id) 
  WHERE deleted_at IS NULL AND linked_loan_id IS NOT NULL;
```

### Formula Integration: `DebtServiceCost`

The True Disposable Surplus calculation uses the composite `DebtServiceCost`:

$$\text{DebtServiceCost}(\text{loan}) = \text{loan.monthly\_payment} + \sum_{\text{linked}} \text{RecurringPayment.amount}$$

```
True Disposable Surplus =
    Average Monthly Income
  − Total Monthly Recurring Obligations (unlinked bills & subscriptions)
  − Total Discretionary Budget Ceiling   (living expense limits)
  − Sum of All DebtServiceCosts          (loan minimums + linked insurance)
  ──────────────────────────────────────────────────────────────────────────
  = Extra Monthly Budget Available for Debt Acceleration
```

### Payoff Notification Flow

When `LoanPaidOff` is emitted by the `debt_planner_service`:
1. `notification_service` receives `LoanPaidOff`.
2. It queries active recurring payments where `linked_loan_id == event.loan_id`.
3. It sends a push notification to the Family Owner: *"Loan [Mortgage] is paid off! Review linked insurance [PMI] to cancel unnecessary premiums."*

### Positive Consequences

* Clean separation of concerns: insurance scheduling, frequency rules, and payments remain in `Recurring Payments`.
* Complete surplus accuracy without domain pollution.
* Clear workflow for insurance policy cancellations on loan payoff.

### Negative Consequences / Trade-offs

* Creating a loan with insurance requires creating two records in the UI (a loan plus a linked recurring payment).
* The debt planner query must perform a cross-context lookup or event-driven projection to determine total `DebtServiceCost`.

## Compliance & Invariants

* `recurring_payments` **must** retain `linked_loan_id UUID` as a nullable field.
* The `debt_planner_service` calculation for `DebtServiceCost` **must** aggregate linked insurance payments.
* On `LoanPaidOff`, the `notification_service` **must** check for linked insurance and deliver appropriate guidance to the family.
