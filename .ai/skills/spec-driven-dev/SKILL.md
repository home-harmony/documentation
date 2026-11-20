---
name: spec-driven-dev
description: >-
  Use when drafting or reviewing declarative behavior specifications in
  backend/specs/, slicing roadmap epics into vertical tracer-bullet tasks,
  generating blind spec-backed Rust tests, or handling SPEC-CODE MISMATCH failures.
---

# Spec-Driven Development & Vertical Slicing

Bridges settled domain design (`domain-and-adr`) and verified Rust execution (`rust-testing`) via declarative specifications in [`backend/specs/README.md`](../../../../backend/specs/README.md) ([`GUIDELINES.md`](../../../../backend/specs/GUIDELINES.md)).

---

## 1. Seams Before Prose (Pre-Spec Alignment)

Before writing a specification or slicing a feature:
1. Read settled design docs (`ubiquitous-language.md`, ADRs, `api-design.md`, `database-schema.md`).
2. Identify and confirm the **testing seams** with the user using [`testing-strategy.md`](../../../testing/testing-strategy.md):
   - **Layer 1 Seam (`domain`)**: Pure aggregates, `nutype` value objects, `Money` arithmetic, `RepaymentPlanCalculator` in `../backend/domain/tests/`.
   - **Layer 2a Seam (`infrastructure`)**: Repository traits (`RPITIT`) verified against Testcontainers PostgreSQL in `../backend/infrastructure/tests/`.
   - **Layer 2b Seam (`api` / `lambdas/*`)**: Axum handlers via `tower::ServiceExt::oneshot()` + fakes/`SpyPublisher` in `../backend/api/tests/` or `../backend/lambdas/<service>/tests/`.

---

## 2. Spec Format Selection & Drafting Protocol

| Behavior Type | Format | Spec Directory | Test Target |
| :--- | :--- | :--- | :--- |
| State machines, permission/decision matrices | **T (Case Table)** | `backend/specs/domain/` | `backend/domain/tests/` |
| Value-object validation, universal invariants | **P (Property)** | `backend/specs/domain/` | `backend/domain/tests/` |
| Multi-step use cases, authorization & API flows | **U (Use Case)** | `backend/specs/api/` | `backend/api/tests/` or `lambdas/*/tests/` |

### Drafting Steps (AI Agent)
1. Register or verify a unique 2–4 uppercase prefix (`VO`, `FM`, `LD`, `LN`, `BG`, `AU`) in [`backend/specs/README.md`](../../../../backend/specs/README.md).
2. Follow [`backend/specs/TEMPLATE.md`](../../../../backend/specs/TEMPLATE.md) (`Test seam` metadata header, `Vocabulary`, `Rules`, `Open Questions`, `Findings`).
3. Every rule MUST include:
   - **ID**: `<PREFIX>-<NN>` (never reused, never renumbered).
   - **Status**: Always **`Draft`** (only humans may set `Approved`).
   - **Origin**: `human`, `adr:NNNN`, `doc:<path>`, or `observed` (inferred from existing code/tests; `spec_traceability` blocks `Approved` rules that still have `Origin: observed`).
   - **Concrete inputs, single action, observable outcome** (exact `DomainError` variant, public state, or HTTP status/error code).
4. Record any doc/code/test contradictions in **Findings** and add **Open Questions** rather than guessing.

---

## 3. Spec Pre-Review (Quality Criteria Q1–Q10)

When asked to review a `Draft` spec before human approval:
1. Check Q1–Q10 from [`GUIDELINES.md`](../../../../backend/specs/GUIDELINES.md) (Atomic, Observable, Concrete boundaries, Closed vocabulary, Intent not implementation, Complete state, Explicit min/max/empty boundaries, Negative cases, No contradictions, Not redundant with Rust type safety).
2. Output a review table **without changing rule statuses or editing spec rules**:
   `| ID | Severity (blocker/major/minor) | Check | Finding | Suggested Resolution |`

---

## 4. Vertical Tracer-Bullet Slicing (Large Specs & Epics)

When breaking a large spec or sprint roadmap item (e.g., [`sprint-roadmap.md`](../../../roadmap/sprint-roadmap.md)) into executable units:
- **Vertical Slices Only**: Never slice by horizontal layers (*"write all migrations, then all structs, then all handlers"*). Every slice must deliver one end-to-end testable behavior (`1-DDL migration(s)` $\rightarrow$ `domain` invariant $\rightarrow$ `infrastructure` adapter $\rightarrow$ `api`/handler $\rightarrow$ spec test) small enough for one focused session.
- **Explicit Blockers & Prefactoring**: Order structural prefactoring ahead of dependent slices and state explicit blocking dependencies (`blocked-by`) between slices.
- **Confirm Slice Plan**: Present the numbered vertical slice graph for human approval before coding.

---

## 5. Blind Test Generation & Mismatch Protocol

When generating tests for **`Approved`** rules only (ignore `Draft` and `Deprecated`):
1. **Context Isolation (Blindness)**:
   - **Allowed**: Approved rules + Vocabulary in `backend/specs/**`, public API signatures/docstrings (`pub` items in `backend/*/src/`), `test_support` fixtures/fakes.
   - **Forbidden**: Do NOT read function implementation bodies or internal `#[cfg(test)]` blocks of the unit under test. To stay blind in Rust (where signatures and bodies share a file), inspect `cargo doc --no-deps` output or delegate `pub` signature/docstring extraction to a separate read-only subagent. (If tooling cannot isolate bodies, explicitly note in the report that blindness was constrained so the human reviews carefully.)
2. **Implement 1:1 Traceable Tests**: Place tests in `<crate>/tests/<prefix_lowercase>_<topic>.rs` containing the exact rule ID (`"VO-01"` or `#[test] fn vo_01_...`). Never generate tests for `Draft` or `Deprecated` rules (blocked by `spec_traceability`).
3. **Verify Traceability**: Run `cargo test -p domain --test spec_traceability` in `../backend/` (enforces the 4 checks documented in [`backend/specs/README.md`](../../../../backend/specs/README.md)).
4. **Red-Test Mismatch Halt**: If a faithful spec test fails against existing code, **DO NOT** weaken the assertion or edit the spec rule. Stop and emit:
   ```markdown
   ### SPEC–CODE MISMATCH DETECTED
   - **Rule ID**: <PREFIX-NN> (<Title>)
   - **Spec File**: backend/specs/...
   - **Expected (from Spec)**: <Exact Then condition>
   - **Actual (from Execution)**: <Observed return/error/state>
   - **Failing Test**: <crate>/tests/<file>::<test_fn>
   - **Awaiting Decision**: Human must decide whether implementation or spec needs revision.
   ```
