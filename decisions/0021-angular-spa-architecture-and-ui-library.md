# 0021. Angular Web SPA Architecture, UI Component Library, and Cognito Auth Flow

* **Status**: Accepted
* **Date**: 2026-11-26
* **Deciders**: Project Owner / HomeHarmony Engineering Team
* **Research**: [Angular UI Libraries for FamilyLedger: Five Newer Candidates](../research/Angular%20UI%20Libraries%20for%20FamilyLedger%20Five%20Newer%20Candidates.md)

---

## Context and Problem Statement

With the Sprint 2 backend (`family_service`, multi-family `X-Family-Id` auth middleware, and 3-stack Option B2 AWS SAM deployment) complete, **FamilyLedger** is ready to build its Angular Web SPA in the sibling `frontend-web/` repository ([ADR-0004](0004-multi-repo-organization.md)).

Per [ADR-0018](0018-ai-native-modular-platform-and-sdui.md), the Angular Web SPA serves both as the full-parity family financial hub (rich forms, tables, date pickers, multi-currency analytics charts) and the **Admin Control Plane** (RBAC governance, member permissions, future categorization rules and AI budget settings). Meanwhile, `template.yaml` already enforces strict production constraints on the SPA:
* **CloudFront CSP**: `script-src 'self'; style-src 'self' 'unsafe-inline'; connect-src 'self' https://*.execute-api.*.amazonaws.com https://cognito-idp.*.amazonaws.com https://*.amazoncognito.com; frame-ancestors 'none'`.
* **API Gateway JWT Authorizer**: Requires the **Cognito `IdToken`** (`aud` == Web Client ID, `email_verified == "true"`) in `Authorization: Bearer <id_token>`, plus `X-Family-Id` for all family-scoped routes ([ADR-0011](0011-multi-family-membership.md)).
* **Error Localization**: Backend returns RFC 9457 `application/problem+json` with machine-readable `error.code`; the frontend must never expose raw backend diagnostic strings as primary user-facing text ([`api-design.md`](../architecture/api-design.md)).

We need to decide the foundational UI component library, Cognito authentication UX, and localization architecture before scaffolding `frontend-web/`.

---

## Decision Drivers

* **Aesthetic & Usability**: Classical, minimalist, low-noise visual design suited for a calm family financial workspace.
* **Small Download Size**: Fast load over S3 + CloudFront (`PriceClass_100`) by bundling only used primitives.
* **Full Component Coverage**: Rich searchable dropdowns/comboboxes, data tables, date pickers, dialogs, sidebars, and charts (needed for Sprints 3–5).
* **Modern Angular Alignment**: Signals-first, standalone components, zoneless-ready, strong accessibility (ARIA, keyboard navigation, focus management).
* **Long-Term Maintenance Stability**: Avoid sub-1.0 libraries likely to break repeatedly over a multi-year personal finance app.
* **Security & Fast Sprint 2 Delivery**: Support both Cognito email/password and optional Google OIDC federation (`FamilyLedgerGoogleIdp`) with zero password handling in the SPA and strict CSP compliance.

---

## Considered Options

1. **Option 1: `spartan/ui` 1.0 + Tailwind CSS + Cognito Hosted UI (OAuth 2.0 PKCE) + EN/RU i18n**
   * `@spartan-ng/brain` (Angular CDK-backed accessible primitives) + copied Tailwind `helm` styles owned in `frontend-web/`.
   * Cognito Hosted UI with OAuth 2.0 Authorization Code + PKCE (`/auth/callback`, `/auth/logout`).
   * EN + RU localization wired from Sprint 2 with a typed RFC 9457 `error.code` dictionary.
2. **Option 2: Newer Lightweight Angular Libraries (`Guild of Gleks UI`, `ng-oat`, `Frakton NG`, `eagami/ui`, `Revo UI`)**
   * Zero-dependency or minimal CSS-variable libraries evaluated in the [research spike](../research/Angular%20UI%20Libraries%20for%20FamilyLedger%20Five%20Newer%20Candidates.md).
