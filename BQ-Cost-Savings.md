# BigQuery Cost Optimization — All Google-Provided Levers

BigQuery costs come from two buckets: **compute (analysis)** and **storage**. Here's every optimization lever Google provides.

---

## 1. Pricing Model Choice

### On-Demand vs Capacity (Slots)

| Model | How It Works | Best For |
| :--- | :--- | :--- |
| **On-Demand** | Pay per TB scanned ($6.25/TB) | Unpredictable, low-volume workloads |
| **Standard Edition** | Pay-as-you-go slots (auto-scales) | Variable workloads, no commitment |
| **Enterprise Edition** | 1-year commitment slots | Steady, high-volume workloads |
| **Enterprise Plus** | 3-year commitment slots | Maximum discount, predictable workloads |

**Savings:** Editions with commitments can be **60–70% cheaper** than on-demand for heavy users.

**Strategic Break-Even:** Transition from On-Demand to Editions if your monthly scanning regularly exceeds **400–500 TiB**.

### Flex Slots (Short-Term)

- Buy slots for as little as **60 seconds** for burst workloads
- No long-term commitment — great for overnight batch jobs

### Idle Slot Sharing (Enterprise Edition)

Unused capacity from one reservation **automatically covers bursts** in another reservation. This means you don't pay for idle slots in one project while another project is throttled.

### Autoscaling Caps

BigQuery autoscales aggressively to finish jobs fast. If your ETL jobs **aren't time-sensitive**, set lower autoscaling caps to avoid paying for sub-minute completion when 10 minutes is perfectly acceptable.

```
Reservation: etl_nightly
  Baseline slots: 100
  Max autoscale slots: 200  ← cap this instead of letting it go to 2000
```

---

## 2. Reduce Data Scanned (Biggest Lever)

Since on-demand charges per TB scanned, reading less data = lower cost.

### 2.1 Partitioning

Split tables into segments so queries only scan relevant partitions:

```sql
-- Table partitioned by date
CREATE TABLE sales
PARTITION BY DATE(order_date)
AS SELECT * FROM raw_sales;

-- This only scans one partition, not the whole table
SELECT * FROM sales WHERE order_date = '2025-01-01';
```

**Partition types:** By ingestion time, DATE/TIMESTAMP/DATETIME column, integer range.

### 2.2 Clustering

Within each partition, **physically sort rows** by frequently filtered columns:

```sql
CREATE TABLE sales
PARTITION BY DATE(order_date)
CLUSTER BY region, product_category
AS SELECT * FROM raw_sales;

-- Clustering lets BQ skip irrelevant data blocks
SELECT * FROM sales
WHERE order_date = '2025-01-01' AND region = 'APAC';
```

**Key:** Cluster by columns you filter/group on most. Up to 4 clustering columns.

### 2.3 SELECT Only What You Need

```sql
-- BAD: scans every column
SELECT * FROM big_table;

-- GOOD: scans only 2 columns
SELECT customer_id, revenue FROM big_table;
```

BigQuery is **columnar** — each column you add costs money.

**Use `SELECT * EXCEPT` when you need most columns:**

```sql
-- Exclude only the heavy columns you don't need
SELECT * EXCEPT(raw_payload, debug_log) FROM events;
```

### 2.4 LIMIT Does NOT Reduce Cost (Gotcha!)

A common mistake — `LIMIT` does **not** reduce bytes scanned on non-clustered tables:

```sql
-- ❌ Still scans the ENTIRE table, then truncates output
SELECT * FROM big_table LIMIT 10;  -- billed for full scan!
```

BigQuery scans first, formats output second. `LIMIT` only limits the output rows, not the scan. Use **preview** or **clustering** instead.

### 2.5 Require Partition Filter (Force Filters)

This is the most direct way to **prevent queries from running without a filter**. When enabled, any query without a `WHERE` clause on the partition column **fails immediately** rather than performing a full scan:

