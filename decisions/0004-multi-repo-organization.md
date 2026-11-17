# 0004. Multi-Repository Structure under Umbrella Workspace

* **Status**: Accepted
* **Date**: 2026-06-16
* **Deciders**: Core Engineering Team

## Context and Problem Statement
The Home Harmony system comprises multiple disparate codebases: backend Rust microservices, central architecture documentation, an Angular Web SPA, and a Flutter mobile application. We need an organizational structure that maintains clear Git boundaries while enabling seamless agentic pair programming across repositories.

## Decision Drivers
* Independent Git versioning, branching, and CI/CD pipelines for backend, documentation, and frontend.
* Unified workspace context for autonomous AI agents (Gemini, Claude).
* Avoidance of git submodule complexities.

## Considered Options
1. **Umbrella Workspace (`home-harmony`) with Independent Git Repositories**
2. **Single Monorepo Git Repository**
3. **Completely Disconnected Repositories with No Shared Local Parent**

## Decision Outcome
Chosen option: **Umbrella Workspace (`home-harmony`) with Independent Repositories**, where `home-harmony` acts as the root parent folder containing individual sub-repositories (`documentation/`, `backend/`, `frontend-web/`, `frontend-mobile/`), and repository-specific agent rules (such as `backend/AGENTS.md`) live inside their respective repositories.

### Positive Consequences
* AI agents can read central documentation and implement changes across backend or frontend simultaneously.
* Each sub-repository maintains self-contained agent rules (`backend/AGENTS.md`) and build configurations.
* Repositories can be pushed to separate GitHub remotes independently.

### Negative Consequences / Trade-offs
* Developers must clone/initiate sub-repos into the umbrella workspace.

## Compliance & Invariants
* Sub-repositories maintain their own agent instruction files (e.g. `backend/AGENTS.md`) alongside central architectural documentation in `documentation/`.
* Each sub-repository manages its own `.gitignore` and build files.

