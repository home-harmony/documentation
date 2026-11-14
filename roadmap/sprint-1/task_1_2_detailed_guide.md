# Task 1.2 Detailed Guide — Database & Event Stream Provisioning

> **Goal**: Provision the **Amazon Aurora DSQL** serverless cluster, configure **IAM database authentication**, create an **Amazon Kinesis Data Stream** for native Change Data Capture (CDC), create a custom **Amazon EventBridge Bus** for semantic domain events, and deploy the infrastructure stack via AWS SAM.

---

## What You Will Build (Mental Model)

```
              ┌─────────────────────────────────────────────────────────┐
              │                   Rust Microservices                    │
              │  (family_service, ledger_service, cards_service, etc.)  │
              └────────────────────────────┬────────────────────────────┘
                                           │
                           1. Read/Write (OCC + IAM Auth)
                                           │
                                           ▼
              ┌─────────────────────────────────────────────────────────┐
              │                   Amazon Aurora DSQL                    │
              │   Serverless Distributed Active-Active PostgreSQL 16    │
              └────────────────────────────┬────────────────────────────┘
                                           │
                                  2. Native CDC Stream
                                           │
                                           ▼
              ┌─────────────────────────────────────────────────────────┐
              │               Amazon Kinesis Data Stream                │
              │          `familyledger-cdc-stream-dev` (On-Demand)      │
              └────────────────────────────┬────────────────────────────┘
                                           │
                                3. Event-Source Trigger
                                           │
                                           ▼
              ┌─────────────────────────────────────────────────────────┐
              │                 CDC Fanout Lambda (Rust)                │
              │            Transforms CDC row diffs to events           │
              └────────────────────────────┬────────────────────────────┘
                                           │
                                4. Publish Domain Events
                                           │
                                           ▼
              ┌─────────────────────────────────────────────────────────┐
              │                Amazon EventBridge Bus                   │
              │             `familyledger-events-dev`                   │
              └────────────────────────────┬────────────────────────────┘
                                           │
                      5. Subscriptions (Rules & Targets)
                                           │
                 ┌─────────────────────────┼─────────────────────────┐
                 ▼                         ▼                        ▼
        ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
        │  Budget Service │       │  Planning Engine│       │  Notifications  │
        └─────────────────┘       └─────────────────┘       └─────────────────┘
```

---

## Prerequisites: AWS IAM Permissions for CloudFormation & SAM

When AWS SAM deploys the data & streaming stack, it creates custom IAM roles (`FamilyLedgerServiceRole` and `FamilyLedgerMigrationRunnerRole`) with embedded policies for Aurora DSQL and EventBridge. 

To allow CloudFormation to manage and pass these execution roles to Lambda services securely, your IAM deployer user (e.g. `programmer-user`) must have IAM management permissions conforming to AWS security best practices (scoping `iam:PassRole` with the `iam:PassedToService` condition).

### How to Add IAM Role Permissions to Deployer User:

