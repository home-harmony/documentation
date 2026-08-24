# Architectural Decision Records (ADRs)

This directory documents key architectural, infrastructural, and design decisions for the **FamilyLedger** project.

---

## What is an ADR?
An **Architectural Decision Record (ADR)** is a lightweight document capturing an important architectural decision, its context, consequences, trade-offs, and alternatives considered.

---

## ADR Index

| ADR Number | Title | Status | Date |
| :--- | :--- | :--- | :--- |
| [0001](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/decisions/0001-aurora-dsql-serverless-db.md) | Use Amazon Aurora DSQL for Serverless Relational Storage | **Accepted** | 2026-06-16 |
| [0002](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/decisions/0002-keyset-pagination.md) | Enforce Keyset (Cursor-Based) Pagination with UUID v7 | **Accepted** | 2026-06-16 |
| [0003](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/decisions/0003-rust-ddd-clean-architecture.md) | Clean Architecture and Domain-Driven Design in Rust Workspace | **Accepted** | 2026-06-16 |
| [0004](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/decisions/0004-multi-repo-organization.md) | Multi-Repository Structure under Umbrella Workspace | **Accepted** | 2026-06-16 |
| [0005](file:///c:/Users/demiu/my-rust-projects/home-harmony/documentation/decisions/0005-decimal-precision-money.md) | Exact Decimal Precision for Monetary Arithmetic | **Accepted** | 2026-06-16 |

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

