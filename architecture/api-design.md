# REST API Specification — FamilyLedger

All API endpoints are hosted via Amazon API Gateway HTTP API and secured with AWS Cognito JWT validation.

---

## 1. Authentication & Security Rules

1. **Bearer Token & Gateway Trust Boundary**: Every request must include `Authorization: Bearer <cognito_jwt>` in the HTTP headers.
   * API Gateway HTTP API validates the Cognito RS256 signature, `iss`, `exp`, and `aud`/`client_id` at the edge.
   * Production Rust Lambdas read verified claims from `requestContext.authorizer.jwt.claims` (never from raw unverified HTTP identity headers) and fail closed (`401 MISSING_CLAIMS`) if missing. Local/test execution uses `LocalJwtVerifier` (`jsonwebtoken`).
2. **User Identity, Token Policy (Option 2B) & Multi-Family Isolation**: 
   * `token_use == "access"` resolves [`UserIdentity`](../../backend/api/src/auth.rs) (`user_id` from `sub`, `email: None`) for both native and Google-federated users.
   * `token_use == "id"` is accepted **only** when `email_verified == "true"` (else `401 UNVERIFIED_EMAIL`), populating `email: Some(Email)`.
   * Onboarding/profile routes (`GET /profile/families`, `POST /families`, `POST /families/{id}/members`) extract `UserIdentity` without requiring `X-Family-Id`.
   * Multi-family workspace context is passed via the `X-Family-Id: <UUID>` request header for all family-scoped endpoints (`400 MISSING_FAMILY_ID` / `INVALID_FAMILY_ID` if absent or malformed).
   * Authorization middleware verifies that `user_id` is an active (`deleted_at IS NULL`) member of `family_id` in Aurora DSQL (`BEGIN READ ONLY`) and resolves sealed [`AuthClaims`](../../backend/api/src/auth.rs) (`403 USER_NOT_IN_FAMILY` if not an active member) per ADR-0011.
   * **Send the Cognito ID token (`IdToken`), not the Access Token**: the API Gateway `CognitoJwtAuthorizer` (`services-template.yaml`) sets `JwtConfiguration.Audience` to the web client ID, and only ID tokens carry the `aud` claim. Access tokens (which carry `client_id`, not `aud`) are rejected at the gateway with `401` before the Lambda runs. The Lambda extractor still treats `email` as optional.
   * **CORS allow-list**: `X-Family-Id` is an allowed request header (alongside `Content-Type`, `Authorization`, `X-Amz-Date`, `X-Api-Key`). Allowed origins are the CloudFront URL exported by the `core` stack, plus `http://localhost:4200` in the `dev` environment only.
3. **Idempotency**: All state-modifying ledger/payment endpoints (`POST /transactions`, `POST /transactions/{id}/amend`, payments, contributions) require a client-generated `idempotency_key: UUID` (uniqueness scoped per `(family_id, idempotency_key)`). Repeating a request with an already-committed `(family_id, idempotency_key)` returns an idempotent `200 OK` / `201 Created` replay (emitting `0` duplicate EventBridge events) when the normalized request matches, or `409 IDEMPOTENCY_CONFLICT` when the payload differs. On `POST /transactions/{id}/amend`, the client's `idempotency_key` is stored on the `Corrected` entry and a deterministic SHA-256 companion UUID is derived for the `Reversal` entry.
4. **Pagination**: List endpoints use keyset pagination via opaque cursor tokens (`cursor=<base64>`) with `limit <= 50` enforced by `PageLimit`. `OFFSET` is strictly banned (ADR-0002).
5. **Ledger Immutability**: `PUT /transactions/{id}` is prohibited (`405 Method Not Allowed`, `METHOD_NOT_ALLOWED`). Transaction adjustments use the amendment pattern (`POST /transactions/{id}/amend`, ADR-0012).
6. **Error Responses (`application/problem+json` per RFC 9457) & Security Headers**: All error responses return RFC 9457 Problem Details containing `type`, `title`, `status`, `detail`, and a stable machine-readable `code` (e.g. `MISSING_CLAIMS`, `INVALID_CLAIMS`, `UNVERIFIED_EMAIL`, `MISSING_FAMILY_ID`, `INVALID_FAMILY_ID`, `USER_NOT_IN_FAMILY`, `INSUFFICIENT_ROLE`, `TOKEN_EXPIRED`, `MEMBER_ALREADY_EXISTS`) mapped in `api/src/errors.rs` and `api/src/auth.rs` for frontend localization, alongside `X-Content-Type-Options: nosniff` and `Cache-Control: no-store`.

