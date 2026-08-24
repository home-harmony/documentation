# System Architecture Overview — FamilyLedger

## 1. Cloud Architecture

FamilyLedger is built entirely on a serverless, event-driven AWS architecture designed for high availability, zero idle costs, and strong data consistency.

```mermaid
graph TD
    ClientWeb[Angular Web SPA] -->|HTTPS| CloudFront[CloudFront CDN]
    CloudFront -->|Static Assets| S3[S3 Static Hosting]
    
    ClientWeb -->|API Requests| APIGW[API Gateway HTTP API]
    ClientMobile[Flutter Android App] -->|API Requests| APIGW
    
    APIGW -->|JWT Validation| Cognito[AWS Cognito User Pool]
    APIGW -->|Proxy| ServiceLambdas[Rust Lambda Microservices]
    
    ServiceLambdas -->|Read / Write (OCC)| DSQL[(Amazon Aurora DSQL)]
    
    DSQL -->|Native CDC Stream| Kinesis[Kinesis Data Streams]
    Kinesis -->|Trigger| CDCFanout[CDC Fanout Lambda]
    CDCFanout -->|Publish Domain Events| EventBridge[AWS EventBridge Bus]
    
    EventBridge -->|Scheduled Events| Scheduler[EventBridge Scheduler]
    EventBridge -->|Async Reactions| BudgetLambda[Budget Service Lambda]
    EventBridge -->|Async Reactions| NotifLambda[Notification Service Lambda]
    EventBridge -->|Async Reactions| PlanningLambda[Planning Service Lambda]
    
    MigrateRunner[Migration Runner Lambda] -->|Schema DDL| DSQL
```

---

## 2. Infrastructure Components

### 2.1 Edge & Frontend Hosting
* **Amazon CloudFront**: Global CDN terminating HTTPS, caching frontend bundles, and enforcing SPA routing fallbacks (routing 404s to `index.html`).
* **Amazon S3**: Private bucket storing compiled Angular artifacts with CloudFront Origin Access Control (OAC).

### 2.2 API & Authentication
* **Amazon API Gateway (HTTP API)**: High-performance, low-latency API gateway routing requests to backend Lambdas.
* **AWS Cognito**: Managed user directory issuing signed JWT tokens. `family_id`, `user_id`, and `role` claims are embedded and verified per request.

### 2.3 Compute (Rust Microservices)
* **AWS Lambda (ARM64 / Graviton)**: Zero-idle microservices written in Rust 1.98 using Axum and compiled via `cargo-lambda` for sub-200ms cold starts.

### 2.4 Database & Streaming
* **Amazon Aurora DSQL**: Distributed serverless PostgreSQL-compatible database with key-ordered storage and Optimistic Concurrency Control (OCC).
* **Amazon Kinesis Data Streams**: Receives native Change Data Capture (CDC) events directly from Aurora DSQL with zero polling.
* **AWS EventBridge**: Enterprise event bus distributing semantic domain events (`TransactionRecorded`, `LoanPaidOff`, `BudgetAlertTriggered`) across bounded contexts.

---

## 3. Microservices Inventory

| Service Name | Primary Responsibility | Trigger Sources |
| :--- | :--- | :--- |
| `family_service` | Workspace, members, invite tokens, roles | API Gateway |
| `cards_service` | Cards, bank accounts, cash envelopes, balances | API Gateway |
| `ledger_service` | Transactions, categories, P2P transfers, cash withdrawals | API Gateway |
| `debt_planner_service` | Loans, payments, avalanche/snowball plan generation | API Gateway |
| `recurring_service` | Fixed bills, subscriptions, committed amounts | API Gateway, EventBridge |
| `budget_service` | Monthly budgets, envelope tracking, alert thresholds | API Gateway, EventBridge (CDC) |
| `planning_service` | Goal-based savings timelines and projections | API Gateway, EventBridge |
| `notification_service` | Web push & FCM push notifications | EventBridge, Scheduler |
| `cdc_fanout` | Ingests DSQL CDC from Kinesis $\rightarrow$ EventBridge | Kinesis Data Stream |
| `migrate_runner` | Runs embedded SQLx schema migrations on deploy | Direct Invoke (CI/CD) |

