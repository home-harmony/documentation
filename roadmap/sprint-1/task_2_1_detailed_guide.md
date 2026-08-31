# Task 2.1 Detailed Guide — Rust Multi-Crate Workspace Initialization

> **Goal**: Initialize the Rust multi-crate Cargo workspace structure in `backend/` following Clean Architecture and Domain-Driven Design (DDD) principles (ADR-0003), establishing zero-I/O domain isolation, infrastructure adapters, shared API middleware, and Lambda binary targets.

---

## What You Will Build (Mental Model)

```
                              ┌───────────────────────────────┐
                              │      AWS Lambda Binaries      │
                              │  (backend/lambdas/*)          │
                              │  - migrate_runner             │
                              │  - family_service             │
                              │  - ledger_service             │
                              │  - cdc_fanout                 │
                              └───────┬───────────────┬───────┘
                                      │               │
                                      ▼               ▼
                 ┌─────────────────────────┐     ┌─────────────────────────┐
                 │    API / HTTP Layer     │     │  Infrastructure Layer   │
                 │    (backend/api)        │     │  (backend/infrastructure│
                 │  - Axum Middlewares     │     │  - Aurora DSQL / SQLx   │
                 │  - Cognito JWT Claims   │     │  - Kinesis / EventBridge│
                 │  - Common API Responses │     │  - Exchange Rate Client │
                 └────────────┬────────────┘     └────────────┬────────────┘
                              │                               │
                              └───────────────┬───────────────┘
                                              ▼
                               ┌─────────────────────────────┐
                               │     Pure Domain Model       │
                               │     (backend/domain)        │
                               │  - Entities & Value Objects │
                               │  - Domain Invariants        │
                               │  - Repayment Algorithms     │
                               │  - ZERO I/O / ZERO Network  │
                               └─────────────────────────────┘
```

---

## Layer Responsibilities & Architectural Invariants (ADR-0003)

| Crate | Purpose | Allowed Dependencies | Invariants & Rules |
| :--- | :--- | :--- | :--- |
| **`domain`** | Pure business entities, value objects, domain errors, and debt repayment algorithms. | `rust_decimal`, `uuid`, `chrono`, `serde`, `nutype`, `thiserror` | **STRICT ZERO I/O**: Never import `sqlx`, `tokio`, `axum`, `reqwest`, `aws-sdk-*`, or filesystem APIs. |
| **`infrastructure`** | Database access (Aurora DSQL via SQLx), EventBridge publisher, BNM/ECB exchange rate fetcher, Testcontainers setup. | `domain`, `sqlx`, `aurora-dsql-sqlx-connector`, `aws-sdk-*`, `reqwest`, `tokio` | Implements domain repository traits; handles OCC retry logic and IAM token refresh. |
| **`api`** | Common web concerns: Axum error responses, `X-Family-Id` extraction, Cognito JWT claims extraction, pagination query params. | `domain`, `axum`, `tower`, `tower-http`, `serde`, `jsonwebtoken` | Extracts authenticated context from HTTP requests and delegates to application use-cases. |
| **`lambdas/*`** | Standalone executables compiled for AWS Lambda (ARM64 / Graviton). | `domain`, `infrastructure`, `api`, `lambda_runtime`, `lambda_http`, `tokio` | Thin composition roots that wire together repositories, config, and invoke use-cases. |

---

## Step-by-Step Implementation

### Step 1: Create the Folder Hierarchy

Create the following folder structure under `backend/`:

```
backend/
├── Cargo.toml                  # Workspace Manifest
├── domain/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
├── infrastructure/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
├── api/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
└── lambdas/
    └── migrate_runner/
        ├── Cargo.toml
        └── src/
            └── main.rs
```

---

### Step 2: Define the Root Workspace Manifest (`backend/Cargo.toml`)

Create `backend/Cargo.toml` with centralized workspace dependencies and strict compiler lint rules:

```toml
[workspace]
resolver = "2"
members = [
    "domain",
    "infrastructure",
    "api",
    "lambdas/*",
]

[workspace.package]
version = "0.1.0"
edition = "2021"
authors = ["FamilyLedger Team"]
license = "MIT OR Apache-2.0"

[workspace.dependencies]
# Internal workspace crates
domain = { path = "domain" }
infrastructure = { path = "infrastructure" }
api = { path = "api" }

# Domain & Math
rust_decimal = { version = "1.36", features = ["serde-with-str", "maths"] }
uuid = { version = "1.10", features = ["v4", "v7", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
nutype = { version = "0.4", features = ["serde"] }
thiserror = "2.0"

# Async & Runtime
tokio = { version = "1.40", features = ["full"] }
futures = "0.3"
async-trait = "0.1"

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Web & API
axum = { version = "0.7", features = ["macros"] }
tower = { version = "0.5", features = ["util"] }
tower-http = { version = "0.6", features = ["cors", "trace", "catch-panic"] }

# AWS & Lambda
lambda_runtime = "0.13"
lambda_http = "0.13"
aws-config = "1.5"
aws-sdk-eventbridge = "1.40"
aws-sdk-kinesis = "1.40"

# Database & Aurora DSQL
sqlx = { version = "0.8", features = ["postgres", "runtime-tokio-rustls", "macros", "migrate", "uuid", "chrono", "rust_decimal"] }
aurora-dsql-sqlx-connector = { version = "0.2", features = ["pool", "occ"] }

# Observability & Logging
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }

# Testing
tokio-test = "0.4"
pretty_assertions = "1.4"

[workspace.lints.rust]
unsafe_code = "forbid"
missing_debug_implementations = "warn"
rust_2018_idioms = "warn"

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
nursery = { level = "warn", priority = -1 }
unwrap_used = "deny"
expect_used = "warn"
module_name_repetitions = "allow"
```