---

## 2. API Endpoints Catalog

### 2.1 User Profile & Workspaces
```http
GET    /profile/families                 # Authenticated user: list all families user belongs to (with role & display name)
```

### 2.2 Family & Membership
*(Note: `POST /families` and `POST /families/{id}/members` require only `Authorization: Bearer` (`UserIdentity`); all other routes below require `X-Family-Id` matching `{id}`.)*
```http
POST   /families                         # Authenticated user: creates a new family workspace (creator becomes Owner)
GET    /families/{id}                    # Any active member (Owner/Member/Child/Guest): get family workspace details and home currency
PUT    /families/{id}                    # Owner only: update family name or home currency
POST   /families/{id}/invites            # Owner only: generates invite link/token (payload includes optional permissions array for Role::Guest)
POST   /families/{id}/members            # Accept invite: adds authenticated user to family using invite token
GET    /families/{id}/members            # Any active member (Owner/Member/Child/Guest): list all active family members and roles
PATCH  /families/{id}/members/{mid}/role # Owner only: updates member role and/or custom permissions
DELETE /families/{id}/members/{mid}      # Owner only: soft-delete a family member (cannot remove Owner)
```

#### 2.2.1 Identity & Family Wire Contracts (`family_service`)

| Route | Auth Scope | Request Body | Success Response | Domain Events Emitted |
| :--- | :--- | :--- | :--- | :--- |
| `GET /profile/families` | `UserIdentity` | — | `200 OK` `{ items: [{ family: { id, name, home_currency }, membership: { member_id, role, permissions, display_name } }] }` | — |
| `POST /families` | `UserIdentity` | `{ name, home_currency, owner_display_name }` | `201 Created` `{ id, name, home_currency, owner_member_id, created_at }` | `FamilyCreated`, `MemberJoined` |
| `GET /families/{id}` | `AuthClaims` (`ViewFamilyWorkspace`) | — | `200 OK` `{ id, name, home_currency, created_at }` | — |
| `PUT /families/{id}` | `AuthClaims` (`ManageFamily`) | `{ name, home_currency }` | `200 OK` `{ id, name, home_currency, created_at }` | *(None — profile metadata update)* |
| `POST /families/{id}/invites` | `AuthClaims` (`ManageFamily`) | `{ role, permissions?: [], relationship?: string, ttl_hours?: i64 }` *(wire `role`: `"owner" \| "member" \| "child" \| "guest"`; `permissions`: `["record_own_transactions", "record_any_transaction", "view_all_transactions", "view_all_accounts", "manage_own_accounts", "view_loans", "view_budget", "view_reports"]`; `ttl_hours` default `168`h = 7 days, max `720`h = 30 days)* | `201 Created` `{ token, family_id, role, permissions, relationship, created_by, created_at, expires_at }` | `MemberInvited` |
| `POST /families/{id}/members` | `UserIdentity` | `{ token, display_name }` | `201 Created` `{ id, family_id, user_id, display_name, role, permissions, relationship, joined_at }` | `MemberJoined` |
| `GET /families/{id}/members` | `AuthClaims` (`ViewFamilyWorkspace`) | — | `200 OK` `{ items: [{ id, family_id, user_id, display_name, role, permissions, relationship, joined_at }] }` *(active `deleted_at IS NULL` only)* | — |
| `PATCH /families/{id}/members/{mid}/role` | `AuthClaims` (`ManageFamily`) | `{ role, permissions?: [] }` | `200 OK` `{ id, family_id, user_id, display_name, role, permissions, relationship, joined_at }` | `MemberRoleChanged` *(0 events on no-op)* |
| `DELETE /families/{id}/members/{mid}` | `AuthClaims` (`ManageFamily`) | — | `204 No Content` | `MemberRemoved` |

