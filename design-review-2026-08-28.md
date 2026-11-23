# FamilyLedger — Design Review Log (2026-08-28)

> **Status:** Consolidated into canonical architecture, schema, and Sprint 1 codebase (updated 2026-11-17).
> **Scope:** Cross-document & backend consistency review.

---

## Remaining Open Issues & Resolutions

| # | File(s) | Issue | Resolution / Status |
|---|---------|-------|---------------------|
| **A** | Repo name / docs | Repo is `home-harmony`; product is `FamilyLedger`. | **Resolved**: `FamilyLedger` is the internal product/service architecture name; `HomeHarmony` is the external brand name. |
| **B** | `database-schema.md` / `domain` | `family_invite_tokens.token` PK format & UUID compliance. | **Resolved**: Updated to `token UUID PRIMARY KEY DEFAULT gen_random_uuid()` in database schema and `token: InviteTokenId` in domain entity. |
| **C** | `domain/events/mod.rs` | `DomainEvent` enum has 5 Sprint 1 events; `event-storming.md` catalogs 25 events. | **Resolved**: Aligned `// TODO: Sprint N` markers in `domain/events/mod.rs` with `event-storming.md`. |
| **D** | `backend/domain/errors.rs` | Missing `InvalidPaymentSplit` variant for ADR-0008. | **Resolved**: Added `DomainError::InvalidPaymentSplit { principal, interest, total }` and aligned ADR-0008. |
| **E** | `backend/Cargo.toml` | Only `migrate_runner` in workspace `members`. | **Resolved**: Confirmed strategy — service Lambdas will be added sprint-by-sprint. |
| **F** | `database-schema.md` / `domain` | `family_invite_tokens` missing `created_at` timestamp. | **Resolved**: Added `created_at TIMESTAMPTZ NOT NULL DEFAULT now()` to schema and `DateTime<Utc>` to domain entity. |
| **G** | `database-schema.md` | `UNIQUE(family_id, code)` NULL-key behavior in Aurora DSQL. | **Documented**: Clarified standard SQL NULL semantics; targeted integration test scheduled for Sprint 4. |
| **H** | `architecture/api-design.md` | `GET /analytics/monthly-surplus` average monthly income calculation path. | **Postponed**: Formally postponed to Sprint 5 as a separate analytics design decision. |
| **I** | `architecture/api-design.md` | Multi-currency budget envelope aggregation logic. | **Postponed**: Formally postponed to Sprint 4 (Debt & Multi-currency Budgeting). |
| **J** | `backend/Cargo.toml` | Dependency versions validation against crates.io. | **Resolved**: Upgraded all dependencies to latest verified crates (`tokio 1.54.1`, `lambda_runtime 1.5.0`, `lambda_http 1.3.2`, `axum 0.8.9`, `tower 0.5.3`, `tower-http 0.7.1`, `sqlx 0.9.0`, `aurora-dsql-sqlx-connector 0.2.2`, `rust_decimal 1.43.0`, `uuid 1.29.3`, `jsonwebtoken 12.0.0`, `nutype 0.8.0`). |

---

## Finalized Sprint 1 Baseline Schema & Role Model

The finalized Sprint 1 schema consists of **18 tables and 16 asynchronous indexes (34 migrations total)**:

| Decision | Final State in Sprint 1 |
|----------|-------------------------|
| Role & Permission Model | 4 system roles (`Owner`, `Member`, `Child`, `Other`) with `Permission` JSONB flags on `family_members` and `family_invite_tokens` (used for `Role::Other`). |
| `budget_monthly_budgets.status` | `CHECK (status IN ('draft','active'))` — `'closed'` removed across schema, API, and ADRs. |
| `budget_envelopes.spent_value` | Omitted from `budget_envelopes`; computed dynamically via `SUM(ledger_transactions)` at read time. |
| `ledger_transactions` pagination index | `idx_tx_family_time ON ledger_transactions (family_id, occurred_at, id)` (DSQL uses backward scan for `ORDER BY ... DESC`). |
| `FamilyName` / `DisplayName` lengths | `FamilyName` max 200 chars (`VARCHAR(200)`); `DisplayName` max 100 chars (`VARCHAR(100)`). |
| Event Publishing (ADR-0017) | Direct Domain Event Publishing to EventBridge (`$0.00` idle cost; Kinesis Data Stream removed). |
| Post-Sprint 1 Additions (`account_frozen_periods`, `ledger_categorization_rules`, `family_system_settings`) | Tracked in design docs (`ADR-0006` / `ADR-0018`) for upcoming feature sprints; Sprint 1 baseline stays at 18 tables + 16 async indexes. |

---

_Document updated: 2026-11-17_
