# Testing Strategy — FamilyLedger

```
Test Pyramid
                    ▲
                   ╱ ╲   Layer 3: E2E (reqwest against deployed test stack)
                  ╱───╲
                 ╱ Integ╲  Layer 2b: Handler integration (Axum tower::ServiceExt)
                ╱────────╲ Layer 2a: DB integration (sqlx + Testcontainers PostgreSQL)
               ╱  Unit    ╲ Layer 1: Pure Domain Unit Tests (Rust domain/ crate)
              ╱────────────╲
             fast ◄──────► slow
```

---

## 1. Test Layers & Scope

### Layer 1: Domain Unit Tests
* **Location**: `backend/domain/tests/` and unit test modules inside `backend/domain/src/`.
* **Execution**: `cargo test -p domain` (Runs in $< 1$ second).
* **Coverage Target**: **95%+**.
* **Scope**:
  * Monetary math (`Money` arithmetic, zero handling, currency mismatch protections).
  * Debt optimization algorithms (`RepaymentPlanCalculator` for Avalanche and Snowball strategies).
  * Budget envelope alert threshold calculations.
  * Goal timeline and contribution projections.
  * Aggregate invariants and value object validations (e.g. transfer source $\neq$ destination).

### Layer 2a: Database Integration Tests
* **Location**: `backend/infrastructure/tests/`.
* **Execution**: `cargo test -p infrastructure` (Requires Docker active).
* **Scope**:
  * Real SQL execution against a local PostgreSQL 16 container via `testcontainers`.
  * Keyset cursor pagination encoding and query range verification.
  * Idempotent insert checks using unique idempotency keys.
  * Soft-delete query filtering (`WHERE deleted_at IS NULL`).

### Layer 2b: Lambda Handler Integration Tests
* **Location**: `backend/lambdas/<service>/tests/`.
* **Execution**: `cargo test -p <service>`.
* **Scope**:
  * In-process HTTP execution using `axum` and `tower::ServiceExt::oneshot()`.
  * Verifies request parsing, status codes, and response JSON shapes without AWS infrastructure.
  * Uses `SpyPublisher` mock to verify domain event emission.
  * Tests role-based access control (e.g., verifying `Role::Child` receives `403 Forbidden` on `/loans`).

### Layer 3: End-to-End (E2E) Tests
* **Location**: `backend/tests/e2e/`.
* **Execution**: `cargo test --test e2e -- --ignored` (Runs against deployed test stage).
* **Scope**:
  * End-to-end user journeys using `reqwest` with real Cognito auth tokens against API Gateway endpoints.

---

## 2. Test Commands Cheat-Sheet

```powershell
# Run pure domain unit tests (Instant feedback)
cargo test -p domain

# Run infrastructure database tests with Testcontainers
cargo test -p infrastructure

# Run specific handler tests
cargo test -p ledger_service

# Prepare SQLx query verification cache for CI
cargo sqlx prepare -- --all-targets
```

