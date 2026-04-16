# SQL Query Optimization — The Complete Interview Arsenal

Sections 1–15 cover **BigQuery-specific** optimizations. Sections 16+ cover **general SQL hacks** that apply across all databases (PostgreSQL, MySQL, SQL Server, Oracle, Snowflake, Spark SQL, etc.).

---

## 1. Never Wrap Partition Columns in Functions

The #1 most common mistake.

```sql
-- ❌ BAD: BQ cannot prune partitions (full table scan)
SELECT * FROM orders
WHERE DATE(order_timestamp) = '2025-01-01';

-- ✅ GOOD: Partition pruning kicks in (scans 1 partition)
SELECT * FROM orders
WHERE order_timestamp >= '2025-01-01'
  AND order_timestamp < '2025-01-02';
```

**Why:** Any function on the partition column (`DATE()`, `CAST()`, `EXTRACT()`, `TIMESTAMP_TRUNC()`) turns it into a **computed expression** — BQ evaluates it row-by-row *after* scanning, so it can't skip partitions.

This applies to **all** function-wrapped filters:

```sql
-- ❌ BAD
WHERE LOWER(email) = 'john@test.com'
WHERE CAST(user_id AS STRING) = '12345'
WHERE EXTRACT(YEAR FROM created_at) = 2025

-- ✅ GOOD
WHERE email = 'john@test.com'  -- store pre-lowered
WHERE user_id = 12345          -- compare in native type
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01'
```

---

## 2. CTE vs Subquery vs Temp Table vs Materialized View

This is the comparison interviewers love. **They are NOT equivalent.**

### How BigQuery Actually Executes Them

| Type | Execution Behavior | Data Persisted? | Scope |
| :--- | :--- | :--- | :--- |
| **CTE** (`WITH ... AS`) | **Inlined** — re-executed every time it's referenced | No | Single query |
| **Subquery** | **Inlined** — identical to CTE in BQ | No | Single query |
| **Temp Table** (`CREATE TEMP TABLE`) | **Materialized once**, stored for session | Yes (session) | Session (multi-query script) |
| **Materialized View** | **Persisted + auto-refreshed** | Yes (permanent) | Project-wide |

### The Hidden Gotcha: CTEs Are NOT Cached

```sql
-- ❌ This scans big_table TWICE (CTE is inlined, not cached)
WITH base AS (
  SELECT * FROM big_table WHERE date = '2025-01-01'
)
SELECT COUNT(*) FROM base
UNION ALL
SELECT SUM(amount) FROM base;  -- scans big_table again!

-- ✅ Use a TEMP TABLE if you reference it multiple times
CREATE TEMP TABLE base AS
SELECT * FROM big_table WHERE date = '2025-01-01';

SELECT COUNT(*) FROM base
UNION ALL
SELECT SUM(amount) FROM base;  -- reads from temp (already materialized)
```

### Decision Matrix

| Scenario | Use |
| :--- | :--- |
| Referenced **once** | CTE or subquery (identical performance) |
| Referenced **2+ times** in same query | **Temp table** (avoids re-scanning) |
| Referenced across **multiple queries** in a session | **Temp table** |
| Heavy aggregation queried **repeatedly by many users** | **Materialized view** |
| Just improving readability | CTE (no perf difference vs subquery) |

> **Interview answer:** "CTEs in BigQuery are syntactic sugar — they're inlined at execution, not materialized. If I reference a CTE multiple times, I use a temp table to avoid redundant scans."

---

## 3. SELECT * is a Crime in Columnar Databases

```sql
-- ❌ Scans ALL columns (pays for every byte in every column)
SELECT * FROM events;  -- 500 GB table = 500 GB billed

-- ✅ Scans only 2 columns (maybe 5 GB)
SELECT user_id, event_type FROM events;
```

BigQuery stores data **column by column**. Every column you add to SELECT is an extra file read. This is the single easiest cost reduction.

**Bonus trap:**

```sql
-- ❌ Subquery with SELECT * forces full column read
SELECT user_id FROM (SELECT * FROM events);

-- ✅ Pushdown works — only scans user_id
SELECT user_id FROM (SELECT user_id FROM events);
```

---

## 4. JOIN Optimization

### 4.1 Put the Largest Table on the LEFT (Broadcast Joins)