```sql
-- Enable on table creation
CREATE TABLE sales
PARTITION BY DATE(order_date)
OPTIONS (require_partition_filter = TRUE)
AS SELECT * FROM raw_sales;

-- Or enable on existing table
ALTER TABLE sales
SET OPTIONS (require_partition_filter = TRUE);
```

```sql
-- ❌ FAILS: no partition filter
SELECT * FROM sales;
-- Error: Cannot query over table 'sales' without a filter on partition column(s)

-- ✅ WORKS: includes partition filter
SELECT * FROM sales WHERE order_date = '2025-01-01';
```

> This is the **#1 governance hack** — enable it on every large partitioned table.

### 2.6 Free Data Preview (Zero Cost Exploration)

Instead of running `SELECT * LIMIT 10` (which still triggers a full scan), use these **completely free** preview methods:

| Method | How | Cost |
| :--- | :--- | :--- |
| **Console Preview Tab** | Table details → click **Preview** tab | Free |
| **CLI: `bq head`** | `bq head -n 10 project:dataset.table` | Free |
| **API: `tabledata.list`** | REST call to retrieve rows directly | Free |

- No bytes billed
- No quota impact
- Best practice for initial data checks before writing queries

---

## 3. Materialized Views

Pre-compute expensive aggregations. BigQuery **auto-refreshes** them and the query optimizer **auto-rewrites** queries to use them:

```sql
CREATE MATERIALIZED VIEW sales_summary AS
SELECT region, DATE(order_date) AS day, SUM(revenue) AS total_revenue
FROM sales
GROUP BY region, day;

-- BQ automatically uses the MV even if you query the base table
SELECT region, SUM(revenue) FROM sales GROUP BY region;
```

**Savings:** Avoids re-scanning the base table for repeated aggregation patterns. You are charged only for the data in the view, not the massive source tables.

---

## 3.5 Search Indexes

For **"needle-in-a-haystack"** text searches in logs, JSON, or string columns, create a search index:

```sql
CREATE SEARCH INDEX my_index ON events(ALL COLUMNS);

-- Now this is orders of magnitude faster and cheaper
SELECT * FROM events WHERE SEARCH(raw_log, 'error_code_502');
```

**Impact:** In one case study, a genomic data query's slot time dropped from **1 hour to 664 milliseconds** after indexing. Scanned bytes dropped from GBs to MBs.

---

## 4. Caching (Automatic & Free)

BigQuery **caches query results for 24 hours** (per user). If you run the same query again, it costs **$0**.

**Conditions:** Same query text, table hasn't changed, not using non-deterministic functions.

---

## 5. Storage Optimization

### 5.1 Long-Term Storage (Automatic)

Any table/partition **not modified for 90 days** automatically moves to long-term storage pricing:

| Storage Type | Price |
| :--- | :--- |
| **Active** | ~$0.02/GB/month |
| **Long-term** (90+ days untouched) | ~$0.01/GB/month (50% cheaper) |

This is **automatic** — no action needed.

**Pro tip:** Avoid "overwriting" entire tables. Instead, **load new data into new partitions** to keep old partitions untouched — this preserves their long-term storage status and the 50% discount.

### 5.2 Table Expiration

Set auto-delete on temp/staging tables:

```sql
CREATE TABLE temp_results
OPTIONS (expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 7 DAY))
AS SELECT * FROM ...;
```

### 5.3 Partition Expiration

Auto-drop old partitions:

```sql
ALTER TABLE sales
SET OPTIONS (partition_expiration_days = 365);
-- Partitions older than 1 year are automatically deleted
```

### 5.4 Drop Unused Tables / Datasets

Use `INFORMATION_SCHEMA` to find tables nobody queries:

```sql
SELECT table_id, row_count, size_bytes
FROM `project.dataset.__TABLES__`
ORDER BY size_bytes DESC;
```

Tables not queried in 90+ days are candidates for **deletion** or **archival to Cloud Storage Coldline** class (much cheaper).

---

