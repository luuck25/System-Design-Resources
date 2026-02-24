# Lambda vs Kappa Architecture -- Complete Summary

## Core Difference

### Lambda Architecture

-   Uses two processing paths:
    -   **Batch layer** (accurate, historical processing)
    -   **Speed layer** (real-time processing)
-   Results are merged in a serving layer.
-   More complex due to dual systems.

### Kappa Architecture

-   Uses a single stream-processing path.
-   All data (real-time + historical) flows through one system.
-   Historical data is handled by replaying the event log.
-   Simpler and unified design.

------------------------------------------------------------------------

## Data Handling Approach

### Lambda

-   Stores data in separate systems:
    -   Batch storage (distributed file systems / data lakes)
    -   Real-time storage (cache / database)
-   Errors are fixed by reprocessing the batch layer.
-   Batch layer eventually corrects stream inaccuracies.

### Kappa

-   Uses a durable, replayable log (e.g., Kafka).
-   Historical analysis is done by replaying past events.
-   Same logic processes both old and new data.

------------------------------------------------------------------------

## Complexity Comparison

<img width="674" height="566" alt="image" src="https://github.com/user-attachments/assets/28169402-9b38-4ab9-835d-95a166c77499" />


## How Kappa Handles Historical Data

-   Stores all data in a permanent log.
-   Replays log when reprocessing is needed.
-   Uses same processing logic for all data.
-   Simplifies architecture but replaying huge datasets can be slower
    than batch systems.

------------------------------------------------------------------------

## Error Correction in Kappa

1.  Fix logic or source data.
2.  Replay log from error point.
3.  System reprocesses events.
4.  Updated results overwrite old state.

Raw data remains immutable for auditing.

------------------------------------------------------------------------

## Exactly-Once Processing (Simple Explanation)

Kappa prevents double counting using:

### Idempotence

Processing the same event multiple times gives the same result.

Example: Pressing an elevator button multiple times still goes to the
same floor once.

### Transactions

Read → Process → Write happens atomically. Either everything succeeds,
or nothing is committed.

<img width="693" height="798" alt="image" src="https://github.com/user-attachments/assets/9ec3ee0b-ba4d-4a7f-a40b-95aac543ca81" />

------------------------------------------------------------------------

## Ignore vs Overwrite During Replay

<img width="679" height="171" alt="image" src="https://github.com/user-attachments/assets/7473768a-627a-4afa-899f-aa8dc77caacf" />


# The Modern Middle Ground: Lambda and Kappa

Most modern architectures don't strictly pick one.

### Lambda Architecture

Runs both a fast stream (for speed) and a batch layer (for 100%
accuracy) in parallel. The batch layer eventually overwrites the stream
to correct errors.

### Kappa Architecture

Treats everything as a stream. Uses a single processing engine (like
Flink or Spark Streaming). If historical reprocessing is needed, it
replays past data through the same stream.

------------------------------------------------------------------------

## Which One Should You Choose?

Ask yourself:

> What is the cost of being wrong for five minutes?

-   If the answer is **millions of dollars** (e.g., credit card fraud
    detection) → Choose **Streaming / Kappa**.
-   If the answer is **a slightly annoyed manager looking at a
    dashboard** → Choose **Batch**.

------------------------------------------------------------------------

## Note on Tooling

Tools like **Apache Beam** allow you to write processing logic once and
run it in either batch or streaming mode.\
This is a great way to future-proof your pipeline architecture.

------------------------------------------------------------------------

# Batch-Only Architecture

Also called **Batch-Oriented Architecture**.

### Characteristics

-   Scheduled processing (hourly/daily)
-   High throughput
-   High latency
-   Optimized for completeness & reporting

### Best For

-   Data warehousing
-   Business reporting
-   Historical analytics

------------------------------------------------------------------------

# Final Takeaway

-   **Lambda** = Accurate but complex (Batch + Stream)
-   **Kappa** = Simple and stream-first (Replay instead of batch)
-   **Batch-only** = Traditional, high-latency reporting systems

Choose based on: - Required latency - Operational complexity tolerance -
Historical vs real-time priority

<img width="1206" height="682" alt="image" src="https://github.com/user-attachments/assets/8af8c796-93e4-43aa-90d0-4559ea9c6cd3" />


# Can u explain in more detail and simple words, exactly once, say out of 100 spark task, 2 are failing and rerun, how it will handle

## How Spark Handles Failing Tasks (Exactly-Once)

In a normal system, those 2 cooks might accidentally put a "half-finished" plate on the table before they dropped it, leading to a mess. Spark prevents this using **Two-Step Commit** and **Temporary Files**.

---

### Step 1: Writing to a "Secret Location" (The Sandbox)
When Spark starts 100 tasks, it doesn't let them write directly to the final database or folder. Instead, each task writes its result to a **private, temporary folder**.

* **Task #99** is working in `/temp/task_99_attempt_1/`
* **Task #100** is working in `/temp/task_100_attempt_1/`

### Step 2: The Failure & The Rerun
If Task #99 and #100 crash (maybe the network blipped), Spark **ignores everything** in those two temporary folders. It treats them like they never existed.

* Spark then launches **Task #99 (Attempt 2)**.
* This new task starts completely fresh, writing to a new folder: `/temp/task_99_attempt_2/`.

### Step 3: The "Commit" (The Exactly-Once Magic)
Only when a task signals *"I am 100% finished and the data is perfect,"* does the Spark Driver move the data.

* **For the 98 successful tasks:** Spark "commits" them (moves the file from the temp folder to the final folder).
* **For the 2 failed tasks:** Spark waits until the **rerun (Attempt 2)** is finished. Once Attempt 2 succeeds, Spark commits that version and deletes the "garbage" left behind by the failed Attempt 1.

---

### In "Production" Terms (The Delta Lake / Idempotent way)
If you are writing to a database (like the Delta Lake code I showed you), it uses a **Transaction Log**.

* **The Log:** Before any data is changed, Spark writes a note in a log file: *"I am starting Job #123 with 100 tasks."*
* **The Rerun:** If tasks 99 and 100 fail and rerun, they still carry the **"Job #123" ID**.
* **The Final Check:** The database looks at the incoming data. If it sees data from Task 99 (Attempt 1) trying to sneak in, but the Log says *"Attempt 1 failed, use Attempt 2,"* the database **rejects Attempt 1**.

---

### Summary: Why you don't get duplicates
* **Failed tasks are "Invisible":** Their partial work is hidden in temp folders or ignored by the transaction log.
* **Successful tasks are "Atomic":** They only appear in the final destination when they are 100% done.
* **The Driver is the "Boss":** It keeps a master list of which "Attempt ID" is the winner. If a failed task suddenly wakes up and tries to write data later, the Boss says, *"Too late, I already accepted Attempt 2. Your data is trash."*

---

### One Catch: The "External Side Effect"
Exactly-once only works if the **Sink** (the place you are writing to) supports transactions (like Delta Lake, Iceberg, or SQL).

If your Spark task sends an **Email** or a **Push Notification**, and that task fails and reruns... the user will get **two emails**. Spark can't "undo" an email once it's sent! This is why for Kappa architecture, we always try to write to a "Smart" database that can handle these re-runs.