BigQuery uses **broadcast joins** when one side is small enough. The smaller right-hand table gets broadcast to all workers:

```sql
-- ✅ Large table LEFT, small table RIGHT
SELECT f.*, d.product_name
FROM fact_sales f              -- 1B rows (stays distributed)
JOIN dim_product d             -- 10K rows (broadcast to all workers)
  ON f.product_id = d.product_id;
```

### 4.2 Filter Before Joining, Not After

```sql
-- ❌ BAD: Joins everything, then filters
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date = '2025-01-01';

-- ✅ GOOD: Filter first, then join smaller result set
SELECT *
FROM (SELECT * FROM orders WHERE order_date = '2025-01-01') o
JOIN customers c ON o.customer_id = c.customer_id;
```

BQ's optimizer *sometimes* does this automatically, but **don't rely on it** — especially with complex queries.

### 4.3 Avoid Cross Joins / Cartesian Products

```sql
-- ❌ Accidental cross join (no ON clause)
SELECT * FROM table_a, table_b;  -- 1M × 1M = 1 TRILLION rows

-- ❌ Implicit cross join from comma syntax
SELECT * FROM table_a a, table_b b WHERE a.id = b.id;

-- ✅ Always explicit
SELECT * FROM table_a a JOIN table_b b ON a.id = b.id;
```

### 4.4 Use INTEGER Keys for Joins, Not Strings

```sql
-- ❌ Slow: string comparison on every row
ON a.user_email = b.user_email

-- ✅ Fast: integer comparison
ON a.user_id = b.user_id
```

String JOINs are **3–5x slower** than integer JOINs on large tables.

---

## 5. Approximate Functions (Often Ignored)

When **exact precision isn't needed** (dashboards, monitoring), approximate functions are dramatically faster:

```sql
-- ❌ Exact distinct count (expensive on 1B rows)
SELECT COUNT(DISTINCT user_id) FROM events;

-- ✅ ~2% error margin, but 10x faster and cheaper
SELECT APPROX_COUNT_DISTINCT(user_id) FROM events;
```

| Function | Approximate Version |
| :--- | :--- |
| `COUNT(DISTINCT x)` | `APPROX_COUNT_DISTINCT(x)` |
| `PERCENTILE_CONT` | `APPROX_QUANTILES(x, 100)` |
| Top-N values | `APPROX_TOP_COUNT(x, N)` |
| Top-N by sum | `APPROX_TOP_SUM(x, weight, N)` |

> **Interview tip:** Mention this proactively — it shows you think about cost/accuracy trade-offs.

---

## 6. Avoid Data Skew in GROUP BY / JOINs

**Data skew** = one key has disproportionately more rows than others. This kills parallelism.

```sql
-- If user_id = 'bot_account' has 500M rows and everyone else has 100
-- One worker gets slammed while others sit idle

-- ✅ Fix: Filter out the skewed key, process separately
WITH normal AS (
  SELECT * FROM events WHERE user_id != 'bot_account'
),
skewed AS (
  SELECT * FROM events WHERE user_id = 'bot_account'
)
SELECT ... FROM normal GROUP BY user_id
UNION ALL
SELECT ... FROM skewed;  -- processed in its own pipeline
```

---

## 7. Window Functions: ROW_NUMBER vs QUALIFY

Most people don't know `QUALIFY` exists in BigQuery — it's like `HAVING` but for window functions:

```sql
-- ❌ Verbose: subquery + ROW_NUMBER
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY ts DESC) AS rn
  FROM events
) WHERE rn = 1;

-- ✅ Clean: QUALIFY (BQ-specific, no subquery needed)
SELECT *
FROM events
QUALIFY ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY ts DESC) = 1;
```

Same performance, but **cleaner and easier to maintain**. Interviewers love this.

---

## 8. The MERGE Trick (Upserts)

For incremental pipelines, `MERGE` is far better than DELETE+INSERT:

```sql
-- ✅ Atomic upsert: update existing, insert new
MERGE INTO gold.customers AS target
USING staging.new_customers AS source
ON target.customer_id = source.customer_id
WHEN MATCHED THEN
  UPDATE SET target.name = source.name, target.email = source.email
WHEN NOT MATCHED THEN
  INSERT (customer_id, name, email)
  VALUES (source.customer_id, source.name, source.email);
```

