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

---

## Planned Post-MVP Investigations

- **STRIDE Threat Modeling**: End-to-end threat analysis across Cognito JWT + `X-Family-Id` tenant boundaries, Aurora DSQL IAM auth, EventBridge, Angular SPA (CSP/PKCE), and future Bedrock/MCP surfaces (`ADR-0018`).
- **GDPR & Privacy Compliance (incl. Child Protection)**: EU GDPR Article 8 parental consent for `Role::Child`, minor-profile data minimization, right to erasure (`DSAR` anonymization/crypto-shredding vs. immutable ledger `ADR-0012`), portability export, and EU residency.
