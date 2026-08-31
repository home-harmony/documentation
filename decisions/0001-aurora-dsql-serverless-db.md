# 0001. Use Amazon Aurora DSQL for Serverless Relational Storage

* **Status**: Accepted
* **Date**: 2026-06-16
* **Deciders**: Core Engineering Team

## Context and Problem Statement
FamilyLedger requires a transactional SQL database capable of scaling to zero to minimize idle hosting costs for family users while supporting ACID transactions, high availability, full relational integrity (Foreign Keys, JSONB), and native event streaming.

## Decision Drivers
* Truly serverless pricing (scale-to-zero, free tier covering initial adoption).
* High availability and distributed active-active consistency.
* Direct Change Data Capture (CDC) streaming without database polling.
* Full PostgreSQL 16 wire compatibility, including Foreign Keys and native `JSONB`.

## Considered Options
1. **Amazon Aurora DSQL** (Distributed serverless SQL)
2. **Amazon Aurora PostgreSQL Serverless v2** (Minimum 0.5 ACU baseline cost ~$45/month)
3. **Amazon DynamoDB** (NoSQL, lacks relational querying and flexible cross-account joins)

## Decision Outcome
Chosen option: **Amazon Aurora DSQL**, because it scales to zero ($0 idle cost) with generous free tier allowances (100k DPUs + 1GB), provides native CDC to Kinesis, supports PostgreSQL 16 `FOREIGN KEY` constraints and `JSONB` data types, and guarantees multi-region active-active availability.

### Positive Consequences
* Zero infrastructure cost when idle (~$0.60/month total family footprint).
* Native row-level CDC streams to Kinesis without transactional outbox polling.
* Automatic IAM authentication and OCC retry support via `aurora-dsql-sqlx-connector`.
* Strong database-level referential integrity via `FOREIGN KEY` references.
* Schema-flexible document attributes via native `JSONB`.

### Negative Consequences / Trade-offs
* Distributed storage constraints: No triggers, no blocking synchronous index creation.
* Must enforce 1 DDL statement per migration file and use `CREATE INDEX ASYNC`.
* Must use random UUID v4/v7 PKs (`gen_random_uuid()`) to prevent distributed write hotspots.

## Compliance & Invariants
* Enforce tenant isolation and domain validation in the Rust domain layer.
* Strictly follow the 1 DDL per migration file rule.