**Why:** Single pass, atomic, no intermediate state where rows are missing.

---

## 9. Clustering Column Order Matters

```sql
-- Clustering by (region, product, date) means:
-- ✅ WHERE region = 'APAC'                     → uses clustering
-- ✅ WHERE region = 'APAC' AND product = 'X'   → uses clustering
-- ❌ WHERE product = 'X'                        → CANNOT use clustering (skipped first column)
```

**Rule:** Clustering works **left to right**, like a phone book. Filter on the **leftmost** cluster column first. Order clusters from **lowest cardinality** to **highest**.

---

## 10. Use `_PARTITIONTIME` / `_PARTITIONDATE` for Ingestion-Time Tables

```sql
-- For ingestion-time partitioned tables, use the pseudo-column directly
-- ❌ BAD
SELECT * FROM events WHERE DATE(event_time) = '2025-01-01';

-- ✅ GOOD (uses built-in partition pseudo-column)
SELECT * FROM events WHERE _PARTITIONDATE = '2025-01-01';
```

---

## 11. Avoid Repeated Parsing of JSON/STRING

```sql
-- ❌ Parsing JSON in every query (slow, repeated cost)
SELECT JSON_EXTRACT_SCALAR(raw_payload, '$.user.id') AS user_id
FROM raw_events
WHERE JSON_EXTRACT_SCALAR(raw_payload, '$.user.id') = '12345';

-- ✅ Parse once into a structured table, query that
CREATE TABLE parsed_events AS
SELECT
  JSON_EXTRACT_SCALAR(raw_payload, '$.user.id') AS user_id,
  JSON_EXTRACT_SCALAR(raw_payload, '$.event_type') AS event_type,
  event_timestamp
FROM raw_events;
```

Same principle applies to repeated `REGEXP_EXTRACT`, `SPLIT`, etc.

---

## 12. EXISTS vs IN vs JOIN for Semi-Joins

```sql
-- ❌ IN with subquery (can be slow with large lists, loads all into memory)
SELECT * FROM orders
WHERE customer_id IN (SELECT customer_id FROM vip_customers);

-- ✅ EXISTS (short-circuits — stops scanning once found)
SELECT * FROM orders o
WHERE EXISTS (
  SELECT 1 FROM vip_customers v WHERE v.customer_id = o.customer_id
);
```

**Rule of thumb:**

- **`IN`** — fine for small, static lists (`IN ('US', 'UK', 'DE')`)
- **`EXISTS`** — better for subqueries against large tables
- **`JOIN`** — when you need columns from both sides

---

## 13. Avoid ORDER BY on Full Result Sets

```sql
-- ❌ Sorts 1 billion rows just to see the top 10
SELECT * FROM events ORDER BY event_time DESC;

-- ✅ Always pair ORDER BY with LIMIT
SELECT * FROM events ORDER BY event_time DESC LIMIT 10;
```

Unbounded `ORDER BY` forces all data onto a **single worker** for final sorting — kills parallelism and can OOM.

---

## 14. Use STRUCT/ARRAY Instead of Repeated JOINs

```sql
-- ❌ Flat table: orders + line items as separate tables (JOIN every time)
SELECT o.order_id, li.product_id, li.quantity
FROM orders o JOIN line_items li ON o.order_id = li.order_id;

-- ✅ Nested: line items stored inside orders (no JOIN needed)
SELECT
  order_id,
  item.product_id,
  item.quantity
FROM orders, UNNEST(line_items) AS item;
```

**Nested/repeated fields** eliminate JOINs and reduce storage. BigQuery was *designed* for this.

---

## 15. `INFORMATION_SCHEMA` for Query Auditing

Find your most expensive queries:

```sql
SELECT
  user_email,
  query,
  total_bytes_processed / POW(1024, 3) AS gb_scanned,
  total_slot_ms / 1000 AS slot_seconds
FROM `region-us`.INFORMATION_SCHEMA.JOBS
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
ORDER BY total_bytes_processed DESC
LIMIT 20;
```

**Interview gold:** Shows you think about observability, not just writing queries.

---

# General SQL Hacks (All Databases)

---

## 16. SARGABLE Queries (Search ARGument ABLE)

