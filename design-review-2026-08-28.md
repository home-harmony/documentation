---

## Remaining Open Issues & Resolutions

| # | File(s) | Issue | Resolution / Status |
|---|---------|-------|---------------------|
| **A** | Repo name / docs | Repo is `home-harmony`; product is `FamilyLedger`. | **Resolved**: `FamilyLedger` is the internal product/service architecture name; `HomeHarmony` is the external brand name. |
| **B** | `database-schema.md` / `domain` | `family_invite_tokens.token` PK format & UUID compliance. | **Resolved**: Updated to `token UUID PRIMARY KEY DEFAULT gen_random_uuid()` in database schema and `token: Uuid` in domain entity. |
| **C** | `domain/events/mod.rs` | `DomainEvent` enum has 5 events; `event-storming.md` catalogs 24 events. | **Resolved**: Added `// TODO: Sprint N` markers for all 19 remaining events across bounded contexts. |
| **D** | `backend/domain/errors.rs` | Missing `InvalidPaymentSplit` variant for ADR-0008. | **Resolved**: Added `DomainError::InvalidPaymentSplit { principal, interest, total }`. |
| **E** | `backend/Cargo.toml` | Only `migrate_runner` in workspace `members`. | **Resolved**: Confirmed strategy — service Lambdas will be added sprint-by-sprint. |
| **F** | `database-schema.md` / `domain` | `family_invite_tokens` missing `created_at` timestamp. | **Resolved**: Added `created_at TIMESTAMPTZ NOT NULL DEFAULT now()` to schema and `DateTime<Utc>` to domain entity. |
| **G** | `database-schema.md` | `UNIQUE(family_id, code)` NULL-key behavior in Aurora DSQL. | **Documented**: Clarified standard SQL NULL semantics; targeted integration test scheduled for Sprint 4. |
| **H** | `architecture/api-design.md` | `GET /analytics/monthly-surplus` average monthly income calculation path. | **Postponed**: Formally postponed to Sprint 5 as a separate analytics design decision. |
| **I** | `architecture/api-design.md` | Multi-currency budget envelope aggregation logic. | **Postponed**: Formally postponed to Sprint 4 (Debt & Multi-currency Budgeting). |
| **J** | `backend/Cargo.toml` | Dependency versions validation against crates.io. | **Resolved**: All dependencies pinned to real published versions (`tokio 1.53.1`, `lambda_runtime 1.3.0`, `lambda_http 1.3.0`, `axum 0.8.9`, `tower 0.5.3`, `sqlx 0.9.0`, `aurora-dsql-sqlx-connector 0.2.2`). |

---

## Summary of Schema Changes

The table below lists all schema modifications implied by the decisions above.

| Change | Type | Decision |
|--------|------|----------|
| Add `family_roles` table | New table | #1 |
| Modify `family_members.role` column + add `role_id` | Alter table | #1 |
| Modify `family_invite_tokens.token` to `UUID PK` + add `created_at` | Alter table | #1, #B, #F |
| Add `account_frozen_periods` table + index | New table | #4 |
| `budget_monthly_budgets.status` CHECK: remove `'closed'` | Alter table | #5 |
| Remove `budget_envelopes.spent_value` column | Alter table | #10 |
| Add `ledger_transactions(category_id)` async index | New index | #13 |

---

_Document updated: 2026-08-31_
