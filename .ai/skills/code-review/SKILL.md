---
name: code-review
description: >-
  Mandatory post-coding self-review and PR readiness audit. Runs two isolated
  review axes (Standards & Architecture vs. Spec, Domain & Security), executes
  automated backend (Rust) and frontend-web (Angular) verification gates, and
  classifies One-Way vs. Two-Way Door risks.
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

Run inside `frontend-web/` whenever Angular, TypeScript, HTML, or CSS files change:
```bash
pnpm lint
pnpm test
pnpm build
```

---

## 2. Two-Axis Isolated Audit

Pin the diff against the base commit/branch (`git diff <base>...HEAD` or working tree diff). Evaluate **Standards** and **Spec Conformance** separately so clean code never excuses missed requirements, and passing tests never excuse architectural violations.

*(For multi-file or cross-crate diffs, launch two isolated parallel subagents—one for Axis A blind to the feature prompt, one for Axis B—and report both findings unblended. For small single-file edits, execute both passes sequentially.)*

### Axis A — Standards & Architecture (`backend/.ai/rules/` & `frontend-web/.ai/rules/`)
- [ ] **Hexagonal Purity (`backend/`)**: `domain/` has zero `sqlx`, `axum`, filesystem, or AWS SDK imports; `tokio::spawn` lives only in `infrastructure/` or `lambdas/*`.
- [ ] **Deep Modules & ACL**: Wire DTOs / DB rows stay separated from domain entities; Rust ports use static-dispatch RPITIT (`Send + Sync`, no unnecessary `'static` on domain traits); no shallow pass-through wrappers.
- [ ] **Clean Code (SLAP & SRP)**: Functions target **10–30 lines** (hard ceiling **40 lines**), max nesting **2 levels** via guard clauses; Newspaper top-down file ordering (`Imports` $\rightarrow$ `Types` $\rightarrow$ `Impls` $\rightarrow$ `Functions` $\rightarrow$ `Tests`).
- [ ] **Rust Type Safety & Prohibitions**: `nutype` / typed UUID newtypes (`try_new`) for domain IDs and scalars; `rust_decimal::Decimal` for money (`Money >= 0`, `SignedMoney` *(planned Sprint 3)* for signed balances; zero `f32`/`f64`); `unsafe_code = "forbid"`, `unwrap_used = "deny"`, and `expect_used = "deny"` in production code (`clippy.toml` permits `unwrap`/`expect` only inside `#[test]` / `#[cfg(test)]`); no `Deref` polymorphism; no unjustified `.clone()`; no `#![deny(warnings)]`.
- [ ] **Aurora DSQL Invariants**:
  - Migrations follow `{YYYYMMDDHHMMSS}_{description}.sql`, start with `-- no-transaction`, **1 DDL statement per file** (`IF NOT EXISTS`), no DML mixed in, `CREATE INDEX ASYNC` (no `ASC`/`DESC`), never editing applied migrations.
  - Primary keys use UUID v4 (low-frequency/tokens) or `Uuid::now_v7()` (transactions/append logs); soft-deletes (`deleted_at`); `retry_on_occ` (`40001`/`OC000`/`OC001`); `BEGIN READ ONLY` on read endpoints; `DSQL_USER` non-admin role for runtime services; keyset pagination only (zero `OFFSET`); all queries parameterized.
- [ ] **Frontend Bounded-Context & Styling Rules (`frontend-web/`)**:
  - One-way DAG (`features/*` $\rightarrow$ `shared/*` $\rightarrow$ `core/*` $\rightarrow$ `libs/ui/*`; zero cross-feature imports); every suffixless file sits inside an explicit role folder (`pages/`, `components/`, `services/`, `stores/`, `guards/`, `interceptors/`, `pipes/`, `directives/`, `models/`, `utils/`) with zero empty directories.
  - Non-trivial components/pages/dialogs use multi-file `<name>/{<name>.ts,.html,.css,.spec.ts}` with `OnPush`, signals, strict TS (zero `any`), and 3-layer styling (reusable Angular/`spartan/ui` primitives $\rightarrow$ global `@layer components` `fl-*` $\rightarrow$ component native CSS + OKLCH `var(--*)`; zero `@apply` in component CSS, zero shared `.css` `@import` inside `styleUrl`, zero `:host-context()`).

### Axis B — Spec, Domain & Security (`documentation/` & `backend/specs/`)
- [ ] **Ubiquitous Language & Invariants**: Code and identifiers match [`ubiquitous-language.md`](../../../domain/ubiquitous-language.md) (e.g., `TransactionAmendment` reversals rather than `UPDATE`, `principal + interest == amount`, `Role::Child` blocked from loans/budgets/reports, `Permission` checklist shown/sent strictly for `Role::Other`).
- [ ] **Zero-Trust & AI Guardrails (`ADR-0011`, `ADR-0018`, `ADR-0021`)**: `family_id`, `user_id`, and `role` come exclusively from verified Cognito JWT (`IdToken`, `email_verified == true`) + `family_members` membership (`X-Family-Id` omitted on identity-only routes), never from body/query/LLM args; zero LLM or JS float monetary math.
- [ ] **API, Localization & CSP Contracts**: HTTP statuses and error codes match [`api-design.md`](../../../architecture/api-design.md) and `api/src/errors.rs`; `frontend-web/` maps every `application/problem+json` `error.code` through EN/RU dictionaries without leaking raw `detail` strings, passes EN/RU key parity, complies with strict CloudFront CSP (zero external script/font CDNs), and respects the UI Mock Approval Gate.
- [ ] **Spec Traceability**: Every `Approved` rule in `backend/specs/` has a matching test in `backend/*/tests/*.rs`; `spec_traceability` passes all checks in [`backend/specs/README.md`](../../../../backend/specs/README.md); no test assertions were weakened; zero secrets committed.

---

## 3. PR & Blast-Radius Summary

When preparing a PR or final handoff summary, output three concise blocks:
1. **Structural Summary**: Minimal diagram, call path, or behavior summary (no repetitive file-by-file prose).
2. **Tiered Evidence**: Exact commands run and observed results (`cargo clippy`, `cargo test --workspace`, Docker PG integration runs if applicable).
3. **Door Classification (Blast Radius)**:
   - **One-Way Doors (High Risk — Spotlight for Human Sign-Off)**: New/altered Aurora DSQL migrations (irreversible once applied), EventBridge event schemas, Cognito/RBAC policy changes, public API breaking changes.
   - **Two-Way Doors (Reversible)**: Internal `domain/` refactors, unit/spec tests, documentation updates.