This is the **generalized version** of "don't wrap partition columns in functions." A query is **sargable** if the database can use an index to speed up the filter. Any function on an indexed column makes it **non-sargable**:

```sql
-- ❌ NON-SARGABLE: index on created_at is useless
WHERE YEAR(created_at) = 2025
WHERE UPPER(name) = 'JOHN'
WHERE amount + 10 > 100
WHERE CAST(id AS VARCHAR) = '123'

-- ✅ SARGABLE: index can be used
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01'
WHERE name = 'JOHN'  -- store pre-uppercased, or use a computed/functional index
WHERE amount > 90
WHERE id = 123
```

> **Interview one-liner:** "A WHERE clause is sargable if the column is naked — no functions, no arithmetic, no casts wrapping it."

---

## 17. EXPLAIN / Execution Plans

The single most important debugging tool. **Always check the execution plan** before optimizing:

```sql
-- PostgreSQL
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;

-- MySQL
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- SQL Server
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
SELECT * FROM orders WHERE customer_id = 42;
```

### What to Look For

| Red Flag | Meaning |
| :--- | :--- |
| **Seq Scan / Full Table Scan** | No index used — scanning every row |
| **Nested Loop on large tables** | Likely missing a JOIN index |
| **Sort** (without index) | Expensive in-memory or disk sort |
| **Hash Join** on huge tables | Both sides loaded into memory |
| **High "Rows Removed by Filter"** | Scanning way more rows than returned |
| **Temp disk usage** | Query spills to disk — needs more memory or better plan |

> **Interview tip:** If asked "how would you optimize a slow query?" — always start with "I'd look at the execution plan first."

---

## 18. Indexing Strategies

### Types of Indexes

| Index Type | Best For | Example |
| :--- | :--- | :--- |
| **B-tree** (default) | Equality and range queries | `WHERE id = 5`, `WHERE date > '2025-01-01'` |
| **Hash** | Exact equality only | `WHERE email = 'john@test.com'` |
| **GIN** (PostgreSQL) | Full-text search, JSONB, arrays | `WHERE tags @> '{urgent}'` |
| **BRIN** | Large, naturally ordered data (timestamps) | `WHERE created_at > '2025-01-01'` on append-only table |
| **Partial index** | Filter on subset | `CREATE INDEX idx ON orders(status) WHERE status = 'pending'` |
| **Composite index** | Multi-column filters | `CREATE INDEX idx ON orders(customer_id, order_date)` |

### Composite Index Column Order Matters

Same rule as clustering — **leftmost prefix**:

```sql
-- Index on (customer_id, order_date, status)

-- ✅ Uses index (left prefix match)
WHERE customer_id = 42
WHERE customer_id = 42 AND order_date = '2025-01-01'
WHERE customer_id = 42 AND order_date > '2025-01-01' AND status = 'shipped'

-- ❌ Cannot use index (skipped customer_id)
WHERE order_date = '2025-01-01'
WHERE status = 'shipped'
```

### Covering Index (Index-Only Scan)

If all columns in SELECT + WHERE are in the index, the database reads **only the index** and never touches the table:

```sql
-- Index on (customer_id, order_date) INCLUDE (total_amount)
SELECT order_date, total_amount
FROM orders
WHERE customer_id = 42;
-- ✅ Index-only scan — never reads the heap/table
```

### Over-Indexing Trap

- Every index **slows down writes** (INSERT/UPDATE/DELETE) because the index must be updated
- Rule: Index columns you **filter, join, and order by** — not everything

---

## 19. UNION ALL vs UNION

```sql
-- ❌ UNION: deduplicates results (expensive sort + compare)
SELECT name FROM customers_us
UNION
SELECT name FROM customers_eu;

-- ✅ UNION ALL: just concatenates (no dedup, much faster)
SELECT name FROM customers_us
UNION ALL
SELECT name FROM customers_eu;
```

**Always use `UNION ALL`** unless you specifically need deduplication. `UNION` does a full sort + distinct pass.

---

## 20. WHERE vs HAVING (Filter Early)

