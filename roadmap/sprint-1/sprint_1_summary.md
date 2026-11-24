# Sprint 1 Completion Summary — Infrastructure & Foundation

- **Status**: ✅ **Completed**

---

## 1. What Was Delivered

### AWS Serverless Infrastructure (`$0.00` Idle Cost)
- **Core Stack (`familyledger-core-dev`)**: Identity-only Amazon Cognito User Pool (`sub`, `email` — no custom `family_id`/`family_role` attributes per [`ADR-0011`](../../decisions/0011-multi-family-membership.md)), Web App Client, Hosted UI Domain, API Gateway HTTP API with JWT Authorizer and `X-Family-Id` CORS support, private versioned S3 SPA bucket, and CloudFront CDN with OAC and 403/404 SPA fallback.
- **Data & Event Stack (`familyledger-data-dev`)**: Custom Amazon EventBridge bus (`familyledger-events-dev`; Kinesis removed per [`ADR-0017`](../../decisions/0017-cdc-and-event-publishing-strategies.md)), least-privilege IAM roles (`FamilyLedgerServiceRole`, `FamilyLedgerMigrationRunnerRole`), and `FamilyLedgerMigrateRunnerFunction` (`familyledger-migrate-dev`, ARM64 `provided.al2023`).
- **Database Cluster**: Active-active Amazon Aurora DSQL cluster (`PostgreSQL 16` wire compatible).
- **Runbooks**: [`aws-deployment-and-teardown.md`](../../architecture/aws-deployment-and-teardown.md) · [`backend/infrastructure/sam/README.md`](../../../backend/infrastructure/sam/README.md)

### Rust Multi-Crate Workspace (`backend/`)
- Clean Architecture workspace (`domain`, `infrastructure`, `api`, `lambdas/migrate_runner`) on Rust `edition = "2024"` with `unsafe_code = "forbid"`, `unwrap_used = "deny"`, and `expect_used = "deny"` ([`backend/README.md`](../../../backend/README.md)).
- **`domain`**: Pure zero-I/O value objects (`Money`, `CurrencyCode`, `FamilyName`, `DisplayName`, `Email`, `Relationship`, `InviteTtl`, `Role`/`Permission`/`DomainAction`, UUID v4/v7 typed IDs), `Family`/`FamilyMember`/`InviteToken` aggregates, `DomainEvent`, and `DomainError`.
- **`api`**: RFC 9457 `application/problem+json` error mapper (`api/src/errors.rs`), `AuthClaims` extractor (`api/src/auth.rs`), and response helpers (`api/src/response.rs`). *(Sprint 2 HTTP/JWT crates `lambda_http`, `tower-http`, `jsonwebtoken` deferred cleanly.)*
- **`infrastructure` & `lambdas/migrate_runner`**: Aurora DSQL pool normalization (`DSQL_ENDPOINT`, `DSQL_USER`), keyset `PageCursor`/`Page` utilities, and shared `infrastructure::db::migrations` engine.

### Database Schema & Live Migration Execution
- Authored **34 SQL migration files** (**18 tables** + **16 `CREATE INDEX ASYNC` indexes**). See [`database-schema.md`](../../architecture/database-schema.md) and [`aurora-dsql.md`](../../architecture/aurora-dsql.md).
- Verified offline (`cargo test -p infrastructure --test migration_invariants`), against local PostgreSQL 16 via Testcontainers (`cargo test -p infrastructure -- --ignored`), and live on AWS Aurora DSQL via `familyledger-migrate-dev` (`applied_now: 34`, followed by idempotency re-run `already_applied: 34`, `active_async_jobs: 0`).

### Spec-Driven Governance & AI Workflow ([`ADR-0019`](../../decisions/0019-declarative-test-specifications.md))
- Accepted `ADR-0019`, established 7 cross-repo skills + modular rules, and enabled the `spec_traceability` CI gate.
- Approved **32 rules** (`VO-01`–`VO-10`, `FM-01`–`FM-22`) with 1:1 blind conformance test coverage (`vo_value_objects.rs`, `fm_family_rbac.rs`).

---

## 2. Key Technical Learnings Applied

- **Read-Only DSQL Transactions**: Aurora DSQL rejects `SET TRANSACTION READ ONLY` after `BEGIN`; always use `pool.begin_with("BEGIN READ ONLY")`.
- **DSQL DDL & Async Index Rules**: Every migration starts with `-- no-transaction`, uses `IF NOT EXISTS` and 1 DDL per file, omits `ASC`/`DESC` in index DDL, disables SQLx advisory locks (`Migrator::set_locking(false)`), retries OCC conflicts (`40001`/`OC000`/`OC001`), and awaits background builds via `CALL sys.wait_for_job`.
- **Windows ARM64 Lambda Builds**: Cross-compiling `aarch64-unknown-linux-gnu` (`aws-lc-sys` / `ring`) requires `cargo-lambda >= 1.9.2` and `zig >= 0.17.1`.

---

## 3. Sprint 2 Handoff Notes

1. Configure a non-admin Aurora DSQL database role mapped via `AWS IAM GRANT` for `FamilyLedgerServiceRole` (`DSQL_USER`) before deploying runtime microservices.
2. Add `lambda_http`, `tower-http`, and `jsonwebtoken` when implementing JWT + `X-Family-Id` auth middleware.
3. Draft `Format U` API specs (`AU` auth isolation, `FM` endpoints) in `backend/specs/api/` ahead of `family_service` implementation.
