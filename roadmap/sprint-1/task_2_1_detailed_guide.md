# Task 2.1 Detailed Guide — Rust Multi-Crate Workspace Initialization

> **Goal**: Initialize the Rust multi-crate Cargo workspace structure in `backend/` following Clean Architecture and Domain-Driven Design (DDD) principles (ADR-0003), establishing zero-I/O domain isolation, infrastructure adapters, shared API middleware, and Lambda binary targets.

---

## What You Will Build (Mental Model)

```
                              ┌───────────────────────────────┐
                              │      AWS Lambda Binaries      │
                              │  (backend/lambdas/*)          │
                              │  - migrate_runner             │
                              │  - family_service (Sprint 2)  │
                              │  - accounts_service (Sprint 3)│
                              │  - ledger_service (Sprint 3)  │
                              └───────┬───────────────┬───────┘
                                      │               │
                                      ▼               ▼
                 ┌─────────────────────────┐     ┌─────────────────────────┐
                 │    API / HTTP Layer     │     │  Infrastructure Layer   │
                 │    (backend/api)        │     │  (backend/infrastructure│
                 │  - Axum AuthClaims      │     │  - Aurora DSQL / SQLx   │
                 │  - X-Family-Id Context  │     │  - Keyset Pagination    │
                 │  - Common API Responses │     │  - EventBridge / FX     │
                 └────────────┬────────────┘     └────────────┬────────────┘
                              │                               │
                              └───────────────┬───────────────┘
                                              ▼
                               ┌─────────────────────────────┐
                               │     Pure Domain Model       │
                               │     (backend/domain)        │
                               │  - Entities & Value Objects │
                               │  - Domain Events & Errors   │
                               │  - Repayment Algorithms     │
                               │  - ZERO I/O / ZERO Network  │
                               └─────────────────────────────┘
```

---

## Layer Responsibilities & Architectural Invariants (ADR-0003)

| Crate | Purpose | Allowed Dependencies | Invariants & Rules |
| :--- | :--- | :--- | :--- |
| **`domain`** | Pure business entities (`Family`, `FamilyMember`, `InviteToken`), value objects (`Money`, `CurrencyCode`, `FamilyName`, `DisplayName`, `Email`, `Relationship`, `InviteTtl`, `Role`, `Permission`, `DomainAction`, typed IDs), domain events, domain errors, and debt repayment algorithms. | `rust_decimal`, `uuid`, `chrono`, `serde`, `nutype`, `email_address`, `thiserror` (`serde_json` in `dev-dependencies`) | **STRICT ZERO I/O**: Never import `sqlx`, `tokio`, `axum`, `reqwest`, `aws-sdk-*`, or filesystem APIs. |
| **`infrastructure`** | Database access (Aurora DSQL via SQLx), keyset pagination, EventBridge publisher, BNM/ECB exchange rate fetcher, Testcontainers setup. | `sqlx`, `aurora-dsql-sqlx-connector`, `tokio`, `serde`, `serde_json`, `rust_decimal`, `uuid`, `chrono`, `thiserror`, `anyhow`, `base64`, `tracing` *(add `domain` in Sprint 2 when implementing repository traits)* | Implements domain repository traits; handles OCC retry logic and IAM token refresh. |
| **`api`** | Common web concerns: RFC 9457 `ApiError` responses, `AuthClaims` extraction (`user_id` + optional `email` from JWT, `family_id` from verified `X-Family-Id` membership), standard JSON helpers (`ok`, `created`, `no_content`). | `domain`, `axum`, `tower`, `tokio`, `serde`, `serde_json`, `uuid`, `chrono`, `thiserror`, `tracing` *(add `lambda_http`, `tower-http`, `jsonwebtoken` in Sprint 2 HTTP/JWT middleware)* | Extracts authenticated context from HTTP requests and delegates to application use-cases. |
| **`lambdas/*`** | Standalone executables compiled for AWS Lambda (ARM64 / Graviton). | `domain`, `infrastructure`, `api`, `lambda_runtime`, `lambda_http`, `tokio`, `sqlx`, `anyhow`, `tracing*` | Thin composition roots that wire together pools, repositories, config, and invoke use-cases. |

---

## Step-by-Step Implementation

### Step 1: Folder Hierarchy

```
backend/
├── Cargo.toml                  # Workspace Manifest
├── AGENTS.md                   # Backend Rust & Aurora DSQL rules
├── domain/
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── errors.rs
│       ├── entities/
│       │   ├── mod.rs
│       │   └── family.rs
│       ├── events/
│       │   └── mod.rs
│       └── value_objects/
│           ├── mod.rs
│           ├── currency_code.rs
│           ├── display_name.rs
│           ├── email.rs
│           ├── family_name.rs
│           ├── ids.rs
│           ├── invite_ttl.rs
│           ├── money.rs
│           ├── relationship.rs
│           └── role.rs
├── infrastructure/
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       └── db/
│           ├── mod.rs
│           ├── pagination.rs
│           └── pool.rs
├── api/
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── auth.rs
│       ├── errors.rs
│       └── response.rs
└── lambdas/
    └── migrate_runner/
        ├── Cargo.toml
        └── src/
            └── main.rs
```

---

### Step 2: Root Workspace Manifest (`backend/Cargo.toml`)