```sql
-- ❌ BAD: HAVING filters AFTER grouping (processes all rows first)
SELECT region, COUNT(*) AS cnt
FROM orders
GROUP BY region
HAVING region != 'test';

-- ✅ GOOD: WHERE filters BEFORE grouping (skips rows early)
SELECT region, COUNT(*) AS cnt
FROM orders
WHERE region != 'test'
GROUP BY region;
```

**Rule:** Use `WHERE` for row-level filters, `HAVING` only for aggregate-level filters (e.g., `HAVING COUNT(*) > 10`).

---

## 21. NOT IN vs NOT EXISTS (The NULL Trap)

```sql
-- ❌ DANGEROUS: If subquery returns ANY NULL, NOT IN returns EMPTY result set!
SELECT * FROM orders
WHERE customer_id NOT IN (SELECT customer_id FROM blacklist);
-- If blacklist has a NULL customer_id, this returns ZERO rows. Silent bug.

-- ✅ SAFE: NOT EXISTS handles NULLs correctly
SELECT * FROM orders o
WHERE NOT EXISTS (
  SELECT 1 FROM blacklist b WHERE b.customer_id = o.customer_id
);
```

### Why NOT IN Breaks — Step by Step

SQL uses **three-valued logic**: `TRUE`, `FALSE`, and `UNKNOWN`. Any comparison with `NULL` returns `UNKNOWN`:

```
101 = 101   → TRUE
101 = 102   → FALSE
101 = NULL  → UNKNOWN  (you can't know if NULL equals 101)
```

Say `blacklist` has these `customer_id` values: **`[101, 105, NULL]`**

For an order with `customer_id = 200` (clearly NOT in the blacklist):

```sql
WHERE 200 NOT IN (101, 105, NULL)
```

SQL expands `NOT IN` internally as:

```sql
WHERE 200 != 101 AND 200 != 105 AND 200 != NULL
```

Evaluate each:

```
200 != 101   → TRUE
200 != 105   → TRUE
200 != NULL  → UNKNOWN   ← the problem
```

Combine with AND:

```
TRUE AND TRUE AND UNKNOWN → UNKNOWN
```

SQL treats `UNKNOWN` as "not TRUE" → **row is excluded**. So `customer_id = 200` is NOT returned, even though 200 is clearly not in the blacklist.

### Every Row Gets Poisoned

```
customer_id = 101 (actual blacklisted):
  101 != 101 → FALSE → FALSE AND ... → FALSE
  Not returned ✅ (correct)

customer_id = 200 (not blacklisted):
  200 != 101 → TRUE, 200 != 105 → TRUE, 200 != NULL → UNKNOWN
  TRUE AND TRUE AND UNKNOWN → UNKNOWN
  Not returned ❌ (WRONG — 200 should be included!)

customer_id = 300 (not blacklisted):
  300 != 101 → TRUE, 300 != 105 → TRUE, 300 != NULL → UNKNOWN
  TRUE AND TRUE AND UNKNOWN → UNKNOWN
  Not returned ❌ (WRONG again!)
```

**Every non-blacklisted row is excluded.** You get zero rows back — no error, no warning. Completely silent bug.

### Why NOT EXISTS Doesn't Have This Problem

```sql
SELECT * FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM blacklist b WHERE b.customer_id = o.customer_id
);
```

For `customer_id = 200`:

```
Does any row in blacklist have customer_id = 200?
  Row 1: 101 = 200? → FALSE
  Row 2: 105 = 200? → FALSE
  Row 3: NULL = 200? → UNKNOWN (not TRUE)

No TRUE found → EXISTS returns FALSE
NOT FALSE → TRUE → Row IS returned ✅
```

`EXISTS` only cares if **any row returned TRUE**. `UNKNOWN` results are skipped — they don't poison the `AND` chain like `NOT IN` does.

### Summary

| | `NOT IN` with NULLs | `NOT EXISTS` with NULLs |
| :--- | :--- | :--- |
| **Internal logic** | `!= NULL` → UNKNOWN, AND chain breaks | EXISTS ignores UNKNOWN rows |
| **Result** | **Zero rows** (silent bug) | **Correct rows** |
| **Error/Warning?** | None. Completely silent. | N/A |

> **Rule: Never use `NOT IN` with a subquery. Always use `NOT EXISTS`.**
>
> **Interview killer:** Most candidates don't know about the NULL trap with `NOT IN`. Mentioning this unprompted shows deep SQL knowledge.

