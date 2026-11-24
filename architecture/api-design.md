# REST API Specification — FamilyLedger

All API endpoints are hosted via Amazon API Gateway HTTP API and secured with AWS Cognito JWT validation.

---

## 1. Authentication & Security Rules

1. **Bearer Token**: Every request must include `Authorization: Bearer <cognito_jwt>` in the HTTP headers.
2. **User Identity & Multi-Family Isolation**: 
   * User identity (`user_id` from `sub`, plus optional `email` when using Cognito ID tokens; Access Tokens carry `sub` without `email`) is extracted from the validated Cognito JWT claims.
   * Multi-family workspace context is passed via the `X-Family-Id: <UUID>` request header for all family-scoped endpoints.
   * Authorization middleware verifies that `user_id` is an active member of `family_id` with sufficient role/permissions (ADR-0011).
3. **Idempotency**: All state-modifying endpoints (`POST` for transactions, payments, contributions) require a client-generated `idempotency_key: UUID`.
4. **Pagination**: List endpoints use keyset pagination via opaque cursor tokens (`cursor=<base64>`). `OFFSET` is strictly banned (ADR-0002).
5. **Ledger Immutability**: `PUT /transactions/{id}` is prohibited (`405 Method Not Allowed`). Transaction adjustments use the amendment pattern (`POST /transactions/{id}/amend`, ADR-0012).
6. **Error Responses (`application/problem+json` per RFC 9457)**: All error responses return RFC 9457 Problem Details containing `type`, `title`, `status`, `detail`, and a stable machine-readable `code` (e.g. `INSUFFICIENT_ROLE`, `TOKEN_EXPIRED`, `MEMBER_ALREADY_EXISTS`) mapped in `api/src/errors.rs` and `api/src/auth.rs` for frontend localization.

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
GET    /families/{id}                    # Any active member (Owner/Member/Child/Other): get family workspace details and home currency
PUT    /families/{id}                    # Owner only: update family name or home currency
POST   /families/{id}/invites            # Owner only: generates invite link/token (payload includes optional permissions array for Role::Other)
POST   /families/{id}/members            # Accept invite: adds authenticated user to family using invite token
GET    /families/{id}/members            # Any active member (Owner/Member/Child/Other): list all active family members and roles
PATCH  /families/{id}/members/{mid}/role # Owner only: updates member role and/or custom permissions
DELETE /families/{id}/members/{mid}      # Owner only: soft-delete a family member (cannot remove Owner)
```

#### 2.2.1 Identity & Family Wire Contracts (`family_service`)

| Route | Auth Scope | Request Body | Success Response | Domain Events Emitted |
| :--- | :--- | :--- | :--- | :--- |
| `GET /profile/families` | `UserIdentity` | — | `200 OK` `{ items: [{ family_id, name, home_currency, role, display_name }] }` | — |
| `POST /families` | `UserIdentity` | `{ name, home_currency, owner_display_name }` | `201 Created` `{ id, name, home_currency, owner_member_id, created_at }` | `FamilyCreated`, `MemberJoined` |
| `GET /families/{id}` | `AuthClaims` (`ViewFamilyWorkspace`) | — | `200 OK` `{ id, name, home_currency, created_at }` | — |
| `PUT /families/{id}` | `AuthClaims` (`ManageFamily`) | `{ name, home_currency }` | `200 OK` `{ id, name, home_currency, created_at }` | *(None — profile metadata update)* |
| `POST /families/{id}/invites` | `AuthClaims` (`ManageFamily`) | `{ role, permissions?: [], relationship?: string, ttl_hours?: i64 }` *(default `168`h = 7 days, max `720`h = 30 days)* | `201 Created` `{ token, family_id, role, permissions, relationship, created_by, created_at, expires_at }` | `MemberInvited` |
| `POST /families/{id}/members` | `UserIdentity` | `{ token, display_name }` | `201 Created` `{ id, family_id, user_id, display_name, role, permissions, relationship, joined_at }` | `MemberJoined` |
| `GET /families/{id}/members` | `AuthClaims` (`ViewFamilyWorkspace`) | — | `200 OK` `{ items: [{ id, family_id, user_id, display_name, role, permissions, relationship, joined_at }] }` *(active `deleted_at IS NULL` only)* | — |
| `PATCH /families/{id}/members/{mid}/role` | `AuthClaims` (`ManageFamily`) | `{ role, permissions?: [] }` | `200 OK` `{ id, family_id, user_id, display_name, role, permissions, relationship, joined_at }` | `MemberRoleChanged` *(0 events on no-op)* |
| `DELETE /families/{id}/members/{mid}` | `AuthClaims` (`ManageFamily`) | — | `204 No Content` | `MemberRemoved` |

**Key RFC 9457 Error Codes (`application/problem+json`)**:
- **Auth / Context (`api/src/auth.rs`)**: `401 MISSING_CLAIMS`, `401 INVALID_CLAIMS`, `400 MISSING_FAMILY_ID`, `400 INVALID_FAMILY_ID`, `403 USER_NOT_IN_FAMILY`, `403 FAMILY_ID_MISMATCH`, `403 INSUFFICIENT_ROLE`.
- **Domain / Lifecycle (`api/src/errors.rs`)**: `404 FAMILY_NOT_FOUND`, `404 MEMBER_NOT_FOUND`, `404 INVITE_TOKEN_NOT_FOUND`, `409 TOKEN_ALREADY_USED`, `409 MEMBER_ALREADY_EXISTS`, `409 DUPLICATE_OWNER`, `409 CANNOT_REMOVE_OWNER`, `410 FAMILY_DELETED`, `410 TOKEN_EXPIRED`, `422 INVALID_FAMILY_NAME` / `INVALID_DISPLAY_NAME` / `INVALID_CURRENCY_CODE` / `INVALID_RELATIONSHIP` / `INVALID_INVITE_TTL` / `INVARIANT_VIOLATION`.

### 2.3 Payment Accounts (Cards, Bank, Cash Envelopes)
```http
GET    /accounts                         # Member+ (Child sees only own accounts; Other based on ViewAllAccounts permission)
POST   /accounts                         # Member+: register a credit card / debit card / bank account / cash envelope
GET    /accounts/{id}                    # Member+: get account details and current derived balance
PUT    /accounts/{id}                    # Account owner or Family Owner: update name / limit / color / included_in_budget
DELETE /accounts/{id}                    # Account owner or Family Owner: soft-delete account
GET    /accounts/{id}/snapshots          # Member+: get monthly closing balance history (AccountBalanceSnapshots)
```

### 2.4 Ledger & Transactions
```http
GET    /transactions?cursor=...&limit=50&category=...&account=...&kind=...&from=...&to=...
POST   /transactions                     # Record expense, income, internal transfer, cash withdrawal
POST   /transactions/{id}/amend          # Amend transaction: creates Reversal entry + new corrected entry (ADR-0012)
DELETE /transactions/{id}                # Soft-delete transaction (triggers balance and envelope reversal)
```

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