## 6. BI Engine (In-Memory Acceleration)

A **dedicated in-memory cache** for BI workloads. Pre-loads frequently accessed data into RAM for **sub-second query responses**.

- Reserve 1–250 GB of memory
- Works transparently with Looker, Data Studio, Tableau, etc.
- Queries hitting BI Engine are **not charged** on-demand bytes (stages accelerated by BI Engine scan **zero bytes** from storage)

---

## 7. Query Optimization Techniques

### 7.1 Avoid Cross Joins & Cartesian Products

```sql
-- BAD: explodes row count
SELECT * FROM table_a, table_b;

-- GOOD: always use explicit JOINs with conditions
SELECT * FROM table_a JOIN table_b ON table_a.id = table_b.id;
```

### 7.2 Use Approximate Functions

```sql
-- Exact (expensive on huge tables)
SELECT COUNT(DISTINCT user_id) FROM events;

-- Approximate (~2% error, much cheaper/faster)
SELECT APPROX_COUNT_DISTINCT(user_id) FROM events;
```

Also: `APPROX_QUANTILES`, `APPROX_TOP_COUNT`, `APPROX_TOP_SUM`.

### 7.3 Filter Early with WHERE

Push filters as early as possible — especially on partition/cluster columns.

### 7.4 Avoid Repeated Subqueries — Use CTEs or Temp Tables

```sql
-- Instead of scanning the same table multiple times in subqueries
WITH base AS (
  SELECT * FROM huge_table WHERE date = '2025-01-01'
)
SELECT ... FROM base JOIN ...;
```

### 7.5 Use INT64 Keys Instead of STRING Joins

Integer comparisons are **faster and cheaper** than string comparisons.

### 7.6 Use Nested/Repeated Fields to Eliminate Joins

For many-to-one relationships, denormalize using **STRUCTs and ARRAYs**:

```sql
-- ❌ Requires JOIN every time
SELECT o.order_id, li.product_id, li.quantity
FROM orders o JOIN line_items li ON o.order_id = li.order_id;

-- ✅ No JOIN needed — line items stored inside orders
SELECT order_id, item.product_id, item.quantity
FROM orders, UNNEST(line_items) AS item;
```

Eliminates JOIN cost and reduces storage.

---

## 8. Cost Controls & Guardrails

### 8.1 Custom Cost Controls (Query-Level)

```sql
-- Fail if query would scan more than 10 GB
SET @@dataset_project_id.max_bytes_billed = 10737418240;
```

Or set in the console: **More → Query Settings → Maximum bytes billed**.

### 8.2 Project-Level & User-Level Quotas (Hard Caps)

Set daily limits to enforce a **hard spend ceiling**:

- **Project-level quota:** Limits total data processed across the entire project per day
- **User-level quota:** Limits how much a specific individual can process per day

Once the limit is hit, **all subsequent queries fail** until the quota resets the next day. Prevents "bill shock" from runaway automated processes.

### 8.3 Dry Runs / Query Validator (Zero-Cost Pre-Check)

Require analysts to perform a **dry run** before executing large jobs:

```bash
# CLI: dry run returns bytes to be scanned at zero cost
bq query --dry_run 'SELECT col1, col2 FROM big_table WHERE date = "2025-01-01"'
```

Or use the **Query Validator** in the console (top-right of the editor) — it shows estimated bytes before you click Run. **Zero cost, zero quota impact.**

### 8.4 Slot Reservations & Assignments

In Editions pricing, create **reservations** and assign them to projects/folders/orgs. This caps compute usage per team.

### 8.5 Row-Level Security (Implicit Cost Saver)

While primarily a security feature, **Row-Level Security (RLS)** acts as a permanent, automatic filter:

- Define access policies that restrict which rows a user can see
- Even if a user runs `SELECT *` without a filter, BQ **silently filters** results to only permitted rows
- This reduces data processed for that specific user — lowering their compute cost

---

## 9. Scheduling & Batch Optimization

