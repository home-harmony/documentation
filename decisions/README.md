# Architectural Decision Records (ADRs)

This directory documents key architectural, infrastructural, and design decisions for the **FamilyLedger** project.

---

## What is an ADR?
An **Architectural Decision Record (ADR)** is a lightweight document capturing an important architectural decision, its context, consequences, trade-offs, and alternatives considered.

---

## ADR Index

| ADR Number | Title | Status | Date |
| :--- | :--- | :--- | :--- |
| [0001](0001-aurora-dsql-serverless-db.md) | Use Amazon Aurora DSQL for Serverless Relational Storage | **Accepted** | 2026-06-16 |
| [0002](0002-keyset-pagination.md) | Enforce Keyset (Cursor-Based) Pagination with UUID v7 | **Accepted** | 2026-06-16 |
| [0003](0003-rust-ddd-clean-architecture.md) | Clean Architecture and Domain-Driven Design in Rust Workspace | **Accepted** | 2026-06-16 |
| [0004](0004-multi-repo-organization.md) | Multi-Repository Structure under Umbrella Workspace | **Accepted** | 2026-06-16 |
| [0005](0005-decimal-precision-money.md) | Exact Decimal Precision for Monetary Arithmetic | **Accepted** | 2026-06-16 |
| [0006](0006-account-balance-snapshot.md) | Separate Account Balance into Snapshot Table | **Accepted** | 2026-08-26 |
| [0007](0007-loan-kind-dictionary.md) | Loan Kind as Extensible Dictionary (System + Family-Defined) | **Accepted** | 2026-08-26 |
| [0008](0008-loan-payment-child-entity.md) | LoanPayment as Explicit Child Entity of the Loan Aggregate | **Accepted** | 2026-08-26 |
| [0009](0009-event-driven-loan-payment-ledger.md) | Event-Driven Loan Payment Ledgerization via EventBridge | **Accepted** | 2026-08-26 |
| [0010](0010-insurance-linked-recurring-payment.md) | Loan-Bound Insurance Modelled as a Linked RecurringPayment | **Accepted** | 2026-08-26 |
| [0011](0011-multi-family-membership.md) | A User May Belong to Multiple Family Workspaces Simultaneously | **Accepted** | 2026-08-26 |
| [0012](0012-immutable-ledger-transactions.md) | Ledger Transactions Are Immutable — Corrections Via Reversal Pattern | **Accepted** | 2026-08-26 |
| [0013](0013-budget-owner-approval-flow.md) | Monthly Budget Requires Owner Review and Approval Before Activation | **Accepted** | 2026-08-26 |
| [0014](0014-credit-card-explicit-loan-link.md) | Credit Card PaymentAccount Explicitly Linked to Loan for Debt Planning | **Accepted** | 2026-08-26 |
| [0015](0015-multi-currency-live-exchange-rates.md) | Multi-Currency Display with Live Exchange Rates and Dual-Amount Cross-Currency Transfers | **Accepted** | 2026-08-26 |
| [0016](0016-pluggable-exchange-rate-providers.md) | Pluggable ExchangeRateProvider Trait with BNM and ECB Frankfurter Implementations | **Accepted** | 2026-08-26 |

---

## ADR Template (MADR Format)

When proposing a new architectural decision, create a file named `YYYY-short-title.md` using this template:

```markdown
# [Number]. [Title]

* **Status**: Proposed | Accepted | Deprecated | Superseded by [ADR-XXXX]
* **Date**: YYYY-MM-DD
* **Deciders**: [List of participants / agents]

## Context and Problem Statement
What is the context, requirement, or technical challenge we are addressing?

## Decision Drivers
* Driver 1 (e.g. Cost, Performance, Serverless Compatibility)
* Driver 2 (e.g. Developer Ergonomics, Consistency)

## Considered Options
1. Option 1: [Name]
2. Option 2: [Name]

## Decision Outcome
Chosen option: "[Option 1]", because [rationale].

### Positive Consequences
* Benefit 1
* Benefit 2

### Negative Consequences / Trade-offs
* Trade-off 1
* Trade-off 2

## Compliance & Invariants
* How this decision is enforced in code or CI pipelines.
```
