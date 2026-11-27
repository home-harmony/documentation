# 0022. Frontend Web Module Organization, Suffixless Role Folders, and Three-Layer Styling

* **Status**: Accepted
* **Date**: 2026-11-27
* **Deciders**: Project Owner / HomeHarmony Engineering Team
* **Related**: [ADR-0004](0004-multi-repo-organization.md), [ADR-0018](0018-ai-native-modular-platform-and-sdui.md), [ADR-0021](0021-angular-spa-architecture-and-ui-library.md)

---

## Context and Problem Statement

Sprint 2 bootstrapped `frontend-web/` ([ADR-0021](0021-angular-spa-architecture-and-ui-library.md)) using a flat `core / shared / features` directory layout, `.component.ts` / `.service.ts` file suffixes with inline templates, Tailwind utility strings in HTML, and `family_service` DTOs/clients placed inside `src/app/core/api/`.

Before scaling `frontend-web/` across Sprints 3–7 (`PaymentAccount`, `Ledger`, `Debt Planner`, `Recurring Payments`, `Budget & Limits`, `SavingsGoal`, and [ADR-0018](0018-ai-native-modular-platform-and-sdui.md) SDUI cards), four structural issues must be resolved:

1. **Central `core/api` Coupling**: Placing bounded-context DTOs and API clients inside `core/api/` couples `core/` to every backend microservice, blurs lazy-chunk boundaries, and diverges from `backend/`'s DDD Bounded Contexts ([`bounded-contexts.md`](../domain/bounded-contexts.md)).
2. **Suffixless Naming vs. Role Clarity**: Adopting Angular 20+'s shorter suffixless file/class convention (`members-directory.ts` instead of `members-directory.component.ts`) becomes ambiguous if components, services, stores, guards, models, and pure utilities sit together in flat folders.
3. **Inline Template & Utility Noise on Complex Views**: Large single-file components mixing TypeScript signals, multi-state HTML control flow, and long Tailwind utility strings reduce readability and complicate diffs across dense financial forms, tables, and dialogs.
4. **Cross-Repo `.ai/` Symmetry**: Unlike `backend/` (which separates `AGENTS.md`, `.agents/skills.json`, `.ai/rules/*`, `.ai/skills/*`, and gitignored `.ai/local/`), `frontend-web/` placed all rules inline inside a single `AGENTS.md` without modular rules or Angular skills.

---

## Decision Drivers

* **DDD Bounded-Context Alignment**: Match `frontend-web/src/app/features/` to [`bounded-contexts.md`](../domain/bounded-contexts.md) with an enforced one-way import DAG.
* **Immediate Folder-Level Role Clarity**: Every suffixless file's architectural role (`pages`, `components`, `services`, `stores`, `guards`, `interceptors`, `pipes`, `directives`, `models`, `utils`) must be unmistakable from its parent directory path.
* **Pragmatic Scaling**: Create only the role subdirectories a module actually uses—zero empty placeholder folders.
* **Clean SLAP & Semantic Styles**: Separate `.ts`, `.html`, `.css`, and `.spec.ts` for non-trivial components; keep HTML semantic and avoid CSS bundle duplication across Angular component styles.
* **Unified AI Governance**: Mirror `backend/`'s `.ai/` native layout so multi-root IDE sessions auto-discover rules and skills consistently across `documentation/`, `backend/`, and `frontend-web/`.

---

## Considered Options

### Area A — Feature & Module Organization
1. **Option A1: Horizontal Layer-by-Type** (`components/`, `services/`, `models/`, `utils/` at root).
2. **Option A2: Flat Feature Slices + Central `core/api/`** (Sprint 2 initial layout).
3. **Option A3: Strict 6-Layer Feature-Sliced Design (FSD)** (`app / pages / widgets / features / entities / shared`).
4. **Option A4: Progressive DDD Bounded-Context Slices with Explicit Role Subdirectories** (`core/` + `shared/` + `features/<bounded-context>/{pages,components,services,stores,models,utils}`).

