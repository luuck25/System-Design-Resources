# Designing a Scalable Pipeline — Interview Deep Dive

> **Prompt:** *"How will you design a scalable pipeline that handles 2B daily fulfilment data with latency less than an hour?"*

---

## Table of Contents

1. [How to Approach This Question](#1-how-to-approach-this-question)
2. [Step 1 — Clarify Requirements](#2-step-1--clarify-requirements)
3. [Step 2 — Back-of-the-Envelope Math](#3-step-2--back-of-the-envelope-math)
4. [Step 3 — Choose an Architecture Paradigm](#4-step-3--choose-an-architecture-paradigm)
5. [Step 4 — Pipeline Components (End-to-End)](#5-step-4--pipeline-components-end-to-end)
6. [Step 5 — Scalability & Performance Strategies](#6-step-5--scalability--performance-strategies)
7. [Step 6 — Reliability & Resilience](#7-step-6--reliability--resilience)
8. [Step 7 — Observability & SLA Management](#8-step-7--observability--sla-management)
9. [Step 8 — Data Governance & Quality](#9-step-8--data-governance--quality)
10. [Step 9 — Cost Considerations](#10-step-9--cost-considerations)
11. [Why Not a 30-Minute Batch Here?](#11-why-not-a-30-minute-batch-here)
12. [When to Choose Batch Pipelines](#12-when-to-choose-batch-pipelines)
13. [When to Choose Streaming Pipelines](#13-when-to-choose-streaming-pipelines)
14. [Batch vs Streaming vs Micro-Batch — Comparison Matrix](#14-batch-vs-streaming-vs-micro-batch--comparison-matrix)
15. [Bonus — Common Follow-Up Questions](#15-bonus--common-follow-up-questions)
16. [Interview Answer Framework (STAR-ish Template)](#16-interview-answer-framework)

---

## 1. How to Approach This Question

**Interviewer intent:** They want to see structured thinking, trade-off awareness, and real-world engineering maturity — not just tool name-dropping.

**Recommended flow:**

1. **Clarify** — Ask about data shape, SLA strictness, downstream consumers, budget.
2. **Quantify** — Do quick napkin math (throughput, storage, partitions).
3. **Propose architecture** — Pick a paradigm and justify it.
4. **Walk through components** — Ingestion → Processing → Storage → Serving.
5. **Address failure modes** — Retries, idempotency, late data, backpressure.
6. **Discuss observability** — How you'll know if the SLA is at risk.
7. **Trade-offs** — What you sacrificed and why.

> **Pro tip:** Always state your assumptions out loud. Interviewers value transparent reasoning over a "perfect" answer.

---

## 2. Step 1 — Clarify Requirements

Before designing anything, ask or state assumptions about:

| Dimension | What to Clarify | Example Assumption |
|---|---|---|
| **Data shape** | Average record size? Nested/flat? | ~500 bytes → ~1 TB/day raw |
| **Latency SLA** | Is "< 1 hour" a P95 or hard ceiling? | Hard ceiling — no record older than 60 min |
| **Data sources** | Single system or multi-source? CDC or API push? | Multiple upstream services pushing events |
| **Downstream consumers** | Dashboard? ML models? APIs? | BI dashboards + downstream microservices |
| **Ordering guarantees** | Strict per-entity ordering or global? | Per-fulfillment-ID ordering is sufficient |
| **Exactly-once needs** | Can downstream tolerate duplicates? | Financial data — need exactly-once semantics |
| **Peak-to-average ratio** | Even distribution or spiky (e.g., Black Friday)? | 3–5× spikes during promotions |
| **Budget & team** | Cloud-native? Managed services preferred? | Cloud (AWS/GCP), lean team prefers managed |
| **Regulatory** | PII? GDPR? Data residency? | Contains customer PII — encryption at rest & transit |

---

## 3. Step 2 — Back-of-the-Envelope Math

This is **critical** in interviews — it shows engineering maturity.

```
Daily records     = 2,000,000,000 (2B)
Seconds per day   = 86,400
Avg throughput     = 2B / 86,400 ≈ 23,148 events/sec (~23K EPS)
Peak (5× burst)   = ~115,000 events/sec

Avg record size   = 500 bytes (assumed)
Daily volume      = 2B × 500B = 1 TB/day raw
Hourly volume     = ~42 GB/hour
Per-second        = ~11.5 MB/s sustained, ~58 MB/s peak
```

**Kafka sizing estimate (rough):**

```
Target per-partition throughput = ~5,000 msg/s (conservative)
Min partitions (sustained)     = 23,000 / 5,000 ≈ 5 partitions
Min partitions (peak)          = 115,000 / 5,000 ≈ 23 partitions
Recommended                    = 30–50 partitions (headroom + consumer parallelism)
Replication factor             = 3
Retention                      = 72 hours (for replay)
```

> **Why this matters:** It grounds the architecture in reality. An interviewer asking about 2B records wants to see you can do the math, not just say "use Kafka."

---

## 4. Step 3 — Choose an Architecture Paradigm

### Option A: Kappa Architecture (Recommended for this scenario)

```
[Sources] → [Kafka] → [Flink / Spark Streaming] → [Bronze] → [Silver] → [Gold] → [Serving]
```

- **Single processing path** — all data treated as an unbounded stream.
- Historical reprocessing = replay from Kafka (with sufficient retention) or from Bronze layer.
- **Simpler codebase** — one engine, one logic path.
- **Best for:** Sub-hour latency with high throughput where the primary access pattern is real-time.

### Option B: Lambda Architecture

```
Speed Layer:  [Kafka] → [Flink]  → [Real-Time Views]
Batch Layer:  [S3/GCS] → [Spark Batch] → [Batch Views]
Serving:      Merge both layers at query time
```

- **Dual path** — speed layer for low-latency, batch layer for "source of truth."
- **Best for:** Financial reconciliation where you need a batch layer as a safety net for long-term accuracy.
- **Downside:** Maintaining two codepaths (one batch, one stream) is operationally expensive. Logic drift between layers is a real risk.

### Option C: Hybrid Micro-Batch (Practical Middle Ground)

```
[Sources] → [Kafka] → [Spark Structured Streaming, trigger=30s] → [Delta Lake / Iceberg] → [Serving]
```

- Uses Spark Structured Streaming with short trigger intervals (e.g., 30 seconds).
- Easier for teams already invested in Spark ecosystem.
- Delivers near-real-time latency (seconds to low minutes) without full streaming complexity.

**Recommendation for this scenario:** Kappa with Flink or Hybrid Micro-Batch with Spark Structured Streaming, depending on team expertise.

---

## 5. Step 4 — Pipeline Components (End-to-End)

### 5.1 Data Ingestion Layer

| Component | Purpose |
|---|---|
| **Apache Kafka** | Distributed commit log; decouples producers/consumers; durable; replayable |
| **Amazon Kinesis / Azure Event Hubs** | Managed alternatives to Kafka |
| **Schema Registry (Confluent)** | Enforces Avro/Protobuf schemas; prevents bad data from entering the pipeline |
| **CDC (Debezium)** | If sources are databases, use Change Data Capture to stream row-level changes without polling |

**Key design choices:**
- **Serialization format:** Avro (compact, schema-evolved) or Protobuf (strongly typed, fast). Avoid JSON at this scale — it's verbose and schema-less.
- **Topic design:** One topic per entity type (e.g., `fulfillment.events`, `fulfillment.status-updates`). Partition by `fulfillment_id` for ordering guarantees per entity.
- **Producer acknowledgments:** `acks=all` for durability; ensures the message is written to all replicas before the producer gets a success response.

### 5.2 Stream Processing Engine

| Engine | Strengths | Latency | Best For |
|---|---|---|---|
| **Apache Flink** | True record-at-a-time, advanced state management, event-time processing, savepoints | Milliseconds | Ultra-low latency, complex event processing |
| **Spark Structured Streaming** | Unified batch+stream API, mature ecosystem, Delta Lake integration | Seconds–minutes | Teams with Spark expertise, micro-batch acceptable |
| **Apache Beam (Dataflow)** | Portable across runners (Flink, Spark, Dataflow), good for GCP | Varies by runner | Multi-cloud portability |
| **Kafka Streams** | Library (not a cluster), embedded in microservices | Milliseconds | Lightweight, per-service stream processing |

**Processing logic at this layer:**
- **Filtering:** Drop invalid/malformed records.
- **Enrichment:** Join with dimension tables (e.g., warehouse lookup, product catalog).
- **Aggregation:** Rolling counts, sums, averages per time window.
- **Deduplication:** Use a stateful dedup window keyed on `fulfillment_id` + `event_timestamp`.

### 5.3 Medallion Architecture (Data Layers)

```
┌──────────┐      ┌──────────┐      ┌──────────┐
│  BRONZE  │ ──→  │  SILVER  │ ──→  │   GOLD   │
│  (Raw)   │      │ (Cleaned)│      │(Business)│
└──────────┘      └──────────┘      └──────────┘
```

| Layer | What Happens | Storage Format | Example |
|---|---|---|---|
| **Bronze** | Raw ingestion, append-only, no transformations | Parquet / Delta / Iceberg on S3/GCS | Raw Kafka messages as-is |
| **Silver** | Schema validation, dedup, joins, null handling, type casting | Delta / Iceberg (partitioned) | Cleaned fulfillment records joined with warehouse dim |
| **Gold** | Business-level aggregations, KPIs, SLA metrics | Delta / Iceberg (optimized for reads) | Hourly fulfillment counts by region, SLA compliance % |

**Why Medallion matters in interviews:** It shows you think about data quality progressively, not as a single monolithic ETL.

### 5.4 Storage / Sink Layer

| Destination | Use Case |
|---|---|
| **Delta Lake / Apache Iceberg** | Lakehouse — ACID transactions, time travel, schema evolution on object storage |
| **BigQuery / Snowflake / Redshift** | Analytical queries, BI dashboards |
| **Elasticsearch / OpenSearch** | Full-text search, operational dashboards |
| **Redis / DynamoDB** | Low-latency serving layer for APIs |
| **Another Kafka Topic** | Downstream microservice consumption |

### 5.5 Serving / Consumption Layer

- **BI Tools:** Looker, Tableau, Power BI querying Gold layer.
- **APIs:** REST/gRPC services reading from Redis/DynamoDB for real-time lookups.
- **Reverse ETL:** Push aggregated data back to operational systems (e.g., CRM, order management).

---

## 6. Step 5 — Scalability & Performance Strategies

### 6.1 Horizontal Partitioning (Sharding)

- **Kafka partitions:** Data partitioned by `hash(fulfillment_id) % N`. Ensures all events for a single fulfillment land on the same partition → preserves ordering per entity.
- **Processing parallelism:** Each Flink/Spark task consumes one or more partitions. Scale consumers = scale partitions.
- **Storage partitioning:** Partition output by `event_date` and `region` (or `warehouse_id`) for efficient downstream queries.

### 6.2 Auto-Scaling

- **Kubernetes (K8s):** HPA (Horizontal Pod Autoscaler) based on Kafka consumer lag or CPU utilization.
- **AWS EMR / GCP Dataproc:** Auto-scaling managed Spark clusters.
- **Flink on K8s:** Reactive scaling — Flink adjusts parallelism based on backlog.
- **Serverless options:** AWS Lambda + Kinesis for lightweight enrichments (careful with cold starts at scale).

### 6.3 Backpressure Handling

- **Flink:** Built-in credit-based backpressure propagates from sink to source. If a downstream operator is slow, upstream operators naturally slow down.
- **Spark:** Rate limiting via `maxOffsetsPerTrigger` to cap how many records are pulled per micro-batch.
- **Kafka:** Consumer `fetch.max.bytes` and `max.poll.records` to control pull rate.

### 6.4 Data Compaction & Optimization

- **Small file problem:** Streaming writes many small files. Use compaction jobs (Delta `OPTIMIZE`, Iceberg `rewrite_data_files`) on a schedule.
- **Z-ordering / Clustering:** Co-locate related data on disk for faster downstream reads.
- **Caching:** Cache hot dimension tables in processing engine memory (Flink broadcast state, Spark broadcast joins).

---

## 7. Step 6 — Reliability & Resilience

### 7.1 Delivery Semantics

| Guarantee | Meaning | How to Achieve |
|---|---|---|
| **At-most-once** | May lose data | Fire-and-forget producer, no retries |
| **At-least-once** | May duplicate | Producer retries + consumer commits after processing |
| **Exactly-once** | No loss, no dups | Kafka transactions + Flink checkpointing + idempotent sinks |

**For fulfillment data → exactly-once is the target.**

### 7.2 Checkpointing & Savepoints

- **Flink checkpoints:** Periodic snapshots of operator state to durable storage (S3/HDFS). On failure, restart from last checkpoint — no data loss, no reprocessing of entire window.
- **Spark checkpointing:** Write-ahead logs + offset tracking in checkpoint directory.
- **Kafka consumer offsets:** Committed only after successful processing (`enable.auto.commit=false`).

### 7.3 Idempotent Writes

- **Delta Lake / Iceberg:** Use `MERGE INTO` (upsert) keyed on `fulfillment_id` + `event_id` → retries don't produce duplicates.
- **Partition overwrite:** `INSERT OVERWRITE` on a time partition ensures re-runs replace rather than append.
- **Database sinks:** Use `ON CONFLICT DO UPDATE` (PostgreSQL) or equivalent.

### 7.4 Dead Letter Queues (DLQ)

- Route poison messages (unparseable, schema-violating, or transformation-failing records) to a separate Kafka topic or S3 bucket.
- **Why it matters:** A single bad record should never halt the entire pipeline. DLQ allows the pipeline to continue while engineers investigate failures offline.

### 7.5 Disaster Recovery & Replay

- **Kafka retention:** Keep 72+ hours of raw data for replay.
- **Bronze layer:** Serves as a permanent replay source. If processing logic changes, reprocess from Bronze.
- **Multi-region:** For critical SLAs, replicate Kafka (MirrorMaker 2) and storage across regions.

---

## 8. Step 7 — Observability & SLA Management

### Key Metrics to Monitor

| Metric | What It Tells You | Alert Threshold |
|---|---|---|
| **Consumer lag** | How far behind processing is from ingestion | > 5 minutes of lag |
| **End-to-end latency** | Time from event generation to availability in Gold layer | > 45 minutes (warning), > 55 minutes (critical) |
| **Throughput (records/sec)** | Whether the pipeline keeps pace with ingestion | Drop > 20% from baseline |
| **Checkpoint duration** | Health of state management | > 60 seconds |
| **Error rate** | Records landing in DLQ | > 0.1% of throughput |
| **Backpressure ratio** | Whether operators are bottlenecked | Sustained > 50% |

### Tooling

- **Metrics:** Prometheus + Grafana (or Datadog, CloudWatch).
- **Logging:** Structured logs → ELK Stack / Splunk.
- **Tracing:** OpenTelemetry for distributed tracing across pipeline stages.
- **Alerting:** PagerDuty / OpsGenie integration for SLA breach warnings.
- **Data quality:** Great Expectations or Deequ for automated data validation checks at each layer.

---

## 9. Step 8 — Data Governance & Quality

Often overlooked in interviews but demonstrates senior-level thinking:

- **Schema Registry:** Enforces backward/forward compatibility. Producers cannot push breaking schema changes without approval. Prevents downstream consumer breakage.
- **Schema Evolution:** Use Avro or Protobuf with schema registry to add fields without breaking consumers (backward-compatible changes).
- **Data Lineage:** Tools like Apache Atlas, OpenLineage, or Marquez track where data comes from and where it flows — critical for debugging and compliance.
- **PII Handling:** Tokenize or encrypt PII fields at the Bronze layer. Use column-level access controls in the serving layer.
- **Data Contracts:** Formalize agreements between producer and consumer teams on schema, SLAs, and data quality expectations.

---

## 10. Step 9 — Cost Considerations

| Component | Cost Driver | Optimization |
|---|---|---|
| **Kafka / Event Hub** | Partition count, retention, throughput | Right-size partitions; use tiered storage for old data |
| **Compute (Flink/Spark)** | Cluster size, always-on vs auto-scaled | Auto-scale; use spot/preemptible instances for non-critical layers |
| **Storage (S3/GCS)** | Volume (1 TB/day raw) | Columnar formats (Parquet); compression (Zstd/Snappy); lifecycle policies to archive/delete old data |
| **Serving (BigQuery/Redshift)** | Query volume, scanned bytes | Partition pruning; clustering; materialized views |

**Streaming vs Batch cost trade-off:**
- Streaming has higher **steady-state compute cost** (always-on cluster).
- Batch has higher **peak compute cost** (massive burst, then idle).
- At 2B records/day, streaming's even resource utilization is typically more cost-effective than batch's burst pattern.

---

## 11. Why Not a 30-Minute Batch Here?

While technically possible, a batch pipeline on a 30-minute schedule has serious risks at this scale:

### 11.1 The "Thundering Herd" Problem

- A 30-minute window means each batch processes **~41.6 million records** in a burst.
- Requires spinning up massive compute resources for each burst, then idling between runs.
- Streaming spreads the same load **evenly at ~23K events/sec**, avoiding resource spikes.

### 11.2 Zero Margin for Failure

```
Timeline with 30-min batch + 1-hour SLA:

00:00 ─── Data starts arriving ───
00:30 ─── Batch job triggers ───
00:30–01:00 ─── Processing (best case) ───
01:00 ─── Data available ─── ✅ Barely meets SLA

If the job FAILS at 00:45:
00:45 ─── Failure detected ───
00:50 ─── Retry starts ───
01:20 ─── Retry completes ─── ❌ SLA breached
```

- One failure + one retry = SLA breach. There is **no buffer**.
- Streaming with checkpoint-based recovery resumes from the exact failure point in seconds, not minutes.

### 11.3 Late-Arriving Data

- Fulfillment data commonly arrives late (network delays, retries from upstream).
- **Batch:** Late data from Window 1 arriving in Window 2 is only corrected in Window 3's run → could exceed 1-hour visibility for those records.
- **Streaming:** Watermarking and event-time processing handle late data gracefully, updating aggregates in near-real-time.

### 11.4 Redundant Reprocessing

- Batch jobs often use full-partition overwrites or re-scan entire windows for consistency.
- Streaming tracks offsets and processes **only new data** — far more compute-efficient at 2B records/day.

### 11.5 Small File Problem (Compounded)

- High-frequency batch (every 30 min) produces many small output files per partition per run.
- Requires additional compaction jobs to merge small files — adding complexity and latency.

### When 30-Min Batch IS Acceptable

- Volume is moderate (millions, not billions per day).
- SLA is lenient (2–4 hours, not strict sub-hour).
- Team lacks streaming expertise and the business can tolerate occasional SLA misses.
- Data arrives in large, predictable dumps (not continuous stream).

---

## 12. When to Choose Batch Pipelines

Batch is not dead — it's the right choice in many scenarios:

### 12.1 High Latency Tolerance

- Acceptable delay of **hours or days** (e.g., monthly sales reports, quarterly reviews).
- No real-time consumer depends on the output.

### 12.2 Complex Historical / Analytical Workloads

- **Backfills:** Reprocessing years of historical data.
- **ML training:** Feature engineering over large historical datasets.
- **Complex joins:** Multi-way joins across massive tables where streaming state would be impractically large.

### 12.3 Financial Reconciliation & Regulatory Compliance

- Need **point-in-time snapshots** that are reproducible and auditable.
- Re-running the same batch on the same input must yield identical results (deterministic).

### 12.4 Simplicity & Lower Operational Overhead

- Easier to reason about: input → transform → output. No state management, no watermarks, no checkpoints.
- Cheaper to maintain for small/medium teams.
- Scheduling tools (Airflow, Dagster, Prefect) provide mature orchestration with dependency management and retry logic.

### 12.5 Data Warehouse / Lakehouse ETL

- Loading data into warehouses (BigQuery, Snowflake, Redshift) on a schedule.
- Dimension table builds, slowly changing dimensions (SCD Type 2).
- Data quality validation runs over complete datasets.

### 12.6 Cost Efficiency for Intermittent Workloads

- Use spot instances / preemptible VMs for batch jobs.
- Clusters spin up, process, and shut down — no idle cost between runs.
- Off-peak scheduling reduces cloud costs.

---

## 13. When to Choose Streaming Pipelines

### 13.1 Low Latency Requirements

- SLA demands data freshness in **seconds to minutes** (dashboards, alerting, fraud detection).

### 13.2 Continuous, Unbounded Data

- Data arrives continuously with no natural "end" (IoT sensors, clickstreams, event logs, CDC).

### 13.3 Event-Driven Architectures

- Downstream systems react to events in real-time (order status updates, inventory changes, notifications).

### 13.4 High Volume with Even Processing

- At extreme volumes (billions/day), streaming's continuous processing is more resource-efficient than batch's burst pattern.

### 13.5 Operational Use Cases

- **Real-time fraud detection** — must act within seconds.
- **Dynamic pricing** — adjust prices based on live demand signals.
- **Live dashboards / monitoring** — operational health, SLA tracking.
- **Recommendation engines** — update recommendations based on recent user behavior.

---

## 14. Batch vs Streaming vs Micro-Batch — Comparison Matrix

| Dimension | Batch | Micro-Batch | Streaming |
|---|---|---|---|
| **Latency** | Hours–days | Seconds–minutes | Milliseconds–seconds |
| **Throughput** | Very high (bounded) | High | Very high (continuous) |
| **Complexity** | Low | Medium | High |
| **State management** | Simple (stateless transforms) | Moderate | Complex (windowing, watermarks) |
| **Failure recovery** | Re-run entire job | Re-run micro-batch | Checkpoint-based resume |
| **Late data handling** | Next batch run | Next trigger | Watermarks + allowed lateness |
| **Resource pattern** | Burst (spike then idle) | Small periodic spikes | Steady-state |
| **Cost model** | Pay per run (spot-friendly) | Moderate steady cost | Higher steady cost |
| **Best tools** | Spark Batch, Hive, dbt | Spark Structured Streaming | Flink, Kafka Streams |
| **Team skill needed** | Lower | Medium | Higher |
| **Example SLA** | "Updated by 8 AM daily" | "Fresher than 5 minutes" | "Available within 10 seconds" |

---

## 15. Bonus — Common Follow-Up Questions

### Q: How does topic-based routing work at the broker level?

Topic-based routing directs messages to specific Kafka topics based on content or metadata:

- **Producer-side routing:** The producer inspects the event type (e.g., `order.created`, `order.shipped`) and publishes to the corresponding topic.
- **Custom partitioner:** Implement a `Partitioner` interface to route within a topic based on key attributes.
- **Kafka Streams / Flink branching:** Use `side outputs` (Flink) or `branch()` (Kafka Streams) to route a single input stream into multiple output topics based on predicates.
- **Header-based routing:** Kafka message headers can carry metadata (e.g., `source-region`, `priority`) that consumers or Kafka Connect SMTs use for routing decisions.

### Q: What is stateless vs stateful filtering?

| Aspect | Stateless Filtering | Stateful Filtering |
|---|---|---|
| **Definition** | Each record evaluated independently | Decision depends on previously seen records |
| **Example** | Drop records where `status = 'TEST'` | Deduplicate based on `fulfillment_id` seen in the last 1 hour |
| **State needed** | None | Keyed state (hash map, RocksDB backend) |
| **Complexity** | Trivial | Must manage state size, TTL, and checkpointing |
| **Performance** | Very fast, horizontally scalable | Bounded by state backend I/O |
| **Flink example** | `.filter(e -> !e.isTest())` | `.keyBy(id).process(new DeduplicationFunction())` |

### Q: How does a Schema Registry prevent unwanted data consumption?

- **Contract enforcement:** Producers must register schemas before publishing. If a schema is not compatible (e.g., removing a required field), the registry rejects it.
- **Compatibility modes:** `BACKWARD` (new schema can read old data), `FORWARD` (old schema can read new data), `FULL` (both). Prevents producers from making breaking changes.
- **Consumer validation:** Consumers deserialize using the schema ID embedded in the message. If the schema is unknown or incompatible, deserialization fails gracefully.
- **Access control:** Schema registry can enforce ACLs — only authorized producers can write to specific subjects (topics).

### Q: How would you handle schema evolution?

- Add new optional fields (backward-compatible).
- Never remove or rename fields without a deprecation cycle.
- Use Avro `default` values so old consumers can read new records.
- Version your topics if a breaking change is unavoidable (`fulfillment.events.v2`).

### Q: How do you handle exactly-once end-to-end?

Three components must align:
1. **Source:** Kafka `enable.idempotence=true` + transactions.
2. **Processing:** Flink's exactly-once checkpointing (barrier alignment).
3. **Sink:** Idempotent writes (`MERGE`/upsert) or transactional sink connectors.

### Q: What windowing strategies would you use?

| Window Type | Use Case | Example |
|---|---|---|
| **Tumbling** | Fixed, non-overlapping intervals | Count fulfillments per 15-minute window |
| **Sliding** | Overlapping windows for smoothing | Average delivery time over last 1 hour, updated every 5 minutes |
| **Session** | Activity-based, gap-triggered | Group all events for a single fulfillment session |
| **Global** | Entire stream (with triggers) | Running total of all fulfillments today |

---

## 16. Interview Answer Framework

Use this ~3 minute structured template:

### Opening (30 sec)

> "Before jumping in, let me clarify a few assumptions: 2B records at ~500 bytes each gives us roughly 1 TB/day and ~23K events/sec sustained. With sub-hour latency, this rules out traditional batch and points toward a streaming or high-frequency micro-batch architecture."

### Architecture (60 sec)

> "I'd propose a **Kappa architecture** using Kafka for ingestion and Flink (or Spark Structured Streaming) for processing, organized in a **Medallion pattern** — Bronze for raw append, Silver for cleaned/enriched data, Gold for business aggregates. Data lands in a lakehouse format like Delta Lake or Iceberg for both streaming writes and analytical reads."

### Scalability (45 sec)

> "Kafka topics partitioned by fulfillment ID — roughly 30–50 partitions for parallelism headroom. Processing scales horizontally with partition count. Auto-scaling on Kubernetes responds to consumer lag. Backpressure is handled natively by Flink's credit-based flow control."

### Reliability (30 sec)

> "Exactly-once semantics via Kafka transactions + Flink checkpointing + idempotent MERGE writes. Dead letter queues for poison messages. Bronze layer serves as a permanent replay source. Checkpoints every 30 seconds so recovery after failure takes seconds, not minutes."

### Observability (15 sec)

> "Prometheus + Grafana monitoring consumer lag, end-to-end latency, error rates. Alert if latency exceeds 45 minutes — giving 15 minutes of buffer before SLA breach."

### Close with Trade-offs (15 sec)

> "The main trade-off is operational complexity versus a simpler batch approach. But at 2B records with a strict sub-hour SLA, the resilience and efficiency of streaming justify the investment."

---

## Quick Reference — Technology Cheat Sheet

| Layer | Recommended | Alternatives |
|---|---|---|
| **Ingestion** | Apache Kafka | Amazon Kinesis, Azure Event Hubs, Google Pub/Sub |
| **Schema** | Confluent Schema Registry (Avro) | AWS Glue Schema Registry, Protobuf |
| **CDC** | Debezium | AWS DMS, Striim, Fivetran |
| **Stream Processing** | Apache Flink | Spark Structured Streaming, Kafka Streams, Apache Beam |
| **Storage** | Delta Lake on S3/GCS | Apache Iceberg, Apache Hudi |
| **Warehouse** | BigQuery / Snowflake | Redshift, Databricks SQL |
| **Orchestration** | (Streaming: N/A — always on) | Airflow, Dagster, Prefect (for batch layers) |
| **Monitoring** | Prometheus + Grafana | Datadog, CloudWatch, New Relic |
| **Data Quality** | Great Expectations / Deequ | Monte Carlo, Soda |
| **Lineage** | OpenLineage / Marquez | Apache Atlas, Datahub |

---

*Last updated: April 2025*
