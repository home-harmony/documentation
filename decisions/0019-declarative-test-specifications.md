# 0019. Declarative Behavior Specifications and AI-First Verification Workflow

* **Status**: Proposed
* **Date**: 2026-11-20
* **Deciders**: Project Owner / AI Architecture Advisor

---

## Context and Problem Statement

`FamilyLedger` (`HomeHarmony`) relies heavily on AI assistance for generating tests, documentation, and implementation slices across two separate Git repositories (`documentation/` and `backend/`). When an AI agent writes tests solely by reading existing Rust implementation bodies, it introduces **tautological test bias**: bugs in monetary arithmetic, currency enforcement, loan payment splits, or RBAC role restrictions get frozen into green assertions.

Additionally, storing AI rules or skills at the umbrella directory (`home-harmony/`) outside `documentation/` and `backend/` leaves AI infrastructure unversioned and unsynced with GitHub ([ADR-0004](0004-multi-repo-organization.md)).

We need a structured, Git-tracked Spec-Driven Development (SDD) and AI-first workflow across `documentation/` and `backend/` so that:
1. Humans retain sole ownership of intended system behavior (`Approved` specs and `Accepted` ADRs).
2. AI agents generate conformance tests **blindly** from approved specifications and public interfaces without inheriting implementation bugs.
3. Spec-to-test traceability is enforced mechanically in `cargo test`.

---

## Decision Drivers

* **Human-Owned Financial Intent**: Monetary invariants (`rust_decimal::Decimal`), `Role::Child` restrictions, and `TransactionAmendment` rules require explicit human sign-off rather than code-inferred assertions.
* **Multi-Repo Git Persistence**: 100% of rules, skills, and specs must live inside `documentation/` and `backend/` (zero untracked umbrella-root files).
* **Low Context Bloat & Zero Skill Overlap**: Skills must use progressive disclosure (`SKILL.md` + `.agents/skills.json`) and consolidate overlapping guidance into a lean set of non-redundant runbooks.
* **Native Rust Tooling**: Traceability checks must run as plain `cargo test` without heavy external BDD frameworks (e.g., Cucumber step-definition glue).

---

## Considered Options

1. **Option 1: Keep Implementation-Derived Unit Tests + Single `backend/AGENTS.md`**
   * Rely on ad-hoc prompts and inline `#[cfg(test)]` tests written alongside production code.
2. **Option 2: Gherkin / Cucumber BDD (`.feature` files + step-definition harness)**
   * Maintain `.feature` files in `documentation/` and step-definition matchers in `backend/`.
3. **Option 3: Self-Contained Declarative Markdown Specifications (`backend/specs/`, `Format T / U / P`) + Blind Test Generation + Automated Guardrails**
   * Author human-approved specs inside `backend/specs/{domain,api}/*.md` using three formats (`T` Case Table, `U` Use Case, `P` Property), generate plain Rust integration tests in `backend/<crate>/tests/*.rs`, and enforce approval guardrails and 1:1 rule coverage via `backend/domain/tests/spec_traceability.rs` with zero cross-repo CI dependencies.

---

## Decision Outcome

Chosen option: **Option 3 (Self-Contained Declarative Markdown Specifications + Blind Test Generation + Automated Guardrails)**.

### Key Architectural Elements

1. **Three Specification Formats (`backend/specs/`)**:
   * **Format T (Case Table)**: Decision matrices and state transitions (e.g., `Role` permissions, `MonthlyBudget` approval flow) $\rightarrow$ table-driven tests in `backend/domain/tests/`.
   * **Format U (Use Case)**: Multi-step application/handler flows with fakes (`Given / When / Then`) $\rightarrow$ `#[tokio::test]` tests in `backend/api/tests/` or `backend/lambdas/*/tests/`.
   * **Format P (Property)**: Universal invariants (`For all / Then / Oracle`) for value objects (`Money`, `CurrencyCode`, `FamilyName`) and financial calculators $\rightarrow$ property/boundary tests in `backend/domain/tests/`.
2. **Strict Human Governance & Automated CI Guardrails**:
   * AI agents write specs only as `Status: Draft` (tagged with `Origin: human | adr:NNNN | doc:<path> | observed`) and ADRs only as `Status: Proposed`.
   * **Only humans** change specification rules to `Status: Approved` (confirming `Origin`) or ADRs to `Status: Accepted`.
   * `spec_traceability.rs` mechanically blocks: (a) tests referencing `Draft` or `Deprecated` rules, (b) `Approved` rules still tagged `Origin: observed`, (c) `Approved` rules still listed under `## Open Questions`, and (d) `Approved` rules missing integration tests.
3. **Blind Generation & Red-Test Mismatch Protocol**:
   * When generating tests for `Approved` rules, AI reads only the spec, public Rust signatures/docstrings, and test fixtures—never function bodies.
   * If a spec-backed test fails, the AI must **never** weaken the assertion; it halts and outputs `SPEC–CODE MISMATCH DETECTED` for human decision.
4. **Self-Contained Traceability Gate**:
   * `backend/domain/tests/spec_traceability.rs` scans `backend/specs/{domain,api}/*.md` directly inside the `backend` repository so `cargo test --workspace` executes cleanly in standalone `backend` CI.
5. **Cross-Repo AI Optimization**:
   * `documentation/.ai/` houses cross-repo governance (`common.md`), project memory (`.ai/context/`), and 3 cross-repo skills (`domain-and-adr`, `spec-driven-dev`, `code-review`).
   * `backend/.ai/` houses Rust/Aurora DSQL rules (`backend-rust.md`, `aurora-dsql.md`) and 4 execution skills (`rust-patterns`, `rust-testing`, `dsql-migrations`, `diagnosing-bugs`).
   * `.agents/skills.json` in both repos cross-registers both skill folders.

### Positive Consequences
* Spec–code discrepancies surface automatically instead of being masked by implementation-derived tests.
* `backend` CI runs `spec_traceability` with zero cross-repo checkout dependencies while keeping specifications tightly paired with the code and tests they govern.
* Mechanical guardrails prevent unreviewed `Draft` or `Origin: observed` rules from slipping into conformance tests.

### Negative Consequences / Trade-offs
* Backend-specific behavior specifications live in `backend/specs/` rather than `documentation/`, while architecture docs, ADRs, and domain glossary remain in `documentation/`.
* Requires human discipline to review `Draft` rules, resolve `Open Questions`, and confirm `Origin` before setting `Status: Approved` (assisted by CI guardrails).

---

## Compliance & Invariants

* `cargo test -p domain --test spec_traceability` enforces metadata validity, approval guardrails, and 1:1 rule ID presence across `backend/specs/` and `backend/*/tests/`.
* Detailed authoring, review (`Q1–Q10`), and test-generation rules are maintained in [backend/specs/GUIDELINES.md](../../backend/specs/GUIDELINES.md) and [documentation/ai-development-guide.md](../ai-development-guide.md).