```toml
[workspace]
members = ["domain", "infrastructure", "api", "lambdas/migrate_runner"]
resolver = "2"

[workspace.package]
version = "0.1.0"
edition = "2024"
license = "MIT"

[workspace.dependencies]
tokio = { version = "1.54.1", features = ["full"] }

sqlx = { version = "0.9.0", features = [
    "postgres",
    "runtime-tokio",
    "tls-rustls-ring",
    "macros",
    "migrate",
    "uuid",
    "chrono",
    "rust_decimal",
    "json",
] }
aurora-dsql-sqlx-connector = { version = "0.2.2", features = ["pool", "occ"] }

rust_decimal = { version = "1.43.0", features = ["serde-with-str"] }
rust_decimal_macros = "1.40.0"
uuid = { version = "1.29.3", features = ["v4", "v7", "serde"] }
chrono = { version = "0.4.45", features = ["serde"] }
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.154"

axum = { version = "0.8.9", features = ["macros"] }
lambda_runtime = "1.5.0"
tower = "0.5.3"
# Note: `lambda_http`, `tower-http`, and `jsonwebtoken` will be added when Sprint 2 HTTP/JWT middleware lands.

tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter", "json"] }

thiserror = "2.0.21"
anyhow = "1.0.104"
base64 = "0.23.1"
nutype = { version = "0.8.0", features = ["serde"] }
email_address = { version = "0.2.9", default-features = false }

[workspace.lints.rust]
unsafe_code = "forbid"

[workspace.lints.clippy]
unwrap_used = "deny"
expect_used = "deny"
```

---

### Step 3: `domain` Crate (`backend/domain/`)

* **`Cargo.toml`**: inherits `version.workspace = true`, `edition.workspace = true`, `license.workspace = true`, `[lints] workspace = true`, and zero-I/O dependencies (`rust_decimal`, `uuid`, `chrono`, `serde`, `thiserror`, `nutype`, `email_address`; dev-deps `rust_decimal_macros`, `serde_json`).
* **`src/lib.rs`**: exports `entities`, `errors`, `events`, and `value_objects`.
* **Value Objects**:
  * `CurrencyCode` (3 uppercase ASCII letters via `nutype`, constructed via `try_new`)
  * `Email` (trimmed, lowercased, RFC-validated via `nutype` + `email_address`)
  * `FamilyName` (trimmed, non-empty, max 200 chars via `nutype`)
  * `DisplayName` (trimmed, non-empty, max 100 chars via `nutype`)
  * `Relationship` (trimmed, non-empty, max 50 chars via `nutype`)
  * `InviteTtl` (bounded positive invite expiration duration `1..=2_592_000` seconds via `nutype`)
  * `Money` (`rust_decimal::Decimal >= 0` + `CurrencyCode`, currency-safe checked arithmetic)
  * `Role` (`Owner`, `Member`, `Child`, `Other`), `Permission` (8 capability flags), & `DomainAction`
  * Strongly-typed UUID v4 / v7 newtypes in `ids.rs` (`FamilyId`, `MemberId`, `UserId`, `InviteTokenId`, `TransactionId`, etc.)
* **Aggregates & Events**:
  * `Family`, `FamilyMember`, `InviteToken` in `entities/family.rs`
  * `DomainEvent` enum in `events/mod.rs`
  * `DomainError` enum in `errors.rs`

---

### Step 4: `infrastructure` Crate (`backend/infrastructure/`)

* **`Cargo.toml`**: inherits workspace metadata and `[lints] workspace = true`.
* **`src/lib.rs`**: exports `pub mod db;`.
* **`src/db/pool.rs`**: `create_pool() -> anyhow::Result<PgPool>` wrapping `aurora_dsql_sqlx_connector::pool::connect(&endpoint)` using `DSQL_ENDPOINT`.
* **`src/db/pagination.rs`**: `PageCursor` (URL-safe base64 JSON cursor `{ t, i }`) and `Page<T>::from_query_result` using the `LIMIT page_size + 1` pattern with `MAX_PAGE_SIZE = 50`.

---

### Step 5: `api` Crate (`backend/api/`)

* **`Cargo.toml`**: inherits workspace metadata and `[lints] workspace = true`.
* **`src/lib.rs`**: exports `pub mod auth; pub mod errors; pub mod response;`.
* **`src/auth.rs`**: `AuthClaims` (`user_id: UserId`, `family_id: FamilyId`, `role: Role`, `permissions: Vec<Permission>`, `email: Option<Email>`) Axum extractor + `AuthError`.
* **`src/errors.rs`**: RFC 9457 `ApiError` (`application/problem+json`) and exhaustive `domain_error_response(DomainError) -> Response`.
* **`src/response.rs`**: `ok`, `created`, and `no_content` JSON helpers.

---

### Step 6: `lambdas/migrate_runner` (`backend/lambdas/migrate_runner/`)

* **`Cargo.toml`**: depends on `infrastructure`, `lambda_runtime`, `sqlx`, `tokio`, `serde`, `serde_json`, `anyhow`, `tracing`, `tracing-subscriber` with `[[bin]] name = "bootstrap"`.
* **`src/main.rs`**: embeds `../../migrations` via `sqlx::migrate!`, initializes JSON tracing logs, connects via `infrastructure::db::pool::create_pool()`, executes `MIGRATOR.run(&pg_pool)`, and returns `Response { message, total_migrations }`.

---

## Verification & Sanity Checks

Run the following checks from the `backend/` directory:

1. **Check workspace compilation**:
   ```powershell
   cargo check --workspace --all-targets
   ```
2. **Run unit & doctests**:
   ```powershell
   cargo test --workspace
   ```
3. **Run Clippy linter**:
   ```powershell
   cargo clippy --workspace --all-targets -- -D warnings
   ```
4. **Format verification**:
   ```powershell
   cargo fmt --all -- --check
   ```
