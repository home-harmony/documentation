# Task 1.2 Detailed Guide — Database & Event Stream Provisioning

> **Goal**: Provision the **Amazon Aurora DSQL** serverless cluster, configure **IAM database authentication**, create an **Amazon Kinesis Data Stream** for native Change Data Capture (CDC), create a custom **Amazon EventBridge Bus** for semantic domain events, and define the necessary IAM roles for Rust microservices and the migration runner.

---

## What You Will Build (Mental Model)

```
              ┌─────────────────────────────────────────────────────────┐
              │                   Rust Microservices                    │
              │  (family_service, ledger_service, cards_service, etc.)   │
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
                 ▼                         ▼                         ▼
        ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
        │  Budget Service │       │  Planning Engine│       │  Notifications  │
        └─────────────────┘       └─────────────────┘       └─────────────────┘
```

---

## Prerequisites: AWS Concepts Explained

### 1. What is Amazon Aurora DSQL?
**Amazon Aurora DSQL** is a serverless, distributed, active-active relational database that provides wire compatibility with PostgreSQL 16.
* **No provisioned instances or VPC requirement**: Connects over secure HTTPS/TLS using public endpoints and AWS IAM authentication.
* **Optimistic Concurrency Control (OCC)**: Transactions commit without blocking locks. If a concurrent write conflict occurs (SQLSTATE `40001`), the transaction rolls back and is retried.
* **Native CDC**: Aurora DSQL can stream committed database mutations directly to an Amazon Kinesis Data Stream without external agents (e.g. Debezium) or polling.

### 2. What is IAM Database Authentication for DSQL?
Aurora DSQL does not use traditional database username/passwords. Instead:
* Microservices obtain a short-lived signed **IAM Authentication Token** (valid for 15 minutes) using their AWS credentials (IAM role).
* The official [`aurora-dsql-sqlx-connector`](https://crates.io/crates/aurora-dsql-sqlx-connector) crate manages token generation, auto-refresh at 80% of token lifetime, and connection pooling in Rust automatically.

### 3. What is Amazon Kinesis Data Streams?
**Amazon Kinesis Data Streams** is a real-time data streaming service.
* In FamilyLedger, it acts as the **CDC ingestion pipe** capturing every `INSERT`, `UPDATE`, and `DELETE` committed in Aurora DSQL in strict chronological sequence per partition key (`family_id`).

### 4. What is Amazon EventBridge?
**Amazon EventBridge** is a serverless event bus.
* While Kinesis carries raw row-level database mutations (data-level), EventBridge routes high-level **Semantic Domain Events** (e.g. `FamilyMemberInvited`, `TransactionRecorded`, `BudgetLimitExceeded`) to asynchronous bounded contexts.

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
    "identifier": "ab1c2d3e4f5g6h7i8j9k0l",
    "arn": "arn:aws:dsql:us-east-1:377161178071:cluster/ab1c2d3e4f5g6h7i8j9k0l",
    "status": "CREATING",
    "creationTime": "2026-08-25T00:45:00Z"
}
```

> **Note**: Save your cluster `identifier` (e.g. `ab1c2d3e4f5g6h7i8j9k0l`). The PostgreSQL endpoint format is:
> `https://<cluster-identifier>.dsql.us-east-1.on.aws:5432`

### 1.2 — Wait for Cluster Status to become `ACTIVE`
```powershell
aws dsql get-cluster --identifier <your-cluster-id> --region us-east-1
```
Ensure `"status": "ACTIVE"` before proceeding.

### 1.3 — Create the Database Admin Role in DSQL
Connect using `psql` or `aws dsql` CLI helper to initialize the `admin` and `app_user` database roles:

```powershell
# Generate a temporary IAM authentication token for admin connection
aws dsql generate-db-connect-admin-auth-token --region us-east-1 --hostname <your-cluster-id>.dsql.us-east-1.on.aws
```

Connect and run:
```sql
-- Grant application role access
CREATE ROLE familyledger_app;
GRANT ALL ON SCHEMA public TO familyledger_app;
```

---

## Step 2: Infrastructure as Code — Data & Event SAM Template

We extend the infrastructure with a dedicated data & streaming SAM template (`backend/infrastructure/sam/data-stream-template.yaml`) or combine it with our environment configuration.

### 2.1 — SAM Template: `data-stream-template.yaml`

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Description: "FamilyLedger — Database Streaming, EventBridge Bus, and IAM Execution Roles"

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
    Description: Deployment environment name

  DsqlClusterIdentifier:
    Type: String
    Description: Aurora DSQL Cluster Identifier (e.g. ab1c2d3e4f5g6h7i8j9k0l)

