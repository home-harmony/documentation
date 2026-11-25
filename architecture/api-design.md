# REST API Specification — FamilyLedger

All API endpoints are hosted via Amazon API Gateway HTTP API and secured with AWS Cognito JWT validation.

---

## 1. Authentication & Security Rules

1. **Bearer Token & Gateway Trust Boundary**: Every request must include `Authorization: Bearer <cognito_jwt>` in the HTTP headers.
   * API Gateway HTTP API validates the Cognito RS256 signature, `iss`, `exp`, and `aud`/`client_id` at the edge.
   * Production Rust Lambdas read verified claims from `requestContext.authorizer.jwt.claims` (never from raw unverified HTTP identity headers) and fail closed (`401 MISSING_CLAIMS`) if missing. Local/test execution uses `LocalJwtVerifier` (`jsonwebtoken`).
2. **User Identity, Token Policy (Option 2B) & Multi-Family Isolation**: 
   * `token_use == "access"` resolves [`AuthenticatedUser`](../../backend/api/src/auth.rs) (`user_id` from `sub`, `email: None`) for both native and Google-federated users.
   * `token_use == "id"` is accepted **only** when `email_verified == "true"` (else `401 UNVERIFIED_EMAIL`), populating `email: Some(Email)`.
   * Onboarding/profile routes (`GET /profile/families`, `POST /families`, `POST /families/{id}/members`) extract `AuthenticatedUser` without requiring `X-Family-Id`.
   * Multi-family workspace context is passed via the `X-Family-Id: <UUID>` request header for all family-scoped endpoints (`400 MISSING_FAMILY_ID` / `INVALID_FAMILY_ID` if absent or malformed).
   * Authorization middleware verifies that `user_id` is an active (`deleted_at IS NULL`) member of `family_id` in Aurora DSQL (`BEGIN READ ONLY`) and resolves sealed [`AuthClaims`](../../backend/api/src/auth.rs) (`403 USER_NOT_IN_FAMILY` if not an active member) per ADR-0011.
3. **Idempotency**: All state-modifying endpoints (`POST` for transactions, payments, contributions) require a client-generated `idempotency_key: UUID` (uniqueness scoped per `(family_id, idempotency_key)`).
4. **Pagination**: List endpoints use keyset pagination via opaque cursor tokens (`cursor=<base64>`) with `limit <= 50` enforced by `PageLimit`. `OFFSET` is strictly banned (ADR-0002).
5. **Ledger Immutability**: `PUT /transactions/{id}` is prohibited (`405 Method Not Allowed`). Transaction adjustments use the amendment pattern (`POST /transactions/{id}/amend`, ADR-0012).
6. **Error Responses (`application/problem+json` per RFC 9457) & Security Headers**: All error responses return RFC 9457 Problem Details containing `type`, `title`, `status`, `detail`, and a stable machine-readable `code` (e.g. `MISSING_CLAIMS`, `INVALID_CLAIMS`, `UNVERIFIED_EMAIL`, `MISSING_FAMILY_ID`, `INVALID_FAMILY_ID`, `USER_NOT_IN_FAMILY`, `INSUFFICIENT_ROLE`, `TOKEN_EXPIRED`, `MEMBER_ALREADY_EXISTS`) mapped in `api/src/errors.rs` and `api/src/auth.rs` for frontend localization, alongside `X-Content-Type-Options: nosniff` and `Cache-Control: no-store`.

---

## 2. API Endpoints Catalog

### 2.1 User Profile & Workspaces
```http
GET    /profile/families                 # Authenticated user: list all families user belongs to (with role & display name)
```

### 2.2 Family & Membership (Requires `X-Family-Id` header)
```http
POST   /families                         # Authenticated user: creates a new family workspace (creator becomes Owner)
GET    /families/{id}                    # Member+: get family workspace details and home currency
PUT    /families/{id}                    # Owner only: update family name or home currency
POST   /families/{id}/invites            # Owner only: generates invite link/token (payload includes optional permissions array for Role::Other)
POST   /families/{id}/members            # Accept invite: adds authenticated user to family using invite token
GET    /families/{id}/members            # Member+: list all active family members and roles
PATCH  /families/{id}/members/{mid}/role # Owner only: updates member role and/or custom permissions
DELETE /families/{id}/members/{mid}      # Owner only: soft-delete a family member (cannot remove Owner)
```

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
