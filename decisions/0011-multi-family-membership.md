# 0011. A User May Belong to Multiple Family Workspaces Simultaneously

* **Status**: Accepted
* **Date**: 2026-08-26
* **Deciders**: Dmitri Astafiev

## Context and Problem Statement

The initial backend authentication design embedded a single `custom:family_id` and `custom:family_role` in the AWS Cognito JWT token. This architecture artificially limited a user to exactly one family workspace.

In real-world households, multi-family participation is essential:
* A person may manage their primary nuclear household finances while simultaneously acting as a financial guardian or contributor for aging parents.
* Adult children may move out and start their own household workspace while retaining read or collaborative access to their family of origin.
* Shared vacation or investment properties between siblings may be organized into a separate family ledger workspace.

The persistence table `family_members` already utilizes `UNIQUE(family_id, user_id)` (which allows the same `user_id` across different `family_id` rows). However, the auth token pipeline and API authorization layers required structural adjustments to support multi-family contexts.

## Decision Drivers

* **Real-World Household Dynamics** — support dual/multi-family membership without requiring users to register multiple email accounts.
* **Security & Multi-Tenant Isolation** — strict enforcement of tenant isolation per request, ensuring that credentials and queries never leak between family workspaces.
* **Stateless API Gateway Compatibility** — seamless extraction and verification of the active tenant context on each HTTP request.

## Considered Options

1. **Single Family Hard-Lock (Status Quo)** — enforce `UNIQUE(user_id)` and embed `family_id` in Cognito claims. (Rejected: fails key real-world user scenarios).
2. **Multiple Cognito Tokens per Family** — client must re-authenticate against Cognito specifying the tenant to obtain a new family-scoped JWT. (Rejected: poor UX and complex token refresh logic).
3. **Global User Identity with `X-Family-Id` Context Header** — the JWT identifies the authenticated user (`sub`), and the client supplies the target family in the `X-Family-Id` HTTP header. The API authorization middleware validates membership and role for that specific workspace. (Chosen).

## Decision Outcome

Chosen option: **Option 3 — Global User Identity with `X-Family-Id` Context Header**.

### Architectural Changes

1. **Cognito Token**: Contains user identity claims (`sub`, `email`) but no static `custom:family_id` or `custom:family_role` claims.
2. **Request Header**: All family-scoped API requests must include:
   ```http
   X-Family-Id: 018f3a5e-9a2b-7c3d-8e4f-123456789abc
   Authorization: Bearer <cognito_jwt>
   ```
3. **API Auth Middleware**:
   ```rust
   // api/src/auth.rs
   pub struct AuthClaims {
       pub user_id: Uuid,
       pub family_id: Uuid,
       pub role: Role,
       pub permissions: Option<Vec<Permission>>,
       pub email: String,
   }
   ```
   The middleware extracts `user_id` from the JWT and `family_id` from `X-Family-Id`, then validates against `family_members`:
   ```sql
   SELECT role, permissions 
   FROM family_members 
   WHERE family_id = $1 AND user_id = $2 AND deleted_at IS NULL;
   ```
4. **Profile & Workspace Switching API**:
   * `GET /profile/families` — returns the list of all families the user belongs to, including their roles and display names.

### Positive Consequences

* Single user account can participate in multiple family workspaces seamlessly.
* Switching between families in the Web SPA or Flutter mobile app is a simple client-side state update of `X-Family-Id`.
* Database multi-tenancy remains clean, with `family_id` acting as the tenant boundary for all domain tables.

### Negative Consequences / Trade-offs

* API Gateway cannot perform 100% of the authorization check statically; the Lambda auth middleware must verify family membership against Aurora DSQL (or a low-latency cached store).
* Client apps must present a family-switching UI when multiple memberships exist.

## Compliance & Invariants

* `family_id` **must never** be accepted from client request JSON bodies or query strings for authorization purposes; it must be verified via the `X-Family-Id` header against the caller's `user_id`.
* The `family_members` table **must retain** `UNIQUE(family_id, user_id)` and must never enforce global uniqueness on `user_id`.