3. **Option 3: Heavyweight Enterprise Suites (`Angular Material` / `PrimeNG` / `Taiga UI`) + Custom SRP Auth Forms**
   * Mature full-featured component suites paired with custom Angular sign-up, verification, password-reset, and MFA screens using `ALLOW_USER_SRP_AUTH`.

---

## Decision Outcome

Chosen option: **Option 1 — `spartan/ui` 1.0 + Tailwind CSS + Cognito Hosted UI (OAuth 2.0 PKCE) + EN/RU i18n**, because:

1. **UI Component Library (`spartan/ui` 1.0)**:
   * None of the five newer candidates documents a chart component, and all are sub-1.0 or unverified with very small adoption.
   * `spartan/ui` 1.0 (MIT, Thoughtworks Radar "Assess", 55+ components) delivers the classical low-noise shadcn aesthetic, small copy-only-what-you-use bundles, signals/standalone/zoneless support, and coverage of Select, Combobox, Table, Data Table, Date Picker, Calendar, Chart, Dialog, and Sidebar.
   * **Slice 0 Verification Gate**: Before Sprint 3–5 analytics work, Sprint 2 Slice 0 will verify what `spartan/ui`'s `Chart` component uses underneath, how `Data Table` handles large datasets, and baseline `ng build` bundle sizes.
2. **Authentication UX (Cognito Hosted UI — OAuth 2.0 Authorization Code + PKCE)**:
   * Immediately supports both Cognito email/password (including 12-char password policy, email verification, and optional TOTP MFA) and Google OIDC federation (`FamilyLedgerGoogleIdp`) already provisioned in `template.yaml`.
   * Avoids building and maintaining custom sign-up, email-code confirmation, forgot-password, and MFA UI flows in Sprint 2.
3. **Localization & Error Boundary (EN + RU + Typed `error.code` Dictionary)**:
   * Aligns with existing bilingual documentation (`README.md` + `README.ru.md`) and enforces the RFC 9457 localization contract from day one.

### Positive Consequences

* Bundle size grows strictly with the components actually copied into `frontend-web/`.
* Full visual ownership of `helm` Tailwind components makes future SDUI action cards ([ADR-0018](0018-ai-native-modular-platform-and-sdui.md)) consistent with hand-built screens.
* Zero credential/password handling in Angular; seamless support for Google OAuth account linking via `CognitoPreSignUpFunction`.
* Localized EN/RU UI and RFC 9457 error handling are built into the foundation rather than retrofitted across 7 sprints.

### Negative Consequences / Trade-offs

* Adopts Tailwind CSS in `frontend-web/`.
* Copied `helm` styling files are maintained in-repo; upstream styling improvements must be pulled deliberately rather than via `npm update`.
* Cognito Hosted UI has a slightly different visual chrome during sign-in than the in-app `spartan/ui` screens (acceptable for MVP; can be branded via Cognito Managed Login CSS or migrated to custom SRP later without changing backend APIs).

---

## Compliance & Invariants

1. **Token Policy**: The Angular HTTP auth interceptor must attach the Cognito **`IdToken`** (`Authorization: Bearer <id_token>`) to API Gateway calls and reject/re-authenticate sessions where `email_verified !== true`.
2. **Multi-Family Context (`ADR-0011`)**: All family-scoped API calls must attach `X-Family-Id: <active_family_id>`; onboarding/profile routes (`GET /profile/families`, `POST /families`, `POST /families/{id}/members`) must not require `X-Family-Id`.
3. **Zero Raw Backend Diagnostics in UI**: Every `application/problem+json` response must be mapped through the typed `error.code` i18n dictionary (with a localized generic fallback for unknown codes).
4. **Strict CSP Compatibility**: All fonts, scripts, and styles must work under `FamilyLedgerSecurityHeadersPolicy` (`script-src 'self'`, `style-src 'self' 'unsafe-inline'`, no external script/font CDNs).
5. **No Client-Side Monetary Float Math**: Display formatting only; all financial calculations remain in deterministic Rust backend services ([ADR-0005](0005-decimal-precision-money.md), [ADR-0018](0018-ai-native-modular-platform-and-sdui.md)).
