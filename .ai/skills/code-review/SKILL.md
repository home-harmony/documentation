---
name: code-review
description: >-
  Mandatory post-coding self-review and PR readiness audit. Runs two isolated
  review axes (Standards & Architecture vs. Spec, Domain & Security), executes
  automated Rust verification gates, and classifies One-Way vs. Two-Way Door risks.
---

# Two-Axis Code Review & PR Readiness

Execute this skill automatically after modifying code or documentation contracts, and before opening a PR or concluding a coding task.

---

## 1. Automated Verification Gates (Must All Pass)

Run inside `backend/` whenever Rust or SQL files change:
```bash
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```
*(Verify `spec_traceability` passes as part of `--workspace` or via `cargo test -p domain --test spec_traceability`.)*

---

## 2. Two-Axis Isolated Audit

Pin the diff against the base commit/branch (`git diff <base>...HEAD` or working tree diff). Evaluate **Standards** and **Spec Conformance** separately so clean code never excuses missed requirements, and passing tests never excuse architectural violations.

*(For multi-file or cross-crate diffs, launch two isolated parallel subagents—one for Axis A blind to the feature prompt, one for Axis B—and report both findings unblended. For small single-file edits, execute both passes sequentially.)*

### Axis A — Standards & Architecture (`backend/.ai/rules/`)
- [ ] **Hexagonal Purity**: `domain/` has zero `sqlx`, `axum`, filesystem, or AWS SDK imports; `tokio::spawn` lives only in `infrastructure/` or `lambdas/*`.
- [ ] **Deep Modules & ACL**: Wire DTOs / DB rows stay separated from domain entities; ports use static-dispatch RPITIT (`Send + Sync`, no unnecessary `'static` on domain traits); no shallow pass-through wrappers.
- [ ] **Clean Code (SLAP & SRP)**: Functions target **10–30 lines** (hard ceiling **40 lines**), max nesting **2 levels** via guard clauses; Newspaper top-down file ordering (`Imports` $\rightarrow$ `Types` $\rightarrow$ `Impls` $\rightarrow$ `Functions` $\rightarrow$ `Tests`).
- [ ] **Type Safety & Prohibitions**: `nutype` / typed UUID newtypes for domain IDs and scalars; `rust_decimal::Decimal` for money (zero `f32`/`f64`); zero `.unwrap()`/`.expect()` in production code; no `Deref` polymorphism; no unjustified `.clone()`; no `#![deny(warnings)]`.
- [ ] **Aurora DSQL Invariants**:
  - Migrations follow `{YYYYMMDDHHMMSS}_{description}.sql`, **1 DDL statement per file**, no DML mixed in, `CREATE INDEX ASYNC`, never editing applied migrations.
  - Primary keys use UUID v4 (low-frequency/tokens) or `Uuid::now_v7()` (transactions/append logs); soft-deletes (`deleted_at`); `retry_on_occ`; `BEGIN READ ONLY` on read endpoints; keyset pagination only (zero `OFFSET`); all queries parameterized.

### Axis B — Spec, Domain & Security (`documentation/` & `backend/specs/`)
- [ ] **Ubiquitous Language & Invariants**: Code and identifiers match [`ubiquitous-language.md`](../../../domain/ubiquitous-language.md) (e.g., `TransactionAmendment` reversals rather than `UPDATE`, `principal + interest == amount`, `Role::Child` blocked from loans/budgets/reports).
- [ ] **Zero-Trust & AI Guardrails (`ADR-0011`, `ADR-0018`)**: `family_id`, `user_id`, and `role` come exclusively from verified Cognito JWT + `family_members` membership (`X-Family-Id`), never from body/query/LLM args; zero LLM monetary math.
- [ ] **API & Error Contracts**: HTTP statuses and error codes match [`api-design.md`](../../../architecture/api-design.md) and `api/src/errors.rs`; any endpoint changes update `api-design.md` in the same change.
- [ ] **Spec Traceability**: Every `Approved` rule in `backend/specs/` has a matching test in `backend/*/tests/*.rs`; `spec_traceability` passes all 4 guardrails (no tests on `Draft` rules, no `Origin: observed` or open questions on `Approved` rules); no spec test assertions were weakened; zero secrets committed.

---

## 3. PR & Blast-Radius Summary

When preparing a PR or final handoff summary, output three concise blocks:
1. **Structural Summary**: Minimal diagram, call path, or behavior summary (no repetitive file-by-file prose).
2. **Tiered Evidence**: Exact commands run and observed results (`cargo clippy`, `cargo test --workspace`, Docker PG integration runs if applicable).
3. **Door Classification (Blast Radius)**:
   - **One-Way Doors (High Risk — Spotlight for Human Sign-Off)**: New/altered Aurora DSQL migrations (irreversible once applied), EventBridge event schemas, Cognito/RBAC policy changes, public API breaking changes.
   - **Two-Way Doors (Reversible)**: Internal `domain/` refactors, unit/spec tests, documentation updates.
