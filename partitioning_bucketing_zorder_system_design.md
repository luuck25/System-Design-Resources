# Partitioning, Bucketing, Z-Ordering & Liquid Clustering

## Big Data Data Layout Optimization Guide (System Design Ready)

------------------------------------------------------------------------

# 1. Partitioning

## Definition

Partitioning splits data into separate physical folders based on
low-cardinality columns.

Example directory layout:

    /orders/order_year=2025/order_month=07/country=IN/

## How It Works

-   Each unique value creates a subdirectory.
-   Query engines skip irrelevant folders (Partition Pruning).

## Best For

-   Low-cardinality columns
-   Frequently filtered columns
-   Date, region, country, category

## Benefits

-   Massive I/O reduction
-   Faster filtering
-   Lower compute cost

## Trade-offs

-   ❌ High cardinality → millions of folders
-   ❌ Small files problem
-   ❌ Metadata overhead in Hive/Glue catalog

------------------------------------------------------------------------

# 2. Bucketing (Clustering)

## Definition

Bucketing distributes rows into a fixed number of files using a hash
function.

    bucket_number = hash(user_id) % 1024

## Best For

-   High-cardinality columns
-   Join keys (user_id, product_id)

## Primary Benefit

Shuffle reduction during joins.

If two tables: - Bucketed on same key - Same number of buckets

→ Spark can perform bucket join (no network shuffle)

## Trade-offs

-   Static bucket count
-   Must match across tables
-   Requires planning ahead
-   Not flexible for evolving workloads

------------------------------------------------------------------------

# 3. Z-Ordering

## Definition

Z-Ordering clusters related data within files using multidimensional
ordering.

Unlike partitioning: - No new folders - Works inside files

## Primary Goal

File skipping using min/max statistics.

Delta Lake stores: - Min value - Max value for each column in every
file.

If filter value not in range → file skipped.

## Example

Before Z-Order: - 395 files - id1='id016' spread across all - Query
time: 4.51s

After Z-Order by id1: - 25 files - All id016 rows in 1 file - Query
time: 0.6s

## Important

-   Optimizes filtering
-   NOT designed for join shuffle reduction
-   Best for multi-column filters

## Trade-offs

-   Compute heavy OPTIMIZE operation
-   Too many columns reduce effectiveness
-   No benefit without filters

------------------------------------------------------------------------

# 4. Liquid Clustering

## Definition

Modern alternative to partitioning & Z-Order.

Allows: - Changing clustering columns - Without rewriting entire table

## Benefits

-   Flexible
-   Evolves with workload
-   No rigid partition hierarchy

## Trade-offs

-   Less manual control
-   More abstracted
-   Engine-managed behavior

------------------------------------------------------------------------

# 5. Partitioning vs Bucketing vs Z-Order

  ------------------------------------------------------------------------
  Feature        Partitioning         Bucketing         Z-Ordering
  -------------- -------------------- ----------------- ------------------
  Main Goal      Filtering            Join Optimization Filtering

  Mechanism      Folder split         Hash distribution Multidimensional
                                                        clustering

  Cardinality    Low                  High              High

  Shuffle        No                   Yes               No
  Reduction                                             

  File Skipping  Directory-level      No                File-level

  Flexibility    Low                  Low               Medium
  ------------------------------------------------------------------------

------------------------------------------------------------------------

# 6. System Design Interview Answer Template

## Step 1: Define Both Clearly

Partitioning: \> Used for low-cardinality filter columns like date or
country to enable partition pruning.

Bucketing: \> Used for high-cardinality join keys to reduce shuffle
during joins.

Z-Ordering: \> Used for multi-column filtering to maximize file
skipping.

------------------------------------------------------------------------

## Step 2: Give Real Example (E-Commerce)

Orders Table: - Partition by: order_year, order_month, country - Bucket
by: customer_id (1024 buckets)

Customers Table: - Bucket by: customer_id (1024 buckets)

Query 1: "July 2025 sales in India" → Partition pruning skips all other
folders.

Query 2: Join orders with customers on customer_id → Bucket join, no
shuffle.

Query 3: Filter by customer_id and product_category → Z-Order improves
file skipping.

------------------------------------------------------------------------

# 7. Trade-Offs Summary for Interview

## Partitioning

-   Excellent for filtering
-   Metadata bloat
-   Small files risk

## Bucketing

-   Massive join speedup
-   Rigid structure
-   Must match across tables

## Z-Ordering

-   Multi-column filtering
-   Heavy reorganization cost
-   No join optimization

## Liquid Clustering

-   Adaptive & flexible
-   Engine dependent

------------------------------------------------------------------------

# 8. Key Design Rule

Partition for filtering\
Bucket for joins\
Z-Order for multi-column pruning\
Liquid Clustering for evolving workloads

------------------------------------------------------------------------

# Final Interview Line

"In large-scale systems, data layout decisions are trade-offs between
I/O reduction, shuffle minimization, and metadata overhead. The correct
choice depends entirely on workload patterns."

------------------------------------------------------------------------

END
