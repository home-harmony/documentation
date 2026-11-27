# AI-First & Spec-Driven Development Guide — HomeHarmony

Practical handbook for working effectively with AI agents across the `documentation/`, `backend/`, and `frontend-web/` repositories using our **9 optimized skills**, modular **rules**, **declarative specifications**, and built-in Antigravity slash commands.

---

## 1. How the System Fits Together

To keep every customization tracked on GitHub across our multi-repository umbrella ([ADR-0004](./decisions/0004-multi-repo-organization.md)), AI configuration lives inside `documentation/`, `backend/`, and `frontend-web/`—never at the untracked `home-harmony/` parent folder. **IDE Setup**: open `documentation/`, `backend/`, and `frontend-web/` together as a **multi-root workspace** rather than opening the parent `home-harmony/` directory; otherwise Antigravity cannot walk downward to discover each repo's `AGENTS.md` or `.agents/skills.json`.

| Layer | Purpose | Location (`documentation/` + `backend/` + `frontend-web/`) | When Loaded |
| :--- | :--- | :--- | :--- |
| **Entry Points** | Orientation, reading order, skill routing, mandatory gates | [`documentation/AGENTS.md`](./AGENTS.md), [`../backend/AGENTS.md`](../backend/AGENTS.md), [`../frontend-web/AGENTS.md`](../frontend-web/AGENTS.md) | Always at session start |
| **Rules** | Short, non-negotiable invariants & prohibitions | [`documentation/.ai/rules/common.md`](./.ai/rules/common.md), `backend/.ai/rules/{backend-rust,aurora-dsql}.md`, `frontend-web/.ai/rules/{frontend-angular,frontend-styling}.md` | Always when working in scope |
| **Skills (9)** | On-demand step-by-step workflows & checklists | `documentation/.ai/skills/*`, `backend/.ai/skills/*`, `frontend-web/.ai/skills/*` (auto-discovered via `.agents/skills.json`) | Loaded only when a task matches |
| **Specs & Memory** | Human-approved truth & chronological decisions | [`../backend/specs/`](../backend/specs/README.md), [`documentation/.ai/context/`](./.ai/context/project.md) | Referenced per task |

---

## 2. Quick-Reference Matrix: Which Skill for Which Task?

| Task Type | Primary Skill(s) / Built-ins | Repo(s) Touched | Artifacts Produced |
| :--- | :--- | :--- | :--- |
| **Design a new feature, clarify edge cases, update glossary, or draft an ADR** | `domain-and-adr` (+ `/grill-me`) | `documentation/` | [`ubiquitous-language.md`](./domain/ubiquitous-language.md), [`research/`](./research/README.md), [`decisions/`](./decisions/README.md) (`Proposed`) |
| **Specify behavior rules or break an epic into vertical slices** | `spec-driven-dev` | `backend/` | [`backend/specs/{domain,api}/*.md`](../backend/specs/README.md) (`Draft`), vertical slice list |
| **Implement a feature, aggregate, port, or handler in Rust** | `rust-patterns` + `rust-testing` | `backend/` | Rust code (`domain/`, `infrastructure/`, `api/`, `lambdas/*`) + `tests/*.rs` |
| **Add an Aurora DSQL table, async index, or SQLx query** | `dsql-migrations` | `backend/` | `migrations/{TS}_{desc}.sql` + updated `.sqlx/` offline cache |
| **Debug a failing test, OCC retry (`40001`), or pagination/Decimal bug** | `diagnosing-bugs` | `backend/` | Minimal failing repro test $\rightarrow$ root-cause fix $\rightarrow$ regression test |
| **Implement an Angular bounded-context view, dialog, store, or CSS layout** | `angular-patterns` + `angular-testing` | `frontend-web/` | Interactive UI mock $\rightarrow$ suffixless role-folder Angular slice + Vitest specs |
| **Review code, check spec coverage, or prepare a PR** | `code-review` | All touched | Two-axis audit report + automated repo gates + One-Way Door summary |
| **Explore multi-step financial math or SDUI cards visually** | Built-in `generative_ui` | Chat / Artifacts | Interactive HTML state-machine / UI widget |
| **Long sessions, big plans, or capturing new lessons** | Built-in `/plan`, `/compact`, `/learn` | Session / Config | Plan review, compacted context, persistent memory |

---