**Key RFC 9457 Error Codes (`application/problem+json`)**:
- **Auth / Context (`api/src/auth.rs`)**: `401 MISSING_CLAIMS`, `401 INVALID_CLAIMS`, `401 UNVERIFIED_EMAIL`, `400 MISSING_FAMILY_ID`, `400 INVALID_FAMILY_ID`, `403 USER_NOT_IN_FAMILY`, `403 FAMILY_ID_MISMATCH`, `403 INSUFFICIENT_ROLE`.
- **Request / Infrastructure (`api/src/errors.rs`)**: `400 INVALID_JSON` *(malformed JSON, unknown enum values for `role`/`permissions`, or malformed body UUIDs)*, `400 INVALID_MEMBER_ID` *(non-UUID `{mid}` path parameter)*, `400 INVALID_CURSOR` / `400 INVALID_QUERY_PARAM`, `405 METHOD_NOT_ALLOWED`, `500 INTERNAL_ERROR` *(sanitized generic `detail`; real cause logged via `tracing::error!`)*.
- **Domain / Lifecycle (`api/src/errors.rs`)**: `404 FAMILY_NOT_FOUND`, `404 MEMBER_NOT_FOUND`, `404 INVITE_TOKEN_NOT_FOUND`, `409 TOKEN_ALREADY_USED`, `409 MEMBER_ALREADY_EXISTS`, `409 DUPLICATE_OWNER`, `409 CANNOT_REMOVE_OWNER`, `410 FAMILY_DELETED`, `410 TOKEN_EXPIRED`, `422 INVALID_FAMILY_NAME` / `INVALID_DISPLAY_NAME` / `INVALID_CURRENCY_CODE` / `INVALID_EMAIL` / `INVALID_RELATIONSHIP` / `INVALID_INVITE_TTL` / `INVARIANT_VIOLATION`.
- **Concurrency Semantics (`ADR-0020`, Accepted)**: Mutating family operations execute atomically with OCC retries; concurrent redemption of the same invite token yields one `201 Created` and `409 TOKEN_ALREADY_USED` for losing callers, and concurrent joins by the same user with distinct tokens yield one `201 Created` and `409 MEMBER_ALREADY_EXISTS` (never `500 INTERNAL_ERROR`).

### 2.3 Payment Accounts (`accounts_service`)
```http
GET    /accounts                         # Requires ViewAllAccounts (sees all active) or ManageOwnAccounts (sees own); else 403 INSUFFICIENT_ROLE
POST   /accounts                         # Requires ManageOwnAccounts: register credit_card / debit_card / cash / bank_account
GET    /accounts/{id}                    # Account owner (ManageOwnAccounts) or ViewAllAccounts: details + derived SignedMoney current_balance
PUT    /accounts/{id}                    # Account owner (ManageOwnAccounts) or Family Owner: update name / credit_limit / last_four / bank_name / color / included_in_budget
DELETE /accounts/{id}                    # Account owner (ManageOwnAccounts) or Family Owner: soft-delete account
GET    /accounts/{id}/snapshots          # Account owner (ManageOwnAccounts) or ViewAllAccounts: monthly closing balance history (AccountBalanceSnapshots)
```

#### 2.3.1 Payment Accounts Wire Contracts (`accounts_service`)

| Route | Auth Scope | Request Body | Success Response | Domain Events Emitted |
| :--- | :--- | :--- | :--- | :--- |
| `GET /accounts` | `ViewAllAccounts \| ManageOwnAccounts` | — | `200 OK` `{ items: [AccountResponse] }` *(with derived `current_balance: { amount: "-120.50", currency: "USD" }`)* | — |
| `POST /accounts` | `ManageOwnAccounts` | `{ name, kind, currency, opening_balance?: string, credit_limit?: string, last_four?: string, bank_name?: string, color?: string, included_in_budget?: bool, owner_member_id?: UUID }` *(only `Role::Owner` may specify another active `owner_member_id`; `credit_limit` allowed strictly when `kind == "credit_card"`)* | `201 Created` `AccountResponse` | `AccountRegistered` |
| `GET /accounts/{id}` | `ViewAllAccounts \| ManageOwnAccounts` (own) | — | `200 OK` `AccountResponse` | — |
| `PUT /accounts/{id}` | Account owner (`ManageOwnAccounts`) or `Role::Owner` | `{ name, credit_limit?: string \| null, last_four?: string \| null, bank_name?: string \| null, color?: string \| null, included_in_budget: bool }` *(`currency`, `kind`, `owner_member_id`, and `opening_balance` are immutable)* | `200 OK` `AccountResponse` | *(None)* |
| `DELETE /accounts/{id}` | Account owner (`ManageOwnAccounts`) or `Role::Owner` | — | `204 No Content` | *(None)* |
| `GET /accounts/{id}/snapshots` | `ViewAllAccounts \| ManageOwnAccounts` (own) | Query: `?limit=24&cursor=...` | `200 OK` `{ items: [{ id, account_id, year_month, closing_balance: { amount, currency }, computed_at }], next_cursor, has_more }` | — |

