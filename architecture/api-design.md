# REST API Specification — FamilyLedger

All API endpoints are hosted via Amazon API Gateway HTTP API and secured with AWS Cognito JWT validation.

---

## 1. Authentication & Security Rules

1. **Bearer Token**: Every request must include `Authorization: Bearer <cognito_jwt>` in the HTTP headers.
2. **User Identity & Multi-Family Isolation**: 
   * User identity (`user_id`, `email`) is extracted from the validated Cognito JWT claims (`sub`, `email`).
   * Multi-family workspace context is passed via the `X-Family-Id: <UUID>` request header for all family-scoped endpoints.
   * Authorization middleware verifies that `user_id` is an active member of `family_id` with sufficient role/permissions (ADR-0011).
3. **Idempotency**: All state-modifying endpoints (`POST` for transactions, payments, contributions) require a client-generated `idempotency_key: UUID`.
4. **Pagination**: List endpoints use keyset pagination via opaque cursor tokens (`cursor=<base64>`). `OFFSET` is strictly banned (ADR-0002).
5. **Ledger Immutability**: `PUT /transactions/{id}` is prohibited (`405 Method Not Allowed`). Transaction adjustments use the amendment pattern (`POST /transactions/{id}/amend`, ADR-0012).

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
GET    /budget/{year}/{month}            # Get monthly budget with status (Draft/Active/Closed), envelopes, and spent amounts
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
