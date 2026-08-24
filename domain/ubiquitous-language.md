# Ubiquitous Language & Core Formulas — FamilyLedger

This document defines the official domain vocabulary, business rules, and calculation formulas used across all services, databases, and UI components.

---

## 1. Domain Glossary

| Term | Bounded Context | Definition |
| :--- | :--- | :--- |
| **Family Workspace** | Identity & Family | The primary multi-tenant boundary. All entities and queries belong to a specific `family_id`. |
| **Payment Instrument** | Payment Cards | Any monetary store held by a family member: Credit Card, Debit Card, Cash Envelope, Bank Account. |
| **Transaction** | Ledger Core | An immutable record of financial movement (`Income`, `Expense`, `Transfer`, `CashWithdrawal`). |
| **Committed Obligation** | Recurring Payments | A non-negotiable regular payment (utility bill, rent, subscription, insurance) occurring on a schedule. |
| **Discretionary Spending** | Budget & Limits | Flexible living expenses (groceries, dining, entertainment) bounded by monthly category limits. |
| **True Disposable Surplus**| Debt Planner | The real disposable monthly savings available for debt acceleration after accounting for committed bills and living expenses. |
| **Avalanche Strategy** | Debt Planner | Debt payoff method prioritizing loans by highest `annual_interest_rate` descending to minimize total interest paid. |
| **Snowball Strategy** | Debt Planner | Debt payoff method prioritizing loans by lowest `current_balance` ascending for psychological momentum. |

---

## 2. Core Business Formulas

### 2.1 True Disposable Surplus Formula

```
True Disposable Surplus =
    Average Monthly Income
  − Total Monthly Recurring Obligations  (from active RecurringPayments)
  − Total Discretionary Budget Ceiling    (living expenses target)
  − Sum of All Loan Minimum Payments
  ──────────────────────────────────────
  = Extra Monthly Budget Available for Debt Acceleration
```

### 2.2 Recurring Payment Proration Formula
To calculate `MonthlyCommittedAmount`, non-monthly recurring payments are prorated:
- **Monthly**: $\text{Amount}$
- **Quarterly**: $\frac{\text{Amount}}{3}$
- **Annual**: $\frac{\text{Amount}}{12}$

### 2.3 Loan Payoff & Payment Rollover
When Loan $L_1$ is paid off ($Balance = 0$):
$$\text{Freed Payment} = \text{Minimum Payment}(L_1)$$
$$\text{New Extra Budget for } L_2 = \text{Previous Extra Budget} + \text{Freed Payment}$$

---

## 3. Strict Domain Invariants

1. **Float-Free Money**: All money calculations must use exact fixed-point decimal arithmetic via `rust_decimal::Decimal`.
2. **Currency Consistency**: All operations between two `Money` values must verify identical `CurrencyCode`. Cross-currency math without explicit exchange conversion is prohibited.
3. **Transfer Invariant**: A `Transfer` or `CashWithdrawal` transaction must have both `source_account_id` and `destination_account_id` present and distinct.
4. **Soft-Delete Invariant**: Entity state changes must never physically erase business records.

