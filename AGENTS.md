# AGENTS.md — HomeHarmony (`documentation/` & Umbrella Entry Point)

> Primary entry point for AI agents working in `documentation/` or across the `home-harmony/` repositories.
> **Two-Repo Rule ([ADR-0004](./decisions/0004-multi-repo-organization.md))**: `documentation/` and `backend/` are separate Git repositories. Never write project files at the untracked `home-harmony/` root; open `documentation/` and `backend/` as a **multi-root workspace** so both repos' `AGENTS.md` and `.agents/skills.json` auto-discover cleanly.

---

## 1. Reading Order (Cold Start)

1. [`.ai/context/project.md`](./.ai/context/project.md) — 1-paragraph architecture & domain seed
2. [`README.md`](./README.md) — full documentation portal index
3. [`domain/ubiquitous-language.md`](./domain/ubiquitous-language.md) — canonical glossary, formulas, and 10 domain invariants
4. [`domain/bounded-contexts.md`](./domain/bounded-contexts.md) — 9 bounded contexts & aggregate boundaries
5. [`architecture/system-overview.md`](./architecture/system-overview.md) & [`architecture/api-design.md`](./architecture/api-design.md)
6. [`ai-development-guide.md`](./ai-development-guide.md) — developer playbook for using skills per task type

When working in Rust code or migrations, also read [`../backend/AGENTS.md`](../backend/AGENTS.md).

---

## 2. Rules & Skills Index

### A. Modular Rules (Always Enforced)
@[Common Workspace Rules](./.ai/rules/common.md)

Backend-specific rules (auto-included by [`../backend/AGENTS.md`](../backend/AGENTS.md)):
- **Backend Rust Rules**: [`../backend/.ai/rules/backend-rust.md`](../backend/.ai/rules/backend-rust.md)
- **Aurora DSQL Rules**: [`../backend/.ai/rules/aurora-dsql.md`](../backend/.ai/rules/aurora-dsql.md)

### B. Optimized Skills (7 Total, Auto-Discovered via `.agents/skills.json`)

| Scope | Task | Skill (`SKILL.md`) |
| :--- | :--- | :--- |
| **Cross-Repo** | Align requirements, update glossary, primary-source research, or draft an ADR | [`domain-and-adr`](./.ai/skills/domain-and-adr/SKILL.md) |
| **Cross-Repo** | Draft/review `Draft` specs (T/U/P), slice epics vertically, generate blind tests | [`spec-driven-dev`](./.ai/skills/spec-driven-dev/SKILL.md) |
| **Cross-Repo** | Post-coding 2-axis self-review (Standards vs. Spec) & One-Way Door PR check | [`code-review`](./.ai/skills/code-review/SKILL.md) |
| **Backend** | Design or refactor clean Rust modules (`nutype`, RPITIT, SLAP, Newspaper layout) | [`../backend/.ai/skills/rust-patterns/SKILL.md`](../backend/.ai/skills/rust-patterns/SKILL.md) |
| **Backend** | Run Red-Green-Refactor TDD & spec tests at Layer 1 / 2a / 2b hexagonal seams | [`../backend/.ai/skills/rust-testing/SKILL.md`](../backend/.ai/skills/rust-testing/SKILL.md) |
| **Backend** | Author Aurora DSQL migrations (`1 DDL/file`), async indexes, OCC/keyset SQLx queries | [`../backend/.ai/skills/dsql-migrations/SKILL.md`](../backend/.ai/skills/dsql-migrations/SKILL.md) |
| **Backend** | Reproduce and isolate bugs scientifically (*"no red test, no theories"*) | [`../backend/.ai/skills/diagnosing-bugs/SKILL.md`](../backend/.ai/skills/diagnosing-bugs/SKILL.md) |

*(For interactive financial/SDUI prototypes, use Antigravity's built-in `generative_ui` skill.)*

---

## 3. Memory & Spec Artifacts

- **Persistent Context**: [`.ai/context/project.md`](./.ai/context/project.md) · [`.ai/context/decisions.md`](./.ai/context/decisions.md)
- **Research Spikes (Facts Only)**: [`research/`](./research/README.md)
- **Architecture Decision Records**: [`decisions/`](./decisions/README.md) (ADRs `0001`–`0019`)
- **Behavior Specifications**: [`../backend/specs/`](../backend/specs/README.md) (`VO`, `FM`, `LD`, `LN`, `BG`, `AU`)

---

## 4. Mandatory Workflow Gates

- **Human Owns Intent**: AI agents write specs only as `Status: Draft` and ADRs only as `Status: Proposed`. Never set `Approved` or `Accepted` autonomously.
- **Blind Spec Testing & Mismatch Halt**: Generate tests from `Approved` specs without reading function bodies. If a faithful spec test fails, halt and emit `SPEC–CODE MISMATCH DETECTED`—never weaken assertions.
- **Mandatory Self-Review**: Run [`code-review`](./.ai/skills/code-review/SKILL.md) and pass `cargo clippy --workspace --all-targets -- -D warnings` + `cargo test --workspace` (including `spec_traceability`) in `../backend/` after code changes.
- **API & Domain Sync**: Keep [`architecture/api-design.md`](./architecture/api-design.md) and [`domain/ubiquitous-language.md`](./domain/ubiquitous-language.md) synchronized with code changes.
- **Local Scratch**: Put temporary personal notes in `.ai/local/` (gitignored).
