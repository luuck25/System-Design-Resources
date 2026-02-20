How do columnar storage formats speed up analytical queries?

### The Engine

- Reads min/max statistics from the footer  
- Skips entire row groups where the condition cannot be true  

#### Example

- If a row group's `max(price)` is 100  
- The engine skips that block entirely  

This can avoid reading **millions of rows**.

---

## Advanced Indexing

Formats like **ORC** include **Bloom filters**, which:

- Probabilistically test whether a value exists in a block  
- Accelerate highly selective point-lookups  

---

## 3. Superior Compression and Encoding

Columnar data is **homogeneous** (same data type per column), which makes it highly compressible.

Specialized encoding techniques reduce logical size before compression:

### Dictionary Encoding

Replaces repeated values (e.g., country names) with small integer IDs.

### Run-Length Encoding (RLE)

Stores a value once with a repetition count — ideal for sorted data.

### Delta Encoding

Stores differences between consecutive values — effective for:

- Timestamps  
- Sequential IDs  

These techniques often deliver **5–10× better compression**, meaning:

- Less disk I/O  
- Less network transfer  
- Faster scans  

---

## 4. Vectorized Execution

Columnar formats are optimized for **vectorized processing**.

Instead of processing one row at a time:

- The engine processes batches ("vectors") of values  
- Enables SIMD (Single Instruction, Multiple Data) parallelism  
- Improves CPU cache utilization  

### Result

- Fewer CPU instructions  
- Higher throughput  
- Lower overhead  

---

## 5. Hierarchical Architecture

Columnar files are organized into:

- **Row Groups (Parquet)** or **Stripes (ORC)**  
- **Column Chunks**  
- **Pages**  

### Parallelism

Each row group can be processed independently, enabling distributed execution across systems like Spark.

### Page-Level Skipping

Pages are the atomic unit of compression.  
If metadata indicates a page is irrelevant, it can be skipped without decompression.
