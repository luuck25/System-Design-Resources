### Question 1: You chose Flink for this system. Why not Spark Streaming?

> “We chose Flink over Spark Streaming because our system requires true real-time processing for feature computation. Since Spark works in micro-batches, it introduces latency at batch boundaries, which would delay feature availability.
> 
> Additionally, our data arrives out of order at scale (~2B events/day), and Flink’s event-time processing with watermarks ensures correctness of aggregations, whereas Spark makes this more complex.
> 
> Flink also provides more advanced state management, which is critical for maintaining large, continuously updated aggregations like session metrics or rolling windows.
> 
> That said, Flink adds operational complexity compared to Spark, but we prioritized low latency and correctness.”

---

### Question 2: You mentioned state management is better in Flink. What exactly do you mean by that?

> “In Flink, state represents intermediate data like aggregations, session data, or pattern detection results. 
> 
> There are two main types:
> * **Keyed state**, which is partitioned by a key (like `user_id`) and distributed across nodes.
> * **Operator state**, which is shared across the operator instance.
> 
> For scalability, Flink stores large state in **RocksDB**, which allows handling state beyond memory limits.
> 
> To ensure fault tolerance, Flink periodically takes **checkpoints**, which are snapshots of the state stored in durable storage like S3.
> 
> If a node crashes, Flink restores the latest checkpoint and resumes processing from the exact point, ensuring no data loss and maintaining consistency.”

---

### Question 3: Failure + Exactly-Once

**Scenario:**
* You are reading from Kafka.
* Writing aggregated results to a database.
* 👉 A failure happens *after* processing but *before* writing to DB.

**Question:** How do you guarantee no data loss and no duplicate writes?
*(Note: Be prepared to discuss Two-Phase Commit (2PC) and transactional sinks here!)*

---

### Question 4: Late Data Edge Case

**Scenario:** You said Flink handles late-arriving data well. You compute a 5-minute window aggregation. The watermark passes → the window closes. Then a late event arrives. What happens?

> “In Flink, once a watermark passes the end of a window, the window is considered complete and results are emitted.
> 
> If a late event arrives after that, the behavior depends on configuration:
> 
> 1. **Allowed Lateness (grace period)**
>    * If configured, Flink keeps the window open for some additional time.
>    * Late events within this period are still processed.
>    * The window result is updated and re-emitted.
> 2. **Beyond allowed lateness**
>    * The event is considered too late.
>    * It is either dropped, or sent to a **side output** (dead-letter stream) for further handling.
> 
> This allows balancing between correctness and system latency.”

---

### Question 5: Backpressure + Stability

**Scenario:** You are processing 2B events/day. Suddenly your sink (DB/Redis) becomes slow. What happens?

> “When a downstream operator (like a sink) becomes slow, Flink applies **backpressure**, which propagates upstream through the operator chain.
> 
> Each operator processes data using bounded buffers. When the sink cannot keep up, its buffers fill up, which causes upstream operators to slow down as well.
> 
> This propagation continues all the way to the source, such as Kafka, reducing the consumption rate.
> 
> Since Kafka retains data, no data is lost—the system effectively self-regulates instead of crashing. This built-in flow control ensures system stability under uneven load.”

---

### 🎙️ 2-Minute Flink Pitch (Polished & Interview-Ready)

> “In our system, we were processing around 2 billion events per day, including telemetry and transactional data, where we needed near real-time aggregations and feature computation.
> 
> We chose **Apache Flink** primarily because of its true streaming model, unlike Spark Streaming which operates on micro-batches. This allowed us to achieve millisecond-level latency, which was critical for timely insights and downstream decision-making.
> 
> Another key reason was **event-time processing with watermarks**. Our data could arrive out of order, and Flink ensured correctness of windowed aggregations by handling late-arriving events effectively using allowed lateness and side outputs.
> 
> Flink’s **stateful processing** was also crucial. We maintained large-scale keyed state for aggregations like session metrics and rolling windows. Using RocksDB as the state backend, we were able to scale state beyond memory limits while maintaining performance.
> 
> For reliability, Flink provides exactly-once guarantees through **checkpointing**. It periodically snapshots both state and Kafka offsets, and in case of failure, it restores from the last consistent checkpoint without data loss. To ensure end-to-end exactly-once, especially when writing to external systems, we used transactional sinks with a **two-phase commit protocol**, preventing duplicate or partial writes.
> 
> Additionally, Flink handles **backpressure** natively, so when downstream systems slow down, it automatically propagates the pressure upstream and throttles ingestion from Kafka, ensuring system stability.
> 
> While Flink introduces some operational complexity, we chose it because our use case required extremely low latency, correctness with out-of-order data, and large-scale state management, which Flink handles very effectively.”