Resources:
  # ─── Kinesis Data Stream for Aurora DSQL CDC ─────────────────────────────────
  FamilyLedgerCdcStream:
    Type: AWS::Kinesis::Stream
    Properties:
      Name: !Sub "familyledger-cdc-stream-${Environment}"
      StreamModeDetails:
        StreamMode: ON_DEMAND
      StreamEncryption:
        EncryptionType: KMS
        KeyId: alias/aws/kinesis

  # ─── Custom EventBridge Event Bus ───────────────────────────────────────────
  FamilyLedgerEventBus:
    Type: AWS::Events::EventBus
    Properties:
      Name: !Sub "familyledger-events-${Environment}"

  # ─── IAM Role for Rust Microservices (DSQL + EventBridge) ───────────────────
  FamilyLedgerServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "familyledger-service-role-${Environment}"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: AuroraDSQLAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: "AllowDSQLDbConnect"
                Effect: Allow
                Action:
                  - dsql:DbConnect
                  - dsql:DbConnectAdmin
                Resource:
                  - !Sub "arn:aws:dsql:${AWS::Region}:${AWS::AccountId}:cluster/${DsqlClusterIdentifier}"
        - PolicyName: EventBridgePublishAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: "AllowEventBusPutEvents"
                Effect: Allow
                Action:
                  - events:PutEvents
                Resource:
                  - !GetAtt FamilyLedgerEventBus.Arn

  # ─── IAM Role for Migration Runner Lambda ────────────────────────────────────
  FamilyLedgerMigrationRunnerRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "familyledger-migration-runner-role-${Environment}"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: AuroraDSQLAdminAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: "AllowDSQLDbConnectAdmin"
                Effect: Allow
                Action:
                  - dsql:DbConnect
                  - dsql:DbConnectAdmin
                Resource:
                  - !Sub "arn:aws:dsql:${AWS::Region}:${AWS::AccountId}:cluster/${DsqlClusterIdentifier}"

Outputs:
  CdcStreamArn:
    Value: !GetAtt FamilyLedgerCdcStream.Arn
    Export:
      Name: !Sub "${AWS::StackName}-CdcStreamArn"
  CdcStreamName:
    Value: !Ref FamilyLedgerCdcStream
    Export:
      Name: !Sub "${AWS::StackName}-CdcStreamName"
  EventBusArn:
    Value: !GetAtt FamilyLedgerEventBus.Arn
    Export:
      Name: !Sub "${AWS::StackName}-EventBusArn"
  EventBusName:
    Value: !Ref FamilyLedgerEventBus
    Export:
      Name: !Sub "${AWS::StackName}-EventBusName"
  ServiceRoleArn:
    Value: !GetAtt FamilyLedgerServiceRole.Arn
    Export:
      Name: !Sub "${AWS::StackName}-ServiceRoleArn"
  MigrationRunnerRoleArn:
    Value: !GetAtt FamilyLedgerMigrationRunnerRole.Arn
    Export:
      Name: !Sub "${AWS::StackName}-MigrationRunnerRoleArn"
```

---

## Step 3: Enable Native CDC on Aurora DSQL to Kinesis

Once both the Aurora DSQL cluster and the Kinesis Stream exist, activate change data capture:

```powershell
aws dsql update-cluster `
  --identifier <your-cluster-id> `
  --region us-east-1
```
*(Or configure the CDC destination policy granting DSQL write access to `arn:aws:kinesis:us-east-1:377161178071:stream/familyledger-cdc-stream-dev`)*.

---

## Step 4: Update Local Configuration (`.env.dev`)

Update `backend/.env.dev` with the newly provisioned database and streaming endpoints:

```bash
# Aurora DSQL Database
DSQL_CLUSTER_ID=<your-cluster-id>
DSQL_ENDPOINT=<your-cluster-id>.dsql.us-east-1.on.aws
DATABASE_URL=postgres://admin@<your-cluster-id>.dsql.us-east-1.on.aws:5432/postgres?sslmode=require

# Event Streaming & Fanout
CDC_KINESIS_STREAM_NAME=familyledger-cdc-stream-dev
EVENT_BUS_NAME=familyledger-events-dev
```

---

## Step 5: Verification & Sanity Checks

1. **Verify DSQL Cluster connectivity**:
   ```powershell
   aws dsql get-cluster --identifier <your-cluster-id> --region us-east-1
   ```
2. **Verify Kinesis Stream**:
   ```powershell
   aws kinesis describe-stream --stream-name familyledger-cdc-stream-dev --region us-east-1
   ```
3. **Verify EventBridge Bus**:
   ```powershell
   aws events describe-event-bus --name familyledger-events-dev --region us-east-1
   ```
4. **Verify IAM Policy simulation**:
   Verify that `familyledger-service-role-dev` is allowed to perform `dsql:DbConnect` and `events:PutEvents`.
