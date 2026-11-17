# 0017. Change Data Capture (CDC) and Event Publishing Strategies

* **Status**: Accepted
* **Date**: 2026-09-05
* **Deciders**: Engineering Team / AI Pair Programmer

---

## Context and Problem Statement

Downstream bounded contexts in FamilyLedger (`budget_service`, `planning_service`, `notification_service`) must react to state changes produced by upstream services (`ledger_service`, `cards_service`, `debt_planner_service`).

Amazon Aurora DSQL offers native Change Data Capture (CDC) streaming exclusively to **Amazon Kinesis Data Streams**. However, running a 24/7 Kinesis stream in On-Demand mode incurs an inescapable base cost of **$0.040/hour (~$28.80/month)**, and in Provisioned mode (1 shard) **$0.015/hour (~$10.80/month)**, even when completely idle and handling zero transactions. During early development (Sprints 1–3) and for low-volume family deployments, this fixed cost is undesirable for a serverless application where all other services scale to **$0.00** when idle.

Furthermore, Aurora DSQL does not support PostgreSQL triggers, logical replication slots, or `LISTEN/NOTIFY`. We need a clear architectural strategy for capturing changes and propagating events across environments.

---

## Decision Drivers

* **Cost Efficiency**: Eliminate or minimize fixed hourly idle costs in development, staging, and low-scale production environments.
* **Data Consistency**: Avoid event loss or silent desynchronization between bounded contexts.
* **Semantic Richness**: Ensure downstream services receive strongly-typed domain events with business context, rather than raw database column diffs.
* **Operational Simplicity**: Minimize moving parts (shards, fanout Lambdas, CloudWatch alarms) during early development iterations.

---

## Considered Options

1. **Option 1: Native Aurora DSQL CDC $\rightarrow$ Amazon Kinesis Data Streams**
   * Engine-level capture into Kinesis, processed by a Rust Fanout Lambda broadcasting to EventBridge.
2. **Option 2: Direct Domain Event Publishing from Application Code**
   * Axum Lambda services save to Aurora DSQL and then call `events:PutEvents` directly on Amazon EventBridge.
3. **Option 3: Transactional Outbox Pattern in Aurora DSQL**
   * Atomically insert business data and outbox event in the same DSQL transaction; poll and publish via scheduled dispatcher Lambda.
4. **Option 4: Keyset Cursor Polling**
   * Periodic batch query scanning `(occurred_at, id) > cursor` for analytical or reporting consumers.

---

## Decision Outcome

Chosen option: **Option 2 — Direct Domain Event Publishing to Amazon EventBridge** (with **Amazon Kinesis Data Stream completely disabled/removed** from SAM templates due to fixed hourly idle cost; **Option 3 — Transactional Outbox** remains the designated future upgrade path if atomic DB-to-EventBridge guarantees are later required at `$0.00` idle cost).

### Active Architecture Details:
1. **SAM Infrastructure (`data-stream-template.yaml`)**: Provisions only the custom EventBridge bus (`familyledger-events-{env}`) and least-privilege IAM execution roles. **Kinesis is removed**, keeping infrastructure idle cost at **$0.00 / month**.
2. **Application Publishing**: Rust microservice Lambdas execute atomic writes against Aurora DSQL, then immediately publish the typed `Vec<DomainEvent>` returned by aggregate methods to EventBridge via `events:PutEvents` (< 25 ms delivery).
3. **Future Upgrade Path**: If strict at-least-once transactional guarantees are needed later without paying Kinesis hourly fees, add an `outbox_events` table (Option 3) while keeping EventBridge consumers unchanged.

---

## Pros and Cons of the Options

### Option 1: Native DSQL CDC $\rightarrow$ Kinesis
* **Positive**: Captures all database mutations (including manual SQL/migrations); zero dual-write hazard; write-set compaction; 24h–365d replay window.
* **Negative**: Continuous base fee ($10.80 – $28.80/month idle); high infrastructure footprint (Kinesis + Fanout Lambda); low-level row diffs require transformation into domain events.

### Option 2: Direct Domain Event Publishing
* **Positive**: 100% serverless ($0.00 idle cost); rich semantic domain events (`Vec<DomainEvent>`); ultra-fast (<25 ms delivery); zero extra infrastructure.
* **Negative**: Dual-write hazard if Lambda crashes between DB commit and EventBridge call; does not capture direct DB migrations or manual SQL.

### Option 3: Transactional Outbox Pattern
* **Positive**: Guaranteed atomic delivery; $0.00 idle cost; resilient to EventBridge outages; typed domain event payloads.
* **Negative**: Polling delay (10–60 seconds); database write amplification (extra write + delete per transaction); requires handling DSQL's 500-row batch delete limit.

### Option 4: Keyset Cursor Polling
* **Positive**: Zero extra infrastructure; $0.00 idle cost; simple SQL query.
* **Negative**: High latency (batch only); unsuitable for real-time reactivity; wastes DSQL compute DPUs on polling.

---

## Compliance & Invariants

1. `backend/infrastructure/sam/data-stream-template.yaml` must not provision any Amazon Kinesis Data Stream resources (keeping `$0.00` idle event-bus cost).
2. All domain aggregates in `domain/` must continue returning `Vec<DomainEvent>` from mutating methods to support direct EventBridge publishing (and potential future outbox storage).
3. Downstream EventBridge consumers must enforce idempotent processing using deterministic event/entity identifiers (`idempotency_key`).

