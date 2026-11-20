# Technical Research Notes

This directory stores **Phase 1 factual investigation notes** produced before proposing Architecture Decision Records (ADRs) in [`decisions/`](../decisions/README.md) (see skill [`domain-and-adr`](../.ai/skills/domain-and-adr/SKILL.md)).

## Rules for Research Notes

1. **Primary Sources Only**: Cite official AWS/Rust/library documentation, RFCs, or upstream crate source code directly. Avoid second-hand blog posts when primary docs exist.
2. **Facts Only — No Opinions**: Record capabilities, hard limits, pricing, API signatures, and open questions without picking an architecture option (recommendations belong in the subsequent `Proposed` ADR).
3. **File Naming**: `{kebab-case-topic}.md` (e.g., `aurora-dsql-occ-limits.md`, `bedrock-converse-tool-schemas.md`).

---

## Research Note Template

```markdown
# Research: [Topic]

- **Date**: YYYY-MM-DD
- **Question**: What concrete engineering question are we answering?
- **Related ADR**: [Link if planned or existing]

## Findings

### Finding 1: [Sub-topic]
- Factual notes, quotes, constraints, benchmarks, or schema limits...

## Open Questions
- Unresolved item requiring prototyping or human decision

## Sources
- [Primary Source Title](url)
```