## 3. Step-by-Step Playbooks by Task Type

### Playbook A: Designing a New Feature, Domain Rule, or ADR
*Use `domain-and-adr` (and `/grill-me` when exploring raw ideas).*

1. **Align with the codebase first**: Ask the agent to explore existing docs and run frontier-round questions with `➡️` recommended defaults so you only answer unblocked decisions.
2. **Keep terminology crisp**: The agent challenges overloaded words and updates [`ubiquitous-language.md`](./domain/ubiquitous-language.md) immediately.
3. **Research before deciding (if external tech/AWS limits apply)**: The agent writes facts-only notes from primary sources into [`documentation/research/`](./research/README.md).
4. **Draft ADR if the 3-Part Gate passes** (*Hard to reverse*, *Surprising without context*, *Real trade-off*): The agent drafts `documentation/decisions/NNNN-title.md` with `* **Status**: Proposed`.
5. **Human sign-off**: Review the draft and tell the agent to mark it `Accepted` (which logs a summary into [`documentation/.ai/context/decisions.md`](./.ai/context/decisions.md)).

**Copy-Paste Prompt Recipe:**
> *"Using `domain-and-adr`, read `bounded-contexts.md` and `ubiquitous-language.md` and grill me in frontier rounds about [feature/topic]. If an external AWS/crate question comes up, write a research note first; if we settle an architectural trade-off, draft a Proposed ADR."*

---

### Playbook B: Writing & Approving Behavior Specifications (Spec-Driven Development)
*Use `spec-driven-dev` before coding new domain or API behavior.*

1. **Agree on Seams first**: Confirm which test layer verifies the behavior (`Layer 1` pure domain, `Layer 2a` Testcontainers PG, or `Layer 2b` Axum `oneshot`).
2. **Draft rules (`Status: Draft`)**: The agent picks the right format (**T** Case Table, **U** Use Case, **P** Property) and drafts rules tagged with `Origin` (`human`, `adr:NNNN`, `doc:...`, or `observed`).
3. **Run AI Pre-Review (`Q1–Q10`)**: Ask the agent to emit a `blocker / major / minor` findings table before you review.
4. **Slice big specs**: For multi-crate epics, ask `spec-driven-dev` to split the spec into vertical tracer bullets (`migration -> domain -> adapter -> handler -> test`).
5. **Human Approval**: **You** edit `Status: Draft` $\rightarrow$ `Status: Approved` and confirm `Origin` (replacing `observed`, and clearing any linked `Open Questions`). CI enforces this automatically via `spec_traceability`.

**Copy-Paste Prompt Recipes:**
> *"Use `spec-driven-dev` to draft a Format T/P spec in `backend/specs/domain/` for [aggregate/feature]. Confirm the test seam with me first, and leave all rules as `Draft`."*
> *"Review `backend/specs/domain/<file>.md` against Q1–Q10 and list any blockers or ambiguities without changing rule statuses."*

---

### Playbook C: Implementing a Feature or Vertical Slice in Rust
*Use `rust-testing` + `rust-patterns` (+ `spec-driven-dev` for blind spec tests).*

1. **Generate Spec Tests Blindly (for `Approved` rules)**: Ask the agent to translate `Approved` rules into `backend/<crate>/tests/<prefix>_<topic>.rs` without peeking at implementation bodies.
   - If a test fails against existing code, expect a **`SPEC–CODE MISMATCH DETECTED`** halt—never let the agent weaken an assertion. Decide whether the code or the spec is wrong.
2. **Run Inner TDD Loop at Seams**: For new code, `rust-testing` confirms seams, writes one failing test (`Red`), writes minimal code (`Green`), and refactors (`Refactor`).
3. **Apply Clean Rust Architecture**: `rust-patterns` enforces Deep Modules (Deletion Test), SLAP (`10–30` lines, max depth `2`), Newspaper file ordering, `nutype` value objects, `rust_decimal::Decimal`, and RPITIT static-dispatch ports.
4. **Verify**: Run `cargo test -p domain --test spec_traceability`, `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`.

**Copy-Paste Prompt Recipes:**
> *"Use `spec-driven-dev` and `rust-testing` to generate tests for the Approved rules in `backend/specs/domain/<file>.md` into `backend/domain/tests/`. Do not read function bodies; report any spec-code mismatch instead of editing assertions."*
> *"Implement [vertical slice] using `rust-patterns` and `rust-testing`. Confirm the test seams first, then work one Red-Green-Refactor behavior slice at a time."*

