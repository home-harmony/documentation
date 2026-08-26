# MVP Business Plan — FamilyLedger

## 1. Problem Statement
Modern families face compound financial challenges:
- **Fragmented Liabilities**: Multiple active loans, mortgages, and credit cards with differing interest rates, due dates, and hidden borrowing costs.
- **Multi-Currency Complexity**: Household income, savings, and expenses frequently span multiple currencies (e.g. local currency salaries in MDL, savings in EUR, and online spending in USD) with bank exchange spreads that cause balances to drift.
- **Lack of Unified Household Visibility**: Fragmented tracking across physical cards, cash envelopes, bank accounts, and multi-family obligations (e.g. supporting elderly parents).
- **Inaccurate Surplus Calculation**: Families struggle to calculate their true disposable surplus because fixed recurring obligations (utilities, rent, subscriptions) and mandatory loan-bound insurance (PMI, homeowners, comprehensive auto) are rarely factored into automated debt payoff algorithms.
- **Result**: Chronic financial stress, missed debt optimization opportunities, and inability to plan future savings goals.

---

## 2. Solution Overview
**FamilyLedger** is a private, family-shared financial hub that:
1. **Tracks Multi-Currency Finances**: Records all income, expenses, and dual-amount cross-currency transfers across payment accounts (cards, cash, bank accounts), converting to a unified family home currency using daily official central bank rates (National Bank of Moldova - BNM / European Central Bank - ECB).
2. **Maintains Account Balance History**: Tracks closing balances via monthly snapshots, allowing families to see monthly balance trends over time.
3. **Manages Multi-Family Workspaces**: Allows users to participate in and switch between multiple family ledgers (e.g. personal household and elderly parents' finances) with role-based and granular permission controls.
4. **Calculates True Disposable Surplus**: Incorporates unlinked recurring obligations and mandatory loan-bound insurance premiums into a composite debt service cost formula to determine exact monthly surplus.
5. **Optimizes Debt Repayment**: Maintains an active loan registry (supporting system standard kinds, custom family loan types, and linked revolving credit cards) and generates automated Avalanche and Snowball payoff plans.
6. **Enforces Household Budget Governance**: Implements a monthly budget lifecycle where the Family Owner reviews, adjusts, and approves draft budgets before limits are locked.
7. **Projects Savings Timelines**: Tracks savings contributions against long-term family goals.

---

## 3. Target Audience
Households of 2–10 members (adults, children, extended family, caretakers) sharing household finances, holding $\ge 2$ active loans/credit lines, operating multi-currency accounts, and managing multiple payment instruments.

---

## 4. MVP Feature Priority Matrix

| Feature Area | Description | Priority |
| :--- | :--- | :--- |
| **Family Workspace & Multi-Tenant Auth** | Multi-family support, `X-Family-Id` context header, role-based access (`Owner`, `Member`, `Child`, `Other`), and customizable `Permission` flags for extended family/caretakers (ADR-0011). | **P0** |
| **Payment Instrument Registry & Snapshots** | Credit cards, debit cards, cash envelopes, bank accounts per member (`payment_accounts`); monthly closing balance snapshots for balance trend history (ADR-0006). | **P0** |
| **Ledger Core & Immutability** | Immutable double-entry transactions (`Income`, `Expense`, `Transfer`, `CashWithdrawal`, `LoanPayment`, `Reversal`), dual-amount cross-currency transfers, and accounting amendments via `POST /transactions/{id}/amend` (ADR-0012, ADR-0015). | **P0** |
| **Recurring Payments & Loan Insurance** | Registry of fixed bills & subscriptions, automated committed obligation calculation, and loan-bound insurance tracking linked via `linked_loan_id` (ADR-0010). | **P0** |
| **Debt Planner & Optimizer** | Loan registry with extensible system/custom loan kinds (ADR-0007), child entity `LoanPayment` with split invariants (ADR-0008), event-driven ledgerization (ADR-0009), credit card balance CDC sync (ADR-0014), and automated Avalanche/Snowball plan calculator. | **P0** |
| **Monthly Budget & Governance** | Monthly budget lifecycle: 1st-of-month auto-seeded `Draft` budget, Owner review and adjustment, and formal Owner approval before locking envelope limits (ADR-0013). | **P1** |
| **Currency & Exchange Rates** | Pluggable `ExchangeRateProvider` trait fetching daily official rates from BNM (Moldova) / ECB Frankfurter; automatic conversion to family home display currency (ADR-0015, ADR-0016). | **P1** |
| **Future Planning & Goals** | Goal-based savings timelines, contribution recording via `GoalContribution` child entities, and on-track progress projections. | **P1** |
| **Analytics & Surplus Calculation** | Trailing-month True Disposable Surplus calculation, balance history, and category spending analytics in home display currency. | **P1** |
| **Notifications & Reminders** | Bill payment reminders, budget threshold alerts (>80%), loan payoff celebrations, and loan insurance policy cancellation reminders. | **P2** |

---

## 5. Cost Projections (Family Scale)

| AWS Service | Cost Model | Estimated Monthly Cost |
| :--- | :--- | :--- |
| **Amazon Aurora DSQL** | Scales to zero; 100k DPUs + 1GB free tier | **$0.00** |
| **AWS Lambda** | First 1M requests free (microservices, CDC fanout, exchange rate fetcher, migration runner) | **$0.00** |
| **API Gateway** | First 1M HTTP API requests free | **$0.00** |
| **AWS Cognito** | First 50k MAU free | **$0.00** |
| **Kinesis Data Streams** | On-demand negligible tier for CDC streams | **$0.00** |
| **CloudFront + S3** | Static web hosting & CDN data transfer | ~\$0.50 |
| **EventBridge & Scheduler** | Domain events bus, month-start budget draft, month-end balance snapshot, and daily rate fetch | ~\$0.10 |
| **Total Estimated Footprint** | Zero idle compute / Serverless architecture | **~\$0.60 / month** |
