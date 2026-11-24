# Common Workspace Rules

Non-negotiable guardrails across both `documentation/` and `backend/` repositories.

---

## 1. Human Governance & Approval Boundaries

- **Human Owns Intent**:
  - AI agents **must NEVER mark a behavior specification rule as `Approved`** (write `Draft` only). Only humans approve rules.
  - AI agents **must NEVER mark an Architecture Decision Record (ADR) as `Accepted`** (write `Proposed` only). Only humans accept ADRs.
  - Do not implement unapproved architectural shifts without an `Accepted` ADR or explicit human instruction.
- **Specification-Driven Testing Integrity**:
  - **Blind Generation**: When generating tests from an `Approved` specification, read only the spec rules, public API signatures/docstrings, and test fixtures—never function bodies or inline tests of the unit under test.
  - **Spec–Code Mismatch**: If a spec-backed test fails against implementation code, **NEVER** edit the test assertion or spec rule to make it pass. Halt immediately and report a `SPEC–CODE MISMATCH DETECTED` for human decision.

---

## 2. Domain Vocabulary & Communication Hygiene

- **Canonical Glossary Only**: Use exact domain terms from [`ubiquitous-language.md`](../../domain/ubiquitous-language.md) (e.g., `Family Workspace`, `HomeDisplayCurrency`, `PaymentAccount`, `TransactionAmendment`, `DebtServiceCost`, `True Disposable Surplus`). Never invent synonyms for established domain concepts.
- **Plain Technical English**: Keep explanations, specs, and agent rules direct, concrete, and free of filler or pretraining trivia. Do not duplicate lookup tables or crate versions already visible in `Cargo.toml` or `README.md`.

---

## 3. Layering, Contracts & Error Schema

- **Pure Domain & ACL**: Core domain logic must stay decoupled from wire DTOs, database rows, HTTP frameworks, and AWS SDKs. Translate external payloads at boundaries via explicit mappers/codecs.
- **Canonical Error Schema & API Docs**:
  - Return structured machine-readable `code` and developer diagnostics (`Content-Type: application/problem+json` per RFC 9457, implemented in `backend/api/src/errors.rs` and `backend/api/src/auth.rs`).
  - Frontends map `error.code` to localized copy; never expose raw backend diagnostic strings as primary user-facing UI text.
  - Any endpoint, auth rule, or error code change must update [`api-design.md`](../../architecture/api-design.md) in the same work unit.

---

## 4. AI-Native & SDUI Platform Invariants (`ADR-0018`)

- **Zero-Trust Context Propagation**: `family_id`, `user_id`, and `role` must **never** be accepted from LLM tool arguments or unverified payloads—always derive them from verified Cognito JWT claims + active `X-Family-Id` membership (`ADR-0011`).
- **Zero LLM Monetary Math**: Prompts and MCP tools must never ask an LLM to sum transactions, calculate balances, or compute amortization schedules. Deterministic Rust (`rust_decimal::Decimal`) and Aurora DSQL compute all numbers.
- **Tiered Cost Funnel**: Keep routine UI actions and DB mutations at `$0.00` (Tier 0 REST / Tier 1 deterministic rules). Never trigger AWS Bedrock implicitly from database CDC events.
- **Direct REST Mutating Actions**: Mutating SDUI cards (`TransactionDraftCard`, `BudgetRebalanceCard`, `DebtScenarioCard`, `ApprovalRequestCard`) must route `primary_action` to deterministic REST endpoints.

---

## 5. Security & Verification Gates

- **Zero Secrets**: Never commit `.env*` secrets (except `.env.example`), Cognito tokens, or AWS credentials. Keep scratch notes in `.ai/local/` (gitignored).
- **Mandatory Post-Coding Review**: Before finishing any code change, run the `code-review` skill checklist and verify automated gates in `backend/`:
  - `cargo clippy --workspace --all-targets -- -D warnings`
  - `cargo test --workspace` (includes `spec_traceability`)