### Area B — Component Layout & Style Reuse
1. **Option B1: Single-File Components + Inline Tailwind Utility Strings Everywhere**.
2. **Option B2: Multi-File Components Using Tailwind v4 `@reference` + `@apply` or Shared Component `@import`s**.
3. **Option B3: Two-Tier Multi-File Components + Three-Layer Styling** (`spartan/ui` & shared Angular primitives $\rightarrow$ global `@layer components` `fl-*` classes in `src/styles/` $\rightarrow$ component-scoped native CSS consuming OKLCH `var(--*)` tokens).

---

## Decision Outcome

Chosen options: **Option A4 (Progressive DDD Bounded-Context Slices + Suffixless Role Folders)** and **Option B3 (Two-Tier Multi-File Components + Three-Layer Styling)**, paired with a **symmetric `frontend-web/.ai/` governance layout**.

### 1. Progressive Bounded-Context Slices & Dependency DAG

* **`libs/ui/`**: Owned `@spartan-ng/helm/*` primitives generated via `@spartan-ng/cli` (kept untouched in Spartan's single-file format; zero `src/app/` imports).
* **`src/app/core/`**: Strictly app-wide infrastructure singletons (`auth`, `config`, `context`, `errors`, `http`, `i18n`, `theme`). Contains cross-workspace session state (`WorkspaceStore`, `ActiveFamilyStore`, `GET /profile/families` bootstrap, and RBAC `Role`/`Permission` kernel types), with **zero** imports from `shared/` or `features/`.
* **`src/app/shared/`**: Reusable UI shells, cross-context pickers (`AppShell`, `TopControls`, `WorkspaceSwitcher`, `CurrencyCombobox`, future `MoneyDisplay` and `sdui/` card host), shared pipes/directives, and generic utilities, with **zero** imports from `features/`.
* **`src/app/features/<bounded-context>/`**: Lazy-loaded slices aligned to [`bounded-contexts.md`](../domain/bounded-contexts.md) (`family`, and upcoming `accounts`, `ledger`, `debt-planner`, `recurring`, `budget`, `goals`). Each context owns its wire DTOs (`models/*-dto.ts`), API client (`services/*-api.ts`), stores, pages, components, and route table (`<context>.routes.ts`). Sibling `features/*` never import from one another directly; shared read-only contracts/pickers graduate to `shared/` or `core/context/models/`.

### 2. Angular 20+ Suffixless Naming + Unambiguous Role Folders

* Drop `.component`, `.service`, `.store`, `.guard`, `.interceptor`, `.pipe`, and `.dto` dot-suffixes on filenames (`members-directory.ts`, `family-api.ts`, `workspace.ts`, `auth.ts`, `translate.ts`, `family-dto.ts`) and drop `Component` suffixes on component classes (`MembersDirectory`, `AppShell`, `InviteMemberDialog`).
* **Mandatory Role Subdirectories**: Every module under `core/<domain>/`, `shared/`, and `features/<context>/` places files inside explicit role folders so context is clear at a glance:
  * `pages/<name>/` — Routed smart page components
  * `components/<name>/` — Non-routed presentational/dialog UI components
  * `services/` — `@Injectable` stateless/API services (`family-api.ts`, `auth.ts`, `i18n.ts`, `theme.ts`)
  * `stores/` — Signal state stores (`workspace.ts`, `active-family.ts`)
  * `guards/` — Functional route guards (`CanActivateFn`)
  * `interceptors/` — Functional HTTP interceptors (`HttpInterceptorFn`)
  * `pipes/` / `directives/` — Standalone template pipes and attribute directives
  * `models/` — Pure TypeScript interfaces, wire DTOs, `InjectionToken`s, and constants (zero side effects)
  * `utils/` — Pure stateless helper functions
* **Pragmatic Scaling**: Create only the role folders that contain files; empty directories are prohibited.

### 3. Two-Tier Component Layout & Three-Layer Styling

* **Component Files**:
  * All `pages/`, forms, dialogs, tables, charts, SDUI cards, and non-trivial `components/` use a **multi-file folder**: `<name>/{<name>.ts, <name>.html, <name>.css, <name>.spec.ts}` (omitting `<name>.css` only when zero component-scoped CSS is needed).
  * Trivial stateless leaf components (`<= ~40` lines of combined TypeScript + template and zero custom CSS, e.g., `auth-logout`) may remain single-file until complexity grows.
* **Three-Layer Styling**:
  1. **Layer 1 — Behavioral/Variant Primitives**: Reusable UI with variants, DOM structure, or ARIA behavior lives in `libs/ui/*` (`spartan/ui`) or `shared/components/*`.
  2. **Layer 2 — Shared Semantic Surfaces (`src/styles/components.css` in `@layer components`)**: Cross-component visual/layout patterns use prefixed `.fl-*` classes (`.fl-standalone-shell`, `.fl-page`, `.fl-page-header`, `.fl-alert-error`, `.fl-banner-notice`, `.fl-dialog-*`, `.fl-field*`, `.fl-role-radio`, `.fl-tabular`) imported once into `src/styles.css`—compiled once with zero per-component bundle duplication and cleanly overridable by `@layer utilities`.
  3. **Layer 3 — Component-Scoped Semantic CSS (`<name>.css`)**: Component-specific layout and state rules use clean semantic class names (no verbose BEM `__`/`--` chains), modern native CSS (Grid, Flexbox, native `&` nesting, `:not()`/`:where()`, `@container`, `:host { display: block }`), and HomeHarmony's OKLCH `var(--*)` custom properties. Avoid `@reference` + `@apply` inside component CSS, never `@import` shared stylesheets across multiple component `.css` files, and never use Chromium-only `:host-context()`.

### 4. Symmetric `.ai/` Governance in `frontend-web/`

* Mirror `backend/` with `frontend-web/AGENTS.md`, `frontend-web/.agents/skills.json`, `frontend-web/.ai/rules/{frontend-angular,frontend-styling}.md`, `frontend-web/.ai/skills/{angular-patterns,angular-testing}/SKILL.md`, and gitignored `frontend-web/.ai/local/`.

### Positive Consequences

* `core/` stays small and completely decoupled from Sprints 3–7 feature growth.
* Every file path in stack traces, imports, and PR diffs immediately communicates whether a file is a page, component, service, store, guard, interceptor, model, or utility.
* Multi-file components keep templates readable and semantic while preserving `spartan/ui` OKLCH theming and avoiding Angular encapsulated CSS duplication.
* ESLint enforces module boundaries automatically on every `pnpm lint` run.

### Negative Consequences / Trade-offs

* Deeper directory nesting (typically 1–2 extra path segments per component folder).
* Global `.fl-*` classes in `src/styles/components.css` require review discipline to stay capped at genuine cross-cutting layout/financial surfaces (~15–30 classes).

---

## Compliance & Invariants

1. **ESLint Layer Boundaries**: `eslint.config.js` enforces one-way imports (`core` cannot import `shared`/`features`; `shared` cannot import `features`; `features/<ctx>` cannot import sibling `features/<other>`).
2. **Role Folder Placement**: No `.ts` source file under `src/app/{core,shared,features}` (except `<context>.routes.ts` and barrel `index.ts` files if needed) may sit outside an explicit role subdirectory.
3. **Component Style Budget**: `angular.json` enforces `anyComponentStyle` (`4 kB` warning / `8 kB` error); shared `.css` files must never be `@import`ed inside `styleUrl` stylesheets.
4. **Verification Gates**: `pnpm lint`, `pnpm test`, and `pnpm build` must pass before completing any frontend work unit.
