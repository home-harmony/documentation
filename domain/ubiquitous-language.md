# Ubiquitous Language & Core Formulas — FamilyLedger

This document defines the official domain vocabulary, business rules, and calculation formulas used across all services, databases, and UI components.

---

## 1. Domain Glossary

| Term | Bounded Context | Definition |
| :--- | :--- | :--- |
| **Family Workspace** | Identity & Family | The primary multi-tenant boundary. All entities and queries belong to a specific `family_id`. |
| **HomeDisplayCurrency** | Identity & Family | The currency specified in `family.home_currency`. All aggregated views (net worth, surplus, budget totals) are converted to this currency using live exchange rates. Individual accounts may use any currency. |
| **Permission** | Identity & Family | A discrete capability flag that can be granted to `Role::Other` members by the Owner at invite time (e.g., `ViewLoans`, `RecordOwnTransactions`). Stored as a JSON array on `FamilyMember`. |
| **Payment Instrument / Account** | Payment Cards & Accounts | Any monetary store held by a family member: Credit Card, Debit Card, Cash Envelope, Bank Account. Managed via `payment_accounts` table. |
| **AccountBalanceSnapshot** | Payment Cards & Accounts | An immutable record of an account's closing balance at the end of a calendar month, computed from ledger transactions. Enables monthly balance history. |
| **Transaction** | Ledger Core | An immutable, append-only record of financial movement (`Income`, `Expense`, `Transfer`, `CashWithdrawal`, `LoanPayment`, `Reversal`). |
| **TransactionAmendment** | Ledger Core | A correction operation: creates a `Reversal` transaction (negating the original) and a new corrected transaction. Both entries carry `amendment_of_id` pointing to the original. The original is never modified in place. |
| **CrossCurrencyTransfer** | Ledger Core | A `Transfer` transaction where source account currency differs from destination account currency. The user records both the deducted source amount and received destination amount from their bank statement. |
| **FXSpread** | Ledger Core | The implicit cost of a `CrossCurrencyTransfer`, computed as source amount converted to home currency minus destination amount converted to home currency. |
| **Committed Obligation** | Recurring Payments | A non-negotiable regular payment (utility bill, rent, subscription, insurance) occurring on a schedule. |
| **LoanBoundInsurance** | Recurring Payments | A `RecurringPayment` of kind `insurance` with a `linked_loan_id` set. Represents mandatory insurance costs tied to a specific loan (e.g., PMI or homeowner's insurance). |
| **Discretionary Spending** | Budget & Limits | Flexible living expenses (groceries, dining, entertainment) bounded by monthly category envelope limits. |
| **BudgetDraft** | Budget & Limits | A `MonthlyBudget` in `Draft` status, auto-created on the 1st of each month. The Owner reviews and adjusts envelope limits before approving. |
| **True Disposable Surplus** | Debt Planner | The real disposable monthly savings available for debt acceleration after accounting for committed bills, loan debt service costs, and living expenses. |
| **LoanKind** | Debt Planner | A categorization label for a loan. System-standard kinds (mortgage, car loan, student loan, etc.) apply to all families. Families may also create custom kinds in their own language. |
| **LoanPayment** | Debt Planner | A child entity of `Loan` recording a single payment with its principal/interest split. Enforces `principal_portion + interest_portion == amount`. |
| **DebtServiceCost** | Debt Planner | The true total monthly cost of servicing a loan: `monthly_payment + sum(linked_insurance_payments)`. Used in the True Disposable Surplus formula. |
| **LinkedCreditCard** | Debt Planner | A `PaymentAccount(CreditCard)` explicitly linked to a `Loan` via `linked_account_id`. Spending on the card auto-increments the loan balance via CDC. |
| **Avalanche Strategy** | Debt Planner | Debt payoff method prioritizing loans by highest `annual_interest_rate` descending to minimize total interest paid. |
| **Snowball Strategy** | Debt Planner | Debt payoff method prioritizing loans by lowest `current_balance` ascending for psychological momentum. |
| **GoalContribution** | Future Planning | A child entity recording a single savings deposit toward a `SavingsGoal`. The goal's `current_saved` is computed as the sum of all its contributions. |
| **ExchangeRate** | Infrastructure | A live rate record: `1 quote_currency = rate HomeDisplayCurrency`. Fetched daily from a pluggable provider (BNM by default). Only latest rate per pair is stored. |

---

## 2. Core Business Formulas

### 2.1 True Disposable Surplus Formula

All monetary amounts are normalized to `HomeDisplayCurrency` using latest live `ExchangeRate` values prior to evaluation:

```
True Disposable Surplus =
    Average Monthly Income                        (in HomeDisplayCurrency)
  − Total Monthly Recurring Obligations           (active unlinked RecurringPayments)
  − Total Discretionary Budget Ceiling            (living expenses target envelopes)
  − Sum of All Loan DebtServiceCosts              (monthly_payment + linked insurance)
  ────────────────────────────────────────────────────────────────────────────────────
  = Extra Monthly Budget Available for Debt Acceleration
```

### 2.2 Loan Debt Service Cost

$$\text{DebtServiceCost}(\text{loan}) = \text{loan.monthly\_payment} + \sum_{\text{linked}} \text{RecurringPayment.amount}$$

### 2.3 Recurring Payment Proration Formula
To calculate `MonthlyCommittedAmount`, non-monthly recurring payments are prorated:
- **Monthly**: $\text{Amount}$
- **Quarterly**: $\frac{\text{Amount}}{3}$
- **Annual**: $\frac{\text{Amount}}{12}$

### 2.4 Loan Payoff & Payment Rollover
When Loan $L_1$ is paid off ($Balance = 0$):
$$\text{Freed Payment} = \text{DebtServiceCost}(L_1) = \text{Minimum Payment}(L_1) + \sum_{\text{linked}} \text{Insurance}(L_1)$$
$$\text{New Extra Budget for } L_2 = \text{Previous Extra Budget} + \text{Freed Payment}$$

---

## 3. Strict Domain Invariants

1. **Float-Free Money**: All money calculations must use exact fixed-point decimal arithmetic via `rust_decimal::Decimal`. Never use floating-point types (`f32`, `f64`).
2. **Currency Consistency**: Arithmetic operations between two `Money` values require matching `CurrencyCode`. Cross-currency math must explicitly convert via `ExchangeRate`.
3. **Transfer Invariant**: A `Transfer` or `CashWithdrawal` transaction must have both `source_account_id` and `destination_account_id` present and distinct.
4. **Cross-Currency Transfer Invariant**: When source and destination account currencies differ, `destination_amount_value` and `destination_amount_currency` must both be populated.
5. **LoanPayment Split Invariant**: For every `LoanPayment`, `principal_portion + interest_portion == amount`.
6. **Ledger Immutability Invariant**: No `UPDATE` or hard-`DELETE` on `ledger_transactions`. Corrections use the `TransactionAmendment` pattern (`Reversal` + new corrected entry).
7. **Budget Approval Invariant**: `BudgetEnvelope` limits are read-only once `MonthlyBudget.status == 'active'`. Only `Role::Owner` can approve a draft budget.
8. **Soft-Delete Invariant**: Entity state changes must never physically erase business records (`deleted_at TIMESTAMPTZ NULL`).
9. **Exchange Rate Staleness Rule**: All currency conversions must use the latest `ExchangeRate`. Rates older than 48 hours must trigger a UI warning banner.
10. **Permission Scope Invariant**: Permission flags are only configurable for `Role::Other` members. `Owner`, `Member`, and `Child` roles have fixed, non-configurable permission sets.