**Accounts Error Codes**: `400 INVALID_ACCOUNT_ID`, `404 ACCOUNT_NOT_FOUND`, `410 ACCOUNT_DELETED`, `422 INVALID_ACCOUNT_NAME`, `422 INVALID_BANK_NAME`, `422 INVALID_LAST_FOUR`, `422 INVALID_HEX_COLOR`, `422 CREDIT_LIMIT_NOT_ALLOWED`, `422 NEGATIVE_AMOUNT`.

### 2.4 Ledger & Categories (`ledger_service`)
```http
GET    /categories                       # Requires any ledger view/record permission: list global system (family_id = null) + family custom categories
POST   /categories                       # Requires RecordAnyTransaction (Owner, Member, or Guest with flag): create custom family category
PUT    /categories/{id}                  # Requires RecordAnyTransaction: update custom family category (system categories return 403/409)
DELETE /categories/{id}                  # Requires RecordAnyTransaction: soft-delete custom family category
GET    /transactions?cursor=...&limit=50&category=...&account=...&kind=...&from=...&to=...&include_audit=false
POST   /transactions                     # Record income, expense, transfer, or cash_withdrawal (PositiveMoney > 0, idempotency_key)
PUT    /transactions/{id}                # Prohibited -> 405 METHOD_NOT_ALLOWED (ADR-0012)
POST   /transactions/{id}/amend          # Atomic Reversal + Corrected entry with amendment_of_id = {id}, original.deleted_at = NULL (ADR-0012)
DELETE /transactions/{id}                # Soft-delete transaction (sets deleted_at = now(); emits TransactionDeleted)
```

#### 2.4.1 Ledger & Categories Wire Contracts (`ledger_service`)

| Route | Auth Scope | Request Body / Query | Success Response | Domain Events Emitted |
| :--- | :--- | :--- | :--- | :--- |
| `GET /categories` | `ViewAllTransactions \| RecordOwnTransactions \| RecordAnyTransaction` | Optional `?kind=income\|expense\|transfer` | `200 OK` `{ items: [{ id, family_id, is_system, name, kind, icon, color }] }` | — |
| `POST /categories` | `RecordAnyTransaction` | `{ name, kind, icon?: string, color?: string }` | `201 Created` `CategoryResponse` | *(None)* |
| `PUT /categories/{id}` | `RecordAnyTransaction` | `{ name, icon?: string \| null, color?: string \| null }` *(`kind` immutable; system categories rejected with `409 CANNOT_MODIFY_SYSTEM_CATEGORY`)* | `200 OK` `CategoryResponse` | *(None)* |
| `DELETE /categories/{id}` | `RecordAnyTransaction` | — | `204 No Content` *(system categories rejected with `409 CANNOT_MODIFY_SYSTEM_CATEGORY`)* | *(None)* |
| `GET /transactions` | `ViewAllTransactions` (all) or `RecordOwnTransactions` (own recorded or touching own accounts); else `403` | `?cursor=<b64>&limit=1..50&category=<uuid>&account=<uuid>&kind=...&from=<iso>&to=<iso>&include_audit=bool` *(default `include_audit=false` hides superseded originals & `Reversal` rows)* | `200 OK` `{ items: [TransactionResponse], next_cursor, has_more }` *(cross-currency rows include `implied_rate: string` and `fx_spread: { amount, currency } \| null`)* | — |
| `POST /transactions` | `RecordAnyTransaction` (any active account) or `RecordOwnTransactions` (own accounts only) | `{ kind: "income"\|"expense"\|"transfer"\|"cash_withdrawal", amount: { amount, currency }, destination_amount?: { amount, currency }, source_account_id?: UUID, destination_account_id?: UUID, category_id?: UUID, tags?: string[], description?: string, occurred_at: ISO8601, idempotency_key: UUID }` | `201 Created` (or `200 OK` on matching idempotent replay) `TransactionResponse` | `TransactionRecorded` *(`0` on replay)* |
| `PUT /transactions/{id}` | Any | Any | `405 Method Not Allowed` (`METHOD_NOT_ALLOWED`) | — |
| `POST /transactions/{id}/amend` | `RecordAnyTransaction` / `Role::Owner`, or `RecordOwnTransactions` (if `recorded_by == caller` & own accounts) | `{ kind, amount, destination_amount?, source_account_id?, destination_account_id?, category_id?, tags?, description?, occurred_at, idempotency_key: UUID }` | `201 Created` (or `200 OK` on matching replay) `{ reversal: TransactionResponse, corrected: TransactionResponse }` | `TransactionReversed`, `TransactionRecorded` *(`0` on replay)* |
| `DELETE /transactions/{id}` | `RecordAnyTransaction` / `Role::Owner`, or `RecordOwnTransactions` (if `recorded_by == caller` & own accounts) | — | `204 No Content` | `TransactionDeleted` |

