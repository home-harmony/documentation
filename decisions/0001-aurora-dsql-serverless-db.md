# 0001. Use Amazon Aurora DSQL for Serverless Relational Storage

* **Status**: Accepted
* **Date**: 2026-06-16
* **Deciders**: Core Engineering Team

## Context and Problem Statement
FamilyLedger requires a transactional SQL database capable of scaling to zero to minimize idle hosting costs for family users while supporting ACID transactions, high availability, and native event streaming.

## Decision Drivers
* Truly serverless pricing (scale-to-zero, free tier covering initial adoption).
* High availability and distributed active-active consistency.
* Direct Change Data Capture (CDC) streaming without database polling.
* PostgreSQL 16 wire compatibility.

## Considered Options
1. **Amazon Aurora DSQL** (Distributed serverless SQL)
2. **Amazon Aurora PostgreSQL Serverless v2** (Minimum 0.5 ACU baseline cost ~\$45/month)
3. **Amazon DynamoDB** (NoSQL, lacks relational querying and flexible cross-account joins)

## Decision Outcome
Chosen option: **Amazon Aurora DSQL**, because it scales to zero ($0 idle cost) with generous free tier allowances (100k DPUs + 1GB), provides native CDC to Kinesis, and guarantees multi-region active-active availability.

### Positive Consequences
* Zero infrastructure cost when idle (~$0.60/month total family footprint).
* Native row-level CDC streams to Kinesis without transactional outbox polling.
* Automatic IAM authentication and OCC retry support via `aurora-dsql-sqlx-connector`.

### Negative Consequences / Trade-offs
* Distributed storage constraints: No DB-enforced foreign keys, no triggers, no `ON DELETE CASCADE`.
* Must enforce 1 DDL statement per migration file and use `CREATE INDEX ASYNC`.
* Must use random UUID v4/v7 PKs to prevent distributed write hotspots.

## Compliance & Invariants
* Enforce referential integrity and tenant isolation in the Rust domain layer.
* Strictly follow the 1 DDL per migration file rule.