---

## 22. Avoid SELECT DISTINCT (When Possible)

```sql
-- ❌ Expensive: sorts/hashes entire result set to remove duplicates
SELECT DISTINCT customer_id FROM orders;

-- ✅ If you just need existence, use EXISTS or GROUP BY
SELECT customer_id FROM orders GROUP BY customer_id;

-- ✅ Or fix the root cause — often DISTINCT hides a bad JOIN
SELECT DISTINCT o.order_id, o.amount
FROM orders o
JOIN line_items li ON o.order_id = li.order_id;  -- this multiplies rows!
-- Fix: rethink the JOIN, don't mask it with DISTINCT
```

---

## 23. Correlated Subqueries → Rewrite as JOINs

A correlated subquery runs **once per row** in the outer query — O(N×M) complexity:

```sql
-- ❌ SLOW: subquery executes for EVERY row in orders
SELECT o.order_id,
  (SELECT MAX(amount) FROM payments p WHERE p.order_id = o.order_id) AS max_payment
FROM orders o;

-- ✅ FAST: single JOIN pass
SELECT o.order_id, p.max_payment
FROM orders o
LEFT JOIN (
  SELECT order_id, MAX(amount) AS max_payment
  FROM payments
  GROUP BY order_id
) p ON o.order_id = p.order_id;
```

---

## 24. Wildcard Searches & Leading % Problem

```sql
-- ❌ Leading wildcard: FULL TABLE SCAN (index cannot help)
WHERE name LIKE '%smith'
WHERE email LIKE '%@gmail.com'

-- ✅ Trailing wildcard: uses index (B-tree can seek to prefix)
WHERE name LIKE 'smith%'
WHERE email LIKE 'john%'

-- ✅ For full-text search, use dedicated indexes
-- PostgreSQL: GIN index + tsvector
-- MySQL: FULLTEXT index
-- BigQuery: SEARCH index
```

---

## 25. Pagination: OFFSET vs Keyset (Seek Method)

`OFFSET` is one of the most common performance killers in web apps:

```sql
-- ❌ OFFSET: scans and discards 1M rows to get 10
SELECT * FROM events ORDER BY id LIMIT 10 OFFSET 1000000;
-- Database reads 1,000,010 rows, throws away 1,000,000

-- ✅ KEYSET: jumps directly to the right spot using an index
SELECT * FROM events
WHERE id > 1000000  -- last seen ID from previous page
ORDER BY id
LIMIT 10;
-- Database reads exactly 10 rows
```

| Method | Page 1 | Page 1000 | Page 100000 |
| :--- | :--- | :--- | :--- |
| **OFFSET** | Fast | Slow | Extremely slow |
| **Keyset** | Fast | Fast | Fast |

> **Interview tip:** If someone mentions pagination, immediately bring up keyset pagination. Most developers use OFFSET and don't realize it's O(N).

---

## 26. Implicit Type Conversions Kill Indexes

```sql
-- Column `phone` is VARCHAR, but you compare with an integer
-- ❌ Database casts EVERY row to integer → full table scan
WHERE phone = 1234567890

-- ✅ Compare with matching type
WHERE phone = '1234567890'

-- Column `id` is INT, but you compare with string
-- ❌ Some databases cast the column, not the literal
WHERE id = '42'

-- ✅ Always match types
WHERE id = 42
```

Implicit casts on the **column side** effectively wrap it in a `CAST()` function — making it non-sargable.

---

## 27. Batch Operations (Avoid Row-by-Row)

```sql
-- ❌ SLOW: 1000 individual INSERT statements (1000 round trips)
INSERT INTO logs VALUES (1, 'event_a');
INSERT INTO logs VALUES (2, 'event_b');
... (998 more)

-- ✅ FAST: single multi-row INSERT (1 round trip)
INSERT INTO logs VALUES
  (1, 'event_a'),
  (2, 'event_b'),
  ... (998 more);

-- ✅ For updates/deletes on large tables, batch in chunks
DELETE FROM logs WHERE created_at < '2024-01-01' LIMIT 10000;
-- Run in a loop until 0 rows affected
```

Row-by-row operations have **network round-trip overhead** per statement + **lock contention**.

---

## 28. TRUNCATE vs DELETE

