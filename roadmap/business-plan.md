# MVP Business Plan — FamilyLedger

## 1. Problem Statement
Modern families face compound financial challenges:
- Multiple active loans and credit cards with differing interest rates and due dates.
- Lack of unified household visibility into spending across cards, cash, and bank accounts.
- Inaccurate surplus calculation: families struggle to calculate their true disposable surplus because fixed recurring obligations (utilities, rent, subscriptions) are rarely factored into automated debt payoff algorithms.
- Result: Chronic financial stress, missed debt optimization opportunities, and inability to plan future savings goals.

---

## 2. Solution Overview
**FamilyLedger** is a private, family-shared financial hub that:
1. Tracks all income and expenses per family member across all payment instruments (cards, cash, bank accounts).
2. Manages peer-to-peer transfers and cash withdrawals within the family.
3. Tracks all committed recurring obligations (utilities, rent, subscriptions) to establish a firm floor on monthly spending.
4. Maintains an active loan registry and generates optimal repayment plans (Avalanche vs. Snowball) based on the **True Disposable Surplus**.
5. Projects timelines for future major savings goals.

---

## 3. Target Audience
Households of 2–10 members (adults, children, extended family) sharing household finances, holding $\ge 2$ active loans/credit lines and multiple payment cards.

---

## 4. MVP Feature Priority Matrix

| Feature Area | Description | Priority |
| :--- | :--- | :--- |
| **Family Workspace** | Multi-tenant workspace, role-based access (Owner, Member, Child, Other) | **P0** |
| **Payment Instrument Registry** | Credit cards, debit cards, cash envelopes, bank accounts per member | **P0** |
| **Ledger Core** | Expense, income, internal P2P transfers, cash withdrawals, categories | **P0** |
| **Recurring Payments** | Registry of fixed bills & subscriptions; automated committed amount calculation | **P0** |
| **Debt Planner** | Loan registry, payment tracking, automated Avalanche/Snowball repayment plans | **P0** |
| **Monthly Budget** | Committed & discretionary envelopes with alert thresholds | **P1** |
| **Future Planning** | Goal-based savings timelines and contribution calculator | **P1** |
| **Analytics & Reports** | Monthly surplus calculations, spending breakdowns by category/account | **P1** |
| **Notifications** | Bill payment reminders, loan due alerts, budget threshold warnings | **P2** |

---

## 5. Cost Projections (Family Scale)

| AWS Service | Cost Model | Estimated Monthly Cost |
| :--- | :--- | :--- |
| **Amazon Aurora DSQL** | Scales to zero; 100k DPUs + 1GB free tier | **$0.00** |
| **AWS Lambda** | First 1M requests free | **$0.00** |
| **API Gateway** | First 1M requests free | **$0.00** |
| **AWS Cognito** | First 50k MAU free | **$0.00** |
| **Kinesis Data Streams** | On-demand negligible tier | **$0.00** |
| **CloudFront + S3** | Static web hosting & CDN data transfer | ~\$0.50 |
| **EventBridge Scheduler** | Payment reminders & scheduled triggers | ~\$0.10 |
| **Total Estimated Footprint** | | **~\$0.60 / month** |

