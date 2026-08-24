# 0003. Clean Architecture and Domain-Driven Design in Rust Workspace

* **Status**: Accepted
* **Date**: 2026-06-16
* **Deciders**: Core Engineering Team

## Context and Problem Statement
Financial applications require rigorous business invariants, exact monetary arithmetic, and high testability. Blending database access, framework routing, and business logic into monolithic handlers leads to test brittleness and domain logic corruption.

## Decision Drivers
* Isolate core financial invariants and algorithms (e.g. debt repayment plans) from database and cloud dependencies.
* Enable fast, zero-I/O unit tests running in milliseconds.
* Facilitate independent compilation of microservice Lambda targets.

## Considered Options
1. **Clean Architecture / DDD Multi-Crate Workspace** (`domain/`, `infrastructure/`, `api/`, `lambdas/`)
2. **Layered Monolith per Lambda** (Each microservice contains its own DB and domain code)
3. **Hexagonal Architecture with Dynamic Traits**

## Decision Outcome
Chosen option: **Clean Architecture / DDD Multi-Crate Workspace**, structuring the Rust codebase into distinct crates where `domain/` has zero external I/O or framework dependencies.

### Positive Consequences
* The `domain/` crate compiles in seconds and supports 95%+ unit test coverage with no mock setups.
* Database repositories and EventBridge publishers are cleanly encapsulated in `infrastructure/`.
* Axum handlers in `lambdas/` act as thin composition roots.

### Negative Consequences / Trade-offs
* Requires careful type mapping between domain entities, SQLx database rows, and Axum DTOs.

## Compliance & Invariants
* The `domain/` crate must never import `sqlx`, `axum`, `lambda_http`, or network libraries.
* Domain entities must maintain their internal invariants via private fields and validated constructors.