---

### Step 3: Configure `domain/Cargo.toml` & `src/lib.rs`

#### `backend/domain/Cargo.toml`
```toml
[package]
name = "domain"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
rust_decimal.workspace = true
uuid.workspace = true
chrono.workspace = true
nutype.workspace = true
thiserror.workspace = true
serde.workspace = true

[dev-dependencies]
pretty_assertions.workspace = true
```

#### `backend/domain/src/lib.rs`
```rust
//! # Pure Domain Crate
//!
//! Contains entities, value objects, and algorithms for FamilyLedger.
//! Strictly zero I/O and zero network dependencies.

#![forbid(unsafe_code)]
#![deny(clippy::unwrap_used)]

pub mod money;

#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error("Invalid monetary amount: {0}")]
    InvalidAmount(String),

    #[error("Invariant violated: {0}")]
    InvariantViolated(String),
}
```

---

### Step 4: Configure `infrastructure/Cargo.toml` & `src/lib.rs`

#### `backend/infrastructure/Cargo.toml`
```toml
[package]
name = "infrastructure"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
domain.workspace = true
sqlx.workspace = true
aurora-dsql-sqlx-connector.workspace = true
tokio.workspace = true
async-trait.workspace = true
tracing.workspace = true
thiserror.workspace = true

[dev-dependencies]
tokio-test.workspace = true
```

#### `backend/infrastructure/src/lib.rs`
```rust
//! # Infrastructure Layer
//!
//! Provides database connection pooling (Aurora DSQL) and external adapters.

#![forbid(unsafe_code)]

pub mod db;

#[derive(Debug, thiserror::Error)]
pub enum InfraError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Configuration error: {0}")]
    Config(String),
}
```

---

### Step 5: Configure `api/Cargo.toml` & `src/lib.rs`

#### `backend/api/Cargo.toml`
```toml
[package]
name = "api"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
domain.workspace = true
axum.workspace = true
tower.workspace = true
tower-http.workspace = true
serde.workspace = true
serde_json.workspace = true
tracing.workspace = true
thiserror.workspace = true
```

#### `backend/api/src/lib.rs`
```rust
//! # Shared API & HTTP Middleware
//!
//! Shared Axum extractors, responses, and error handlers.

#![forbid(unsafe_code)]

pub mod error;
pub mod extractors;
```

---

### Step 6: Configure `lambdas/migrate_runner/Cargo.toml` & `src/main.rs`

#### `backend/lambdas/migrate_runner/Cargo.toml`
```toml
[package]
name = "migrate_runner"
version.workspace = true
edition.workspace = true
license.workspace = true

[[bin]]
name = "migrate_runner"
path = "src/main.rs"

[dependencies]
infrastructure.workspace = true
lambda_runtime.workspace = true
tokio.workspace = true
serde.workspace = true
serde_json.workspace = true
tracing.workspace = true
tracing-subscriber.workspace = true
```

#### `backend/lambdas/migrate_runner/src/main.rs`
```rust
use lambda_runtime::{service_fn, Error, LambdaEvent};
use serde::{Deserialize, Serialize};
use tracing::info;

#[derive(Debug, Deserialize)]
struct MigrationRequest {}

#[derive(Debug, Serialize)]
struct MigrationResponse {
    message: String,
    status: String,
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .json()
        .init();

    info!("Starting FamilyLedger Migration Runner Lambda");
    lambda_runtime::run(service_fn(handler)).await
}

async fn handler(_event: LambdaEvent<MigrationRequest>) -> Result<MigrationResponse, Error> {
    info!("Migration Runner invoked");
    Ok(MigrationResponse {
        message: "Migration runner placeholder ready".to_string(),
        status: "SUCCESS".to_string(),
    })
}
```

---

## Verification & Sanity Checks

Run the following checks from the `backend/` directory:

1. **Check workspace compilation**:
   ```powershell
   cargo check --workspace --all-targets
   ```
2. **Run Clippy linter**:
   ```powershell
   cargo clippy --workspace --all-targets
   ```
3. **Format verification**:
   ```powershell
   cargo fmt --all -- --check
   ```