### 9.1 Scheduled Queries

Run heavy queries during off-peak hours. Combined with Flex Slots, you buy cheap capacity at night and release it.

### 9.2 Batch Priority

BigQuery offers **`BATCH` priority** — queries are queued and run when resources are available, reducing contention with interactive queries. Only available with Editions pricing.

---

## 10. Data Format & Ingestion

### 10.1 Use Nested/Repeated Fields (STRUCT, ARRAY)

Instead of flattening into wide tables, use nested structures. Reduces storage and makes queries more efficient:

```sql
SELECT event.user.name, event.user.email
FROM events;
-- Only scans the nested columns you reference
```

### 10.2 Load Data in Bulk, Not Row-by-Row

Streaming inserts cost **$0.05/GB**. Batch loads (from GCS) are **free**. Always prefer batch loading when real-time isn't required.

| Method | Cost |
| :--- | :--- |
| **Batch load** (GCS → BQ) | Free |
| **Streaming insert** (legacy) | $0.05/GB |
| **Storage Write API** (modern) | Free (default stream), $0.025/GB (committed) — first **2 TiB/month free** |

> **Always prefer Storage Write API** over legacy streaming — it's ~50% cheaper and includes a generous free tier.

---

## 11. Compression & Physical Bytes

BigQuery stores data in **compressed columnar format** internally. You're billed on **logical bytes** by default, but you can switch to **physical bytes billing**:

```sql
ALTER TABLE sales SET OPTIONS (storage_billing_model = 'PHYSICAL');
```

Physical bytes billing charges for the **compressed size** (~70–80% smaller), which can be significantly cheaper for highly compressible data. Trade-off: time travel bytes also count.

---

## 12. Monitoring & Auditing

### 12.1 Find Your Most Expensive Queries

Regularly audit with `INFORMATION_SCHEMA.JOBS` — your top 10% of queries often account for **90% of spend**:

```sql
SELECT
  user_email,
  query,
  total_bytes_processed / POW(1024, 3) AS gb_scanned,
  total_slot_ms / 1000 AS slot_seconds,
  creation_time
FROM `region-us`.INFORMATION_SCHEMA.JOBS
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND job_type = 'QUERY'
ORDER BY total_bytes_processed DESC
LIMIT 20;
```

### 12.2 Find Unused Tables

```sql
SELECT
  t.table_name,
  t.creation_time,
  TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), t.creation_time, DAY) AS age_days,
  t.row_count,
  t.size_bytes / POW(1024, 3) AS size_gb
FROM `project.dataset.INFORMATION_SCHEMA.TABLES` t
WHERE t.table_name NOT IN (
  SELECT DISTINCT referenced_table.table_id
  FROM `region-us`.INFORMATION_SCHEMA.JOBS, UNNEST(referenced_tables) AS referenced_table
  WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
)
ORDER BY size_bytes DESC;
```

---

## Quick Reference: Cost Levers by Impact

| Lever | Impact | Effort |
| :--- | :--- | :--- |
| **Partitioning + Clustering** | Very High | Low |
| **Require partition filter** | Very High | Zero |
| **Avoid SELECT \*** | High | Zero |
| **Editions (committed slots)** | Very High | Medium |
| **Materialized Views** | High | Low |
| **Search Indexes** | Very High (text search) | Low |
| **BI Engine** | High (for BI) | Low |
| **Physical bytes billing** | Medium–High | Low |
| **Table/partition expiration** | Medium | Low |
| **Approximate functions** | Medium | Low |
| **Storage Write API (over streaming)** | Medium | Low |
| **Batch loading** | Medium | Low |
| **Dry runs / Query Validator** | Preventive | Zero |
| **Cost controls / quotas** | Preventive | Low |
| **Row-Level Security** | Medium | Medium |
| **Free Preview (not SELECT * LIMIT)** | Low–Medium | Zero |
| **Autoscaling caps** | Medium | Low |
| **Archive unused to Coldline** | Medium | Low |