```sql
-- ❌ DELETE: logs every row, fires triggers, slow on large tables
DELETE FROM staging_table;
-- Can take hours on 100M rows, fills up transaction log

-- ✅ TRUNCATE: instant, resets the table, minimal logging
TRUNCATE TABLE staging_table;
```

| Aspect | DELETE | TRUNCATE |
| :--- | :--- | :--- |
| **Speed** | Slow (row-by-row) | Instant |
| **Logging** | Full (every row logged) | Minimal |
| **WHERE clause** | Supported | Not supported (all or nothing) |
| **Triggers** | Fires per row | Does NOT fire |
| **ROLLBACK** | Yes | Depends on DB (PostgreSQL: yes, MySQL: no) |
| **Auto-increment** | Keeps counter | Resets counter |

---

## 29. Window Functions vs Self-JOINs

```sql
-- ❌ Self-JOIN to get previous row's value (scans table twice)
SELECT a.id, a.revenue, b.revenue AS prev_revenue
FROM sales a
LEFT JOIN sales b ON a.id = b.id + 1;

-- ✅ Window function: single pass over the data
SELECT id, revenue,
  LAG(revenue) OVER (ORDER BY id) AS prev_revenue
FROM sales;
```

Window functions (`LAG`, `LEAD`, `ROW_NUMBER`, `RANK`, `SUM() OVER`, etc.) are almost always faster than self-joins because they process data in a **single pass**.

---

## 30. OR Conditions Can Prevent Index Usage

```sql
-- ❌ OR on different columns: optimizer may give up and do full scan
SELECT * FROM users
WHERE email = 'john@test.com' OR phone = '555-1234';

-- ✅ Rewrite as UNION ALL (each branch can use its own index)
SELECT * FROM users WHERE email = 'john@test.com'
UNION ALL
SELECT * FROM users WHERE phone = '555-1234'
  AND email != 'john@test.com';  -- avoid duplicates
```

Not all optimizers handle `OR` across columns well. The `UNION ALL` rewrite lets each branch do an **index seek** independently.

---

## 31. COUNT(*) vs COUNT(column) vs COUNT(1)

```sql
-- COUNT(*) — counts ALL rows (including NULLs). Fastest.
SELECT COUNT(*) FROM orders;  -- ✅

-- COUNT(column) — counts non-NULL values only. Reads the column.
SELECT COUNT(email) FROM customers;  -- slower if email has NULLs

-- COUNT(1) — identical to COUNT(*) in all modern databases
SELECT COUNT(1) FROM orders;  -- ✅ same as COUNT(*)
```

**Use `COUNT(*)`** unless you specifically need to count non-NULL values.

---

## 32. Avoid N+1 Query Pattern (Application Layer)

The most common performance killer in ORMs:

```python
# ❌ N+1: 1 query for orders + N queries for customers
orders = db.query("SELECT * FROM orders LIMIT 100")
for order in orders:
    customer = db.query(f"SELECT * FROM customers WHERE id = {order.customer_id}")
    # 101 total queries!

# ✅ Single JOIN query
results = db.query("""
    SELECT o.*, c.name
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    LIMIT 100
""")
# 1 query!
```

---

## 33. Use EXISTS Instead of COUNT for Existence Checks

```sql
-- ❌ Counts ALL matching rows just to check if any exist
SELECT CASE WHEN COUNT(*) > 0 THEN 'yes' ELSE 'no' END
FROM orders WHERE customer_id = 42;

-- ✅ Stops at first match (short-circuits)
SELECT CASE WHEN EXISTS (
  SELECT 1 FROM orders WHERE customer_id = 42
) THEN 'yes' ELSE 'no' END;
```

`EXISTS` returns as soon as it finds **one row**. `COUNT(*)` scans every matching row.

---

## 34. Predicate Pushdown in Views / CTEs

Not all databases push filters **into** views:

```sql
-- View returns ALL orders
CREATE VIEW all_orders AS SELECT * FROM orders;

-- ❌ Some databases scan entire view first, then filter
SELECT * FROM all_orders WHERE order_date = '2025-01-01';

-- ✅ If predicate pushdown doesn't work, use a parameterized function or inline the query
```

Check with `EXPLAIN` whether your database pushes the `WHERE` clause into the view/CTE.