**Ledger Error Codes**: `400 INVALID_TRANSACTION_ID`, `400 INVALID_CATEGORY_ID`, `400 INVALID_CURSOR`, `400 INVALID_PAGE_LIMIT`, `404 TRANSACTION_NOT_FOUND`, `404 CATEGORY_NOT_FOUND`, `405 METHOD_NOT_ALLOWED`, `409 IDEMPOTENCY_CONFLICT`, `409 TRANSACTION_ALREADY_AMENDED`, `409 CANNOT_MODIFY_SYSTEM_CATEGORY`, `410 TRANSACTION_DELETED`, `410 ACCOUNT_DELETED`, `410 CATEGORY_DELETED`, `422 NON_POSITIVE_AMOUNT`, `422 CURRENCY_MISMATCH`, `422 CROSS_CURRENCY_DESTINATION_REQUIRED`, `422 SAME_CURRENCY_DESTINATION_FORBIDDEN`, `422 SAME_ACCOUNT_TRANSFER`, `422 INVALID_CASH_WITHDRAWAL_ACCOUNTS`, `422 CATEGORY_KIND_MISMATCH`, `422 CANNOT_AMEND_REVERSAL`, `422 INVALID_CATEGORY_NAME`, `422 INVALID_CATEGORY_ICON`, `422 INVALID_TAG`, `422 INVALID_DESCRIPTION`.

### 2.5 Debt Planner & Repayment Optimizer
```http
GET    /loan-kinds                       # List system standard loan kinds + family custom loan kinds (ADR-0007)
POST   /loan-kinds                       # Owner/Member: create a custom family loan kind
GET    /loans                            # Member+ with ViewLoans permission: list all active loans
POST   /loans                            # Register a loan/credit with principal, interest, minimum payment, and loan_kind_id
GET    /loans/{id}                       # Get loan details, payments history, and linked account/insurance
PUT    /loans/{id}                       # Update loan terms or linked credit card account (ADR-0014)
DELETE /loans/{id}                       # Soft-delete loan
POST   /loans/{id}/payments              # Record a loan payment (principal + interest breakdown; emits LoanPaymentRecorded)
GET    /loans/repayment-plan             # Get latest cached Avalanche/Snowball repayment plan projection
POST   /loans/repayment-plan/generate    # Recalculate plan projection: { strategy: "avalanche" | "snowball", extra_budget }
```

### 2.6 Recurring Payments (Committed Obligations)
```http
GET    /recurring?linked_loan_id=...     # List active utility bills, subscriptions, rent, and loan-bound insurance
POST   /recurring                        # Register new recurring obligation with frequency config and optional linked_loan_id
GET    /recurring/{id}                   # Get recurring obligation details
PUT    /recurring/{id}                   # Update recurring obligation
DELETE /recurring/{id}                   # Deactivate/soft-delete obligation
POST   /recurring/{id}/records           # Record actual payment made for a period
```

### 2.7 Monthly Budget & Envelopes
```http
GET    /budget/{year}/{month}            # Get monthly budget with status (Draft/Active), envelopes, and spent amounts
PUT    /budget/{year}/{month}/envelopes  # Draft status only: adjust discretionary envelope spending limits
POST   /budget/{year}/{month}/approve    # Owner only: approve draft budget -> transitions to Active, locks envelope limits (ADR-0013)
```

### 2.8 Savings Goals (Future Planning)
```http
GET    /goals                            # List savings goals, calculated current_saved, and on-track status
POST   /goals                            # Create new savings goal with target date, target amount, and monthly target
GET    /goals/{id}                       # Get goal details and contribution history
PUT    /goals/{id}                       # Update goal details
DELETE /goals/{id}                       # Soft-delete goal
POST   /goals/{id}/contributions         # Record a deposit/contribution toward the savings goal
```

### 2.9 Currency & Exchange Rates
```http
GET    /exchange-rates                   # Get latest live exchange rates against family home_currency (ADR-0015, ADR-0016)
```

### 2.10 Analytics & Surplus Calculation
```http
GET    /analytics/monthly-surplus?months=3 # Computes true disposable surplus over trailing months in home display currency
GET    /analytics/by-account?from=...&to=... # Balance history & cash flows
GET    /analytics/by-category?month=...      # Spending breakdowns by category
```
