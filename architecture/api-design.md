# REST API Specification — FamilyLedger

All API endpoints are hosted via Amazon API Gateway HTTP API and secured with AWS Cognito JWT validation.

---

## 1. Authentication & Security Rules

1. **Bearer Token**: Every request must include `Authorization: Bearer <cognito_jwt>`.
2. **Identity & Tenant Isolation**: `family_id`, `user_id`, and `role` are extracted exclusively from the validated JWT claims.
3. **Idempotency**: All state-modifying endpoints (`POST` for transactions, payments) require a client-generated `idempotency_key: UUID`.
4. **Pagination**: List endpoints use keyset pagination via opaque cursor tokens (`next_cursor`). `OFFSET` is banned.

---

## 2. API Endpoints Catalog

### 2.1 Family & Membership
```http
POST   /families                         # Any authenticated user: creates a new family workspace
POST   /families/{id}/invites            # Owner only: generates invite link/token
POST   /families/{id}/members            # Accept invite: adds authenticated user to family
GET    /families/{id}/members            # Member+: list all active family members
PATCH  /families/{id}/members/{mid}/role # Owner only: updates member role
```

### 2.2 Payment Cards & Accounts
```http
GET    /accounts                         # Member+ (Child sees only own accounts)
POST   /accounts                         # Member+: register a card / bank account / cash envelope
PUT    /accounts/{id}                    # Owner or account owner: update account name/limit/color
DELETE /accounts/{id}                    # Soft-delete account
```

### 2.3 Ledger & Transactions
```http
GET    /transactions?cursor=...&limit=50&category=...&account=...&kind=...&from=...&to=...
POST   /transactions                     # Record expense, income, transfer, cash withdrawal
PUT    /transactions/{id}                # Update transaction details
DELETE /transactions/{id}                # Soft-delete transaction (triggers balance reversal)
```

### 2.4 Debt Planner & Repayment Optimizer
```http
GET    /loans                            # Member+ (Child: 403 Forbidden)
POST   /loans                            # Register a loan/credit with principal, interest, minimum
PUT    /loans/{id}                       # Update loan terms or balance
DELETE /loans/{id}                       # Soft-delete loan
POST   /loans/{id}/payments              # Record a loan payment (principal + interest breakdown)
GET    /loans/repayment-plan             # Get latest generated avalanche/snowball plan
POST   /loans/repayment-plan/generate   # Generate optimized plan: { strategy, extra_budget }
```

### 2.5 Recurring Payments (Committed Obligations)
```http
GET    /recurring                        # List active utility bills, subscriptions, rent
POST   /recurring                        # Register new recurring obligation with frequency config
PUT    /recurring/{id}                   # Update recurring obligation
DELETE /recurring/{id}                   # Deactivate/soft-delete obligation
POST   /recurring/{id}/records           # Record actual payment made for a period
```

### 2.6 Monthly Budget & Envelopes
```http
GET    /budget/{year}/{month}            # Get monthly budget with committed & discretionary envelopes
PUT    /budget/{year}/{month}/envelopes  # Adjust discretionary envelope spending limits
```

### 2.7 Savings Goals (Future Planning)
```http
GET    /goals                            # List savings goals and on-track status
POST   /goals                            # Create new savings goal with target date and amount
PUT    /goals/{id}                       # Update goal details
DELETE /goals/{id}                       # Soft-delete goal
POST   /goals/{id}/contributions         # Record savings contribution
```

### 2.8 Analytics & Surplus Calculation
```http
GET    /analytics/monthly-surplus?months=3 # Computes true disposable surplus over trailing months
GET    /analytics/by-account?from=...&to=... # Balance history & cash flows
GET    /analytics/by-category?month=...      # Spending breakdowns by category
```