---

## 35. Transaction & Locking Tips

```sql
-- ❌ Long transactions hold locks, blocking other queries
BEGIN;
  SELECT * FROM inventory WHERE product_id = 1 FOR UPDATE;
  -- ... application does 30 seconds of processing ...
  UPDATE inventory SET stock = stock - 1 WHERE product_id = 1;
COMMIT;
-- Other queries on product_id = 1 are BLOCKED for 30 seconds

-- ✅ Keep transactions as SHORT as possible
BEGIN;
  UPDATE inventory SET stock = stock - 1 WHERE product_id = 1;
COMMIT;
-- Lock held for milliseconds
```

### Isolation Level Trade-offs

| Level | Dirty Read | Non-Repeatable Read | Phantom | Performance |
| :--- | :--- | :--- | :--- | :--- |
| **READ UNCOMMITTED** | Yes | Yes | Yes | Fastest |
| **READ COMMITTED** | No | Yes | Yes | Fast |
| **REPEATABLE READ** | No | No | Yes | Medium |
| **SERIALIZABLE** | No | No | No | Slowest |

Use the **lowest isolation level** that meets your correctness requirements.

---

## Quick Reference: Cheat Sheet

### BigQuery-Specific

| # | Trick | Category | Impact |
| :--- | :--- | :--- | :--- |
| 1 | Don't wrap partition cols in functions | Partition pruning | Very High |
| 2 | CTE = inlined; use temp tables for multi-ref | Execution model | High |
| 3 | Avoid `SELECT *` | Column pruning | Very High |
| 4 | Largest table on LEFT (broadcast join) | Join performance | High |
| 5 | Filter before JOIN, not after | Query planning | High |
| 6 | `APPROX_COUNT_DISTINCT` | Approximate | Medium |
| 7 | Avoid data skew (split hot keys) | Parallelism | High |
| 8 | `QUALIFY` instead of subquery | Window functions | Readability |
| 9 | `MERGE` for upserts | DML | Correctness |
| 10 | Cluster column order (left→right) | Clustering | High |
| 11 | `_PARTITIONDATE` pseudo-column | Partition pruning | High |
| 12 | Parse JSON once, not per query | Compute cost | Medium |
| 13 | `EXISTS` over `IN` for subqueries | Semi-joins | Medium |
| 14 | Always `LIMIT` with `ORDER BY` | Sorting | High |
| 15 | Use STRUCT/ARRAY, avoid JOINs | Schema design | Medium |

### General SQL (All Databases)

| # | Trick | Category | Impact |
| :--- | :--- | :--- | :--- |
| 16 | Keep WHERE clauses sargable | Index usage | Very High |
| 17 | Always check EXPLAIN before optimizing | Debugging | Very High |
| 18 | Composite index order (leftmost prefix) | Indexing | Very High |
| 18 | Covering indexes (index-only scans) | Indexing | High |
| 19 | `UNION ALL` over `UNION` | Deduplication | High |
| 20 | `WHERE` before `HAVING` | Filter early | Medium |
| 21 | `NOT EXISTS` over `NOT IN` (NULL trap) | Correctness | High |
| 22 | Avoid `SELECT DISTINCT` (fix the JOIN) | Deduplication | Medium |
| 23 | Rewrite correlated subqueries as JOINs | Query planning | High |
| 24 | Avoid leading `%` wildcards | Index usage | High |
| 25 | Keyset pagination over OFFSET | Pagination | Very High |
| 26 | Match data types (no implicit casts) | Index usage | High |
| 27 | Batch INSERTs (not row-by-row) | Write perf | High |
| 28 | `TRUNCATE` over `DELETE` for full wipe | Write perf | High |
| 29 | Window functions over self-JOINs | Query planning | Medium |
| 30 | Rewrite `OR` as `UNION ALL` | Index usage | Medium |
| 31 | `COUNT(*)` over `COUNT(column)` | Aggregation | Low |
| 32 | Avoid N+1 queries (use JOINs) | Application | Very High |
| 33 | `EXISTS` over `COUNT` for existence | Short-circuit | Medium |
| 34 | Check predicate pushdown in views | Query planning | Medium |
| 35 | Keep transactions short, use lowest isolation | Locking | High |