---

### Playbook D: Creating or Modifying Aurora DSQL Migrations & Queries
*Use `dsql-migrations` whenever touching `backend/migrations/` or `sqlx` repository code.*

1. **Author Migrations Safely**: Enforces `{YYYYMMDDHHMMSS}_{description}.sql`, **strictly 1 DDL statement per file**, `CREATE INDEX ASYNC`, UUID v4/v7 PKs, `deleted_at` soft-deletes, and zero edits to already-applied migrations.
2. **Author Queries Safely**: Enforces `retry_on_occ` on mutations, `BEGIN READ ONLY` on reads, `WHERE family_id = $1`, and keyset cursor pagination (zero `OFFSET`).
3. **Verify**: Run `cargo test -p infrastructure -- --ignored` (with Docker running) and refresh offline metadata (`cargo sqlx prepare --workspace -- --all-targets`).

**Copy-Paste Prompt Recipe:**
> *"Use `dsql-migrations` to add the migration(s) and SQLx repository methods for [table/query], keeping strictly one DDL statement per file and keyset pagination."*

---

### Playbook E: Debugging Failing Tests, OCC Retries, or Math Bugs
*Use `diagnosing-bugs` instead of letting an agent guess-edit code.*

1. **Phase 1 Hard Gate**: Agent must run a single deterministic failing command (`cargo test ...`) before forming theories.
2. **Minimize & Hypothesize**: Agent shrinks the failing test input, writes 3–5 falsifiable hypotheses, and probes one variable at a time.
3. **Permanent Regression Test**: Converts the minimal repro into a permanent test at the proper seam (`Layer 1` / `2a` / `2b`), applies the minimal fix, verifies Green, and cleans up debug instrumentation.

**Copy-Paste Prompt Recipe:**
> *"Use `diagnosing-bugs` to investigate why `[test/endpoint]` fails on `[symptom]`. Get a deterministic red `cargo test` command first before proposing or applying any edits."*

---

### Playbook F: Reviewing Code & Preparing a Pull Request
*Use `code-review` automatically after coding or before committing/merging.*

1. **Automated Gates**: Runs `cargo clippy --workspace --all-targets -- -D warnings` and `cargo test --workspace` (including `spec_traceability`).
2. **Two-Axis Audit**: Evaluates **Axis A (Standards & Architecture)** separately from **Axis B (Spec, Domain & Security)** so neither masks the other.
3. **PR & Door Summary**: Produces a concise structural diagram/summary, exact test outputs, and spotlights any **One-Way Doors** (e.g., Aurora DSQL migrations, EventBridge contracts, RBAC rules).

**Copy-Paste Prompt Recipe:**
> *"Run `code-review` against our current diff in `backend/` (and any doc changes in `documentation/`), checking both Standards and Spec axes and flagging any One-Way Doors."*

---

### Playbook G: Interactive Prototyping (State Machines & SDUI Cards)
*Use Antigravity's built-in `generative_ui` skill.*

- When stress-testing complex multi-step financial logic (e.g., Avalanche/Snowball payoff rollovers, `CrossCurrencyTransfer` FX spreads, or `ADR-0018` SDUI action cards), ask the agent to render an interactive widget via `generative_ui` so you can click through state transitions before writing specs or Rust code.

**Copy-Paste Prompt Recipe:**
> *"Before we write Rust code, build an interactive state-machine widget with `generative_ui` so I can test edge cases of [Loan payoff rollover / TransactionAmendment / SDUI card]."*

---

### Playbook H: Managing Long Sessions & Evolving Rules
*Use built-in Antigravity slash commands.*

- **`/plan`**: Switch on step-by-step planning before large multi-crate changes.
- **`/compact`**: Free context window tokens mid-session without losing key workspace state.
- **`/learn`**: Persist recurring corrections or hard-won workflow fixes across future sessions.
- **Human Governance Reminder**: Periodically review [`backend/specs/`](../backend/specs/README.md) to resolve open questions, replace `Origin: observed` with confirmed intent, and promote validated `Draft` rules to `Approved`—`spec_traceability` enforces all checks listed in [`backend/specs/README.md`](../backend/specs/README.md).
