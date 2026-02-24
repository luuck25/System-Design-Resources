# The CAP Theorem (Brewer's Theorem)

## 1. Introduction

The **CAP Theorem**, also known as *Brewer's Theorem*, is a foundational
concept in distributed systems. It states that a distributed data store
can guarantee **only two out of the following three properties at the
same time**:

-   **Consistency (C)**
-   **Availability (A)**
-   **Partition Tolerance (P)**

When a network partition occurs (which is inevitable in real distributed
systems), a system must choose between **Consistency** and
**Availability**.

------------------------------------------------------------------------

## 2. The Three Guarantees Explained

### 2.1 Consistency (C)

Every read receives the most recent write (or an error).\
All users see the same data at the same time.

**Example:**\
If you transfer \$100 from Account A to Account B: - After the transfer,
every user must immediately see the updated balances. - No one should
see the old balance once the transaction is complete.

This is critical in: - Banking systems - Payment processing - Inventory
management

------------------------------------------------------------------------

### 2.2 Availability (A)

Every request receives a response --- even if the response may not
contain the latest data.

**Example:**\
In a social media app: - If you post a photo, another user in a
different region might not see it immediately. - But the system always
responds instead of failing.

This is critical in: - Social networks - E-commerce browsing - Messaging
apps

------------------------------------------------------------------------

### 2.3 Partition Tolerance (P)

The system continues operating even when network failures occur between
nodes.

**Example:**\
If servers in Europe cannot communicate with servers in the US: - Both
sides must continue functioning independently. - The system cannot
completely shut down.

In real-world distributed systems, **Partition Tolerance is mandatory**,
because: - Networks fail - Cables are cut - Servers crash - Cloud zones
disconnect

------------------------------------------------------------------------

## 3. The Core Trade-off (What Really Happens)

Since partitions are unavoidable, systems must choose:

### Option 1: CP (Consistency + Partition Tolerance)

When a partition occurs: - The system may reject requests. - It refuses
to return stale data.

**Example Scenario:**\
Imagine a banking system split into two regions. If Region A cannot
confirm the latest account balance from Region B: - It may block
withdrawals. - This prevents incorrect transactions.

This sacrifices availability to protect correctness.

------------------------------------------------------------------------

### Option 2: AP (Availability + Partition Tolerance)

When a partition occurs: - The system always responds. - Data might be
outdated (eventual consistency).

**Example Scenario:**\
In an online shopping cart: - Even if servers cannot sync, - Customers
can still add items. - Conflicts are resolved later.

This sacrifices strict consistency to keep the system responsive.

------------------------------------------------------------------------

## 4. Real-World Systems

### CP Systems

-   MongoDB (by MongoDB Inc.)
-   Redis (by Redis Ltd.)
-   Google Spanner (by Google)

These systems prioritize correctness over immediate availability during
partitions.

**Google Spanner** uses: - Atomic clocks - GPS synchronization -
TrueTime API

It achieves extremely high availability (\~99.999%) while maintaining
strong consistency.

------------------------------------------------------------------------

### AP Systems

-   Amazon Dynamo (by Amazon)
-   Cassandra (Apache Software Foundation)
-   CouchDB

These systems prioritize being always available.

They follow the **BASE model**: - Basically Available - Soft state -
Eventual consistency

**Example:**\
Amazon chose availability for shopping carts: - An unavailable cart
means lost sales. - Slight inconsistencies are resolved later.

------------------------------------------------------------------------

## 5. CAP in Practice

Modern systems often allow tuning between consistency and availability.

For example: - Some databases allow "read consistency levels". - You can
choose strong consistency for payments. - You can choose eventual
consistency for analytics.

------------------------------------------------------------------------

## 6. Key Takeaways

-   You cannot guarantee C, A, and P simultaneously in distributed
    systems.
-   Partition tolerance is non-negotiable.
-   During partitions, you must choose between:
    -   Correctness (CP)
    -   Responsiveness (AP)
-   The right choice depends on business needs.

------------------------------------------------------------------------

## 7. Simple Analogy

Imagine two bank branches that lose phone connection:

-   If they stop transactions → CP (safe but unavailable)
-   If they continue independently → AP (available but possibly
    inconsistent)

------------------------------------------------------------------------

**Generated on:** 2026-02-24
