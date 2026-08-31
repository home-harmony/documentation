# Amazon Aurora DSQL — Architectural Constraints & Developer Guide

## 1. Overview

Amazon Aurora DSQL is a distributed, serverless relational database offering active-active multi-region availability with wire compatibility for PostgreSQL 16. With full support for transactional foreign keys and native `JSONB` data types, Aurora DSQL combines the operational simplicity of serverless distributed computing with PostgreSQL relational integrity.

---

## 2. Aurora DSQL vs. Single-Instance PostgreSQL

| Feature / Behavior | PostgreSQL 16 | Amazon Aurora DSQL | FamilyLedger Strategy |
| :--- | :--- | :--- | :--- |
| **Storage Architecture** | Heap-based tables | Primary key-ordered distributed B-trees | **Mandatory UUID v4 / v7 PKs** (`gen_random_uuid()`). Sequential/SERIAL PKs cause severe write hot-spots. |
| **Foreign Keys** | DB-enforced (`REFERENCES`) | **Fully Supported** | Use explicit `FOREIGN KEY (parent_id) REFERENCES parent_table(id)` to enforce referential integrity across bounded contexts. |
| **JSON Data Types** | Native `JSON` / `JSONB` | **Fully Supported (`JSONB`)** | Use native `JSONB` for schema-flexible attributes (`permissions`, `metadata`, `extra_params`). |
| **Identity / Serials** | `SERIAL`, `BIGSERIAL` | **Not recommended for PKs** | Use `gen_random_uuid()` for PKs; `GENERATED ALWAYS AS IDENTITY` only for non-PK sequences. |
| **Truncation** | `TRUNCATE TABLE` | **Not supported** | Use `DELETE FROM table WHERE ...` in batches $\le 500$ rows. |
| **Database Triggers** | `CREATE TRIGGER` | **Not supported** | Implement all calculations, validations, and downstream events in Rust code. |
| **Cascading Deletes** | `ON DELETE CASCADE` | **Supported (RESTRICT preferred)** | Prefer `ON DELETE RESTRICT` with explicit application-level soft-deletes (`deleted_at TIMESTAMPTZ NULL`). |
| **DDL Transactions** | Multi-statement DDL | **1 DDL statement per transaction** | Exactly **1 DDL operation per migration file**. Never mix DDL and DML in the same file. |
| **Index Builds** | Blocking `CREATE INDEX` | `CREATE INDEX ASYNC` only | Write index creations in dedicated migration files; monitor status via `SELECT * FROM sys.jobs`. |
| **Write Tx Limit** | Unlimited | **3,000-row transaction limit** | Keep batch writes $\le 500$ rows per transaction with explicit individual commits. |
| **Concurrency Model** | Row-level locking (Pessimistic) | OCC (Optimistic Concurrency Control) | Use `aurora-dsql-sqlx-connector` with `occ` feature to auto-retry commit conflicts (SQLSTATE 40001). |
| **Read Transactions** | Standard `SELECT` | `BEGIN READ ONLY` | Use `BEGIN READ ONLY` on all read endpoints to eliminate OCC conflict overhead. |
| **Pagination** | `OFFSET` / `LIMIT` | Keyset (Cursor) only | `OFFSET` is banned (consumes DPUs proportional to depth). Keyset cursor with `(occurred_at, id) < ($2, $3)`. |
| **Authentication** | Password / IAM | IAM Authentication only | Managed automatically by `aurora-dsql-sqlx-connector` with background token refresh. |

---

## 3. Rust Integration (`aurora-dsql-sqlx-connector`)

We use version **`0.2.2`** of the official AWS connector with SQLx **`0.9.0`**:

```toml
# backend/Cargo.toml
[workspace.dependencies]
sqlx = { version = "0.9.0", features = ["postgres", "runtime-tokio-rustls", "macros", "migrate", "uuid", "chrono", "rust_decimal", "json"] }
aurora-dsql-sqlx-connector = { version = "0.2.2", features = ["pool", "occ"] }
```

### 3.1 Connection Pool Initialization
```rust
use aurora_dsql_sqlx_connector::pool;
use sqlx::PgPool;

pub async fn create_dsql_pool(endpoint: &str) -> Result<PgPool, sqlx::Error> {
    // Connector auto-detects AWS region, generates IAM auth tokens,
    // and refreshes tokens at 80% of their 15-minute validity window.
    pool::connect(endpoint).await
}
```

### 3.2 OCC Write Retry Pattern
```rust
use aurora_dsql_sqlx_connector::{retry_on_occ, OCCRetryConfig};

pub async fn save_transaction_with_retry(
    pool: &PgPool,
    tx_data: &Transaction,
) -> Result<(), AppError> {
    // Closure must be idempotent because it will be re-executed on OCC conflict
    retry_on_occ(&OCCRetryConfig::default(), || async {
        let mut tx = pool.begin().await?;
        
        sqlx::query!(
            r#"
            INSERT INTO ledger_transactions (
                id, family_id, recorded_by, kind, amount_value, amount_currency,
                destination_amount_value, destination_amount_currency,
                category_id, occurred_at, idempotency_key, metadata
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12)
            "#,
            tx_data.id,
            tx_data.family_id,
            tx_data.recorded_by,
            tx_data.kind.as_str(),
            tx_data.amount.value,
            tx_data.amount.currency.as_str(),
            tx_data.destination_amount.as_ref().map(|m| m.value),
            tx_data.destination_amount.as_ref().map(|m| m.currency.as_str()),
            tx_data.category_id,
            tx_data.occurred_at,
            tx_data.idempotency_key,
            tx_data.metadata // JSONB supported directly via serde_json::Value
        )
        .execute(&mut *tx)
        .await?;
        
        tx.commit().await?;
        Ok(())
    })
    .await
    .map_err(Into::into)
}
```

### 3.3 Read-Only Optimization
```rust
pub async fn list_transactions_read_only(
    pool: &PgPool,
    family_id: Uuid,
) -> Result<Vec<TransactionRow>, sqlx::Error> {
    let mut tx = pool.begin().await?;
    sqlx::query!("SET TRANSACTION READ ONLY").execute(&mut *tx).await?;
    
    let rows = sqlx::query_as!(
        TransactionRow,
        r#"
        SELECT id, family_id, kind, amount_value, amount_currency, occurred_at, metadata
        FROM ledger_transactions
        WHERE family_id = $1 AND deleted_at IS NULL
        ORDER BY occurred_at DESC, id DESC
        LIMIT 50
        "#,
        family_id
    )
    .fetch_all(&mut *tx)
    .await?;
    
    tx.commit().await?;
    Ok(rows)
}
```
