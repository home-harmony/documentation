---
name: domain-and-adr
description: >-
  Use when designing new features, clarifying ambiguous requirements, maintaining
  domain terminology in documentation/domain/ubiquitous-language.md, researching
  technical options from primary sources into documentation/research/, or
  drafting/updating Architecture Decision Records (ADRs) in documentation/decisions/.
---

# Domain Alignment, Research & ADR Workflow

Unifies pre-spec discovery (frontier grilling + ubiquitous language enforcement) with primary-source research and the 4-phase MADR decision pipeline.

---

## 1. Frontier-Round Grilling (Design Alignment)

When requirements or design choices are ambiguous:
1. **Inspect Codebase & Docs First**: Read [`ubiquitous-language.md`](../../../domain/ubiquitous-language.md), [`bounded-contexts.md`](../../../domain/bounded-contexts.md), relevant [`decisions/`](../../../decisions/README.md), and `../backend/domain/` before asking anything the repo already answers.
2. **Build a Decision Tree & Compute the Frontier**: Identify root architectural/domain choices and dependent sub-questions. Never bundle a dependent question alongside its prerequisite.
3. **Batch One Unblocked Frontier Round at a Time**: Present all currently unblocked questions together with a concrete recommended default (`➡️`):
   ```markdown
   ❓ **Q1 — <Short Title>**: <Question, options, and trade-offs>
   ➡️ <Recommended default>
   ```
4. **Stop Until Settled**: Wait for user answers (`"yes"` accepts defaults), prune/expand the tree, and iterate until the frontier is empty.

---

## 2. Ubiquitous Language & Bounded-Context Guard

During every design discussion, enforce [`ubiquitous-language.md`](../../../domain/ubiquitous-language.md) and [`bounded-contexts.md`](../../../domain/bounded-contexts.md):
- **Challenge Overloaded Terms**: If a prompt uses vague terms (e.g., `"account"` without specifying `PaymentAccount` vs. `Family Workspace`, or `"update transaction"` instead of `TransactionAmendment` reversal), stop and resolve to canonical terminology.
- **Inline Domain Updates**: As soon as a new domain concept, formula, or invariant is agreed upon, update [`ubiquitous-language.md`](../../../domain/ubiquitous-language.md) immediately:
  - Keep glossary definitions strictly implementation-free (no SQL types, Rust crate paths, or HTTP verbs).
  - Map the term to one owning Bounded Context in [`bounded-contexts.md`](../../../domain/bounded-contexts.md).

---

## 3. The 3-Part ADR Gate

Only propose a new ADR when a decision satisfies **all three criteria**:
1. **Hard to reverse** (database schema contracts, cross-service EventBridge payloads, security/auth boundaries, crate/module splits).
2. **Surprising without context** (a future engineer would ask *"Why was it built this way?"*).
3. **Result of a real trade-off** (at least two viable alternatives were weighed and rejected).

---

## 4. Four-Phase Research → ADR Pipeline

```text
1. INVESTIGATE  ──►  documentation/research/{kebab-case-topic}.md  (Facts only, primary sources)
2. PROPOSE      ──►  documentation/decisions/{NNNN}-{short-title}.md  (Status: Proposed)
3. DECIDE       ──►  Human reviews & approves  ──►  Status: Accepted
4. LOG          ──►  documentation/.ai/context/decisions.md + decisions/README.md index
```

> **GOVERNANCE INVARIANT**: Never set an ADR status to `Accepted` autonomously, and never implement unapproved architectural changes before human acceptance.

### Phase 1 — Investigate (`documentation/research/`)
- Consult **primary sources only** (official AWS docs, RFCs, crate docs/source code) and record factual findings using [`documentation/research/README.md`](../../../research/README.md).
- Include zero opinions or architecture picks in research notes.

### Phase 2 — Propose (`documentation/decisions/`)
- Check [`documentation/decisions/README.md`](../../../decisions/README.md) for the next 4-digit number (`0019`, `0020`, ...).
- Follow the MADR template (`Context and Problem Statement`, `Decision Drivers`, `Considered Options` $\ge 2$, `Decision Outcome`, `Positive/Negative Consequences`, `Compliance & Invariants`).
- Set `* **Status**: Proposed`, link any research note, and add the row to [`documentation/decisions/README.md`](../../../decisions/README.md). Present the draft and stop for human decision.

### Phase 3 & 4 — Decide & Log
- Once the user explicitly approves (`"accept"`, `"go with option 2"`), update `Status` to `Accepted` (or mark superseded ADRs `Superseded by ADR-NNNN`) and append a concise summary entry to [`documentation/.ai/context/decisions.md`](../../context/decisions.md).