1. Sign in to the **AWS Management Console** as an Administrator or Root user.
2. Navigate to **IAM** $\rightarrow$ **Users** $\rightarrow$ select your user (`programmer-user`).
3. Under the **Permissions** tab, click **Add permissions** $\rightarrow$ **Create inline policy**.
4. Switch to the **JSON** editor tab and paste the following policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowIamRoleLifecycleManagement",
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:GetRole",
                "iam:AttachRolePolicy",
                "iam:DetachRolePolicy",
                "iam:PutRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:GetRolePolicy",
                "iam:TagRole"
            ],
            "Resource": "arn:aws:iam::377161178071:role/familyledger-*"
        },
        {
            "Sid": "AllowPassRoleToLambda",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::377161178071:role/familyledger-*",
            "Condition": {
                "StringEquals": {
                    "iam:PassedToService": "lambda.amazonaws.com"
                }
            }
        }
    ]
}
```
5. Click **Next**, name the policy (e.g. `FamilyLedgerDeployerRoleManagementPolicy`), and click **Create policy**.

---

## Step 1: Provision Aurora DSQL Cluster via AWS CLI

Aurora DSQL clusters are created via the AWS CLI or AWS Console.

### 1.1 — Create the Aurora DSQL Cluster
Run the following command in PowerShell:

```powershell
aws dsql create-cluster --region us-east-1 --tags Environment=dev,Project=FamilyLedger
```

**Expected output:**
```json
{
    "identifier": "4bublzlvvtnwjpmgtotwrulv7m",
    "arn": "arn:aws:dsql:us-east-1:377161178071:cluster/4bublzlvvtnwjpmgtotwrulv7m",
    "status": "CREATING",
    "creationTime": "2026-08-31T14:47:01.702000+03:00"
}
```

### 1.2 — Wait for Cluster Status to become `ACTIVE`
```powershell
aws dsql get-cluster --identifier 4bublzlvvtnwjpmgtotwrulv7m --region us-east-1
```
Ensure `"status": "ACTIVE"` before proceeding.

---

## Step 2: Review the SAM Infrastructure Template

The template [`backend/infrastructure/sam/data-stream-template.yaml`](backend/infrastructure/sam/data-stream-template.yaml) provisions:
1. **`FamilyLedgerCdcStream`** *(Conditional)*: AWS Kinesis Data Stream with KMS encryption.
   * > [!WARNING]
     > **Cost Alert**: Kinesis charges a flat hourly fee even when completely idle!
     > - `DeployKinesisStream="false"`: **$0.00 / month** (Recommended for Sprints 1–3).
     > - `KinesisCapacityMode="PROVISIONED"` (1 shard): **$0.015/hour** (~$0.36/day, ~$10.80/month).
     > - `KinesisCapacityMode="ON_DEMAND"`: **$0.040/hour** (~$0.96/day, ~$28.80/month).
     > See [Kinesis Modes & Pricing Guide](../../architecture/kinesis-cdc-streaming.md) for full details.
2. **`FamilyLedgerEventBus`**: Custom EventBridge bus for domain events ($0.00 when idle).
3. **`FamilyLedgerServiceRole`**: IAM Execution Role allowing Rust Lambdas to connect to DSQL (`dsql:DbConnect`) and publish to EventBridge (`events:PutEvents`).
4. **`FamilyLedgerMigrationRunnerRole`**: IAM Execution Role for migration execution with `dsql:DbConnectAdmin`.

---

## Step 3: Deploy Data & Event Infrastructure via SAM

We deploy this stack as `familyledger-data-dev` with `--capabilities CAPABILITY_NAMED_IAM` so that data/event infrastructure is decoupled from API/Cognito core infrastructure.

### 3.1 — Validate the Template
Run inside `backend/infrastructure/sam/`:

```powershell
sam validate -t data-stream-template.yaml --lint
```

### 3.2 — Deploy using `sam deploy`

#### Option A: Zero-Cost Deployment ($0.00 Idle Cost — Recommended for Sprint 1)
Provisions the custom EventBridge bus and IAM roles without creating Kinesis:

```powershell
sam deploy `
  --template-file data-stream-template.yaml `
  --config-file samconfig-data.toml `
  --capabilities CAPABILITY_NAMED_IAM
```
*(Uses the default in `samconfig-data.toml`: `DeployKinesisStream="false"`)*

#### Option B: Deploy with 1 Provisioned Shard ($0.36/Day — For CDC Testing)
```powershell
sam deploy `
  --template-file data-stream-template.yaml `
  --config-file samconfig-data.toml `
  --capabilities CAPABILITY_NAMED_IAM `
  --parameter-overrides `
    Environment="dev" `
    DsqlClusterIdentifier="4bublzlvvtnwjpmgtotwrulv7m" `
    DeployKinesisStream="true" `
    KinesisCapacityMode="PROVISIONED" `
    KinesisShardCount=1
```

When SAM prompts:
`Apply this changeset? [y/N]:` — type **`y`** and press Enter.

---

## Step 4: Update Local Configuration (`backend/.env.dev`)

Add the newly deployed identifiers to [`backend/.env.dev`](backend/.env.dev):

```bash
# Aurora DSQL Database
DSQL_CLUSTER_ID=4bublzlvvtnwjpmgtotwrulv7m
DSQL_ENDPOINT=4bublzlvvtnwjpmgtotwrulv7m.dsql.us-east-1.on.aws
DATABASE_URL=postgres://admin@4bublzlvvtnwjpmgtotwrulv7m.dsql.us-east-1.on.aws:5432/postgres?sslmode=require

# Event Streaming & Fanout
CDC_KINESIS_STREAM_NAME=familyledger-cdc-stream-dev
EVENT_BUS_NAME=familyledger-events-dev
```

---

## Step 5: Verification & Sanity Checks

1. **Verify DSQL Cluster connectivity**:
   ```powershell
   aws dsql get-cluster --identifier 4bublzlvvtnwjpmgtotwrulv7m --region us-east-1
   ```
2. **Verify Kinesis Stream**:
   ```powershell
   aws kinesis describe-stream --stream-name familyledger-cdc-stream-dev --region us-east-1
   ```
3. **Verify EventBridge Bus**:
   ```powershell
   aws events describe-event-bus --name familyledger-events-dev --region us-east-1
   ```
