How do columnar storage formats speed up analytical queries?

# How Columnar Storage Formats Speed Up Analytical Queries

Columnar storage formats speed up analytical queries by fundamentally changing how data is organized, compressed, and accessed. Unlike traditional row-based formats (like CSV or JSON) that store records sequentially, columnar formats like **Apache Parquet** and **ORC** group all values for a single field together.

This vertical organization enables several critical optimizations that drastically reduce I/O overhead and processing time.

---

## 1. Column Pruning (I/O Reduction)

The most direct performance benefit comes from **column pruning**, which allows a query engine to read only the specific columns needed for a query.

In a row-based system, reading even a single metric requires loading the entire row into memory—including hundreds of irrelevant columns—only to discard them.

In a columnar format:
- The engine uses metadata to identify exact byte offsets for required columns
- Skips the rest of the file entirely

For wide tables, this can reduce the data footprint read from disk by **80% to 99%**.

---

## 2. Predicate Pushdown and Data Skipping

Columnar formats support **predicate pushdown**, an optimization that filters data at the storage layer before loading it into memory.

These files contain self-describing metadata in a **file footer**, including:
- Minimum and maximum values
- Null counts
- Distinct counts
- Row group / stripe statistics

### How It Works

If a query includes a filter:

```sql
SELECT * FROM products WHERE price > 500;

The engine:

Reads min/max statistics from the footer

Skips entire row groups where the condition cannot be true

Example:

If a row group's max(price) is 100

The engine skips that block entirely

This can avoid reading millions of rows.

Advanced Indexing

Formats like ORC also include Bloom filters, which:

Probabilistically test whether a value exists in a block

Accelerate highly selective point-lookups

3. Superior Compression and Encoding

Columnar data is homogeneous (same data type per column), which makes it highly compressible.

Specialized encoding techniques reduce logical size before compression:

Dictionary Encoding

Replaces repeated values (e.g., country names) with small integer IDs.

Run-Length Encoding (RLE)

Stores a value once with a repetition count — ideal for sorted data.

Delta Encoding

Stores differences between consecutive values — effective for:

Timestamps

Sequential IDs

These techniques often deliver 5–10× better compression, meaning:

Less disk I/O

Less network transfer

Faster scans

4. Vectorized Execution

Columnar formats are optimized for vectorized processing.

Instead of processing one row at a time:

The engine processes batches ("vectors") of values

Enables SIMD (Single Instruction, Multiple Data) parallelism

Improves CPU cache utilization

Result:

Fewer CPU instructions

Higher throughput

Lower overhead

5. Hierarchical Architecture

Columnar files are organized into:

Row Groups (Parquet) or Stripes (ORC)

Column Chunks

Pages

Parallelism

Each row group can be processed independently, enabling distributed execution across systems like Spark.

Page-Level Skipping

Pages are the atomic unit of compression. If metadata indicates a page is irrelevant, it can be skipped without decompression.
